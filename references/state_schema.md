# State Schema Reference — `plan_queue.json`

> This document defines the schema for the `plan_queue.json` artifact file that the orchestrator uses to track the agentic loop's state.

---

## File Location

```
<artifact_dir>/plan_queue.json
```

Where `<artifact_dir>` is the orchestrator's brain directory:
```
~/.gemini/antigravity/brain/<orchestrator-conversation-id>/plan_queue.json
```

---

## Full Schema

```json
{
  "version": 1,
  "repo": "owner/repo-name",
  "base_branch": "main",
  "created_at": "2026-06-24T08:00:00-05:00",
  "updated_at": "2026-06-24T08:15:00-05:00",
  "config": {
    "max_review_iterations": 5,
    "poll_interval_seconds": 60,
    "timeout_cycles": 3,
    "sequential": true,
    "max_plan_chars": 50000,
    "warn_plan_chars": 20000
  },
  "plans": [
    {
      "id": 1,
      "slug": "auth-flow-refactor",
      "title": "Auth Flow Refactor",
      "plan_source": "inline|file_path",
      "plan_content": "Full markdown plan text",
      "plan_file": null,
      "status": "pending",
      "branch": null,
      "pr_url": null,
      "pr_number": null,
      "review_iterations": 0,
      "coder_conversation_id": null,
      "reviewer_conversation_id": null,
      "started_at": null,
      "completed_at": null,
      "escalation_reason": null,
      "failure_reason": null,
      "review_comments": [],
      "timeout_minutes": null,
      "char_count": 1500
    }
  ],
  "current_plan_index": 0,
  "completed_count": 0,
  "escalated_count": 0,
  "failed_count": 0
}
```

---

## Field Reference

### Root Fields

| Field | Type | Description |
|-------|------|-------------|
| `version` | `number` | Schema version. Always `1`. |
| `repo` | `string` | GitHub repo in `owner/name` format. |
| `base_branch` | `string` | Branch to merge PRs into (usually `main`). |
| `created_at` | `string` | ISO-8601 timestamp of queue creation. |
| `updated_at` | `string` | ISO-8601 timestamp of last state update. |
| `config` | `object` | Orchestration configuration. |
| `plans` | `array` | Ordered array of plan objects. |
| `current_plan_index` | `number` | Index of the plan currently being executed (0-based). |
| `completed_count` | `number` | Number of plans successfully merged. |
| `escalated_count` | `number` | Number of plans that hit the review cap. |
| `failed_count` | `number` | Number of plans that failed during coding. |

### Config Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_review_iterations` | `number` | `5` | Max review loop iterations before escalation. |
| `poll_interval_seconds` | `number` | `60` | Safety timer interval in seconds. |
| `timeout_cycles` | `number` | `3` | Number of timer cycles without response before a subagent is considered stuck. |
| `sequential` | `boolean` | `true` | If true, execute plans one at a time. |
| `max_plan_chars` | `number` | `50000` | Maximum allowed character count for a single plan. |
| `warn_plan_chars` | `number` | `20000` | Character count threshold for size warnings. |

### Plan Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `number` | 1-based plan index. |
| `slug` | `string` | URL-safe slug derived from title (e.g., `auth-flow-refactor`). Max 40 chars. |
| `title` | `string` | Human-readable plan title. |
| `plan_source` | `string` | How the plan was provided: `"inline"` or `"file_path"`. |
| `plan_content` | `string` | Full markdown content of the plan. Present when `plan_source` is `"inline"`. |
| `plan_file` | `string\|null` | Absolute path to the plan file. Present when `plan_source` is `"file_path"`. |
| `status` | `string` | Current plan status (see Status Enum below). |
| `branch` | `string\|null` | Git branch name (e.g., `agentic-loop/auth-flow-refactor`). |
| `pr_url` | `string\|null` | Full GitHub PR URL. |
| `pr_number` | `number\|null` | GitHub PR number. |
| `review_iterations` | `number` | Count of completed review iterations (0-5). |
| `coder_conversation_id` | `string\|null` | Antigravity conversation ID of the coder subagent. |
| `reviewer_conversation_id` | `string\|null` | Antigravity conversation ID of the reviewer subagent. |
| `started_at` | `string\|null` | ISO-8601 timestamp when coding started. |
| `completed_at` | `string\|null` | ISO-8601 timestamp when plan was merged/escalated/failed. |
| `escalation_reason` | `string\|null` | Why the plan was escalated (if status is `"escalated"`). |
| `failure_reason` | `string\|null` | Why the plan failed (if status is `"failed"`). |
| `review_comments` | `array` | Array of review comment objects from all iterations. |
| `timeout_minutes` | `number\|null` | Optional per-plan timeout override. Default: uses `config.timeout_cycles * config.poll_interval_seconds / 60`. |
| `char_count` | `number` | Character count of plan content. Set at ingestion. |

---

## Status Enum

```
pending            → Plan is in the queue, not yet started
coding             → Coder subagent is actively implementing
pr_open            → PR has been created, waiting for reviewer
reviewing          → Reviewer subagent is actively reviewing
review_iteration_1 → First round of review feedback being addressed
review_iteration_2 → Second round
review_iteration_3 → Third round
review_iteration_4 → Fourth round
review_iteration_5 → Fifth round (will escalate after this)
approved           → PR approved, ready to merge
merged             → PR successfully merged
escalated          → Exceeded max review iterations, needs human
failed             → Coder subagent failed to implement
```

### State Transitions

```
pending → coding → pr_open → reviewing → approved → merged
                                      ↘ review_iteration_N → reviewing (loop)
                                                           ↘ escalated (N >= max)
                   ↘ failed (coder error)
```

---

## Review Comment Object

```json
{
  "iteration": 1,
  "timestamp": "2026-06-24T08:10:00-05:00",
  "verdict": "request_changes",
  "comments": [
    {
      "path": "src/auth/login.ts",
      "line": 42,
      "body": "Missing null check on user object before accessing properties"
    }
  ]
}
```

---

## Example: Initialized Queue

```json
{
  "version": 1,
  "repo": "danielraffel/my-project",
  "base_branch": "main",
  "created_at": "2026-06-24T08:34:00-05:00",
  "updated_at": "2026-06-24T08:34:00-05:00",
  "config": {
    "max_review_iterations": 5,
    "poll_interval_seconds": 60,
    "timeout_cycles": 3,
    "sequential": true,
    "max_plan_chars": 50000,
    "warn_plan_chars": 20000
  },
  "plans": [
    {
      "id": 1,
      "slug": "auth-flow-refactor",
      "title": "Auth Flow Refactor",
      "plan_source": "inline",
      "plan_content": "## Auth Flow Refactor\n\nRefactor the authentication...",
      "plan_file": null,
      "status": "pending",
      "branch": null,
      "pr_url": null,
      "pr_number": null,
      "review_iterations": 0,
      "coder_conversation_id": null,
      "reviewer_conversation_id": null,
      "started_at": null,
      "completed_at": null,
      "escalation_reason": null,
      "failure_reason": null,
      "review_comments": [],
      "timeout_minutes": null,
      "char_count": 1500
    },
    {
      "id": 2,
      "slug": "add-rate-limiting",
      "title": "Add Rate Limiting",
      "plan_source": "inline",
      "plan_content": "## Rate Limiting\n\nAdd rate limiting to...",
      "plan_file": null,
      "status": "pending",
      "branch": null,
      "pr_url": null,
      "pr_number": null,
      "review_iterations": 0,
      "coder_conversation_id": null,
      "reviewer_conversation_id": null,
      "started_at": null,
      "completed_at": null,
      "escalation_reason": null,
      "failure_reason": null,
      "review_comments": [],
      "timeout_minutes": null,
      "char_count": 2200
    }
  ],
  "current_plan_index": 0,
  "completed_count": 0,
  "escalated_count": 0,
  "failed_count": 0
}
```
