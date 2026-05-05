---
name: fix-bugs
description: Autonomously fix every unfixed bug in one or more markdown bug-list files, one bug per fresh-context iteration, until none remain. Files are processed sequentially in the order given. Tag each file with @.
argument-hint: "@bug-list.md [@bug-list-2.md ...]"
allowed-tools: Bash, Read, Grep, Glob
---

# fix-bugs — headless bug-fix loop

Drive a loop that fixes bugs from `$ARGUMENTS` one at a time. Each iteration spawns a fresh `claude -p` so the working context stays small across the whole run.

The argument is one or more `@`-tagged file references separated by spaces (e.g. `/fix-bugs @BUGS.md` or `/fix-bugs @bugs-foo.md @bugs-bar.md`). The leading `@` is stripped per file before resolving — bare paths still work for backward compat. When multiple files are given, they are processed **sequentially** in the order supplied: file 2's loop only starts after file 1's loop finishes (whether by completing all bugs or hitting a safety rail).

## Canonical bug-file format

Every file in the list must conform to this shape — no heuristics, no tolerance:

- Every bug is a `### N. Title` heading (numbered, unique within its file).
- A bug is **fixed** iff its `### ` heading contains the literal string `FIXED` (e.g. `### 3. Foo — FIXED`).
- Severity groups are `## High severity`, `## Medium severity`, `## Low severity` — exactly these three strings. The inner prompt walks them in order.
- No other `### ` headings exist in the file. Section labels, commentary, and summaries live at `##` or below-`###` levels only.

Non-conforming files are normalised in step 2 before the loop runs. The counters can't tell a stray `### Critical` section label from a real bug heading, so the skill fixes the file instead of hardening the counters. Numbering is independent per file; cross-file collisions are fine.

## What you (the model running this skill) do

1. **Validate the arguments.** If `$ARGUMENTS` is empty, stop and tell the user to tag the bug file(s) with `@` (e.g. `/fix-bugs @BUGS.md` or `/fix-bugs @a.md @b.md`). Split `$ARGUMENTS` on whitespace, strip a leading `@` from each token, and resolve each to an absolute path. If any file doesn't exist, stop and report the bad path before doing anything else (fail fast — don't normalise some files and then fail).
2. **Normalise the bug-file format.** For each file in the list, in order: read it, and if it already matches the canonical format above exactly, skip it. Otherwise rewrite it in place so the loop can address every bug:
   - **Bullet-item bugs** (`- **Title** — body…` under a heading) → promote each bullet to a `### N. Title` heading with the bullet body verbatim as the entry body. Preserve any existing `— FIXED` marker.
   - **Sub-severity `### ` labels** (e.g. `### Critical`, `### Medium`, `### Minor` used as section headers, not as numbered bugs) → delete the label and treat its children as bugs under the mapped canonical severity section (Critical → High, Medium → Medium, Minor → Low).
   - **Non-canonical `## ` sections** (e.g. `## Bugs found in review of commit X`, `## Highest-leverage fixes`) → keep the prose/context as a paragraph, but move the bugs under it into one of the three canonical severity sections. Pick the severity from the section name's intent; if genuinely ambiguous, ask the user which severity to use before rewriting.
   - **Renumber** `### N.` entries sequentially within each file once restructuring is done, so no two bugs collide on a number inside that file.
   - **Lossless on bodies**: do not rephrase, summarise, or drop existing bug descriptions or FIXED postmortems. Only restructure headings and severity grouping.
   - Tell the user in one short sentence per modified file what you changed (e.g. "Normalised 4 bullet-item bugs in `bugs-foo.md` into `### ` headings; no content touched.") before kicking off the loop. Do not ask for confirmation — the user reviews the diff before committing.
3. **Pick model + effort for the inner `claude -p` sessions.** Subprocesses do NOT inherit the parent session's `/model` or `/effort` overrides — those live only in the parent's runtime. You must pass them explicitly. Defaults if the user hasn't said otherwise: `opus` + `xhigh`. If the user recently mentioned a different model/effort for this work (e.g. "I'm running sonnet today"), use that. If genuinely unsure, ask in one sentence. Export `CLAUDE_FIX_MODEL` and `CLAUDE_FIX_EFFORT` before kicking off the loop — the script reads them.
4. **Kick off the loop script below via Bash with `run_in_background: true`.** The script body is wrapped in `bash <<'OUTERSCRIPT'` … `OUTERSCRIPT` so it always runs under bash regardless of the user's login shell — without the wrapper, hosts that default to zsh choke on bash-only syntax (`read -ra`, `<<<`, process substitution `<(...)`). It writes progress to `.fix-bugs.log` in the same directory as each bug file (so files in the same directory share one log; files in different directories each get their own). Each time a bug flips to FIXED the loop writes a `NOTIFY fixed: [<basename>] <title> (N unfixed remaining)` line so the parent session can surface it as a live console-style update.
5. **Monitor** the log(s) (`Monitor` on the bash process, or periodic tail). Do NOT sleep-poll tightly — Monitor notifies on new lines. Whenever you see a new `NOTIFY fixed:` line, **immediately echo it to the user as a short one-liner** ("Fixed [bugs-foo.md] #N: title — M left") so they get live updates in the conversation without having to read the log themselves. Use these lines for the post-run summary too.
6. When the loop exits, **read each bug file's final state** and report to the user, per file: how many bugs were fixed this run, what remains unfixed, and any newly-discovered bugs that were appended. If processing stopped early on a file due to a safety rail (subprocess failure or stuck-streak), note which file and why.

Do not commit anything. The user reviews and commits themselves.

## The loop

Run this exactly — do not inline-edit the prompt body unless the user asks. Both `BUGS_FILE` paths and the log path must be absolute.

```bash
# Wrap in `bash <<'OUTERSCRIPT'` so the script always runs under bash, even when
# the parent shell is zsh (macOS default). Without this, `read -ra`, `<<<`, and
# process substitution `<(...)` fail under zsh with `read: bad option: -a`.
# The single-quoted delimiter prevents the outer shell from expanding anything
# inside, so bash sees the script body verbatim.
bash <<'OUTERSCRIPT'
set -u

# --- Parse arguments: space-separated list of @-tagged or bare paths. ---
# Backward compat: if ARGUMENTS_PATH (single path) is set and ARGUMENTS_RAW is
# not, treat ARGUMENTS_PATH as a one-element list.
RAW="${ARGUMENTS_RAW:-${ARGUMENTS_PATH:-}}"
if [ -z "$RAW" ]; then
  echo "[ERROR] fix-bugs: no bug-list files supplied" >&2
  exit 1
fi

declare -a BUGS_FILES=()
read -ra TOKENS <<< "$RAW"
for tok in "${TOKENS[@]}"; do
  raw="${tok#@}"   # strip leading @ if present
  [ -z "$raw" ] && continue
  if [ ! -f "$raw" ]; then
    echo "[ERROR] fix-bugs: file not found: $tok" >&2
    exit 1
  fi
  abs="$(cd "$(dirname "$raw")" && pwd)/$(basename "$raw")"
  BUGS_FILES+=("$abs")
done

if [ "${#BUGS_FILES[@]}" -eq 0 ]; then
  echo "[ERROR] fix-bugs: no valid bug-list files after parsing" >&2
  exit 1
fi

MODEL="${CLAUDE_FIX_MODEL:-opus}"
EFFORT="${CLAUDE_FIX_EFFORT:-xhigh}"

# --- Truncate each unique log path exactly once before any file runs. ---
# Two bug files in the same directory share .fix-bugs.log; we don't want the
# second file's loop to wipe the first file's history. Linear-scan dedup keeps
# this compatible with bash 3.2 (macOS default).
declare -a SEEN_LOGS=()
log_seen() {
  local target="$1"
  if [ "${#SEEN_LOGS[@]}" -gt 0 ]; then
    for s in "${SEEN_LOGS[@]}"; do [ "$s" = "$target" ] && return 0; done
  fi
  return 1
}
for f in "${BUGS_FILES[@]}"; do
  L="$(dirname "$f")/.fix-bugs.log"
  if ! log_seen "$L"; then
    : > "$L"
    SEEN_LOGS+=("$L")
  fi
done

# --- Outer loop: one bug-file at a time, in the order given. ---
MULTI=0
[ "${#BUGS_FILES[@]}" -gt 1 ] && MULTI=1

for BUGS_FILE in "${BUGS_FILES[@]}"; do
  LOG="$(dirname "$BUGS_FILE")/.fix-bugs.log"
  BASENAME="$(basename "$BUGS_FILE")"

  # File-start banner. Only printed when there's more than one file, so the
  # single-file log is byte-identical to the pre-multi-file behaviour.
  if [ "$MULTI" -eq 1 ]; then
    {
      echo ""
      echo "##########################################"
      echo "[$(date +%H:%M:%S)] === fix-bugs: starting $BUGS_FILE ==="
      echo "##########################################"
    } >> "$LOG"
  fi

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
  prev_fixed=$(grep -E '^### .*FIXED' "$BUGS_FILE" 2>/dev/null | sort)
  stuck_streak=0

  # notify_fix <heading-line> — log a live update for one newly-FIXED bug.
  # The NOTIFY line is the easiest grep target for the parent session to
  # surface as a console-style update without re-parsing the whole log.
  # When MULTI=1, the line carries the bug-file basename in brackets so the
  # parent session can disambiguate updates from different files in the same
  # shared log.
  notify_fix() {
    local line="$1"
    local title
    title=$(printf '%s' "$line" | sed -E 's/^### //; s/ — FIXED.*$//')
    local remaining
    remaining=$(count_unfixed)
    if [ "$MULTI" -eq 1 ]; then
      echo "[$(date +%H:%M:%S)] NOTIFY fixed: [$BASENAME] $title ($remaining unfixed remaining)" >> "$LOG"
    else
      echo "[$(date +%H:%M:%S)] NOTIFY fixed: $title ($remaining unfixed remaining)" >> "$LOG"
    fi
  }

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

    # Live update: diff the FIXED set and notify for anything newly-flipped.
    cur_fixed=$(grep -E '^### .*FIXED' "$BUGS_FILE" 2>/dev/null | sort)
    new_fixes=$(comm -13 <(printf '%s\n' "$prev_fixed") <(printf '%s\n' "$cur_fixed"))
    if [ -n "$new_fixes" ]; then
      while IFS= read -r line; do
        [ -z "$line" ] && continue
        notify_fix "$line"
      done <<< "$new_fixes"
    fi
    prev_fixed=$cur_fixed

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
  if [ "$MULTI" -eq 1 ]; then
    echo "[$(date +%H:%M:%S)] Loop done for $BASENAME. Final state: $(state_line)" >> "$LOG"
  else
    echo "[$(date +%H:%M:%S)] Loop done. Final state: $(state_line)" >> "$LOG"
  fi
done
OUTERSCRIPT
```

Before running it, in the same bash invocation:
- `export ARGUMENTS_RAW="$ARGUMENTS"` — the full space-separated argument string. Each token may be `@`-tagged or a bare path. The script splits and strips `@` per token.
- For backward compat with anyone wrapping this skill: `export ARGUMENTS_PATH="..."` (single path) still works as a fallback when `ARGUMENTS_RAW` is unset.
- Optionally `export CLAUDE_FIX_MODEL=...` and `export CLAUDE_FIX_EFFORT=...` if the user wants a non-default model/effort for the inner sessions. Defaults are `opus` + `xhigh`. Valid effort levels: `low, medium, high, xhigh, max`.

## Safety rails

- `MAX_ITER` auto-sizes per file to `unfixed_at_start * 2 + 5` (min 5) — enough headroom for one retry per bug plus a few rule-4 appends, without blind over-spinning. Each file gets its own cap.
- **Stuck detection** runs per file: if an iteration neither flipped a bug to `FIXED` nor appended a new `### ` entry, it counts as no-progress. One retry is allowed; two consecutive stalls stop **that file's** loop and the outer loop advances to the next file.
- `--permission-mode bypassPermissions` is required for autonomy. Every inner iteration is sandboxed to one bug and cannot commit.
- If `claude -p` returns non-zero on any iteration, that file's loop stops and the outer loop advances to the next file. The user investigates via the log.
- Validation is fail-fast: the script bails before any file runs if any path in the argument list is missing.
- The log lives next to each bug file as `.fix-bugs.log`. Files in the same directory share one log, separated by `### fix-bugs: starting <path> ###` banners. Files in different directories get separate logs. Gitignore them.
- **Live updates**: after each iteration the loop diffs the FIXED set and logs a `NOTIFY fixed: [<basename>] <title> (N unfixed remaining)` line (the `[<basename>]` prefix appears only when more than one file is being processed). The parent session watches for these and surfaces each one to the user as a short console-style update — no system notifications, just inline messages in the conversation.
