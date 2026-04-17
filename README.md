# fix-bugs

A Claude Code skill that autonomously fixes every bug in a markdown bug-list, one per fresh-context iteration, until none remain.

Each bug gets its own `claude -p` subprocess with an empty context — so the parent session stays small no matter how long the run is, and the inner session stays focused on exactly one bug.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/ishtihoss/fix-bugs.git ~/.claude/skills/fix-bugs
```

Then invoke it from Claude Code by tagging the bug file with `@`:

```
/fix-bugs @bugs.md
```

The `@`-tag lets Claude Code resolve the file for you — no need to type the full path. A bare path still works if you prefer.

## Bug-file format

- Each bug is a `### ` heading.
- A bug is considered **fixed** iff its `### ` heading contains the literal string `FIXED` (e.g. `### 3. Foo — FIXED`).
- Optional severity groups: `## High severity`, `## Medium severity`, `## Low severity` — the loop prefers higher severity first.

Example:

```markdown
## High severity

### 1. Race condition in init
**File:** `src/foo.js` (lines 42-58)

Describe the bug here…

### 2. Memory leak on detach — FIXED
**File:** `src/bar.js`

What the bug was, and how it was fixed.

## Medium severity

### 3. Off-by-one in pager
…
```

## How it works

For each unfixed bug:

1. Spawn `claude -p` with a fresh context and a tight prompt that picks the highest-severity unfixed entry.
2. Let it read the referenced files, implement a fix, and append ` — FIXED` to the bug's heading with a short post-mortem body.
3. If it discovers a *new* bug while fixing, it appends a new `### ` entry instead of fixing it — keeping one-bug-per-iteration honest.
4. Loop until every `### ` heading contains `FIXED`, or a safety rail trips.

Progress is logged to `.fix-bugs.log` next to the bug file. Gitignore it.

Each time a bug flips to `FIXED` the loop writes a `NOTIFY fixed: <title> (N unfixed remaining)` line. The parent Claude Code session watches for these and surfaces each one to you as a short inline update in the conversation — so you get live console-style progress without tailing the log.

## Safety rails

- **Iteration cap:** `unfixed_at_start * 2 + 5` (min 5). Enough for one retry per bug plus a few rule-4 appends, without blindly over-spinning.
- **Stuck detection:** if an iteration neither flipped a bug to `FIXED` nor appended a new one, it's a stall. One retry allowed; two consecutive stalls stop the loop.
- **Hard stop on subprocess failure:** if `claude -p` exits non-zero, the loop halts so you can investigate.
- **No commits:** the skill never commits. You review the diffs and commit yourself.
- **Fresh context per fix:** no cross-bug contamination, no runaway context growth.

## Configuration

Defaults: `opus` + `xhigh` for the inner sessions.

Override before invoking if you want something else — either tell the parent session (e.g. "use sonnet + high for fix-bugs") or export directly:

```bash
export CLAUDE_FIX_MODEL=sonnet
export CLAUDE_FIX_EFFORT=high
```

Valid effort levels: `low`, `medium`, `high`, `xhigh`, `max`.

## When to reach for this

- You've done a review pass and produced a bug list you trust.
- The bugs are reasonably localized — each has a file pointer or clear description.
- You want to walk away and come back to a stack of diffs to review.

If the bugs are vague, interdependent, or need architectural decisions, drive them interactively instead.

## License

MIT
