---
name: claude-review
description: >-
  A Codex-primary adversarial PLAN-review loop where OpenAI Codex is the
  builder/orchestrator and Claude Code is the critic. Use this when
  you are working in Codex, already have a plan or clear idea, and want Claude
  Code to stress-test the plan before implementation. Codex writes and revises
  a run-scoped PLAN.md, invokes Claude Code with its normal tool access for a
  high-quality review, captures Claude's session_id from JSON output, resumes
  that exact Claude session across review rounds, and stops on VERDICT:
  APPROVED, VERDICT: REVISE plus MAX_ROUNDS, or unresolved reviewer failure.
  NOT for Claude-primary review; use /codex-review for Claude Code calling
  Codex.
---

# Claude-Review - Codex-Primary Plan Review

Codex is the builder and orchestrator. Claude Code is the critic.
This is the inverse of `codex-review`: **Codex calls Claude**, not the other way around.

Use this when you are already working in Codex, have a plan or clear implementation idea, and want a second-model adversarial review before writing code. Skip it for trivial changes.

## Prerequisites

- Claude Code installed and authenticated: `claude --version`.
- Claude Code supports non-interactive JSON output: `claude -p --output-format json`.
- Claude Code supports resuming a `-p` session by explicit id: `claude -p --resume "$CLAUDE_SESSION_ID"`.
- Do not use `claude --continue`, bare `claude --resume`, or any "most recent session" behavior. Every run must resume only the session id captured for that run.

Verified locally:

- `claude -p --output-format json` returns top-level `session_id` and `result`.
- `claude -p --resume "$session_id"` can resume a session created by `claude -p` and preserve context.

## Tunable Variables

| Var | Default | Meaning |
|-----|---------|---------|
| `MAX_ROUNDS` | `5` | Hard cap on review rounds. The loop always terminates here. |
| `RUN_ROOT` | `.grill-codex/runs/<timestamp>-<slug>` | Isolated per-run artifact directory. |
| `CACHE_FILE` | `.grill-codex/cache/WORKSPACE-CONTEXT.md` | Optional, non-authoritative workspace navigation cache. |

If the user supplies an argument like `rounds=3`, use that for `MAX_ROUNDS`.

## Run Artifacts

Each invocation creates a unique run directory:

```text
.grill-codex/runs/<timestamp>-<slug>/
|-- PLAN.md
|-- PLAN-REVIEW-LOG.md
|-- WORKSPACE-CONTEXT.snapshot.md
`-- metadata.json
```

`metadata.json` schema:

```json
{
  "run_id": "2026-06-10T08-35-00-add-auth-check",
  "claude_session_id": "92ad2038-6670-4dbe-96dd-5ad8e76cfbe0",
  "max_rounds": 5,
  "repo_root": "/absolute/path/to/repo",
  "context_cache_fingerprint": {
    "git_head": "abc123...",
    "dirty_fingerprint": "sha256-or-summary",
    "cache_generated_at": "2026-06-10T08:35:00+08:00"
  }
}
```

Run artifacts under `.grill-codex/` are ephemeral by design and should be gitignored. If the final plan should be committed or shared, export it intentionally to a tracked path.

## Workspace Context Cache

The workspace cache is optional. It is only a startup accelerator, never a source of truth.

Cache path:

```text
.grill-codex/cache/WORKSPACE-CONTEXT.md
```

Minimum cache content:

- Generation timestamp.
- Git HEAD.
- Dirty status fingerprint.
- Truncated file tree.
- Key manifest/config summaries.
- Major modules or entrypoints.

Reuse the cache only when Git HEAD and dirty fingerprint match. At run start, copy it into `WORKSPACE-CONTEXT.snapshot.md`. After that, the run uses its own snapshot. If correctness depends on a file, Codex or Claude must inspect the actual file.

## Flow

### Step 0 - Initialize the Run

1. Confirm the task scope in one sentence. If no task was provided, ask one concise question.
2. Create a unique `RUN_ROOT`.
3. Write the initial plan to `RUN_ROOT/PLAN.md`:

```markdown
# Plan: <task>
_Round 0 - initial draft by Codex_

## Goal
<one paragraph>

## Approach
<numbered, concrete steps>

## Key decisions & tradeoffs
<contestable choices Claude should review>

## Risks / open questions
<material uncertainty>

## Out of scope
<bounds>
```

4. Initialize `RUN_ROOT/PLAN-REVIEW-LOG.md`.
5. Copy or generate `RUN_ROOT/WORKSPACE-CONTEXT.snapshot.md`.
6. Initialize `metadata.json` with `claude_session_id` set to `null`.

### Step 1 - Round 1 Creates the Claude Session

Invoke Claude with its normal tool access:

```bash
claude -p \
  --output-format json \
  --permission-mode plan \
  "<round 1 review prompt>"
```

Round 1 JSON response shape:

```json
{
  "type": "result",
  "subtype": "success",
  "is_error": false,
  "result": "Concrete review text...\nVERDICT: REVISE",
  "session_id": "92ad2038-6670-4dbe-96dd-5ad8e76cfbe0",
  "stop_reason": "end_turn"
}
```

Parse exactly:

- `session_id` from the top-level JSON field `session_id`; write it to `metadata.json`.
- Review text from the top-level JSON string field `result`.
- Verdict from the last non-empty line of `result`.

Valid verdict lines:

```text
VERDICT: APPROVED
VERDICT: REVISE
```

### Step 2 - Rounds 2..MAX Resume the Same Claude Session

For every follow-up round, resume only the explicit session captured in Round 1:

```bash
claude -p \
  --resume "$CLAUDE_SESSION_ID" \
  --output-format json \
  --permission-mode plan \
  "<follow-up review prompt>"
```

Hard rules:

- Never use `claude --continue`.
- Never use `claude --resume` without an explicit id.
- Never share a Claude session id across runs.
- Claude may inspect the repository and run checks as needed for review quality.
- Claude should not implement the plan; Codex remains responsible for plan revisions and code changes.

Claude session memory provides multi-round continuity. Run-local artifacts remain the authority and recovery point.

### Step 3 - Prompt Template

Each Claude prompt must be self-contained enough to recover from stale memory and must point Claude at the run artifacts:

```text
You are an adversarial reviewer for an implementation plan.

Codex is the builder/orchestrator. You are the critic.
Use the tools you need to inspect the repository and validate the plan. Your job
is to produce a high-quality review, not to implement the plan. Do not make
source changes unless Codex explicitly asks for implementation after the review
loop has completed.

Repo root: <absolute repo root>
Run root: <absolute RUN_ROOT>

Read the current plan at:
<RUN_ROOT>/PLAN.md

Read the review log at:
<RUN_ROOT>/PLAN-REVIEW-LOG.md

Read the workspace context snapshot at:
<RUN_ROOT>/WORKSPACE-CONTEXT.snapshot.md

Review the plan skeptically. Identify concrete flaws: security holes, race
conditions, missing edge cases, schema conflicts, wrong assumptions,
observability gaps, migration/compatibility risks, and simpler alternatives.
For each issue, give a one-line fix.

End your reply with exactly one final line:
VERDICT: APPROVED
or
VERDICT: REVISE
```

For follow-up rounds, add:

```text
This is round <n>. Re-check whether prior findings in PLAN-REVIEW-LOG.md were
addressed. Do not re-litigate resolved points unless the revision introduced a
new problem.
```

Known v1 boundary: full plans, logs, and context snapshots can become large in big repositories or late rounds. If context limits become a problem, stop and report unresolved rather than silently dropping important history.

### Step 4 - After Each Claude Response

Append to `PLAN-REVIEW-LOG.md`:

```markdown
## Round <n> - Claude

<full Claude result text>
```

If verdict is `VERDICT: APPROVED`, stop and present the final plan plus the round count.

If verdict is `VERDICT: REVISE`, Codex decides what is actually worth acting on. Codex is the final arbiter: incorporate good critiques, reject bad critiques with a reason, revise `PLAN.md`, and append:

```markdown
### Codex response

- Changed: <what changed>
- Rejected: <what was rejected and why>
```

Then continue to the next round.

If `ROUND > MAX_ROUNDS`, stop as unresolved. List the remaining Claude objections and Codex's counter-position. Do not fake approval.

## Error Handling

Treat these as Claude call failures:

- `claude -p` exits non-zero.
- JSON cannot be parsed.
- Round 1 JSON lacks `session_id`.
- JSON lacks a string `result`.
- `result` has no valid verdict as its last non-empty line.
- `claude -p --resume "$CLAUDE_SESSION_ID"` fails because the session is missing, stale, expired, or corrupted.

For any Claude call failure:

1. Retry the same prompt up to 2 times.
2. Append the failure and retry count to `PLAN-REVIEW-LOG.md`.
3. If still failing, stop as unresolved. Do not pretend the plan was approved.

## Multi-Session Isolation

Multiple runs in the same workspace must not share:

- `PLAN.md`
- `PLAN-REVIEW-LOG.md`
- Verdict files
- `metadata.json`
- `claude_session_id`
- Codex state

The optional workspace context cache may be shared only by copying it into a run-local snapshot. It must never be used as mutable session state.

## What Not To Do

- Do not use this from Claude Code as a Claude-primary skill. Use `codex-review` for that direction.
- Do not restrict Claude to `Read` only; high-quality review needs normal exploration tools.
- Do not use `--continue` or resume the "latest" session.
- Do not store run artifacts in root-level `PLAN.md` unless the user explicitly wants a committed/exported plan.
