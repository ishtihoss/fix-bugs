---
name: fix-bugs
description: Autonomously fix every unfixed bug in a markdown bug-list file, one per fresh-context iteration, until none remain. Takes the markdown file as its argument (tag it with @).
argument-hint: "@bug-list.md"
allowed-tools: Bash, Read, Grep, Glob
---

# fix-bugs — headless bug-fix loop

Drive a loop that fixes bugs from `$ARGUMENTS` one at a time. Each iteration spawns a fresh `claude -p` so the working context stays small across the whole run.

The argument is expected to be an `@`-tagged file reference (e.g. `/fix-bugs @BUGS.md`). The leading `@` is stripped before resolving the path — a bare path still works for backward compat.

## Canonical bug-file format

The loop's counters and per-iteration prompt both key off this shape — no heuristics, no tolerance:

- Every bug is a `### N. Title` heading (numbered, unique across the file).
- A bug is **fixed** iff its `### ` heading contains the literal string `FIXED` (e.g. `### 3. Foo — FIXED`).
- Severity groups are `## High severity`, `## Medium severity`, `## Low severity` — exactly these three strings. The inner prompt walks them in order.
- No other `### ` headings exist in the file. Section labels, commentary, and summaries live at `##` or below-`###` levels only.

Non-conforming files are normalised in step 2 before the loop runs. The counters can't tell a stray `### Critical` section label from a real bug heading, so the skill fixes the file instead of hardening the counters.

## What you (the model running this skill) do

1. **Validate the argument.** If `$ARGUMENTS` is empty, stop and tell the user to tag the bug file with `@` (e.g. `/fix-bugs @BUGS.md`). Strip any leading `@` from the argument before treating it as a path. If the resolved file doesn't exist, stop and tell the user.
2. **Normalise the bug-file format.** Read the file. If it already matches the canonical format above exactly, skip this step. Otherwise rewrite it in place so the loop can address every bug:
   - **Bullet-item bugs** (`- **Title** — body…` under a heading) → promote each bullet to a `### N. Title` heading with the bullet body verbatim as the entry body. Preserve any existing `— FIXED` marker.
   - **Sub-severity `### ` labels** (e.g. `### Critical`, `### Medium`, `### Minor` used as section headers, not as numbered bugs) → delete the label and treat its children as bugs under the mapped canonical severity section (Critical → High, Medium → Medium, Minor → Low).
   - **Non-canonical `## ` sections** (e.g. `## Bugs found in review of commit X`, `## Highest-leverage fixes`) → keep the prose/context as a paragraph, but move the bugs under it into one of the three canonical severity sections. Pick the severity from the section name's intent; if genuinely ambiguous, ask the user which severity to use before rewriting.
   - **Renumber** `### N.` entries sequentially across the file once restructuring is done, so no two bugs collide on a number.
   - **Lossless on bodies**: do not rephrase, summarise, or drop existing bug descriptions or FIXED postmortems. Only restructure headings and severity grouping.
   - Tell the user in one sentence what you changed (e.g. "Normalised 4 bullet-item bugs under Critical/Minor into `### ` headings; no content touched.") before kicking off the loop. Do not ask for confirmation — the user reviews the diff before committing.
3. **Pick model + effort for the inner `claude -p` sessions.** Subprocesses do NOT inherit the parent session's `/model` or `/effort` overrides — those live only in the parent's runtime. You must pass them explicitly. Defaults if the user hasn't said otherwise: `opus` + `xhigh`. If the user recently mentioned a different model/effort for this work (e.g. "I'm running sonnet today"), use that. If genuinely unsure, ask in one sentence. Export `CLAUDE_FIX_MODEL` and `CLAUDE_FIX_EFFORT` before kicking off the loop — the script reads them.
4. **Kick off the loop script below via Bash with `run_in_background: true`.** It writes progress to `.fix-bugs.log` in the same directory as the bug file.
5. **Monitor** the log (tail it periodically, or use `Monitor` on the bash process). Do NOT sleep-poll tightly — the Monitor tool notifies on new lines.
6. When the loop exits, **read the final bug file** and report to the user: how many bugs were fixed this run, what remains unfixed, and any newly-discovered bugs that were appended.

Do not commit anything. The user reviews and commits themselves.

## The loop

Run this exactly — do not inline-edit the prompt body unless the user asks. `BUGS_FILE` must be absolute.

```bash
set -u
RAW_ARG="${ARGUMENTS_PATH#@}"   # strip leading @ if the user tagged the file
BUGS_FILE="$(cd "$(dirname "$RAW_ARG")" && pwd)/$(basename "$RAW_ARG")"
LOG="$(dirname "$BUGS_FILE")/.fix-bugs.log"
MODEL="${CLAUDE_FIX_MODEL:-opus}"
EFFORT="${CLAUDE_FIX_EFFORT:-xhigh}"
: > "$LOG"

count_unfixed() { grep -E '^### ' "$BUGS_FILE" 2>/dev/null | grep -cv 'FIXED' | tr -d '\n'; }
count_total()   { grep -cE '^### ' "$BUGS_FILE" 2>/dev/null | tr -d '\n'; }
state_line()    { printf '%s total, %s unfixed' "$(count_total)" "$(count_unfixed)"; }

# Cap scales with actual work: 2× the initial unfixed count, + buffer for bugs
# rule 4 may append. Minimum 5 to keep trivial runs sane.
UNFIXED_START=$(count_unfixed)
MAX_ITER=$(( UNFIXED_START * 2 + 5 ))
if [ "$MAX_ITER" -lt 5 ]; then MAX_ITER=5; fi
echo "[$(date +%H:%M:%S)] Start: $(state_line). MAX_ITER=$MAX_ITER, model=$MODEL, effort=$EFFORT" >> "$LOG"

prev_unfixed=$UNFIXED_START
prev_total=$(count_total)
stuck_streak=0

for i in $(seq 1 $MAX_ITER); do
  if [ "$(count_unfixed)" -eq 0 ]; then
    echo "[$(date +%H:%M:%S)] All bugs marked FIXED after $((i-1)) iteration(s). Stopping." >> "$LOG"
    break
  fi
  {
    echo ""
    echo "=========================================="
    echo "[$(date +%H:%M:%S)] iteration $i — $(state_line)"
    echo "=========================================="
  } >> "$LOG"

  claude -p "You are fixing exactly ONE bug from the markdown file at $BUGS_FILE.

Rules:
1. Read $BUGS_FILE. The highest-severity unfixed bug is the first '### ' heading under '## High severity' (then Medium, then Low) that does NOT contain the word FIXED.
2. Read the code files referenced by that bug entry. Understand the root cause — do not pattern-match a shallow fix.
3. Implement the fix.
4. Regression check: re-read your diff and mentally trace the modified code paths. If you introduced a new bug, fix it in the same iteration. If you *discover* a bug that existed before but wasn't listed, append it as a new '### N. <title>' entry under the appropriate '## <severity> severity' section of $BUGS_FILE — do NOT fix it this iteration.
5. Update the bug's entry in $BUGS_FILE: append ' — FIXED' to its '### ' heading, and rewrite the body to describe (a) what the bug was and (b) what the fix was. Keep it concise — match the style of existing FIXED entries.
6. Do NOT commit. Do NOT touch any other unfixed bug. Do NOT run the app.
7. End with a one-line summary: 'Fixed: <bug title>'." \
    --model "$MODEL" \
    --effort "$EFFORT" \
    --permission-mode bypassPermissions \
    --max-turns 60 \
    >> "$LOG" 2>&1 || {
      echo "[$(date +%H:%M:%S)] iteration $i FAILED (claude exit $?). Stopping." >> "$LOG"
      break
    }

  # Stuck detection: iteration ran but neither flipped a bug to FIXED nor
  # appended a new one. One retry allowed; bail on the second consecutive stall.
  cur_unfixed=$(count_unfixed)
  cur_total=$(count_total)
  if [ "$cur_unfixed" -ge "$prev_unfixed" ] && [ "$cur_total" -le "$prev_total" ]; then
    stuck_streak=$((stuck_streak + 1))
    echo "[$(date +%H:%M:%S)] iteration $i made no progress (stuck_streak=$stuck_streak)" >> "$LOG"
    if [ "$stuck_streak" -ge 2 ]; then
      echo "[$(date +%H:%M:%S)] No progress for 2 iterations. Stopping." >> "$LOG"
      break
    fi
  else
    stuck_streak=0
  fi
  prev_unfixed=$cur_unfixed
  prev_total=$cur_total
done

echo "" >> "$LOG"
echo "[$(date +%H:%M:%S)] Loop done. Final state: $(state_line)" >> "$LOG"
```

Before running it, in the same bash invocation:
- `export ARGUMENTS_PATH="$ARGUMENTS"` — path to the bug file.
- Optionally `export CLAUDE_FIX_MODEL=...` and `export CLAUDE_FIX_EFFORT=...` if the user wants a non-default model/effort for the inner sessions. Defaults are `opus` + `xhigh`. Valid effort levels: `low, medium, high, xhigh, max`.

## Safety rails

- `MAX_ITER` auto-sizes to `unfixed_at_start * 2 + 5` (min 5) — enough headroom for one retry per bug plus a few rule-4 appends, without blind over-spinning.
- **Stuck detection**: if an iteration neither flipped a bug to `FIXED` nor appended a new `### ` entry, it counts as no-progress. One retry is allowed; two consecutive stalls stop the loop. Prevents burning the cap on a single unfixable bug.
- `--permission-mode bypassPermissions` is required for autonomy. Every inner iteration is sandboxed to one bug and cannot commit.
- If `claude -p` returns non-zero, the loop stops — the user investigates via the log.
- The log lives next to the bug file as `.fix-bugs.log`. Gitignore it.
