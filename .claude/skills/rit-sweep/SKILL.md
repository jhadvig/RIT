---
name: rit-sweep
description: RIT health check — flag triaged bugs stuck in New with no activity, and detect PR-closed resets invisible to normal triage
argument-hint: "[--stale-days N] [--unassign] — days without activity to flag as stale (default: 14); add --unassign to also remove assignments beyond the threshold"
allowed-tools: Read, mcp__plugin_jira_atlassian__searchJiraIssuesUsingJql, mcp__plugin_jira_atlassian__getJiraIssue, mcp__plugin_jira_atlassian__addCommentToJiraIssue, mcp__plugin_jira_atlassian__editJiraIssue, mcp__plugin_jira_atlassian__getJiraIssueRemoteIssueLinks
---

# RIT Sweep

Health check skill. Finds triaged bugs stuck in `New` status that are invisible to both
`/rit-triage` (already triaged) and `/rit-end` (not in the current tracker). Handles
two cases:

1. **PR-closed reset** — bug was reset to `New` by Prow Bot after a PR was closed.
2. **Stale triaged** — bug has been triaged and assigned but has had no visible activity
   for an extended period.

By default this skill is **comment only**. With `--unassign`, stale bugs whose assignee is
in the RIT roster are also unassigned after confirmation.

## Usage

```
/rit-sweep
/rit-sweep --stale-days 30
/rit-sweep --stale-days 30 --unassign
```

Run periodically during the rotation week (e.g. mid-week or on-demand). Use `--unassign`
for a deeper cross-week cleanup when bugs have been stale long enough that reassignment
makes sense.

## Instructions

### Step 1: Load configuration

Read `rit_manual.md` from the **current working directory** and extract:

1. **Shared Constants** — Read the "Shared Constants" section to load the canonical PIXAA
   base filter and Prow Bot reset string. Use these in Step 2 and Step 3 respectively.

2. **Comment templates** — Find the "Comment templates for status transitions" section.
   Load the templates for:
   - "PR-closed reset (Prow Bot)"
   - "Sweep — stale triaged bug"
   - "EOW cleanup — unassignment" *(only needed if `--unassign` is set)*

3. **Stale threshold** — Use the `--stale-days` argument if provided, otherwise default
   to **14 days**. A bug is considered stale if its `updated` field is older than this
   threshold relative to today.

4. **Unassign mode** — If `--unassign` is set, find the most recent
   `triaged_bugs_YYYY-MM-DD.md` file in the **current working directory** and read the
   "Engineering" table to get the list of active RIT engineers (name + Jira Account ID).
   If no tracker file exists, **stop and tell the user to run `/rit-start` first**. Only
   bugs whose current assignee is in this list are eligible for unassignment — external
   engineers are never unassigned, even in `--unassign` mode.

### Step 2: Fetch bugs

Run a single JQL query for triaged, assigned bugs currently stuck in `New`:

```
filter in ("operator-framework-all-bugs", "PIXAA HIVE bugs", "PIXAA CCO bugs", "PIXAA OCPCLOUD bugs", "PIXAA Console bugs", "PIXAA MCP Server Bugs", "PIXAA Serverless Bugs", "All OTA Bugs")
AND status = New
AND labels in (triaged)
AND assignee is not EMPTY
AND (labels not in (ocp-sustaining) or labels is EMPTY)
AND statusCategory != done
ORDER BY updated asc, key
```

Request fields: `summary, status, assignee, priority, labels, components, created, updated`.
Paginate until all bugs are fetched.

### Step 3: Classify each bug

For each bug, run the following checks **in order** — first match wins:

**Class 1 — Stale threshold check (in-memory, no API call):**
`updated` field is older than the stale threshold.

- **Not stale → `active`**: skip entirely. No further checks needed.
- **Stale → continue to Class 2.**

**Class 2 — PR-closed reset:**
Fetch the full issue with comments (`getJiraIssue` with `comment` field). Check whether any
comment from **OpenShift Prow Bot** contains `"Bug status changed to NEW as previous linked PR"`.

- **Found → `pr_closed_reset`**: flag for a comment. Never unassign, even in `--unassign` mode.

**Class 3 — Stale:**
No Prow Bot comment found.

- **→ `stale`**: flag for a stale comment.
- In `--unassign` mode: check if the current assignee is in the RIT roster.
  - If yes → check for active linked PR (`getJiraIssueRemoteIssueLinks`). If a PR is found → `stale_comment_only` (work in progress, do not unassign).
  - If yes and no linked PR → `stale_unassign`: propose unassignment.
  - If no → `stale_comment_only`: post the stale comment, keep assignee.

### Step 4: Present summary to user

Show:
- Total bugs found
- `pr_closed_reset`: N (comment only)
- `stale` in comment-only mode: N — grouped by assignee with days-since-update
- In `--unassign` mode:
  - `stale_unassign` (RIT roster — comment + propose unassignment): N — grouped by assignee
  - `stale_comment_only` (external — comment only): N
- `active`: N (skipped)

Ask: "Proceed? (yes/no)"

If the user says no, stop.

### Step 5: Post comments and unassign

#### 5a. PR-closed reset bugs

For each `pr_closed_reset` bug — automatically, no user confirmation needed:
- Post the "PR-closed reset (Prow Bot)" comment.

#### 5b. Stale bugs (comment only)

For each `stale` or `stale_comment_only` bug — automatically, no user confirmation needed:
- Post the "Sweep — stale triaged bug" comment, substituting `{N}` with the actual number
  of days since the last update.

#### 5c. Stale bugs with unassignment (`--unassign` mode only)

Group `stale_unassign` bugs by assignee and propose each engineer's batch:

> "Propose removing assignee from N bugs stale for >X days assigned to [engineer]:
> OCPBUGS-XXXXX, ... Confirm? (yes/no/skip)"

- **Wait for user confirmation** per engineer batch.
- If confirmed:
  1. Remove the assignee (`assignee: null`) via `editJiraIssue`.
  2. Post the "EOW cleanup — unassignment" comment (explains the removal clearly).
- If skipped: post the "Sweep — stale triaged bug" comment only (no unassignment).

### Step 6: Final summary

Show:
- PR-closed reset comments posted: N
- Stale comments posted (comment only): N — list by assignee
- In `--unassign` mode:
  - Unassigned + commented: N — list by engineer
  - Comment only (external assignee): N
- Active (skipped): N

---

## Important Notes

- **PR-closed reset bugs are never unassigned** — even in `--unassign` mode; the developer
  is still the right owner.
- **External assignees are never unassigned** — in `--unassign` mode, only engineers in the
  current RIT roster are eligible; external specialists keep their bugs.
- **`--unassign` requires confirmation per engineer batch** — never silently removes an assignee.
- **`active` bugs are completely ignored** — recently updated bugs are healthy; no comment.
- **Stale threshold is configurable** — default 14 days for comment mode; recommend 30+ days
  before using `--unassign` to avoid cleaning up bugs that are simply slow-moving.
- **Paths use the current working directory** — never hardcode paths.
- **All comments must be Red Hat Employee only** — every `addCommentToJiraIssue` call must include `commentVisibility: {"type": "group", "value": "Red Hat Employee"}`. These are internal process notes and must not be publicly visible.
