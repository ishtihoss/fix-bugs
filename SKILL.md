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
   - **Strip XML-illegal control bytes** (anything in `\x00–\x08\x0B\x0C\x0E–\x1F\x7F` — NUL, BS, VT, FF, SO–US, DEL). These can creep in when an author pastes from a source that interpreted `\x00` etc. as actual bytes; they make plain `grep` flip into binary-file mode (the script defends against this with `-a`, but the file should still be human-readable). Replace each byte with its textual escape (`\x00`, `\x08`, …) so the prose intent is preserved. Silent — no user prompt — but mention in the per-file summary line if any were stripped.
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

# Cross-repo support: $CLAUDE_EXTRA_DIRS is a colon-separated list of absolute
# paths (e.g. sibling repos) the inner sessions need tool access to. Each valid
# path becomes a --add-dir flag on the per-iteration `claude -p` call. Unset /
# empty / non-existent paths are silently skipped. Parsed once, reused for every
# file and iteration. macOS-bash-3-safe expansion via ${arr[@]+"${arr[@]}"}.
# (No log line here: $LOG is only defined inside the per-file loop below.)
declare -a EXTRA_DIR_FLAGS=()
if [ -n "${CLAUDE_EXTRA_DIRS:-}" ]; then
  IFS=':' read -ra _XDIRS <<< "$CLAUDE_EXTRA_DIRS"
  for d in "${_XDIRS[@]}"; do
    [ -n "$d" ] && [ -d "$d" ] && EXTRA_DIR_FLAGS+=(--add-dir "$d")
  done
fi

# --- Multi-model review plumbing --------------------------------------------
# Each inner iteration gets an automatic second opinion from a two-model panel —
# GLM 5.2 (`consult_glm52`) AND the latest Kimi K2.7 (`consult_kimi27`), both in
# the `porkicoder-consult` MCP server. Headless `claude -p` does NOT auto-load
# user-scope MCP servers, so we extract that one server entry from ~/.claude.json
# and hand it to the inner sessions via `--mcp-config` (one config exposes BOTH
# tools). Parsed once, reused for every file and iteration. If the server isn't
# configured, MCP_FLAGS stays empty and the inner prompt silently falls back to
# its own regression check — behaviour unchanged. macOS-bash-3-safe expansion
# via ${arr[@]+"${arr[@]}"} at the call site below.
MCP_TMPDIR="$(mktemp -d -t fix-bugs-mcp-XXXXXX 2>/dev/null || echo "")"
MCP_CONFIG="${MCP_TMPDIR:+$MCP_TMPDIR/mcp-consult.json}"
declare -a MCP_FLAGS=()
if [ -n "$MCP_CONFIG" ] && [ -f "$HOME/.claude.json" ] && command -v python3 >/dev/null 2>&1; then
  python3 - "$HOME/.claude.json" "$MCP_CONFIG" <<'PYEOF' || true
import json, sys
src, dst = sys.argv[1], sys.argv[2]
try:
    cfg = json.load(open(src))
    entry = (cfg.get("mcpServers") or {}).get("porkicoder-consult")
    if entry:
        # Serialize fully, then write in one call so a mid-write failure can
        # never leave a partial JSON that --mcp-config would choke on.
        out = json.dumps({"mcpServers": {"porkicoder-consult": entry}})
        open(dst, "w").write(out)
except Exception:
    pass
PYEOF
fi
if [ -n "$MCP_CONFIG" ] && [ -s "$MCP_CONFIG" ]; then
  MCP_FLAGS=(--mcp-config "$MCP_CONFIG")
  MCP_REVIEW_MSG="Multi-model review ENABLED (GLM 5.2 + Kimi 2.7 via $MCP_CONFIG)"
else
  MCP_REVIEW_MSG="Multi-model review DISABLED (porkicoder-consult not in ~/.claude.json) — regression check only"
fi
# Best-effort cleanup of the temp MCP config when the script exits. ($LOG is
# only defined inside the per-file loop, so the ENABLED/DISABLED state is logged
# there, per file, via $MCP_REVIEW_MSG.)
[ -n "$MCP_TMPDIR" ] && trap 'rm -rf "$MCP_TMPDIR"' EXIT

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

  # `-a` forces grep to treat the file as text. Without it, a single stray
  # control byte (NUL/BS/VT/FF/SO–US) makes grep emit "Binary file ... matches"
  # instead of line content, which collapses the counter to 1 and breaks
  # MAX_ITER + stuck-detection. Step 2 also scrubs these on normalisation, but
  # `-a` is the belt-and-braces guarantee — the script must remain robust if a
  # future inner iteration writes a control char into a FIXED postmortem.
  count_unfixed() { grep -aE '^### ' "$BUGS_FILE" 2>/dev/null | grep -acv 'FIXED' | tr -d '\n'; }
  count_total()   { grep -caE '^### ' "$BUGS_FILE" 2>/dev/null | tr -d '\n'; }
  state_line()    { printf '%s total, %s unfixed' "$(count_total)" "$(count_unfixed)"; }

  # Cap scales with actual work: 2× the initial unfixed count, + buffer for bugs
  # rule 4 may append. Minimum 5 to keep trivial runs sane.
  UNFIXED_START=$(count_unfixed)
  MAX_ITER=$(( UNFIXED_START * 2 + 5 ))
  if [ "$MAX_ITER" -lt 5 ]; then MAX_ITER=5; fi
  echo "[$(date +%H:%M:%S)] Start: $(state_line). MAX_ITER=$MAX_ITER, model=$MODEL, effort=$EFFORT" >> "$LOG"
  echo "[$(date +%H:%M:%S)] $MCP_REVIEW_MSG" >> "$LOG"

  prev_unfixed=$UNFIXED_START
  prev_total=$(count_total)
  prev_fixed=$(grep -aE '^### .*FIXED' "$BUGS_FILE" 2>/dev/null | sort)
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
4. Regression check: re-read your diff and mentally trace the modified code paths. If you introduced a new bug, fix it in the same iteration. If you *discover* a bug that existed before but wasn't listed, append it as a new '### N. <title>' entry under the appropriate '## <severity> severity' section of $BUGS_FILE — do NOT fix it this iteration. Then get an automatic second opinion from a two-model panel — GLM 5.2 (MCP tool consult_glm52) AND the latest Kimi K2.7 (MCP tool consult_kimi27):
   - These tools are only present when the MCP server is configured. If neither consult_glm52 nor consult_kimi27 is available, SKIP this entirely and SILENTLY — your own regression check above stands; do not mention it, do not fail.
   - Neither model has any repo access, so curate ONE context bundle for both: this bug's intent (the entry's title + body), your full \`git diff\` (strip lockfile / build-output / node_modules churn), and the full current contents of the hand-written files you changed (summarize very large or generated files rather than pasting them whole).
   - Call BOTH consult_glm52 AND consult_kimi27 with a 'question' scoped ONLY to correctness, regressions, and security — explicitly NOT style or nits — passing the SAME bundle as 'context'.
   - If ONE tool errors, proceed with whatever ran; never let a consult error abort the iteration.
   - VALIDATE each issue either model raises against the real code before acting on it (a model can be wrong about code it never received). Fix every CONFIRMED real issue IN THIS SAME ITERATION, before marking the bug FIXED. Discard unconfirmed or style-only findings.
5. Update the bug's entry in $BUGS_FILE: append ' — FIXED' to its '### ' heading, and rewrite the body to describe (a) what the bug was and (b) what the fix was. Keep it concise — match the style of existing FIXED entries.
6. Do NOT commit. Do NOT touch any other unfixed bug. Do NOT run the app.
7. End with a one-line summary: 'Fixed: <bug title>'." \
      --model "$MODEL" \
      --effort "$EFFORT" \
      --permission-mode bypassPermissions \
      --max-turns 60 \
      ${EXTRA_DIR_FLAGS[@]+"${EXTRA_DIR_FLAGS[@]}"} \
      ${MCP_FLAGS[@]+"${MCP_FLAGS[@]}"} \
      >> "$LOG" 2>&1 || {
        echo "[$(date +%H:%M:%S)] iteration $i FAILED (claude exit $?). Stopping." >> "$LOG"
        break
      }

    # Live update: diff the FIXED set and notify for anything newly-flipped.
    cur_fixed=$(grep -aE '^### .*FIXED' "$BUGS_FILE" 2>/dev/null | sort)
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
- Optionally `export CLAUDE_EXTRA_DIRS="/abs/path1:/abs/path2"` (colon-separated absolute paths) if the inner sessions need tool access to directories outside the cwd. Common case: a sibling repo whose code a bug fix spans. Each existing path becomes a `--add-dir <path>` flag on the per-iteration `claude -p` call. Non-existent paths are silently skipped.

## Safety rails

- `MAX_ITER` auto-sizes per file to `unfixed_at_start * 2 + 5` (min 5) — enough headroom for one retry per bug plus a few rule-4 appends, without blind over-spinning. Each file gets its own cap.
- **Stuck detection** runs per file: if an iteration neither flipped a bug to `FIXED` nor appended a new `### ` entry, it counts as no-progress. One retry is allowed; two consecutive stalls stop **that file's** loop and the outer loop advances to the next file.
- `--permission-mode bypassPermissions` is required for autonomy. Every inner iteration is sandboxed to one bug and cannot commit.
- If `claude -p` returns non-zero on any iteration, that file's loop stops and the outer loop advances to the next file. The user investigates via the log.
- Validation is fail-fast: the script bails before any file runs if any path in the argument list is missing.
- The log lives next to each bug file as `.fix-bugs.log`. Files in the same directory share one log, separated by `### fix-bugs: starting <path> ###` banners. Files in different directories get separate logs. Gitignore them.
- **Live updates**: after each iteration the loop diffs the FIXED set and logs a `NOTIFY fixed: [<basename>] <title> (N unfixed remaining)` line (the `[<basename>]` prefix appears only when more than one file is being processed). The parent session watches for these and surfaces each one to the user as a short console-style update — no system notifications, just inline messages in the conversation.
- **Automatic multi-model review (advisory).** When the `porkicoder-consult` MCP server is configured in `~/.claude.json`, each iteration's regression check (rule 4) is augmented with a second opinion from a two-model panel — GLM 5.2 (`consult_glm52`) AND the latest Kimi K2.7 (`consult_kimi27`). The MCP config is extracted once (before the file loop) and passed to every inner session via `--mcp-config` (headless `claude -p` does **not** auto-load user-scope MCP servers); each per-file loop logs the ENABLED/DISABLED state. The inner session curates a context bundle (bug intent + diff + changed-file contents — the models have no repo access), asks each model only about correctness/regressions/security, VALIDATES every raised issue against the real code, and fixes any confirmed-real issue in the SAME iteration before marking the bug FIXED. It is purely advisory: a single tool error is tolerated (proceed with whatever ran), and if the server is absent the step is skipped silently and behaviour is identical to the regression check only. The trade-off is added latency and token cost per iteration; the review never aborts the loop.
