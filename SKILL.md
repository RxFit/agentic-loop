---
name: agentic-loop
description: >
  Autonomous PR workflow orchestrator. Spawns coder and reviewer subagents
  to execute a queue of implementation plans sequentially. Each plan goes
  through: code → PR → review → fix loop (max 5 iterations) → merge → next plan.
  Triggered by requests to autonomously execute implementation plans, loop through
  coding tasks, or run an agentic PR workflow.
---

# Agentic Loop — Autonomous PR Workflow Orchestrator

> **You are now the orchestrator.** You manage a queue of implementation plans, executing each one by spawning specialized subagents, managing the PR review loop, and advancing to the next plan upon merge.

---

## 1. Activation

This skill activates when the user:
- Provides a list of implementation plans to execute autonomously
- Asks you to "loop through" a set of coding tasks
- References "agentic loop", "autonomous PR workflow", or "code-review-merge cycle"
- Uses a slash command like `/agentic-loop` or `/loop-plans`

**On activation**, confirm with the user:
> 🔄 **Agentic Loop activated.** I'll orchestrate [N] plans sequentially: code → PR → review loop → merge → next plan. Each coder gets an isolated workspace. Max 5 review iterations per plan before escalation. Ready to start?

---

## 2. State Initialization

When the user provides plans and confirms, create `plan_queue.json` in your artifact directory:

```
<artifact_dir>/plan_queue.json
```

Use the schema defined in `references/state_schema.md`. Initialize all plans with `status: "pending"`. Set `current_plan_index: 0`.

### Plan Ingestion

Plans can be provided as:
1. **Inline markdown** — The user pastes plan content directly
2. **File references** — The user points to existing `.md` files (read them via `view_file`)
3. **Generated plans** — You generated implementation plans earlier in the conversation

For each plan, generate a `slug` (lowercase, hyphens, max 40 chars) from the plan title.

---

## 3. Main Orchestration Loop

Execute the following protocol **sequentially** for each plan in the queue:

### Phase 1: Spawn Coder

1. Update `plan_queue.json`: Set current plan's `status` → `"coding"`, `started_at` → now, `updated_at` → now
2. Read the coder system prompt template from `resources/coder_prompt.md`
3. Construct the coder's prompt by injecting:
   - The plan content (full markdown)
   - The target repo (`repo` field from state)
   - The base branch (`base_branch` field from state)
   - The branch name: `agentic-loop/<plan-id>-<plan-slug>`
4. Spawn the coder subagent:

```
invoke_subagent:
  TypeName: "self"
  Role: "Coder for <plan-title>"
  Workspace: "branch"
  Prompt: <constructed prompt with plan + coder instructions>
```

5. Store the `coder_conversation_id` in the plan's state
6. Set a 60-second safety timer:

```
schedule:
  DurationSeconds: 60
  Prompt: "Check coder status for plan: <plan-title>"
```

7. **Wait** for the coder to send a message. Parse the message using this priority:

   **Primary (structured):** Look for a JSON code block containing status:
   ```json
   {"status": "pr_created", "pr_url": "...", "branch": "...", "files_changed": N}
   ```
   or
   ```json
   {"status": "failed", "reason": "..."}
   ```

   **Fallback (fuzzy):** If no JSON block found, use pattern matching:
   - Look for a GitHub PR URL pattern (`github.com/.*/pull/\d+`) → extract as PR URL → treat as success
   - Look for failure keywords (`failed`, `error`, `could not`, `unable to`) → treat as failure
   - If neither pattern matches after 2 message exchanges, ask the subagent explicitly: "Please confirm: did you successfully create a PR? Reply with the PR URL or explain what failed."

### Phase 2: Spawn Reviewer

1. Update `plan_queue.json`: Set `status` → `"pr_open"`, store `pr_url` and `pr_number`, `updated_at` → now
2. Read the reviewer system prompt template from `resources/reviewer_prompt.md`
3. Construct the reviewer's prompt by injecting:
   - The PR URL
   - The PR number
   - The repo owner/name
   - The plan title (for context)
4. Spawn the reviewer subagent:

```
invoke_subagent:
  TypeName: "self"
  Role: "Reviewer for <plan-title>"
  Workspace: "inherit"
  Prompt: <constructed prompt with PR details + reviewer instructions>
```

5. Store the `reviewer_conversation_id` in the plan's state
6. Update `plan_queue.json`: Set `status` → `"reviewing"`, `updated_at` → now
7. Set a 60-second safety timer
8. **Wait** for the reviewer to send a message with the verdict

### Phase 3: Review Loop (Max 5 Iterations)

When the reviewer sends a message, parse the verdict using this priority:

**Primary (structured):** Look for a JSON code block:
```json
{"verdict": "approved", "summary": "..."}
```
or
```json
{"verdict": "changes_needed", "comments": ["...", "..."]}
```

**Fallback (fuzzy):**
- Look for approval keywords: `approved`, `approve`, `LGTM`, `looks good` → treat as APPROVED
- Look for rejection keywords: `changes requested`, `request changes`, `needs fixes`, `issues found` → treat as REQUEST_CHANGES
- If ambiguous, ask the reviewer: "Please confirm your verdict: APPROVE or REQUEST_CHANGES?"

**If verdict is APPROVED:**
1. Update `plan_queue.json`: Set `status` → `"approved"`, `updated_at` → now
2. Merge the PR:

```
call_mcp_tool:
  ServerName: "github-mcp-server"
  ToolName: "merge_pull_request"
  Arguments: { owner: "<owner>", repo: "<repo>", pullNumber: <pr_number> }
```

3. **Verify the merge succeeded** by reading the PR status:

```
call_mcp_tool:
  ServerName: "github-mcp-server"
  ToolName: "pull_request_read"
  Arguments: { owner: "<owner>", repo: "<repo>", pullNumber: <pr_number> }
```

   - If PR status shows `merged: true` → proceed to step 4
   - If PR status shows `merged: false` → retry merge once. If still false, mark as `"escalated"` with reason `"merge_verification_failed"`, `updated_at` → now

4. Update `plan_queue.json`: Set `status` → `"merged"`, `completed_at` → now, `updated_at` → now, increment `completed_count`
5. **Advance** to the next plan (back to Phase 1)

**If verdict is REQUEST_CHANGES:**
1. Increment `review_iterations` in `plan_queue.json`, `updated_at` → now
2. Check iteration count:
   - **If `review_iterations` < 5:**
     - Update `status` → `"review_iteration_<N>"`, `updated_at` → now
     - Extract review comments from the reviewer's message
     - Store comments in `review_comments` array
     - Send the comments to the coder subagent:

     ```
     send_message:
       Recipient: <coder_conversation_id>
       Message: "Review feedback (iteration <N>/5). Please address these comments and push fixes:\n<comments>"
     ```

     - Set a 60-second safety timer
     - Wait for the coder to confirm fixes are pushed
     - Once pushed, send the reviewer a message to re-review:

     ```
     send_message:
       Recipient: <reviewer_conversation_id>
       Message: "The coder has pushed fixes for iteration <N>. Please re-review the PR and file a new GitHub review."
     ```

     - Wait for the reviewer's new verdict
     - **Loop back** to the top of Phase 3

   - **If `review_iterations` >= 5:**
     - Update `status` → `"escalated"`, `updated_at` → now
     - Set `escalation_reason` to a summary of unresolved comments
     - Increment `escalated_count`
     - Generate an escalation artifact:

     ```markdown
     # ⚠️ Escalation: <plan-title>

     **Plan:** <plan-title>
     **PR:** <pr_url>
     **Iterations exhausted:** 5/5

     ## Unresolved Review Comments
     <all review comments from all iterations>

     ## Recommended Action
     A human reviewer should examine the PR and either:
     1. Approve and merge with known issues
     2. Provide specific guidance for the coder to follow
     3. Close the PR and re-plan
     ```

     - **Notify the user** that this plan has been escalated
     - **Advance** to the next plan (if any)

### Phase 4: Completion

When all plans have been processed (merged, escalated, or failed):

1. Generate a summary artifact:

```markdown
# 🔄 Agentic Loop — Execution Summary

| Plan | Status | PR | Iterations | Duration |
|------|--------|----|------------|----------|
| <plan-1> | ✅ Merged | #<pr> | 2 | 12m |
| <plan-2> | ⚠️ Escalated | #<pr> | 5 | 34m |
| <plan-3> | ✅ Merged | #<pr> | 1 | 8m |

## Statistics
- Total plans: N
- Merged: X
- Escalated: Y
- Failed: Z
- Total duration: Xm
```

2. Write a vault memory log to `C:/AntigravityHQ/chats/<project-slug>/Agentic_Loop_Execution.md`

---

## 4. Safety Rules

### Hard Stops
- **Never exceed 5 review iterations** per plan. Always escalate.
- **Never run plans in parallel.** Always sequential — Plan N must be merged/escalated/failed before Plan N+1 starts.
- **Never merge without an APPROVE review.** The reviewer must explicitly approve via GitHub review.
- **Never auto-resolve escalations.** Always notify the human.

### Timer Discipline
- Always set a 60-second safety timer via `schedule` after spawning or messaging a subagent
- If the timer fires and no message has been received, check the subagent's status via `manage_subagents` → `list`
- If the subagent appears stuck (no response after 3 timer cycles = 3 minutes), kill it and mark the plan as `"failed"` with reason `"subagent_timeout"`

### State Consistency
- **Always update `plan_queue.json` BEFORE taking the next action.** This ensures recoverability if the orchestrator session is interrupted.
- Read `plan_queue.json` at the start of every timer wakeup to re-establish context (in case earlier context was truncated from the window).

---

## 5. Error Recovery

### Coder Subagent Fails
- Mark plan as `"failed"` with the coder's error message
- Kill the coder subagent
- Advance to next plan

### Reviewer Subagent Fails
- Kill the reviewer subagent
- Spawn a NEW reviewer subagent for the same PR
- If the second reviewer also fails, mark plan as `"escalated"` with reason `"reviewer_failure"`

### Merge Conflict
- If `merge_pull_request` fails with a conflict error, send the coder a message to rebase
- If rebase fails, mark plan as `"escalated"` with reason `"merge_conflict"`

### Orchestrator Session Interrupted
- On re-activation, read `plan_queue.json` to determine current state
- Resume from the last incomplete plan based on its `status` field
- If a subagent conversation ID is stored but the subagent is no longer active, restart that phase

---

## 6. Context Retrieval Strategy

### For Coder Agents
The coder system prompt (see `resources/coder_prompt.md`) instructs the coder to:
1. **First**: Read the project's `_INDEX.md` via `view_file` if available
2. **Then**: Use `find_related` (Smart Connections) to find files related to the plan's scope
3. **Then**: Use `semantic_search_brain` for broader codebase understanding
4. **Finally**: Use `grep_search` for exact symbol/pattern lookups

### For Reviewer Agents
The reviewer system prompt (see `resources/reviewer_prompt.md`) instructs the reviewer to:
1. Read the full PR diff via `pull_request_read`
2. Optionally use `view_file` to see the full context of changed files
3. Check for test coverage by searching for related test files
