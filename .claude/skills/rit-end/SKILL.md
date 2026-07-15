---
name: rit-end
description: End-of-week RIT cleanup — remove stale assignments from bugs assigned by /rit-triage so the next rotation starts with a clean slate
argument-hint: "(no arguments required)"
allowed-tools: Read, Edit, Write, mcp__plugin_jira_atlassian__searchJiraIssuesUsingJql, mcp__plugin_jira_atlassian__getJiraIssue, mcp__plugin_jira_atlassian__editJiraIssue, mcp__plugin_jira_atlassian__addCommentToJiraIssue, mcp__plugin_jira_atlassian__getJiraIssueRemoteIssueLinks
---

# RIT End

End-of-week cleanup skill. Reads the current rotation tracker file to identify exactly which
bugs were assigned by `/rit-triage` this week, then removes stale assignments for bugs not
picked up. Only operates on bugs in the tracker whose current assignee is in the RIT roster.

## Usage

```
/rit-end
```

Run at the end of the RIT rotation week, before the next rotation begins.

## Instructions

### Step 1: Load configuration

Read `rit_manual.md` from the **current working directory** and extract:

1. **RIT team roster** — Find the most recent `triaged_bugs_YYYY-MM-DD.md` file in the
   **current working directory** (sort by date in filename, pick the latest). Extract the
   week-start date from the heading. If that date is more than 7 days in the past, **stop
   and warn the user**: "⚠️ The tracker file appears stale (week of YYYY-MM-DD). Running
   cleanup against a stale roster will target the wrong engineers. Please run `/rit-start`
   to set up the current rotation before running cleanup." Do not proceed until the user
   confirms. Then parse the "Engineering" table in the tracker to get the list of active
   engineers (name + Jira Account ID). This is used to decide which assignees are eligible
   for cleanup — **only bugs currently assigned to someone in this roster will be considered
   for unassignment**. Skip engineers marked PTO (they are still in the roster for
   eligibility purposes).

2. **Comment templates** — Find the "Comment templates for status transitions" section.
   Load the template for:
   - "EOW cleanup — unassignment"

3. **Tracker file** — The tracker file found in step 1 is also the source of truth for
   which bugs were assigned by `/rit-triage`. Read:
   - The "Assignment Distribution" table: engineer → assigned keys
   - The "Triaged This Week" table: all bugs with their assigned engineer

   Collect the full list of bug keys assigned by RIT this week.

4. **Shared Constants** — Read the "Shared Constants" section of `rit_manual.md` to load the canonical PIXAA base filter and Prow Bot reset string. Use these when building JQL queries and checking for Prow Bot comments.

### Step 2: Fetch current state of tracker bugs

Build a JQL query from the tracker keys to get their current status in one call:

```
key in (OCPBUGS-XXXXX, OCPBUGS-YYYYY, ...) AND statusCategory != done
```

Request fields: `summary, status, assignee, priority, labels, components`.
Paginate if needed. Discard any bug that is now Done/Closed (it was resolved — skip it).

### Step 3: Classify each tracker bug

For each bug still open, classify **in order** — first match wins:

**Class 0 — External assignee:**
- The current Jira assignee is **not** in the RIT team roster → **skip entirely**. This
  bug was intentionally assigned to a specialist or external engineer; `/rit-sweep`
  will flag it if it goes stale. No comment, no unassignment.

**Class 1 — Already progressed:**
- Status is ASSIGNED, POST, Modified, ON_QA, or In Progress → the engineer picked it up
  and is actively working. **Do not touch.**
- Compare the **current Jira assignee** against the tracker-recorded assignee for this bug.
  - If they match → record as "in progress — skip".
  - If they differ → record as "transferred out": the bug was originally assigned by RIT
    but someone outside the team took it over. Still do not touch, but note it in the
    final summary as `transferred_out` with the new assignee's name.

**Class 2 — PR-closed reset:**
- Status is `New` AND assignee is still set → fetch full issue with comments (`getJiraIssue`
  with `comment` field). Check for an **OpenShift Prow Bot** comment containing
  `"Bug status changed to NEW as previous linked PR"`.
- If found → `pr_closed_reset`: **skip** — do not unassign, do not post any comment.
  `/rit-sweep` handles the notification. Record as skipped.

**Class 3 — Active PR linked:**
- Status is `New` AND no Prow Bot comment → fetch remote links
  (`getJiraIssueRemoteIssueLinks`). Check for any link to a GitHub Pull Request.
- If found → `pr_linked_active`: the engineer has an open PR but hasn't moved the bug.
  Do not unassign. Record as skipped.

**Class 4 — Stale assignment:**
- Status is `New`, no Prow Bot comment, no linked PR → not picked up.
  Candidate for unassignment.

**Class 5 — Unassigned already:**
- Assignee field is now empty → already cleaned up externally. Record as "already
  unassigned — skip".

### Step 4: Present summary to user

Show:
- Total tracker bugs reviewed
- Skipped (external assignee — not in RIT roster): N
- Already progressed (in progress — skip): N
- Transferred out (progressed, different assignee — informational): N
- `pr_closed_reset` (skipped — `/rit-sweep` handles): N
- `pr_linked_active` (skip): N
- Already unassigned: N
- `stale_assignment` (will be unassigned): N — grouped by engineer

Ask: "Proceed with EOW cleanup? (yes/no)"

If the user says no, stop.

### Step 5: Process stale assignments

Group `stale_assignment` bugs by current assignee. For each engineer, propose their full
batch together:

> "Propose removing assignee from N bugs assigned to [engineer]: OCPBUGS-XXXXX, ...
> Confirm? (yes/no/skip)"

- **Wait for user confirmation** — the user may skip an individual engineer's batch.
- If confirmed: for each bug in the batch:
  1. Remove the assignee (`assignee: null`)
  2. Post the "EOW cleanup — unassignment" comment
- Record each in the tracker under "Unassigned This Week (EOW Cleanup)".

### Step 6: Bulk tracker update

After all bugs are processed, perform a single bulk write to the tracker:

1. Update "Assignment Distribution": decrement counts and remove keys for unassigned bugs.
2. Append "Unassigned This Week (EOW Cleanup)" table.

### Step 7: Final summary

Show:
- Total tracker bugs reviewed
- Unassigned: N bugs across M engineers
- Skipped (in progress): N
- Transferred out (picked up by external engineer): N — list each as "[key]: was [tracker assignee] → now [current assignee]"
- Skipped (PR-closed reset — deferred to /rit-sweep): N
- Skipped (active PR): N
- Already unassigned: N

---

## Tracker Format additions

```markdown
## Unassigned This Week (EOW Cleanup)

| Bug | Summary | Previous Assignee | Reason |
|-----|---------|-------------------|--------|
| OCPBUGS-XXXXX | ... | Engineer Name | Not picked up during rotation |

```

---

## Important Notes

- **Tracker is the only source of truth** — only bugs recorded in the current week's
  `triaged_bugs_YYYY-MM-DD.md` are eligible for cleanup.
- **Only unassign RIT roster engineers** — if the current assignee is not in the
  Engineering table of the tracker file, skip the bug entirely. It was intentionally
  assigned to a specialist; `/rit-sweep` handles the stale case later.
- **Never unassign bugs that have progressed** — any status other than `New` means the
  engineer picked it up; skip entirely.
- **Never unassign PR-closed reset bugs** — Prow Bot comment is the signal; skip silently.
  `/rit-sweep` owns the notification for these bugs.
- **Never unassign bugs with linked PRs** — active PR means work is in progress without
  a status update; skip it.
- **Batch by engineer** — propose all of one engineer's stale bugs at once for efficiency.
- **Always comment before unassigning** — never silently remove an assignee.
- **Paths use the current working directory** — never hardcode paths.
- **All comments must be Red Hat Employee only** — every `addCommentToJiraIssue` call must include `commentVisibility: {"type": "group", "value": "Red Hat Employee"}`. These are internal process notes and must not be publicly visible.
- **Skip EOW comment if bug is already closed** — after calling `editJiraIssue`, check the returned `status.statusCategory.key`. If it is `done`, the bug was closed concurrently; skip posting the comment (the unassignment is a no-op).
