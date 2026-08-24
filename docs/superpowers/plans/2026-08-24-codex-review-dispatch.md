# Codex Review Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route SDD's three review touchpoints (task-review, re-review, final-review) through `mcp__codex__review` instead of a Claude Agent reviewer persona, with a new server-side `ReviewStore` giving `codex-mcp-server` a queryable record of what it reviewed.

**Architecture:** Two repos, worked in order. Tasks 1-3 are in `codex-mcp-server` (`C:\Users\erifkim\my_projects\codex-mcp-server`, TypeScript, existing test suite via Jest): add a `ReviewStore` alongside the existing `JobStore`, extend the `review` tool's schema with `planId`/`taskId`/`round`/`phase`, add `reviewStatus`/`reviewList` tools. Tasks 4-7 are in this repo (`superpowers-codex-fork`, forked from superpowers 6.3.0): a new prep script `codex-review-dispatch` (same division of labor as the existing `review-package`/`task-brief` — the script prepares files, the controller performs the actual MCP tool call, since a bash script cannot itself invoke an MCP tool), then `subagent-driven-development/SKILL.md` rewired to use it at all three touchpoints. `cd` into the relevant repo before running each task's commands — the two repos have independent git roots.

**Tech Stack:** TypeScript + Zod + Jest (codex-mcp-server, existing); Bash (superpowers-codex-fork scripts, existing convention).

**Spec:** `docs/superpowers/specs/2026-08-24-codex-review-dispatch-design.md` (this repo)

## Global Constraints

- No new npm dependencies in `codex-mcp-server` — `ReviewStore` is in-memory TypeScript only, same as the existing `JobStore`.
- Server-side stores (`ReviewStore`, `JobStore`, session storage) stay in-memory. No SQLite/disk persistence — the SDD ledger remains the durable recovery record.
- Codex review calls use `gpt-5.6-sol` only. `codex-mcp-server/src/types.ts` records (verified empirically 2026-08-05 against codex-cli 0.146.0) that the entire `*-codex` model family returns HTTP 400 under this account's ChatGPT-subscription auth — do not reference `gpt-5.2-codex`, `gpt-5.1-codex`, or any other `*-codex` name anywhere in this plan's code or scripts.
- The implementer role stays on Claude's native Agent tool. Codex never dispatches implementation — only the three review touchpoints change backend.
- Scope is exactly three SDD touchpoints in `subagent-driven-development`: task-review, re-review, final-review. The standalone `requesting-code-review` skill (ad hoc `{BASE_SHA, HEAD_SHA, DESCRIPTION, PLAN_OR_REQUIREMENTS}` shape, no plan file) is untouched — out of scope, tracked as a spec follow-up.
- All imports in `codex-mcp-server` TypeScript files use the `.js` extension (ESM requirement already established in that codebase's `CLAUDE.md`).

---

### Task 1: ReviewStore

**Files:**
- Create: `C:\Users\erifkim\my_projects\codex-mcp-server\src\tracking\review-store.ts`
- Test: `C:\Users\erifkim\my_projects\codex-mcp-server\src\__tests__\review-store.test.ts`

**Interfaces:**
- Produces: `ReviewStatus` (`'running' | 'completed' | 'failed'`), `ReviewPhase` (`'task-review' | 're-review' | 'final-review'`), `ReviewRecord` interface, `ReviewStoreMeta` interface, `ReviewStore` interface (`create(meta): string`, `markCompleted(reviewId, output): void`, `markFailed(reviewId, error): void`, `get(reviewId): ReviewRecord | undefined`, `list(filter?): ReviewRecord[]`), `InMemoryReviewStore` class implementing it. Task 3 consumes all of these.

- [ ] **Step 1: Write the failing test**

```typescript
// C:\Users\erifkim\my_projects\codex-mcp-server\src\__tests__\review-store.test.ts
import { InMemoryReviewStore } from '../tracking/review-store.js';

describe('InMemoryReviewStore', () => {
  test('create returns a reviewId and starts the record as running', () => {
    const store = new InMemoryReviewStore();
    const id = store.create({ planId: 'plan-a', taskId: '3', phase: 'task-review' });

    expect(id).toBeTruthy();
    const record = store.get(id);
    expect(record?.status).toBe('running');
    expect(record?.planId).toBe('plan-a');
    expect(record?.taskId).toBe('3');
    expect(record?.phase).toBe('task-review');
  });

  test('markCompleted transitions to completed and stores output', () => {
    const store = new InMemoryReviewStore();
    const id = store.create({ phase: 'final-review' });

    store.markCompleted(id, 'Spec ✅ - all requirements met.');

    const record = store.get(id);
    expect(record?.status).toBe('completed');
    expect(record?.output).toBe('Spec ✅ - all requirements met.');
    expect(record?.completedAt).toBeInstanceOf(Date);
  });

  test('markFailed transitions to failed and stores the error', () => {
    const store = new InMemoryReviewStore();
    const id = store.create({ phase: 're-review', round: 2 });

    store.markFailed(id, 'codex exec exited 1');

    const record = store.get(id);
    expect(record?.status).toBe('failed');
    expect(record?.error).toBe('codex exec exited 1');
  });

  test('get on unknown reviewId returns undefined', () => {
    const store = new InMemoryReviewStore();
    expect(store.get('does-not-exist')).toBeUndefined();
  });

  test('list returns newest first', () => {
    const store = new InMemoryReviewStore();
    const first = store.create({ planId: 'p', phase: 'task-review' });
    const second = store.create({ planId: 'p', phase: 'task-review' });

    const results = store.list();
    expect(results[0].id).toBe(second);
    expect(results[1].id).toBe(first);
  });

  test('list filters by planId and taskId', () => {
    const store = new InMemoryReviewStore();
    const wanted = store.create({ planId: 'plan-a', taskId: '1', phase: 'task-review' });
    store.create({ planId: 'plan-b', taskId: '1', phase: 'task-review' });
    store.create({ planId: 'plan-a', taskId: '2', phase: 'task-review' });

    const results = store.list({ planId: 'plan-a', taskId: '1' });
    expect(results.map((r) => r.id)).toEqual([wanted]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run (from `C:\Users\erifkim\my_projects\codex-mcp-server`): `npx jest src/__tests__/review-store.test.ts`
Expected: FAIL — `Cannot find module '../tracking/review-store.js'`

- [ ] **Step 3: Write the implementation**

```typescript
// C:\Users\erifkim\my_projects\codex-mcp-server\src\tracking\review-store.ts
import { randomUUID } from 'crypto';

export type ReviewStatus = 'running' | 'completed' | 'failed';
export type ReviewPhase = 'task-review' | 're-review' | 'final-review';

export interface ReviewRecord {
  id: string;
  status: ReviewStatus;
  planId?: string;
  taskId?: string;
  round?: number;
  phase?: ReviewPhase;
  model?: string;
  startedAt: Date;
  completedAt?: Date;
  output?: string;
  error?: string;
}

export interface ReviewStoreMeta {
  planId?: string;
  taskId?: string;
  round?: number;
  phase?: ReviewPhase;
  model?: string;
}

export interface ReviewListFilter {
  planId?: string;
  taskId?: string;
}

export interface ReviewStore {
  create(meta: ReviewStoreMeta): string;
  markCompleted(reviewId: string, output: string): void;
  markFailed(reviewId: string, error: string): void;
  get(reviewId: string): ReviewRecord | undefined;
  list(filter?: ReviewListFilter): ReviewRecord[];
}

/**
 * In-memory review tracker, parallel to InMemoryJobStore. A "review" is one
 * `codex review` invocation dispatched via the `review` tool. Unlike jobs,
 * reviews are dispatched synchronously (the caller already blocks on the
 * result — see ReviewToolHandler), so this store's only job is to give
 * callers a queryable record of what was reviewed (plan/task/round/phase
 * tagged) after the fact — via reviewStatus/reviewList — independent of any
 * one caller's own ledger file surviving. It is not a mechanism for making
 * review non-blocking.
 */
export class InMemoryReviewStore implements ReviewStore {
  private reviews = new Map<string, ReviewRecord>();
  private readonly maxReviews = 200;
  private readonly reviewTtl = 24 * 60 * 60 * 1000; // 24 hours

  create(meta: ReviewStoreMeta): string {
    this.cleanupExpired();
    const id = randomUUID();
    this.reviews.set(id, {
      id,
      status: 'running',
      startedAt: new Date(),
      ...meta,
    });
    this.enforceMax();
    return id;
  }

  markCompleted(reviewId: string, output: string): void {
    const review = this.reviews.get(reviewId);
    if (!review) return;
    review.status = 'completed';
    review.output = output;
    review.completedAt = new Date();
  }

  markFailed(reviewId: string, error: string): void {
    const review = this.reviews.get(reviewId);
    if (!review) return;
    review.status = 'failed';
    review.error = error;
    review.completedAt = new Date();
  }

  get(reviewId: string): ReviewRecord | undefined {
    return this.reviews.get(reviewId);
  }

  list(filter?: ReviewListFilter): ReviewRecord[] {
    this.cleanupExpired();
    let results = Array.from(this.reviews.values());
    if (filter?.planId) {
      results = results.filter((r) => r.planId === filter.planId);
    }
    if (filter?.taskId) {
      results = results.filter((r) => r.taskId === filter.taskId);
    }
    return results.sort((a, b) => b.startedAt.getTime() - a.startedAt.getTime());
  }

  private cleanupExpired(): void {
    const now = Date.now();
    for (const [id, review] of this.reviews) {
      const anchor = review.completedAt ?? review.startedAt;
      if (now - anchor.getTime() > this.reviewTtl) {
        this.reviews.delete(id);
      }
    }
  }

  private enforceMax(): void {
    if (this.reviews.size <= this.maxReviews) return;
    const sorted = Array.from(this.reviews.values()).sort(
      (a, b) => a.startedAt.getTime() - b.startedAt.getTime()
    );
    for (const review of sorted.slice(0, this.reviews.size - this.maxReviews)) {
      this.reviews.delete(review.id);
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx jest src/__tests__/review-store.test.ts`
Expected: PASS (6/6)

- [ ] **Step 5: Commit**

```bash
cd "C:\Users\erifkim\my_projects\codex-mcp-server"
git add src/tracking/review-store.ts src/__tests__/review-store.test.ts
git commit -m "feat: add InMemoryReviewStore, parallel to InMemoryJobStore"
```

---

### Task 2: Extend tool surface — types.ts + definitions.ts

**Files:**
- Modify: `C:\Users\erifkim\my_projects\codex-mcp-server\src\types.ts:4-14` (TOOLS constant), `:131-139` (ReviewToolSchema), after `:139` (new schemas), `:175-182` (type exports)
- Modify: `C:\Users\erifkim\my_projects\codex-mcp-server\src\tools\definitions.ts:171-204` (REVIEW properties), after `:214` (new tool definitions)

**Interfaces:**
- Consumes: nothing from Task 1 directly (this task only touches schemas/definitions).
- Produces: `TOOLS.REVIEW_STATUS = 'reviewStatus'`, `TOOLS.REVIEW_LIST = 'reviewList'`; extended `ReviewToolSchema` (adds optional `planId: string`, `taskId: string`, `round: number`, `phase: 'task-review'|'re-review'|'final-review'`); `ReviewStatusToolSchema` (`reviewId: string`, required), `ReviewListToolSchema` (`planId?: string`, `taskId?: string`); `ReviewStatusToolArgs`, `ReviewListToolArgs` types; `toolDefinitions` entries for `reviewStatus`/`reviewList`. Task 3 consumes all of these.

- [ ] **Step 1: Add the two new tool names to `TOOLS`**

In `src/types.ts`, replace:

```typescript
export const TOOLS = {
  CODEX: 'codex',
  CODEX_START: 'codexStart',
  CODEX_JOB_STATUS: 'codexJobStatus',
  CODEX_JOB_LIST: 'codexJobList',
  REVIEW: 'review',
  PING: 'ping',
  HELP: 'help',
  LIST_SESSIONS: 'listSessions',
  WEBSEARCH: 'websearch',
} as const;
```

with:

```typescript
export const TOOLS = {
  CODEX: 'codex',
  CODEX_START: 'codexStart',
  CODEX_JOB_STATUS: 'codexJobStatus',
  CODEX_JOB_LIST: 'codexJobList',
  REVIEW: 'review',
  REVIEW_STATUS: 'reviewStatus',
  REVIEW_LIST: 'reviewList',
  PING: 'ping',
  HELP: 'help',
  LIST_SESSIONS: 'listSessions',
  WEBSEARCH: 'websearch',
} as const;
```

- [ ] **Step 2: Extend `ReviewToolSchema` and add the two new schemas**

Replace:

```typescript
// Review tool schema
export const ReviewToolSchema = z.object({
  prompt: z.string().optional(),
  uncommitted: z.boolean().optional(),
  base: z.string().optional(),
  commit: z.string().optional(),
  title: z.string().optional(),
  model: z.string().optional(),
  workingDirectory: z.string().optional(),
});
```

with:

```typescript
// Review tool schema
export const ReviewToolSchema = z.object({
  prompt: z.string().optional(),
  uncommitted: z.boolean().optional(),
  base: z.string().optional(),
  commit: z.string().optional(),
  title: z.string().optional(),
  model: z.string().optional(),
  workingDirectory: z.string().optional(),
  // Fork-local addition: correlate a review to the caller's plan/task/round
  // so codex-mcp-server can answer reviewStatus/reviewList queries. Never
  // interpreted server-side beyond storage — the caller's ledger remains
  // the durable record.
  planId: z.string().optional(),
  taskId: z.string().optional(),
  round: z.number().int().min(1).optional(),
  phase: z.enum(['task-review', 're-review', 'final-review']).optional(),
});

// reviewStatus / reviewList tool schemas — mirror JobStatusToolSchema /
// JobListToolSchema, but for review records instead of async jobs.
export const ReviewStatusToolSchema = z.object({
  reviewId: z.string().min(1, 'reviewId is required'),
});

export const ReviewListToolSchema = z.object({
  planId: z.string().optional(),
  taskId: z.string().optional(),
});
```

- [ ] **Step 3: Export the two new arg types**

Replace:

```typescript
export type CodexToolArgs = z.infer<typeof CodexToolSchema>;
export type CodexStartToolArgs = z.infer<typeof CodexStartToolSchema>;
export type JobStatusToolArgs = z.infer<typeof JobStatusToolSchema>;
export type JobListToolArgs = z.infer<typeof JobListToolSchema>;
export type ReviewToolArgs = z.infer<typeof ReviewToolSchema>;
export type PingToolArgs = z.infer<typeof PingToolSchema>;
export type ListSessionsToolArgs = z.infer<typeof ListSessionsToolSchema>;
export type WebSearchToolArgs = z.infer<typeof WebSearchToolSchema>;
```

with:

```typescript
export type CodexToolArgs = z.infer<typeof CodexToolSchema>;
export type CodexStartToolArgs = z.infer<typeof CodexStartToolSchema>;
export type JobStatusToolArgs = z.infer<typeof JobStatusToolSchema>;
export type JobListToolArgs = z.infer<typeof JobListToolSchema>;
export type ReviewToolArgs = z.infer<typeof ReviewToolSchema>;
export type ReviewStatusToolArgs = z.infer<typeof ReviewStatusToolSchema>;
export type ReviewListToolArgs = z.infer<typeof ReviewListToolSchema>;
export type PingToolArgs = z.infer<typeof PingToolSchema>;
export type ListSessionsToolArgs = z.infer<typeof ListSessionsToolSchema>;
export type WebSearchToolArgs = z.infer<typeof WebSearchToolSchema>;
```

- [ ] **Step 4: Add `planId`/`taskId`/`round`/`phase` to the REVIEW tool's input properties**

In `src/tools/definitions.ts`, inside the `TOOLS.REVIEW` object's `inputSchema.properties`, replace:

```typescript
        workingDirectory: {
          type: 'string',
          description:
            'Working directory to run the review in (passed via -C as a global Codex option)',
        },
      },
      required: [],
    },
    annotations: {
      title: 'Code Review',
```

with:

```typescript
        workingDirectory: {
          type: 'string',
          description:
            'Working directory to run the review in (passed via -C as a global Codex option)',
        },
        planId: {
          type: 'string',
          description:
            'Fork-local: correlate this review to a plan (e.g. the plan file slug) for reviewStatus/reviewList lookups. Purely a tracking tag — never interpreted.',
        },
        taskId: {
          type: 'string',
          description:
            'Fork-local: correlate this review to a task number within the plan. Tracking tag only.',
        },
        round: {
          type: 'number',
          description:
            'Fork-local: fix-loop round number, for re-review calls. Tracking tag only.',
        },
        phase: {
          type: 'string',
          enum: ['task-review', 're-review', 'final-review'],
          description:
            'Fork-local: which SDD review touchpoint this call is for. Tracking tag only.',
        },
      },
      required: [],
    },
    annotations: {
      title: 'Code Review',
```

- [ ] **Step 5: Add `reviewStatus` and `reviewList` tool definitions**

In `src/tools/definitions.ts`, immediately after the `TOOLS.REVIEW` object's closing `},` (the one right before the `TOOLS.PING` object begins), insert:

```typescript
  {
    name: TOOLS.REVIEW_STATUS,
    description:
      'Check the status of a review started with review. Returns "running" until the process exits, then "completed" (with the review text) or "failed" (with an error message).',
    inputSchema: {
      type: 'object',
      properties: {
        reviewId: {
          type: 'string',
          description: 'The reviewId returned by review',
        },
      },
      required: ['reviewId'],
    },
    annotations: {
      title: 'Check Review Status',
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: false,
    },
  },
  {
    name: TOOLS.REVIEW_LIST,
    description:
      'List tracked review calls (running, completed, and failed), newest first. Optionally filter by planId and/or taskId.',
    inputSchema: {
      type: 'object',
      properties: {
        planId: {
          type: 'string',
          description: 'Only list reviews tagged with this planId',
        },
        taskId: {
          type: 'string',
          description: 'Only list reviews tagged with this taskId',
        },
      },
      required: [],
    },
    annotations: {
      title: 'List Reviews',
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: false,
    },
  },
```

- [ ] **Step 6: Verify it compiles**

Run (from `C:\Users\erifkim\my_projects\codex-mcp-server`): `npm run build`
Expected: succeeds with no TypeScript errors (the new schema/definition fields are additive and optional; nothing consumes `ReviewStatusToolArgs`/`ReviewListToolArgs`/`TOOLS.REVIEW_STATUS`/`TOOLS.REVIEW_LIST` yet, so no unused-export errors under this project's `tsconfig.json`, which does not enable `noUnusedLocals` for exported symbols)

- [ ] **Step 7: Commit**

```bash
cd "C:\Users\erifkim\my_projects\codex-mcp-server"
git add src/types.ts src/tools/definitions.ts
git commit -m "feat: add planId/taskId/round/phase to review schema; add reviewStatus/reviewList tool definitions"
```

---

### Task 3: Wire handlers.ts — tracking + two new handlers + registry

**Files:**
- Modify: `C:\Users\erifkim\my_projects\codex-mcp-server\src\tools\handlers.ts:1-25` (imports), `:586-708` (`ReviewToolHandler`), after `:708` (two new classes), `:786-800` (registry)
- Test: `C:\Users\erifkim\my_projects\codex-mcp-server\src\__tests__\review-tracking.test.ts`

**Interfaces:**
- Consumes: `InMemoryReviewStore`, `ReviewStore`, `ReviewRecord` from Task 1 (`../tracking/review-store.js`); `ReviewStatusToolArgs`, `ReviewListToolArgs`, `ReviewStatusToolSchema`, `ReviewListToolSchema`, extended `ReviewToolArgs`/`ReviewToolSchema`, `TOOLS.REVIEW_STATUS`, `TOOLS.REVIEW_LIST` from Task 2 (`../types.js`).
- Produces: `ReviewToolHandler` now takes a `ReviewStore` in its constructor and tracks every call; `ReviewStatusToolHandler`, `ReviewListToolHandler` classes; `toolHandlers[TOOLS.REVIEW_STATUS]`, `toolHandlers[TOOLS.REVIEW_LIST]` entries.

- [ ] **Step 1: Write the failing test**

```typescript
// C:\Users\erifkim\my_projects\codex-mcp-server\src\__tests__\review-tracking.test.ts
jest.mock('chalk', () => ({
  default: {
    blue: (text: string) => text,
    yellow: (text: string) => text,
    green: (text: string) => text,
    red: (text: string) => text,
  },
}));

const mockExecuteCommand = jest.fn();
jest.mock('../utils/command.js', () => ({
  executeCommand: (...args: unknown[]) => mockExecuteCommand(...args),
  executeCommandStreaming: (...args: unknown[]) => mockExecuteCommand(...args),
}));

import { InMemoryReviewStore } from '../tracking/review-store.js';
import {
  ReviewToolHandler,
  ReviewStatusToolHandler,
  ReviewListToolHandler,
} from '../tools/handlers.js';

describe('review tracking (review / reviewStatus / reviewList)', () => {
  test('a completed review is tracked with its planId/taskId/phase and readable via reviewStatus', async () => {
    mockExecuteCommand.mockResolvedValue({ stdout: 'Spec ✅ - clean.', stderr: '' });

    const store = new InMemoryReviewStore();
    const review = new ReviewToolHandler(store);
    const status = new ReviewStatusToolHandler(store);

    const result = await review.execute({
      base: 'main',
      planId: 'my-plan',
      taskId: '3',
      phase: 'task-review',
    });
    const reviewId = (result.content[0]._meta as { reviewId: string }).reviewId;

    expect(reviewId).toBeTruthy();
    expect(store.get(reviewId)?.status).toBe('completed');
    expect(store.get(reviewId)?.planId).toBe('my-plan');
    expect(store.get(reviewId)?.phase).toBe('task-review');

    const statusResult = await status.execute({ reviewId });
    expect((statusResult.content[0]._meta as { status: string }).status).toBe('completed');
    expect(statusResult.content[0].text).toContain('Spec ✅ - clean.');
  });

  test('a failed codex exec marks the review failed, not thrown past the handler', async () => {
    mockExecuteCommand.mockRejectedValue(new Error('codex exec exited 1'));

    const store = new InMemoryReviewStore();
    const review = new ReviewToolHandler(store);

    await expect(review.execute({ base: 'main', phase: 'final-review' })).rejects.toThrow();

    const [record] = store.list();
    expect(record.status).toBe('failed');
    expect(record.error).toContain('codex exec exited 1');
  });

  test('reviewList filters by planId and taskId', async () => {
    mockExecuteCommand.mockResolvedValue({ stdout: 'ok', stderr: '' });

    const store = new InMemoryReviewStore();
    const review = new ReviewToolHandler(store);
    const list = new ReviewListToolHandler(store);

    await review.execute({ base: 'main', planId: 'plan-a', taskId: '1', phase: 'task-review' });
    await review.execute({ base: 'main', planId: 'plan-b', taskId: '1', phase: 'task-review' });

    const listResult = await list.execute({ planId: 'plan-a' });
    const reviews = JSON.parse(listResult.content[0].text as string);
    expect(reviews).toHaveLength(1);
    expect(reviews[0].planId).toBe('plan-a');
  });

  test('reviewStatus on unknown reviewId returns isError, not a thrown exception', async () => {
    const store = new InMemoryReviewStore();
    const status = new ReviewStatusToolHandler(store);

    const result = await status.execute({ reviewId: 'does-not-exist' });
    expect(result.isError).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run (from `C:\Users\erifkim\my_projects\codex-mcp-server`): `npx jest src/__tests__/review-tracking.test.ts`
Expected: FAIL — `ReviewToolHandler` does not accept a constructor argument yet, and `ReviewStatusToolHandler`/`ReviewListToolHandler` do not exist.

- [ ] **Step 3: Add the two new imports**

In `src/tools/handlers.ts`, replace the `from '../types.js'` import block:

```typescript
import {
  TOOLS,
  DEFAULT_CODEX_MODEL,
  DEFAULT_REVIEW_MODEL,
  CODEX_DEFAULT_MODEL_ENV_VAR,
  CODEX_REVIEW_MODEL_ENV_VAR,
  type ToolResult,
  type ToolHandlerContext,
  type CodexToolArgs,
  type CodexStartToolArgs,
  type JobStatusToolArgs,
  type JobListToolArgs,
  type ReviewToolArgs,
  type PingToolArgs,
  type WebSearchToolArgs,
  CodexToolSchema,
  CodexStartToolSchema,
  JobStatusToolSchema,
  JobListToolSchema,
  ReviewToolSchema,
  PingToolSchema,
  HelpToolSchema,
  ListSessionsToolSchema,
  WebSearchToolSchema,
} from '../types.js';
import {
  InMemorySessionStorage,
  type SessionStorage,
  type ConversationTurn,
} from '../session/storage.js';
import { InMemoryJobStore, type JobStore } from '../jobs/store.js';
```

with:

```typescript
import {
  TOOLS,
  DEFAULT_CODEX_MODEL,
  DEFAULT_REVIEW_MODEL,
  CODEX_DEFAULT_MODEL_ENV_VAR,
  CODEX_REVIEW_MODEL_ENV_VAR,
  type ToolResult,
  type ToolHandlerContext,
  type CodexToolArgs,
  type CodexStartToolArgs,
  type JobStatusToolArgs,
  type JobListToolArgs,
  type ReviewToolArgs,
  type ReviewStatusToolArgs,
  type ReviewListToolArgs,
  type PingToolArgs,
  type WebSearchToolArgs,
  CodexToolSchema,
  CodexStartToolSchema,
  JobStatusToolSchema,
  JobListToolSchema,
  ReviewToolSchema,
  ReviewStatusToolSchema,
  ReviewListToolSchema,
  PingToolSchema,
  HelpToolSchema,
  ListSessionsToolSchema,
  WebSearchToolSchema,
} from '../types.js';
import {
  InMemorySessionStorage,
  type SessionStorage,
  type ConversationTurn,
} from '../session/storage.js';
import { InMemoryJobStore, type JobStore } from '../jobs/store.js';
import { InMemoryReviewStore, type ReviewStore } from '../tracking/review-store.js';
```

- [ ] **Step 4: Track the review call in `ReviewToolHandler`**

Replace the whole `ReviewToolHandler` class:

```typescript
export class ReviewToolHandler {
  async execute(
    args: unknown,
    context: ToolHandlerContext = defaultContext
  ): Promise<ToolResult> {
    try {
      const {
        prompt,
        uncommitted,
        base,
        commit,
        title,
        model,
        workingDirectory,
      }: ReviewToolArgs = ReviewToolSchema.parse(args);

      if (prompt && uncommitted) {
        throw new ValidationError(
          TOOLS.REVIEW,
          'The review prompt cannot be combined with uncommitted=true. Use a base/commit review or omit the prompt.'
        );
      }

      // Resolve to absolute path once so -C and spawn cwd agree
      const resolvedWorkDir = workingDirectory
        ? path.resolve(workingDirectory)
        : undefined;

      // Build command arguments for codex review
      const cmdArgs: string[] = [];

      if (resolvedWorkDir) {
        cmdArgs.push('-C', resolvedWorkDir);
      }

      // Add model parameter via config
      // Reviewer role uses its own default (DEFAULT_REVIEW_MODEL), separate
      // from the implementer role's DEFAULT_CODEX_MODEL, so each can be
      // pinned independently (fork-local change).
      const selectedModel =
        model ||
        process.env[CODEX_REVIEW_MODEL_ENV_VAR] ||
        DEFAULT_REVIEW_MODEL;
      cmdArgs.push('-c', `model="${selectedModel}"`);

      cmdArgs.push('review');

      // Add review-specific flags
      if (uncommitted) {
        cmdArgs.push('--uncommitted');
      }

      if (base) {
        cmdArgs.push('--base', base);
      }

      if (commit) {
        cmdArgs.push('--commit', commit);
      }

      if (title) {
        cmdArgs.push('--title', title);
      }

      // Add custom review instructions if provided
      if (prompt) {
        cmdArgs.push(prompt);
      }

      // Send initial progress notification
      await context.sendProgress('Starting code review...', 0);

      const useStreaming = !!context.progressToken;
      // Pass cwd to spawn so the child process starts in the correct directory.
      // This works around openai/codex#9084 where -C is ignored by `review`.
      const cmdOptions = { cwd: resolvedWorkDir };
      const result = useStreaming
        ? await executeCommandStreaming('codex', cmdArgs, {
            ...cmdOptions,
            onProgress: (message) => {
              context.sendProgress(message);
            },
          })
        : await executeCommand('codex', cmdArgs, cmdOptions);

      // Codex CLI outputs to stderr, so check both stdout and stderr
      const response =
        result.stdout || result.stderr || 'No review output from Codex';

      // Prepare metadata for dual approach:
      // - content[0]._meta: For Claude Code compatibility (avoids structuredContent bug)
      // - structuredContent: For other MCP clients that properly support it
      const metadata: Record<string, unknown> = {
        model: selectedModel,
        ...(base && { base }),
        ...(commit && { commit }),
      };

      return {
        content: [
          {
            type: 'text',
            text: response,
            _meta: metadata,
          },
        ],
        structuredContent: isStructuredContentEnabled() ? metadata : undefined,
      };
    } catch (error) {
      if (error instanceof ZodError) {
        throw new ValidationError(TOOLS.REVIEW, error.message);
      }
      if (error instanceof ValidationError) {
        throw error;
      }
      throw new ToolExecutionError(
        TOOLS.REVIEW,
        'Failed to execute code review',
        error
      );
    }
  }
}
```

with:

```typescript
export class ReviewToolHandler {
  constructor(private reviewStore: ReviewStore) {}

  async execute(
    args: unknown,
    context: ToolHandlerContext = defaultContext
  ): Promise<ToolResult> {
    let reviewId: string | undefined;
    try {
      const {
        prompt,
        uncommitted,
        base,
        commit,
        title,
        model,
        workingDirectory,
        planId,
        taskId,
        round,
        phase,
      }: ReviewToolArgs = ReviewToolSchema.parse(args);

      if (prompt && uncommitted) {
        throw new ValidationError(
          TOOLS.REVIEW,
          'The review prompt cannot be combined with uncommitted=true. Use a base/commit review or omit the prompt.'
        );
      }

      // Resolve to absolute path once so -C and spawn cwd agree
      const resolvedWorkDir = workingDirectory
        ? path.resolve(workingDirectory)
        : undefined;

      // Build command arguments for codex review
      const cmdArgs: string[] = [];

      if (resolvedWorkDir) {
        cmdArgs.push('-C', resolvedWorkDir);
      }

      // Add model parameter via config
      // Reviewer role uses its own default (DEFAULT_REVIEW_MODEL), separate
      // from the implementer role's DEFAULT_CODEX_MODEL, so each can be
      // pinned independently (fork-local change).
      const selectedModel =
        model ||
        process.env[CODEX_REVIEW_MODEL_ENV_VAR] ||
        DEFAULT_REVIEW_MODEL;
      cmdArgs.push('-c', `model="${selectedModel}"`);

      cmdArgs.push('review');

      // Add review-specific flags
      if (uncommitted) {
        cmdArgs.push('--uncommitted');
      }

      if (base) {
        cmdArgs.push('--base', base);
      }

      if (commit) {
        cmdArgs.push('--commit', commit);
      }

      if (title) {
        cmdArgs.push('--title', title);
      }

      // Add custom review instructions if provided
      if (prompt) {
        cmdArgs.push(prompt);
      }

      // Fork-local: track this call before it runs, so a crash mid-review
      // still leaves a 'failed' record instead of no record at all.
      reviewId = this.reviewStore.create({
        planId,
        taskId,
        round,
        phase,
        model: selectedModel,
      });

      // Send initial progress notification
      await context.sendProgress('Starting code review...', 0);

      const useStreaming = !!context.progressToken;
      // Pass cwd to spawn so the child process starts in the correct directory.
      // This works around openai/codex#9084 where -C is ignored by `review`.
      const cmdOptions = { cwd: resolvedWorkDir };
      const result = useStreaming
        ? await executeCommandStreaming('codex', cmdArgs, {
            ...cmdOptions,
            onProgress: (message) => {
              context.sendProgress(message);
            },
          })
        : await executeCommand('codex', cmdArgs, cmdOptions);

      // Codex CLI outputs to stderr, so check both stdout and stderr
      const response =
        result.stdout || result.stderr || 'No review output from Codex';

      this.reviewStore.markCompleted(reviewId, response);

      // Prepare metadata for dual approach:
      // - content[0]._meta: For Claude Code compatibility (avoids structuredContent bug)
      // - structuredContent: For other MCP clients that properly support it
      const metadata: Record<string, unknown> = {
        reviewId,
        model: selectedModel,
        ...(base && { base }),
        ...(commit && { commit }),
      };

      return {
        content: [
          {
            type: 'text',
            text: response,
            _meta: metadata,
          },
        ],
        structuredContent: isStructuredContentEnabled() ? metadata : undefined,
      };
    } catch (error) {
      if (reviewId) {
        const message = error instanceof Error ? error.message : String(error);
        this.reviewStore.markFailed(reviewId, message);
      }
      if (error instanceof ZodError) {
        throw new ValidationError(TOOLS.REVIEW, error.message);
      }
      if (error instanceof ValidationError) {
        throw error;
      }
      throw new ToolExecutionError(
        TOOLS.REVIEW,
        'Failed to execute code review',
        error
      );
    }
  }
}
```

- [ ] **Step 5: Add `ReviewStatusToolHandler` and `ReviewListToolHandler`**

Immediately after the `ReviewToolHandler` class's closing `}` (right before the `WebSearchToolHandler` class comment), insert:

```typescript
export class ReviewStatusToolHandler {
  constructor(private reviewStore: ReviewStore) {}

  async execute(
    args: unknown,
    _context: ToolHandlerContext = defaultContext
  ): Promise<ToolResult> {
    try {
      const { reviewId }: ReviewStatusToolArgs = ReviewStatusToolSchema.parse(args);
      const review = this.reviewStore.get(reviewId);

      if (!review) {
        return {
          content: [
            {
              type: 'text',
              text: `No review found with id ${reviewId} (expired or never existed).`,
            },
          ],
          isError: true,
        };
      }

      const text =
        review.status === 'running'
          ? `Review ${review.id}: running (started ${review.startedAt.toISOString()})`
          : review.status === 'completed'
            ? `Review ${review.id}: completed (${review.completedAt?.toISOString()})\n\n${review.output || 'No output'}`
            : `Review ${review.id}: failed (${review.completedAt?.toISOString()})\n\n${review.error}`;

      return {
        content: [
          {
            type: 'text',
            text,
            _meta: {
              reviewId: review.id,
              status: review.status,
              planId: review.planId,
              taskId: review.taskId,
              phase: review.phase,
            },
          },
        ],
      };
    } catch (error) {
      if (error instanceof ZodError) {
        throw new ValidationError(TOOLS.REVIEW_STATUS, error.message);
      }
      throw new ToolExecutionError(
        TOOLS.REVIEW_STATUS,
        'Failed to check review status',
        error
      );
    }
  }
}

export class ReviewListToolHandler {
  constructor(private reviewStore: ReviewStore) {}

  async execute(
    args: unknown,
    _context: ToolHandlerContext = defaultContext
  ): Promise<ToolResult> {
    try {
      const { planId, taskId }: ReviewListToolArgs = ReviewListToolSchema.parse(args);
      const reviews = this.reviewStore.list({ planId, taskId }).map((review) => ({
        id: review.id,
        status: review.status,
        planId: review.planId,
        taskId: review.taskId,
        round: review.round,
        phase: review.phase,
        model: review.model,
        startedAt: review.startedAt.toISOString(),
        completedAt: review.completedAt?.toISOString(),
      }));

      return {
        content: [
          {
            type: 'text',
            text: reviews.length > 0 ? JSON.stringify(reviews, null, 2) : 'No reviews',
          },
        ],
      };
    } catch (error) {
      if (error instanceof ZodError) {
        throw new ValidationError(TOOLS.REVIEW_LIST, error.message);
      }
      throw new ToolExecutionError(
        TOOLS.REVIEW_LIST,
        'Failed to list reviews',
        error
      );
    }
  }
}

```

- [ ] **Step 6: Wire the registry**

Replace:

```typescript
// Tool handler registry
const sessionStorage = new InMemorySessionStorage();
const jobStore = new InMemoryJobStore();

export const toolHandlers = {
  [TOOLS.CODEX]: new CodexToolHandler(sessionStorage),
  [TOOLS.CODEX_START]: new CodexStartToolHandler(jobStore),
  [TOOLS.CODEX_JOB_STATUS]: new JobStatusToolHandler(jobStore),
  [TOOLS.CODEX_JOB_LIST]: new JobListToolHandler(jobStore),
  [TOOLS.REVIEW]: new ReviewToolHandler(),
  [TOOLS.PING]: new PingToolHandler(),
  [TOOLS.HELP]: new HelpToolHandler(),
  [TOOLS.LIST_SESSIONS]: new ListSessionsToolHandler(sessionStorage),
  [TOOLS.WEBSEARCH]: new WebSearchToolHandler(),
} as const;
```

with:

```typescript
// Tool handler registry
const sessionStorage = new InMemorySessionStorage();
const jobStore = new InMemoryJobStore();
const reviewStore = new InMemoryReviewStore();

export const toolHandlers = {
  [TOOLS.CODEX]: new CodexToolHandler(sessionStorage),
  [TOOLS.CODEX_START]: new CodexStartToolHandler(jobStore),
  [TOOLS.CODEX_JOB_STATUS]: new JobStatusToolHandler(jobStore),
  [TOOLS.CODEX_JOB_LIST]: new JobListToolHandler(jobStore),
  [TOOLS.REVIEW]: new ReviewToolHandler(reviewStore),
  [TOOLS.REVIEW_STATUS]: new ReviewStatusToolHandler(reviewStore),
  [TOOLS.REVIEW_LIST]: new ReviewListToolHandler(reviewStore),
  [TOOLS.PING]: new PingToolHandler(),
  [TOOLS.HELP]: new HelpToolHandler(),
  [TOOLS.LIST_SESSIONS]: new ListSessionsToolHandler(sessionStorage),
  [TOOLS.WEBSEARCH]: new WebSearchToolHandler(),
} as const;
```

- [ ] **Step 7: Run the new test to verify it passes**

Run: `npx jest src/__tests__/review-tracking.test.ts`
Expected: PASS (4/4)

- [ ] **Step 8: Run the full existing suite to confirm no regression**

Run: `npm run build && npm test`
Expected: build succeeds; all pre-existing test files (`async-jobs.test.ts`, `context-building.test.ts`, `default-model.test.ts`, `edge-cases.test.ts`, `error-scenarios.test.ts`, `index.test.ts`, `mcp-stdio.test.ts`, `model-selection.test.ts`, `resume-functionality.test.ts`, `session.test.ts`, `windows-utf8-safety-note.test.ts`, `working-directory.test.ts`) plus the two new files still pass. `ReviewToolHandler` now requires a constructor argument everywhere it is instantiated — if any existing test constructs `new ReviewToolHandler()` directly, this step will surface it; fix by passing a fresh `new InMemoryReviewStore()` at that call site.

- [ ] **Step 9: Commit**

```bash
cd "C:\Users\erifkim\my_projects\codex-mcp-server"
git add src/tools/handlers.ts src/__tests__/review-tracking.test.ts
git commit -m "feat: track review calls in ReviewStore; add reviewStatus/reviewList handlers"
```

---

### Task 4: `codex-review-dispatch` prep script

**Files:**
- Create: `C:\Users\erifkim\my_projects\superpowers-codex-fork\skills\subagent-driven-development\scripts\codex-review-dispatch`

**Interfaces:**
- Consumes: `sdd-workspace` and `review-package` (both already in the same `scripts/` directory, unchanged) via subprocess call, matching how `task-brief`/`review-package` already call `sdd-workspace`.
- Produces: a single prompt file per invocation, path printed to stdout along with the `mcp__codex__review(...)` call the controller should make next — this script never calls the MCP tool itself (a bash script structurally cannot; only the calling agent turn can — same division of labor as `review-package`, which prepares a diff file for the controller to hand to a dispatch, not a dispatch itself). Task 6 consumes this script's output shape.

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
# Prepare a Codex review dispatch: generate the diff file (via review-package),
# extract this plan's Global Constraints, and compose the full reviewer
# prompt (phase template + diff path + constraints + brief/report paths) into
# one file. Prints that file's path plus the planId/taskId/round/phase values
# to pass into mcp__codex__review.
#
# This script never calls the MCP tool itself — a bash script cannot; only
# the calling agent turn can. Same division of labor as review-package: this
# script prepares, the controller dispatches.
#
# Usage:
#   codex-review-dispatch PLAN_FILE BASE HEAD final-review
#   codex-review-dispatch PLAN_FILE BASE HEAD task-review TASK_NUMBER
#   codex-review-dispatch PLAN_FILE BASE HEAD re-review TASK_NUMBER ROUND
set -euo pipefail

usage() {
  echo "usage: codex-review-dispatch PLAN_FILE BASE HEAD final-review" >&2
  echo "       codex-review-dispatch PLAN_FILE BASE HEAD task-review TASK_NUMBER" >&2
  echo "       codex-review-dispatch PLAN_FILE BASE HEAD re-review TASK_NUMBER ROUND" >&2
}

if [ $# -lt 4 ] || [ $# -gt 6 ]; then
  usage
  exit 2
fi

plan=$1
base=$2
head=$3
phase=$4
task_num=${5:-}
round=${6:-}

[ -f "$plan" ] || { echo "no such plan file: $plan" >&2; exit 2; }

case "$phase" in
  final-review)
    [ $# -eq 4 ] || { echo "final-review takes exactly 4 args" >&2; usage; exit 2; }
    ;;
  task-review)
    [ $# -eq 5 ] || { echo "task-review requires TASK_NUMBER (5 args)" >&2; usage; exit 2; }
    ;;
  re-review)
    [ $# -eq 6 ] || { echo "re-review requires TASK_NUMBER and ROUND (6 args)" >&2; usage; exit 2; }
    ;;
  *)
    echo "unknown phase: $phase (must be task-review, re-review, or final-review)" >&2
    exit 2
    ;;
esac

git rev-parse --verify --quiet "$base" >/dev/null || { echo "bad BASE: $base" >&2; exit 2; }
git rev-parse --verify --quiet "$head" >/dev/null || { echo "bad HEAD: $head" >&2; exit 2; }

script_dir="$(cd "$(dirname "$0")" && pwd)"
dir=$("$script_dir/sdd-workspace" "$plan")

diff_out="$dir/review-$(git rev-parse --short "$base")..$(git rev-parse --short "$head").diff"
"$script_dir/review-package" "$plan" "$base" "$head" "$diff_out" >/dev/null

case "$phase" in
  task-review) template="$script_dir/../task-reviewer-prompt.md" ;;
  re-review) template="$script_dir/../re-review-prompt.md" ;;
  final-review) template="$script_dir/../../requesting-code-review/code-reviewer.md" ;;
esac
[ -f "$template" ] || { echo "template not found: $template" >&2; exit 2; }

constraints=$(awk '
  /^##[ \t]+Global Constraints/ { grab=1; next }
  grab && /^##/ { exit }
  grab { print }
' "$plan")
[ -n "$constraints" ] || constraints="(none stated in plan)"

case "$phase" in
  task-review)
    prompt_out="$dir/codex-review-task-${task_num}-prompt.md"
    brief="$dir/task-${task_num}-brief.md"
    report="$dir/task-${task_num}-report.md"
    extra_files="Task brief (read first — exact requirements): ${brief}
Implementer report (read for what was tried/tested): ${report}"
    ;;
  re-review)
    prompt_out="$dir/codex-review-task-${task_num}-round-${round}-prompt.md"
    brief="$dir/task-${task_num}-brief.md"
    report="$dir/task-${task_num}-report.md"
    extra_files="Task brief: ${brief}
Implementer report (fix round appended): ${report}
This is fix round ${round} — verdict each prior finding ADDRESSED or NOT ADDRESSED; flag only NEW Critical/Important breakage introduced by the fix diff. Out-of-scope observations are deferred minors, not loop-extending findings."
    ;;
  final-review)
    prompt_out="$dir/codex-review-final-prompt.md"
    extra_files="This is the final whole-branch review — the plan's full task list is the scope."
    ;;
esac

{
  cat "$template"
  echo
  echo "---"
  echo "Diff file (read this — full diff with 10 lines context): ${diff_out}"
  echo
  echo "Global Constraints (binding for this review):"
  echo "$constraints"
  echo
  echo "$extra_files"
} > "$prompt_out"

plan_slug=$(basename "$plan" .md)
echo "wrote ${prompt_out}"
echo "dispatch: mcp__codex__review(base=\"${base}\", prompt=<contents of ${prompt_out}>, planId=\"${plan_slug}\", taskId=\"${task_num:-final}\", round=${round:-1}, phase=\"${phase}\")"
```

- [ ] **Step 2: Make it executable**

```bash
cd "C:\Users\erifkim\my_projects\superpowers-codex-fork"
chmod +x skills/subagent-driven-development/scripts/codex-review-dispatch
```

- [ ] **Step 3: Commit**

```bash
git add skills/subagent-driven-development/scripts/codex-review-dispatch
git commit -m "feat: add codex-review-dispatch prep script"
```

---

### Task 5: Verify `codex-review-dispatch` output shape against a real diff

**Files:**
- No files created or modified — this task runs the Task 4 script against the diff Task 4's own commit introduced, as the pilot the spec's Testing section calls for.

**Interfaces:**
- Consumes: `codex-review-dispatch` from Task 4, `docs/superpowers/plans/2026-08-24-codex-review-dispatch.md` (this file, already committed once Task-writing finished) as `PLAN_FILE`.
- Produces: nothing new — a verification step only.

- [ ] **Step 1: Run the script against Task 4's own commit as a real diff**

```bash
cd "C:\Users\erifkim\my_projects\superpowers-codex-fork"
prev=$(git rev-parse HEAD~1)
head=$(git rev-parse HEAD)
skills/subagent-driven-development/scripts/codex-review-dispatch \
  docs/superpowers/plans/2026-08-24-codex-review-dispatch.md "$prev" "$head" final-review
```

Expected output: two lines — `wrote .superpowers/sdd/2026-08-24-codex-review-dispatch/codex-review-final-prompt.md` and a `dispatch: mcp__codex__review(...)` line naming that same plan slug and `phase="final-review"`.

- [ ] **Step 2: Confirm the composed prompt file has the expected shape**

```bash
prompt_file=".superpowers/sdd/2026-08-24-codex-review-dispatch/codex-review-final-prompt.md"
grep -q "^# " "$prompt_file" && echo "PASS: template heading present"
grep -q "^Diff file (read this" "$prompt_file" && echo "PASS: diff pointer present"
grep -q "^Global Constraints (binding for this review):" "$prompt_file" && echo "PASS: constraints block present"
grep -q "No new npm dependencies" "$prompt_file" && echo "PASS: this plan's actual Global Constraints text was extracted, not a stale/empty block"
```

Expected: all four `PASS` lines print. The last check confirms the `awk` extraction in Task 4 actually pulled this plan's real `## Global Constraints` section (not the "(none stated in plan)" fallback) — a real end-to-end signal, not a placeholder assertion.

- [ ] **Step 3: Note the manual follow-up (not scriptable)**

This step has no further shell command. Record in the task's completion note (ledger, if executed under `subagent-driven-development`) that the actual `mcp__codex__review(...)` round-trip — confirming Codex's free-form output reads as a usable spec ✅/❌ + findings verdict — must be exercised once for real the first time Task 7's rewired SKILL.md runs a live task-review, since no bash script can invoke an MCP tool to verify this ahead of time. This is the spec's flagged "output-format drift" risk, closed by first real use rather than by a fabricated automated test.

---

### Task 6: Rewire `subagent-driven-development/SKILL.md`

**Files:**
- Modify: `C:\Users\erifkim\my_projects\superpowers-codex-fork\skills\subagent-driven-development\SKILL.md`

**Interfaces:**
- Consumes: `codex-review-dispatch` from Task 4 (script path), the `mcp__codex__review` tool (external, already available in this session as confirmed at the start of this design work).
- Produces: none for later tasks — this is the final behavioral change.

- [ ] **Step 1: Replace the task-review dispatch instructions**

In `SKILL.md`, under `### 2. Handle the report`, in the `**DONE:**` line, replace:

```markdown
**DONE:** Generate the review package (`scripts/review-package PLAN_FILE BASE HEAD`, from this skill's directory — it prints the unique file path it wrote; BASE is the commit you recorded before dispatching the implementer — never `HEAD~1`, which silently drops all but the last commit of a multi-commit task), then dispatch the task reviewer with the printed path.
```

with:

```markdown
**DONE:** Run `scripts/codex-review-dispatch PLAN_FILE BASE HEAD task-review N` (from this skill's directory — BASE is the commit you recorded before dispatching the implementer, never `HEAD~1`, which silently drops all but the last commit of a multi-commit task; N is this task's number). It prints a prompt file path and the `mcp__codex__review(...)` call to make — read the file with one Read call, then make that call with the file's contents as `prompt`. The returned text is the task reviewer's verdict (fork-local: reviews route to Codex, not a Claude Agent reviewer persona). **If the `mcp__codex__review` call errors, retry it once unchanged. If it errors again, fall back to dispatching a Claude Agent reviewer with `../requesting-code-review/code-reviewer.md` (the pre-fork behavior) for this review only, and ledger `Task <N>: reviewer fallback (codex failed twice) — used Claude Agent`.**
```

- [ ] **Step 2: Replace the fix-loop re-review dispatch instructions**

Under `### 4. The fix loop`, in the `**The re-review is scoped.**` paragraph, replace:

```markdown
**The re-review is scoped.** Run `scripts/review-package PLAN_FILE FIX_BASE HEAD`
where FIX_BASE is the head the previous review saw, and dispatch
[re-review-prompt.md](re-review-prompt.md) with the findings list, the
brief, the report file, and the printed diff path. The re-reviewer verdicts
each finding ADDRESSED or NOT ADDRESSED and flags new breakage in the fix
diff only. New Critical/Important breakage in the fix diff joins the open
findings list. Out-of-scope observations go to the ledger as deferred
minors — they never extend the loop.
```

with:

```markdown
**The re-review is scoped.** Run `scripts/codex-review-dispatch PLAN_FILE
FIX_BASE HEAD re-review N R` where FIX_BASE is the head the previous review
saw, N is the task number, and R is this fix round (1-5). It prints a
prompt file (already containing the re-review rubric, the diff, and the
round-scoping instructions) and the `mcp__codex__review(...)` call to make
— read the file, make the call, its text is the re-reviewer's verdict.
(Fork-local: routes to Codex, not a Claude Agent reviewer persona.) The
re-reviewer verdicts each finding ADDRESSED or NOT ADDRESSED and flags new
breakage in the fix diff only. New Critical/Important breakage in the fix
diff joins the open findings list. Out-of-scope observations go to the
ledger as deferred minors — they never extend the loop. If the
`mcp__codex__review` call errors, retry it once unchanged; if it errors
again, fall back to [re-review-prompt.md](re-review-prompt.md) dispatched to
a Claude Agent for this round only, and ledger `Task <N>: reviewer fallback
(codex failed twice, round <R>) — used Claude Agent`.
```

- [ ] **Step 3: Replace the final whole-branch review dispatch instructions**

Under `## Final Review`, replace:

```markdown
The final whole-branch review gets a package too: run
`scripts/review-package PLAN_FILE MERGE_BASE HEAD` (MERGE_BASE = the commit the
branch started from, e.g. `git merge-base main HEAD`) and include the
printed path in the final review dispatch, so the final reviewer reads
one file instead of re-deriving the branch diff with git commands. Dispatch
on the most capable available model (see Model Selection), using
superpowers:requesting-code-review's
[code-reviewer.md](../requesting-code-review/code-reviewer.md).
```

with:

```markdown
The final whole-branch review gets a package too: run
`scripts/codex-review-dispatch PLAN_FILE MERGE_BASE HEAD final-review`
(MERGE_BASE = the commit the branch started from, e.g.
`git merge-base main HEAD`). It reuses
[code-reviewer.md](../requesting-code-review/code-reviewer.md) as the
rubric, composed with the branch diff, into a prompt file — read the file,
then make the printed `mcp__codex__review(...)` call. (Fork-local: routes
to Codex, not a Claude Agent reviewer persona. See Model Selection below —
only one Codex review model works under this account's auth, so there is
no "most capable" tier to pick between for this step.) If the
`mcp__codex__review` call errors, retry it once unchanged; if it errors
again, fall back to superpowers:requesting-code-review's
[code-reviewer.md](../requesting-code-review/code-reviewer.md) dispatched to
a Claude Agent for this final review only, and ledger `Final review:
reviewer fallback (codex failed twice) — used Claude Agent`.
```

- [ ] **Step 4: Add the model note to `## Model Selection`**

At the end of the `## Model Selection` section (after the "Always specify the model explicitly when dispatching a subagent." paragraph, before `## The Task Loop`), insert:

```markdown
**Fork-local: review model.** All three review touchpoints (task-review,
re-review, final-review) call `mcp__codex__review`, which defaults to
`gpt-5.6-sol`. `codex-mcp-server/src/types.ts` records that the `*-codex`
model family returns HTTP 400 under this account's ChatGPT-subscription
auth — do not pass `model` overrides referencing that family. With only
`gpt-5.6-sol`/`gpt-5.6-terra` confirmed working and no documented
cost/capability delta between them, there is no per-phase review tiering
here (unlike the implementer-role tiering above, which still applies —
only the reviewer role moved to Codex).
```

- [ ] **Step 5: Commit**

```bash
cd "C:\Users\erifkim\my_projects\superpowers-codex-fork"
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat: route task-review, re-review, and final-review through codex-review-dispatch"
```

---

### Task 7: Fork repo shell-lint + final check

**Files:**
- No files modified — verification only.

**Interfaces:**
- Consumes: everything from Tasks 4 and 6.
- Produces: nothing — this is the plan's closing gate.

- [ ] **Step 1: Lint the new shell script**

```bash
cd "C:\Users\erifkim\my_projects\superpowers-codex-fork"
bash scripts/lint-shell.sh --all
```

Expected: `codex-review-dispatch` (extensionless, `#!/usr/bin/env bash` shebang, same shape as `sdd-workspace`/`task-brief`) is picked up and passes ShellCheck at warning severity. Fix any reported warnings in the script from Task 4 and re-run this step until clean.

- [ ] **Step 2: Run the fork's own test suite**

```bash
bash tests/shell-lint/test-lint-shell.sh
```

Expected: `All shell lint script tests passed` (this suite tests the linter itself, not our new script — confirms Task 4's script didn't somehow break the linter's own fixture-based tests).

- [ ] **Step 3: Confirm the working tree is clean**

```bash
git status --short
```

Expected: empty output — every task's commit (Tasks 4, 6) already captured its changes; Task 5 produced no new tracked files (its output lives under the git-ignored `.superpowers/sdd/` workspace, per `sdd-workspace`'s own `.gitignore` write).
