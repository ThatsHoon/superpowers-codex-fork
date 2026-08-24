# Codex Review Dispatch — Design

Date: 2026-08-24
Status: approved by user (in-session, brainstorming)
Base: superpowers 6.3.0 (forked wholesale into this repo)

## Background

Original motivation: Claude Code's parallel-subagent usage moved to separately-metered "Pool 2" billing (2026-06-15), turning included subagent fan-out into 12x-175x priced usage once the credit pool runs out. This fork offloads the SDD (subagent-driven-development) review seat — not the implementer seat — onto Codex CLI via the already-owned `codex-mcp-server` bridge, to cut Claude subagent spend without changing who implements.

**Existing operating principle, kept unchanged:** review = freely delegated, implementation = supervised directly (Claude Agent tool). This design only changes *which backend* the review seat calls — SDD's task loop, fix loop, model-tiering philosophy, and ledger-as-recovery-mechanism are otherwise untouched.

## Two repos change

1. `codex-mcp-server` (`C:\Users\erifkim\my_projects\codex-mcp-server`, TypeScript, owned/modifiable) — add review tracking.
2. This repo (`superpowers-codex-fork`, forked from superpowers 6.3.0) — route SDD's three review touchpoints through Codex instead of a Claude Agent reviewer persona.

## Current state (grounding, verified by reading source)

- `codex-mcp-server/src/jobs/store.ts`: `InMemoryJobStore` backs `codexStart`/`codexJobStatus`/`codexJobList` only (async fire-and-forget jobs). Pure in-memory `Map`, 24h TTL, 200-job cap. No metadata field linking a job to any external plan/task.
- `codex-mcp-server/src/session/storage.ts`: in-memory session Map for the `codex` tool's `sessionId` resume, 24h TTL, 100-session cap. Unrelated to review.
- The `review` MCP tool (`mcp__codex__review`) is **fully synchronous**: takes `base`/`commit`/`uncommitted`, an optional custom `prompt` (instructions/focus, mutually exclusive with `uncommitted`), `title`, `model`, `workingDirectory`; runs `codex exec review ...` and returns the review text directly. **Nothing is persisted server-side today** — the only record of a past review is whatever the caller (Claude) captured in its own files.
- MCP transport cannot push unsolicited completion notifications (confirmed in `store.ts`'s own comment) — any tracking is poll-based by construction, permanently.
- superpowers 6.3.0's SDD review touchpoints, confirmed by file:
  - Per-task review: `skills/subagent-driven-development/task-reviewer-prompt.md`, dispatched via `scripts/review-package PLAN_FILE BASE HEAD`.
  - Fix-loop scoped re-review: `skills/subagent-driven-development/re-review-prompt.md`, dispatched via `scripts/review-package PLAN_FILE FIX_BASE HEAD`.
  - Final whole-branch review and the standalone `requesting-code-review` skill: both delegate to `skills/requesting-code-review/code-reviewer.md`.
- Ledger (`<repo-root>/.superpowers/sdd/<plan-basename>/progress.md`) is the durable recovery record; the codex-mcp-server stores stay in-memory by design (matches existing `JobStore`/`SessionStorage` pattern) — durability continues to come from the ledger, not the server.

## Goals

- All three SDD review touchpoints (task-review, re-review, final-review) call `mcp__codex__review` instead of dispatching a Claude Agent reviewer persona.
- `codex-mcp-server` gains a lightweight, queryable record of what it reviewed (plan/task/round/phase-tagged), independent of any one ledger file, without taking on new dependencies or persistence machinery.
- SDD's existing verdict contract (spec ✅/❌, Critical/Important/Minor findings, task quality approved/not) is preserved by injecting the existing reviewer prompt templates as Codex's custom `prompt`, not by rewriting the templates.
- A broken/unavailable Codex CLI does not strand a plan — one retry, then fallback to the original Claude Agent reviewer, ledgered.

## Non-goals

- Codex as implementer. Out of scope per explicit decision — implementation stays on Claude's Agent tool.
- Durable (disk/DB) persistence in `codex-mcp-server`. The ledger is already the durable record; adding a second durable store is unjustified duplication (YAGNI).
- Webhook/callback plumbing. MCP transport cannot push notifications (verified in current source); building for it now is speculative.
- Changing SDD's fix-loop, breaker, adjudication, or model-tiering *logic* — only the review backend changes.
- The standalone `requesting-code-review` skill (used outside SDD, ad hoc `{BASE_SHA, HEAD_SHA, DESCRIPTION, PLAN_OR_REQUIREMENTS}` shape, no plan file). Only the three plan-file-scoped SDD touchpoints are in scope — see Open items.

## Design

### A. `codex-mcp-server` changes

- **New `ReviewStore`** (`src/tracking/review-store.ts`), sibling to `InMemoryJobStore`, same shape (`create`/`markCompleted`/`markFailed`/`get`/`list`), same in-memory/TTL(24h)/cap(200) policy. Kept separate from `JobStore` rather than generalizing both into one class — smaller diff, zero risk to the existing async-job path.
  ```typescript
  export interface ReviewRecord {
    id: string;
    status: 'running' | 'completed' | 'failed';
    planId?: string;
    taskId?: string;
    round?: number;
    phase?: 'task-review' | 're-review' | 'final-review';
    model?: string;
    startedAt: Date;
    completedAt?: Date;
    output?: string;   // raw review text, unparsed
    error?: string;
  }
  ```
- **`review` tool input schema** (`src/tools/definitions.ts`): add optional `planId`, `taskId`, `round`, `phase` fields. Passed straight through to `ReviewStore.create()`; never interpreted or validated against any external system — the server stays a dumb, generic recorder.
- **`review` handler** (`src/tools/handlers.ts`): create the `ReviewRecord` (running) before invoking `codex exec review`, `markCompleted`/`markFailed` after, still returns the review text synchronously to the caller (no behavior change to the calling contract — this is additive).
- **New tools** `reviewStatus` (by `reviewId`) and `reviewList` (optional `planId`/`taskId` filter) — mirror `codexJobStatus`/`codexJobList` exactly, giving audit/introspection independent of any single ledger file or session.
- No new npm dependencies.

### B. `superpowers-codex-fork` changes

- **New script** `skills/subagent-driven-development/scripts/codex-review-dispatch`, same calling convention as `review-package`/`task-brief`: takes `PLAN_FILE BASE HEAD PHASE` (`task-review`|`re-review`|`final-review`), plus round number for re-reviews.
  1. Builds the diff file via the existing `review-package` script (unchanged).
  2. Loads the phase-appropriate template verbatim (`task-reviewer-prompt.md` / `re-review-prompt.md` / `../requesting-code-review/code-reviewer.md`) plus the plan's Global Constraints block, as the `prompt` override.
  3. Calls `mcp__codex__review` with `base`, the composed `prompt`, `planId` (plan-file slug), `taskId`, `round`, `phase`, and the phase's model (see Model Selection below).
  4. Writes the returned text to a review-package-shaped file under the plan's workspace, prints `reviewId` + file path.
- **SKILL.md updates** (`subagent-driven-development/SKILL.md` only — see Non-goals): every "dispatch task reviewer / scoped re-review / final code reviewer" step now reads "run `codex-review-dispatch`" instead of dispatching a Claude Agent reviewer persona. Process diagrams updated to match. `requesting-code-review/code-reviewer.md` is reused as the final-review template (see Data flow) but the standalone `requesting-code-review` skill's own dispatch step is untouched — it takes ad hoc `{BASE_SHA, HEAD_SHA, DESCRIPTION, PLAN_OR_REQUIREMENTS}` outside any plan file, a different call shape than `codex-review-dispatch` (which is plan-file-scoped, matching `review-package`/`task-brief`). Building a second script signature for that ad hoc shape now, before this one has run once for real, is scope the confirmed decision ("3자리 다 codex" = task-review/re-review/final-review, the three SDD touchpoints) didn't ask for. Tracked as follow-up.
- **Model Selection section** gains a Codex-model note, correcting an initial assumption from this design's first draft: `codex-mcp-server/src/types.ts` records (verified empirically 2026-08-05 against codex-cli 0.146.0, comment on `AVAILABLE_CODEX_MODELS`) that the entire `*-codex` model family (`gpt-5.3-codex`, `gpt-5.2-codex`, `gpt-5.1-codex`, etc.) returns HTTP 400 "not supported when using Codex with a ChatGPT account" under this account's auth (ChatGPT subscription, not API key). Only `gpt-5.6-sol` and `gpt-5.6-terra` are confirmed working. With two working models and no documented cost/capability delta between them, there is no real tiering to build: **all three review phases use `gpt-5.6-sol`** (the existing `DEFAULT_REVIEW_MODEL`, already the reviewer role's pinned default) unless `CODEX_REVIEW_MODEL_ENV_VAR` overrides it. If the account later moves to API-key auth, the `*-codex` family becomes available and a real phase-tiering can be reconsidered then — not speculatively built now.
- **Ledger line format** gains a reviewer tag, appended to the existing completion/fix-round/parked lines:
  `Task <N>: complete (commits a1b2c3d..d4e5f6a, review clean) [reviewer: codex#<reviewId> model:gpt-5.6-sol]`

### Data flow

1. SDD controller reaches a review point (task-review, re-review, or final-review — unchanged trigger conditions).
2. Calls `codex-review-dispatch` → diff file (existing `review-package`) + rubric prompt → `mcp__codex__review(base, prompt, planId, taskId, round, phase, model)`.
3. `codex-mcp-server`: `ReviewStore.create()` (running) → shells out via the existing `command.ts` path → on completion, stores raw output (completed) → returns `{reviewId, output}` synchronously.
4. Controller reads the returned text, applies the **unchanged** fix-loop/adjudication logic from `subagent-driven-development/SKILL.md`, writes the ledger line with the `reviewId` tag.
5. `reviewList --planId <slug>` is available afterward for cross-session/cross-repo audit, independent of the ledger file surviving.

### Error handling

- Codex CLI failure (nonzero exit / crash) → `ReviewStore.markFailed`, MCP tool error surfaces to the dispatch script → **retry once**; if it fails again, **fall back to the original Claude Agent reviewer dispatch** (unmodified 6.3.0 behavior), and ledger the fallback: `Task <N>: reviewer fallback (codex failed twice) — used Claude Agent`.
- Output-format drift (Codex's free-form review text doesn't exactly match the injected rubric's expected shape) is a soft risk, not a hard failure: the controller (Claude) is reading and interpreting the text regardless of exact formatting, same as it already tolerates variance from Claude Agent reviewers. Flagged as the first validation task below rather than built as an error branch.

### Testing

- `codex-mcp-server`: unit tests for `ReviewStore` mirroring the existing `JobStore` test suite; schema validation tests for the new `review` input fields; a handler lifecycle test (mocked `codex exec`) verifying running → completed/failed transitions and that `reviewList`/`reviewStatus` reflect them.
- `superpowers-codex-fork`: a first implementation task (not a unit test) that runs `codex-review-dispatch` against a real small diff and manually confirms the returned text is readable as a spec ✅/❌ + findings verdict, before the rest of the SKILL.md rewiring proceeds. This is the practical check on the "output-format drift" risk above.

## Open items carried forward (not this spec's scope)

- A second dispatch script (or a mode flag on `codex-review-dispatch`) for the standalone `requesting-code-review` skill's ad hoc `{BASE_SHA, HEAD_SHA, DESCRIPTION, PLAN_OR_REQUIREMENTS}` shape, once the plan-file-scoped version has run for real.
- `CLAUDE.md` (both global and this project's) point at a memory file `codex-mcp-orchestration` that does not exist (`~/.claude/projects/.../memory/` has no such file). Once this fork ships, that memory should be written to reflect the actual, current operating principles — out of scope for this spec.
