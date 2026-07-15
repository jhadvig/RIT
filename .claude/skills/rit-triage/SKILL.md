---
name: rit-triage
description: Triage bugs from a PIXAA dashboard panel — assess, prioritize, assign to RIT team, and record
argument-hint: "<panel-name> — e.g. 'With Customer Cases', 'Component Regressions', 'Untriaged'"
allowed-tools: Read, Edit, Write, mcp__plugin_jira_atlassian__searchJiraIssuesUsingJql, mcp__plugin_jira_atlassian__getJiraIssue, mcp__plugin_jira_atlassian__editJiraIssue, mcp__plugin_jira_atlassian__addCommentToJiraIssue, mcp__plugin_jira_atlassian__getJiraIssueRemoteIssueLinks
---

# RIT Triage

Triage bugs from a PIXAA Bugs Dashboard panel. Fetches all bugs, loops through each one, applies triage actions (status transitions, priority, assignee, labels, comments), and records everything in the tracker file.

## Usage

```
/rit-triage With Customer Cases
/rit-triage Component Regressions
/rit-triage Untriaged
/rit-triage With Due Date
/rit-triage Release Blockers
/rit-triage With OCPPRIO link
/rit-triage In Progress
/rit-triage All Open
```

The panel name is matched fuzzily against the "PIXAA Bugs Dashboard JQL Queries" table in `rit_manual.md` — partial matches and missing leading words (e.g. "Customer Cases" → "With Customer Cases", "Release Blocker" → "Release Blockers") are accepted.

## Instructions

### Step 1: Load configuration

Read `rit_manual.md` from the **current working directory** (not a hardcoded path) and extract:

1. **JQL query** — Read the "Shared Constants" section to load the canonical PIXAA base filter and Prow Bot reset string. Then find the "PIXAA Bugs Dashboard JQL Queries" section. Locate the table row whose panel name fuzzy-matches the `<panel-name>` argument (case-insensitive, partial match, ignore leading "With"). Construct the full JQL:
   - Base filter: `filter in ("operator-framework-all-bugs", "PIXAA HIVE bugs", "PIXAA CCO bugs", "PIXAA OCPCLOUD bugs", "PIXAA Console bugs", "PIXAA MCP Server Bugs", "PIXAA Serverless Bugs", "All OTA Bugs")`
   - Plus the panel's extra conditions from the table
   - Plus common triage condition: `(labels is EMPTY or labels not in (triaged) or assignee is EMPTY or priority is EMPTY) and (labels not in (ocp-sustaining) or labels is EMPTY)`
   - Plus `statusCategory != done`
   - Plus the panel's sort order

   If no row matches even fuzzily, tell the user and list valid panel names.

   Also note whether the panel JQL already filters by Release Blocker status (e.g. `"Release Blocker" in (Approved, Proposed)`). If it does, set a flag `panel_is_release_blockers = true` — this suppresses Action 3 (release blocker assessment) since the field is already set.

2. **RIT team roster** — Find the most recent `triaged_bugs_YYYY-MM-DD.md` file in the **current working directory** (sort by date in filename, pick the latest). Extract the week-start date from the heading. If that date is more than 7 days in the past, **stop and warn the user**: "⚠️ The tracker file appears stale (week of YYYY-MM-DD). Please run `/rit-start` to set up the current rotation before running triage." Do not proceed until the user confirms. Then parse the "Engineering" table in the tracker to get: name, email, Jira Account ID, area of expertise, notes (e.g. PTO). Include **all** engineers regardless of their role label. Skip only engineers explicitly marked as PTO.

3. **Comment templates** — Find the "Comment templates for status transitions" section. Load the templates for "ASSIGNED — no linked PRs (ownership check)", "PR-closed reset (Prow Bot)", and "Setting priority (from Undefined)".

4. **Release Blocker rules** — Read `release_blocker_rules.md` from the same directory as this skill file. Load the A/C/R/P rule tables for use in Action 3. Also read the `CURRENT_RELEASE` value from the top of that file — use it when evaluating version-scoped rules (e.g. flagging bugs whose Affects Version does not match the current GA target).

5. **Tracker file** — The tracker file found in step 2 is also the source for assignment distribution. Read the "Assignment Distribution" table to know each engineer's current bug count and already-assigned keys. If the tracker file does not exist, **stop and tell the user to run `/rit-start` first**.

### Step 2: Fetch all bugs

1. Run the Jira search MCP tool with the constructed JQL. Request fields: `summary, status, assignee, priority, labels, components, created, updated`. Set limit to 50.

2. If there are more results (next page token), paginate until all bugs are fetched.

3. Build an in-memory list of all bugs with their full context: key, summary, status, priority, assignee, labels, components.

4. **Fast-path identification** — Classify each bug (evaluated in order; first match wins):
   - `possible_sustaining`: created < 60 minutes ago AND has any label matching `arc:*` → the `ocp-sustaining` label may not have been applied yet by the sustaining automation; **automatically skip** — do not label, assign, or set Release Blocker; report at the end of the session
   - `needs_only_assignee`: has `triaged` label ✓, has priority ✓, missing assignee only, **status is New or ASSIGNED** → the bug was fully triaged in a prior rotation (release blocker and priority are already assessed); still run Action 2 (component check — always mandatory), then skip Actions 3 and 4, go directly to Action 5 to assign an engineer
   - `needs_only_label`: has assignee ✓, has priority ✓, missing `triaged` label only → will be auto-applied, no confirmation needed
   - `needs_triage`: missing one or more of priority, assignee, or needs status transition
   - `post_missing_fields`: status is POST/ON_QA/Modified AND (missing priority OR missing `triaged` label) → apply missing fields only, skip status/assignment changes
   - `post_clean`: status is POST/ON_QA/Modified AND all fields set → fully skip

### Step 3: Present summary to user

Show:
- Total bug count
- Grouped by status: how many in New, ASSIGNED, POST, ON_QA, other
- How many need: priority (Undefined), assignee (Unassigned), `triaged` label only, assignee only (previously triaged), status transition (ASSIGNED)
- How many POST/ON_QA bugs still have missing fields (will be partially acted on)
- How many auto-skipped as `possible_sustaining` (created < 1h ago with `arc:*` labels)
- Current RIT team load from tracker (engineer → bug count)

If the panel has more than 10 bugs, offer a processing shortcut:
> "This panel has N bugs. Process all at once, or start with a subset (e.g. Critical only)?"

Ask user: "Proceed with triage? (yes/no)"

If the user says no, stop.

### Step 4: Process bugs

#### 4a. Assign engineer to `needs_only_assignee` bugs

For bugs classified as `needs_only_assignee` (previously fully triaged, missing assignee only):
- **Always run Action 2 first** (component check — mandatory for every PIXAA bug). If Action 2 reveals the bug is not PIXAA, stop and skip it entirely; do not assign.
- If component is confirmed PIXAA: batch propose assignments using the same even-load strategy as Action 5 — group by component where possible, propose to the user, and on confirmation assign via Jira.
- Skip Actions 3 and 4 (release blocker and priority already set in prior triage).
- Record each in the tracker.

#### 4b. Auto-apply `triaged` label to `needs_only_label` bugs

For all bugs classified as `needs_only_label`: apply the `triaged` label immediately in parallel. No user confirmation needed. Record each in the tracker.

#### 4c. Apply missing fields to `post_missing_fields` bugs

For bugs in POST/ON_QA/Modified status that are still missing fields:
- If priority is Undefined: assess and propose priority (see Action 4 below). Wait for confirmation, then set it and add the priority comment.
- If `triaged` label is missing: add it automatically (preserve existing labels).
- Do NOT change assignee or status for these bugs.
- Record each in the tracker under "Triaged This Week".

#### 4d. Full triage loop for `needs_triage` bugs

Process in priority order (Critical first, then Major, Normal, Minor, Undefined).

**Group proposals by component/engineer when possible.** Rather than one-by-one confirmation, batch bugs of the same component and propose their assignments together (e.g. "Console bugs — propose assigning 4 to Jackson Lee, 3 to Robert Luby, 4 to Jon Jackson. Confirm?"). Always pause for each batch.

For each bug:

##### Action 1: Assess status
- If status is POST, ON_QA, or Modified → handled in 4c above; skip here.
- If status is ASSIGNED and the bug has NO linked PRs:
  - Fetch remote links (`getJiraIssueRemoteIssueLinks`) to confirm no PR is linked.
  - Post the "ASSIGNED — no linked PRs (ownership check)" comment, @-mentioning the assignee.
  - Do NOT transition the status or change the assignee. No user confirmation needed.
- If status is `New` AND assignee is already set:
  - Fetch the full issue with comments (`getJiraIssue` with `comment` field). Check for an **OpenShift Prow Bot** comment containing `"Bug status changed to NEW as previous linked PR"`.
  - If found: post the "PR-closed reset (Prow Bot)" comment, @-mentioning the assignee. Continue with Actions 2–6 as normal (the bug still needs its missing fields set; do NOT change the assignee).
  - If not found: continue to next actions normally.
- If status is New AND assignee is empty → continue to next actions.

##### Action 2: Assess component
- **Always run — never skip, regardless of which panel the bug came from.**
- Check if the component is a known PIXAA component (Console, OLM, Hive, Cloud Compute, CCO, CVO, OSUS, Serverless, HyperShift, or subcomponents).
- **Also flag if the bug summary prominently names a different component than the Jira `components` field** (e.g. summary says "CCO" but component is "Storage / Operators") — treat this as a mismatch and ask the user.
- If obviously not PIXAA → **ask user**: "OCPBUGS-XXXXX component is [component]. This doesn't look like a PIXAA component. Transfer? To which component?"
- If the user confirms it's not PIXAA: do NOT run Actions 3–6. Do NOT assign, label, or set Release Blocker. Remove from tracker if already recorded. Continue to next bug.
- If unclear, assume correctly aligned and continue.

##### Action 3: Set Release Blocker
- **Mandatory for every PIXAA bug — always set to Approved, Proposed, or Rejected. Never leave null.**
- If the panel is Release Blockers (`panel_is_release_blockers = true`): the field is already set — skip to Action 4.
- For all other bugs, evaluate using the rules in `release_blocker_rules.md` (same directory as this skill). Read that file at the start of any triage session — it contains the full A/C/R/P rule tables and evaluation flow.
- Fetch the full issue (`getJiraIssue`) if you need the description or labels not already loaded.
- Apply the evaluation flow in order: A-series (auto-approve) → C-series (conditional/proposed) → R-series (reject). Include the rule ID(s) and one-line reason in your proposal.
- Propose: `"OCPBUGS-XXXXX: Release Blocker → [Approved / Proposed / Rejected] [rule ID — reason]?"`
- **Wait for user confirmation.**
- If confirmed: set `customfield_10847` via Jira.

##### Action 4: Set priority
- If priority is Undefined:
  - Read the bug summary and description (fetch full issue if needed) to assess.
  - Apply the manual's priority criteria:
    - CVE → normally Critical
    - Regression already shipped, customers affected → Major or Critical
    - High customer impact, no workaround → Major
    - Workaround exists, low impact → Normal
    - Cosmetic, no functional impact → Minor
  - Propose: "OCPBUGS-XXXXX: propose [Priority] — [one-line reasoning]"
  - **Wait for user confirmation**
  - If confirmed: set priority, add comment using the "Setting priority (from Undefined)" template.

##### Action 5: Assign engineer
- If assignee is Unassigned:
  - **Default strategy: even load distribution across all engineers** (regardless of role label). Expertise area is used only as a tiebreaker when multiple engineers have equal bug counts.
  - Pick the engineer(s) with the lowest current bug count.
  - When proposing multiple bugs of the same component in a batch, distribute them round-robin among the lowest-loaded engineers.
  - Propose the full batch: "Console bugs (N total): assign X to [engineer A], Y to [engineer B], Z to [engineer C]?"
  - **Wait for user confirmation** — the user may adjust individual assignments.
  - If confirmed: update assignees via Jira.

##### Action 6: Add `triaged` label
- If the bug doesn't have the `triaged` label: add it (preserve all existing labels). No confirmation needed.

##### Action 7: Handle unclear cases
- No description, no repro steps, missing logs, possible duplicate, needs SME → **Pause and ask user**: "OCPBUGS-XXXXX: [describe issue]. How do you want to handle this?"

### Step 5: Bulk tracker update

After all bugs in the panel are processed, perform a **single bulk write** to the tracker file:

1. Append all newly triaged bugs to the "Triaged This Week" table.
2. Rewrite the "Assignment Distribution" table with updated counts and keys (see Tracker Format).
3. If any bugs were closed during triage, add them to "Closed This Week".

### Step 6: Final summary

Show:
- Total bugs processed
- Breakdown: fully triaged, label-only, POST/partially triaged, skipped (POST clean), closed
- Updated assignment distribution table
- Any bugs paused/skipped for user follow-up
- **Possible sustaining (auto-skipped)**: list any bugs skipped due to sustaining lag (created < 1h ago with `arc:*` labels) with a note: "These were skipped — created recently with `arc:*` labels; the `ocp-sustaining` label may still be pending. Re-check in ~1 hour."

---

## Tracker Format

```markdown
# RIT Triage Tracker — Week of YYYY-MM-DD (Pod Name)

## Assignment Distribution

| Engineer | Bugs Assigned | Keys |
|----------|--------------|------|
| Name | N | OCPBUGS-XXXXX, OCPBUGS-YYYYY, ... |

## Triaged This Week

| Bug | Summary | Status | Priority | Assignee | Actions Taken |
|-----|---------|--------|----------|----------|---------------|

## Closed This Week

| Bug | Summary | Resolution |
|-----|---------|------------|

## Skipped (POST/ON_QA/Modified)

| Bug | Summary | Status | Notes |
|-----|---------|--------|-------|
```

The **Assignment Distribution** table must include **all engineers** from the roster (even those with 0 bugs assigned this week) and a **Keys** column listing every bug key assigned to that engineer, comma-separated.

---

## Important Notes

- **Automate the obvious, pause on judgment** — `triaged` labels are applied automatically. Priority, assignee, component transfer, and release blocker always require a user proposal + confirmation.
- **Action 2 is the gate — always run it, never skip it** — Even for the Release Blockers panel. If a bug is not PIXAA, make zero changes (no label, no assignee, no release blocker, no tracker entry) and move on. A bug appearing in a PIXAA panel does not guarantee it belongs to PIXAA.
- **Summary ≠ Component** — If the bug summary prominently names a PIXAA component but the Jira `components` field points elsewhere (or vice versa), treat it as a mismatch and apply full Action 2 scrutiny before touching anything.
- **Release Blocker is mandatory for every PIXAA bug** — Always set to Approved, Proposed, or Rejected. Never leave null after triage. Use the full rule set from `release_blocker_rules.md` (A/C/R/P framework); Rejected is the default when no A or C rule matches. Always cite the rule ID(s) in your proposal. Skip entirely for non-PIXAA bugs (Action 2 handles those before reaching Action 3).
- **POST/ON_QA bugs: partial triage only** — Never change status or assignee. Do apply missing priority (with comment) and `triaged` label.
- **Even load distribution is the default** — Expertise area is a tiebreaker, not the primary criterion. All engineers (regardless of role label) receive bugs and appear in the distribution table.
- **Batch proposals for efficiency** — Group assignment proposals by component/engineer rather than confirming one bug at a time.
- **Bulk tracker write at end of panel** — Record all bugs in one write, not incrementally.
- **Always comment on status/priority changes** — Use the templates from rit_manual.md.
- **Preserve existing labels** — When adding `triaged`, keep all existing labels on the bug.
- **Respect PTO** — Skip engineers explicitly marked as PTO in the roster.
- **Release Blockers panel** — Bugs fetched via the Release Blockers panel already have the Release Blocker field set; skip the assessment part of Action 3 but still verify the component in Action 2.
- **PR-closed reset** — Two cases: (1) Bug was **fully triaged** (triaged ✓, priority ✓, assignee ✓) before the PR closed — all fields are retained, bug is invisible to this skill's JQL, handled exclusively by `/rit-sweep`. (2) Bug was **partially triaged** (missing at least one field) — it appears in the JQL and Action 1 detects the Prow Bot comment, posts the PR-closed reset notification, then continues normally through Actions 2–6 to finish setting the missing fields.
- **Sustaining label lag** — The `ocp-sustaining` label can take up to an hour to be applied by the sustaining bot after a bug is created. Bugs created < 60 minutes ago that carry `arc:*` labels are automatically skipped and listed at the end — do not triage them.
- **Paths use the current working directory** — Never use hardcoded absolute paths for `rit_manual.md` or tracker files.
- **If the Jira API can't update a field** (screen configuration error), tell the user to do it manually in the UI and continue with the next action.
- **All comments must be Red Hat Employee only** — every `addCommentToJiraIssue` call must include `commentVisibility: {"type": "group", "value": "Red Hat Employee"}`. These are internal process notes and must not be publicly visible.
