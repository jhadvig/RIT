---
name: rit-triage
description: Triage and assign bugs across all PIXAA dashboard panels — auto-prioritized, with parallel triage and assign sub-agents
allowed-tools: Read, Edit, Write, Agent, mcp__plugin_jira_atlassian__searchJiraIssuesUsingJql, mcp__plugin_jira_atlassian__getJiraIssue, mcp__plugin_jira_atlassian__editJiraIssue, mcp__plugin_jira_atlassian__addCommentToJiraIssue, mcp__plugin_jira_atlassian__getJiraIssueRemoteIssueLinks
---

# RIT Triage

Triage and assign bugs across all PIXAA dashboard panels. Fetches all panels, deduplicates, sorts by panel tier and bug priority, then pipelines bugs through a triage sub-agent (assessment) and assign sub-agent (engineer selection). Includes a catch pass for Major/Critical outliers and optional Slack summary output.

## Usage

```
/rit-triage
```

No arguments. All panels are fetched and processed automatically in priority order.

## Panel Priority Order

### Tier 1 (high priority)

1. With OCPPRIO link
2. Component Regressions
3. Release Blockers

### Tier 2 (remaining)

4. With Customer Cases
5. With Due Date
6. Untriaged

### Catch pass (report only)

7. In Progress
8. All Open

## Sort Order

After fetch and dedup, all bugs are sorted by:

1. **Panel tier** — Tier 1 before Tier 2.
2. **Bug priority** — Critical → Major → Normal → Minor → Undefined.
3. **Panel order within tier** — position within the tier list above.

**Anomaly detection:** Normal or Minor bugs in Tier 1 panels are flagged for user review — these panels should not contain low-priority bugs under normal circumstances.

## Instructions

### Phase 1: Fetch & Prepare (parent agent)

#### Step 1: Load configuration

Read `rit_manual.md` from the **current working directory** and extract:

1. **Shared Constants** — Load the canonical PIXAA base filter, Prow Bot reset string, `SOFT_LIMIT`, and `HARD_LIMIT`.

2. **JQL queries** — Read the "PIXAA Bugs Dashboard JQL Queries" section. Load the JQL for all 8 panels (6 triage panels + 2 catch panels). For each panel, construct the full JQL:
   - Base filter from Shared Constants
   - Plus the panel's extra conditions from the table
   - Plus common triage condition: `(labels is EMPTY or labels not in (triaged) or assignee is EMPTY or priority is EMPTY) and (labels not in (ocp-sustaining) or labels is EMPTY)`
   - Plus `statusCategory != done`
   - Plus the panel's sort order

   Note which panels filter by Release Blocker status (e.g. Release Blockers panel). Set `panel_is_release_blockers = true` for those — this suppresses Action 3 (release blocker assessment).

3. **Comment templates** — Load templates for "ASSIGNED — no linked PRs (ownership check)", "PR-closed reset (Prow Bot)", and "Setting priority (from Undefined)".

4. **Release Blocker rules** — Read `release_blocker_rules.md` from the same directory as this skill file. Load the A/C/R/P rule tables. Read `CURRENT_RELEASE`.

5. **RIT team roster** — Find the most recent `triaged_bugs_YYYY-MM-DD.md` file in the **current working directory** (sort by date in filename, pick latest). Extract week-start date. If that date is more than 7 days in the past, **stop and warn**: "⚠️ The tracker file appears stale (week of YYYY-MM-DD). Please run `/rit-start` to set up the current rotation before running triage." Parse the "Engineering" table for: name, email, Jira Account ID, area of expertise, notes (PTO). Include all engineers regardless of role label. Skip engineers marked PTO.

6. **Assignment distribution** — Read the "Assignment Distribution" table from the tracker. Parse each engineer's RIT assigned count and keys. Load the soft and hard limit values and **capacity mode** from the header line (`Soft limit: N | Hard limit: M | Capacity mode: baseline|total`). If `Capacity mode` is absent, default to `total` for backwards compatibility. (The "Total Open" column is a snapshot from the last triage run — it will be refreshed by step 7.)

7. **Total workload** — For each engineer in the roster, run a Jira query: `assignee = "<accountId>" AND project = OCPBUGS AND statusCategory != done`. Request fields: `priority`. Store per engineer:
   - `total_open`: total count of all open bugs (regardless of source)
   - `priority_breakdown`: count by priority (Critical, Major, Normal, Minor) — used by the assign sub-agent to determine whether an engineer at capacity can still receive a Critical or Major bug
   Soft and hard limits apply against `total_open`, not just RIT-assigned count.

#### Step 2: Fetch all panels

1. Fetch all 6 triage panels (Tier 1 then Tier 2) via Jira JQL. Request fields: `summary, status, assignee, priority, labels, components, created, updated`. Paginate until all bugs fetched per panel.

2. Tag each bug with its source panel name and tier.

3. **Deduplicate** — Process panels in priority order. If a bug key already appeared in an earlier panel, discard the duplicate. First panel occurrence wins.

4. **Sort** — Apply the three-tier sort: panel tier → bug priority → panel order within tier.

5. **Classify** each bug (evaluated in order, first match wins):
   - `possible_sustaining`: created < 60 minutes ago AND has any `arc:*` label → auto-skip.
   - `needs_only_assignee`: has `triaged` label ✓, has priority ✓, missing assignee only, status is New or ASSIGNED → run Action 2 (component check), skip Actions 3-4, pass directly to assign sub-agent.
   - `needs_only_label`: has assignee ✓, has priority ✓, missing `triaged` label only → auto-apply.
   - `needs_triage`: missing one or more of priority, assignee, or needs status transition.
   - `post_missing_fields`: status is POST/ON_QA/Modified AND (missing priority OR missing `triaged` label) → apply missing fields only, skip status/assignment changes.
   - `post_clean`: status is POST/ON_QA/Modified AND all fields set → skip.

#### Step 3: Present summary

Show:
- Total bug count across all panels
- Bugs by tier: Tier 1 count, Tier 2 count
- Bugs by classification: needs_triage, needs_only_assignee, needs_only_label, post_missing_fields, post_clean, possible_sustaining
- Anomalies: Normal/Minor bugs in Tier 1 panels
- Current team bandwidth status from tracker:
  ```
  Team bandwidth: X% (N/M soft capacity) | A at limit | B over | C available
  ```
- Per-engineer load: name → "X of Y" (where X = total open bugs, Y = soft limit)

Ask: "Proceed with triage? (yes/no)"

If no, stop.

### Phase 2: Pipeline (parent orchestrates sub-agents)

Process bugs in sort order. Parent hands bugs to sub-agents in tier-sized batches:
- Batch 1: all Tier 1 bugs
- Batch 2: all Tier 2 bugs

For each batch:
1. Send bugs to **triage sub-agent**. Wait for results.
2. Collect triaged-but-unassigned bugs from triage results.
3. Send those bugs to **assign sub-agent**. While assign works on Batch 1, triage can start Batch 2.
4. Parent serializes all user-facing questions — when either sub-agent needs user input, parent relays the question and returns the answer.

#### Triage sub-agent

Spawned via the Agent tool. Receives a batch of bugs with their classifications, panel source, and all loaded configuration (comment templates, release blocker rules, PIXAA component list).

For each bug in sort order:

##### Auto-apply `triaged` label to `needs_only_label` bugs

Apply `triaged` label immediately. No confirmation. Preserve existing labels.

##### Apply missing fields to `post_missing_fields` bugs

- If priority is Undefined: assess and propose priority. Wait for confirmation. Set and add priority comment.
- If `triaged` label missing: add automatically.
- Do NOT change assignee or status.

##### Full triage for `needs_triage` bugs

**Action 1: Assess status**
- POST/ON_QA/Modified → handled above in post_missing_fields.
- ASSIGNED + no linked PRs: fetch remote links to confirm. Post "ASSIGNED — no linked PRs (ownership check)" comment. Do NOT transition status or change assignee.
- New + assignee set: fetch full issue with comments. Check for Prow Bot comment containing the reset string. If found: post "PR-closed reset (Prow Bot)" comment. Continue with Actions 2-5 normally (do NOT change assignee).
- New + no assignee → continue.

**Action 2: Assess component (always run, never skip)**
- Check if component is a known PIXAA component (Console, OLM, Hive, Cloud Compute, CCO, CVO, OSUS, Serverless, HyperShift, or subcomponents).
- Flag if bug summary names a different component than Jira `components` field.
- If not PIXAA → ask user. If confirmed non-PIXAA: skip all remaining actions, do not assign, label, or set Release Blocker. Remove from processing.

**Action 3: Set Release Blocker**
- Mandatory for every PIXAA bug. Never leave null.
- If bug came from Release Blockers panel (`panel_is_release_blockers = true`): skip (already set).
- Otherwise: evaluate using A/C/R/P rules from `release_blocker_rules.md`. Propose with rule ID(s) and one-line reason. Wait for user confirmation. Set `customfield_10847` via Jira.

**Action 4: Set priority**
- If Undefined: assess from summary/description. Propose with reasoning. Wait for confirmation. Set priority and post "Setting priority (from Undefined)" comment.

**Action 5: Add `triaged` label**
- If missing: add (preserve existing labels). No confirmation.

##### Handle `needs_only_assignee` bugs

- Run Action 2 (component check — mandatory). If not PIXAA, skip entirely.
- Skip Actions 3-4 (release blocker and priority already set).
- Pass directly to assign sub-agent via parent.

##### Handle unclear cases

No description, no repro steps, missing logs, possible duplicate, needs SME → pause and ask user.

##### Output

Return each bug to parent with: key, assessed state (priority, release blocker, component verified, labels), classification, whether it needs assignment.

#### Assign sub-agent

Spawned via the Agent tool. Receives triaged-but-unassigned bugs (already sorted by priority from triage), plus the current workload per engineer (total open count, priority breakdown, assigned keys) and roster (name, Jira Account ID, expertise area).

Assignment proceeds in three waves. Each wave proposes a batch to the user before executing.

**Wave 1 — Critical bugs (always assigned)**
- All Critical priority and Release Blocker Approved bugs.
- Capacity limits do NOT apply — these are always assigned.
- Rank engineers by: expertise match first (prefer component expert even with up to 2 more total bugs), then lowest total open count.
- Propose batch: "Critical bugs (N): assign OCPBUGS-X to Jackson Lee (4 of 6), OCPBUGS-Y to Jon Jackson (7 of 6 ⚠️)?"
- Wait for user confirmation. User may adjust.
- Execute confirmed assignments via Jira. Update running totals.

**Wave 2 — Major bugs (always assigned)**
- All Major priority bugs.
- Capacity limits do NOT apply — these are always assigned.
- Distribute as evenly as possible across all engineers, using updated totals from Wave 1.
- Rank by: expertise match (up to 2-bug advantage), then lowest total open count.
- Propose batch. Wait for confirmation. Execute. Update totals.

**Wave 3 — Normal and lower priority bugs (backfill)**
- All Normal, Minor, and Undefined priority bugs.
- The **load counter** used for capacity checks depends on the capacity mode from the tracker header:
  - **`baseline` mode**: use `rit_assigned` (bugs assigned by `/rit-triage` this week only). Pre-existing bugs are informational and do not gate assignment.
  - **`total` mode**: use `total_open` (all open bugs in Jira, regardless of source).
- Capacity limits apply to the load counter:
  - **Under soft limit** (load < `SOFT_LIMIT`): assign normally.
  - **At or above soft limit** (load >= `SOFT_LIMIT` and < `HARD_LIMIT`): assign, but show ⚠️ warning.
  - **At hard limit** (load >= `HARD_LIMIT`): skip this engineer, pick next-lowest-loaded.
  - **All engineers at or above soft limit**: stop assigning. Report: "All engineers at capacity for normal-priority bugs. N bugs remain unassigned."
- Rank by: expertise match (up to 2-bug advantage), then lowest load counter.
- Propose batch. Wait for confirmation. Execute. Update totals.

**Proposal format**

All wave proposals show the engineer's current load counter against the soft limit:
- **`baseline` mode**: show RIT assigned count — `"Console bugs (3): assign OCPBUGS-X to Jackson Lee (1 of 6), OCPBUGS-Y to Jon Jackson (0 of 6)"`
- **`total` mode**: show total open count — `"Console bugs (3): assign OCPBUGS-X to Jackson Lee (4 of 6), OCPBUGS-Y to Jon Jackson (2 of 6)"`

Add ⚠️ when load counter >= `SOFT_LIMIT` and < `HARD_LIMIT`. Add 🛑 when load counter >= `HARD_LIMIT`.

**Output**

Return to parent: bug key, assigned engineer, updated total count per wave.

### Phase 3: Catch & Report (parent agent)

#### Step 1: Fetch catch panels

Fetch In Progress and All Open panels via Jira JQL. Same field set.

#### Step 2: Filter outliers

- Keep only Major or Critical bugs.
- Keep only bugs missing one or more of: `triaged` label, priority, assignee, release blocker.
- Remove any bug already processed in Phase 2 (dedup against processed set).

#### Step 3: Report

- Display outliers in terminal: "⚠️ These Major/Critical bugs may need attention:" with bug key, summary, priority, panel, and which fields are missing.
- No auto-action — RIT lead decides.

#### Step 4: Slack summary (optional)

Ask: "Generate Slack summary for posting? (yes/no)"

If yes, output copy-paste-ready Slack mrkdwn with all assigned bugs (full plate) grouped by engineer:

```
🐛 RIT Triage Update — Week of YYYY-MM-DD

*Engineer Name*
• <https://issues.redhat.com/browse/KEY|KEY> — Summary
• <https://issues.redhat.com/browse/KEY|KEY> — Summary

*Engineer Name*
• ...
```

- All bugs from Assignment Distribution (not just this session).
- Grouped by engineer, sorted alphabetically by last name.
- Slack link format: `<URL|KEY>`.
- No capacity numbers or bandwidth info.

#### Step 5: Bulk tracker update

Single bulk write to tracker file:

1. Rewrite "Assignment Distribution" with updated total open counts ("X of Y" format), RIT assigned counts, keys, bandwidth summary, and capacity indicators (⚠️/🛑).
2. Append all newly triaged bugs to "Triaged This Week".
3. Add any closed bugs to "Closed This Week".
4. Write outliers to "Outliers (In Progress / All Open)".

#### Step 6: Final summary

Show:
- Total bugs processed across all panels
- Breakdown: fully triaged, label-only, POST/partially triaged, skipped (POST clean), closed
- Updated bandwidth status
- Anomalies flagged (Normal/Minor in Tier 1)
- Outliers found (Major/Critical in catch pass)
- Bugs paused/skipped for user follow-up
- Possible sustaining (auto-skipped): list with note to re-check in ~1 hour

---

## Tracker Format

```markdown
# RIT Triage Tracker — Week of YYYY-MM-DD (Pod Name)

## Assignment Distribution

Soft limit: 6 | Hard limit: 8

Team bandwidth: X% (N/M soft capacity) | A at limit | B over | C available

| Engineer | Total Open | RIT Assigned | Keys |
|----------|------------|--------------|------|
| Name | X of 6 | N | OCPBUGS-XXXXX, OCPBUGS-YYYYY, ... |
| Name | 6 of 6 ⚠️ | N | OCPBUGS-XXXXX, ... |
| Name | 8 of 6 🛑 | N | OCPBUGS-XXXXX, ... |

## Triaged This Week

| Bug | Summary | Status | Priority | Assignee | Actions Taken |
|-----|---------|--------|----------|----------|---------------|

## Closed This Week

| Bug | Summary | Resolution |
|-----|---------|------------|

## Skipped (POST/ON_QA/Modified)

| Bug | Summary | Status | Notes |
|-----|---------|--------|-------|

## Outliers (In Progress / All Open)

| Bug | Summary | Priority | Panel | Missing |
|-----|---------|----------|-------|---------|
```

The **Assignment Distribution** table must include **all engineers** from the roster (even those with 0 bugs). Columns:
- **Total Open**: "X of Y" format where X = engineer's total open bugs in Jira (all sources), Y = `SOFT_LIMIT`. Add ⚠️ when X >= Y and < `HARD_LIMIT`. Add 🛑 when X >= `HARD_LIMIT` (🛑 takes precedence).
- **RIT Assigned**: count of bugs assigned by `/rit-triage` this week (keys in the Keys column).
- **Keys**: comma-separated bug keys assigned by `/rit-triage` this week. Used by `/rit-end` for cleanup.

The **bandwidth summary** line uses the load counter for each engineer (RIT assigned in `baseline` mode, total open in `total` mode): capacity = `(sum_load / (engineer_count * SOFT_LIMIT)) * 100`. Bandwidth categories apply to the load counter: "available" = load < `SOFT_LIMIT`, "at limit" = `SOFT_LIMIT` <= load < `HARD_LIMIT`, "over" = load >= `HARD_LIMIT`.

---

## Important Notes

- **No panel argument** — all panels are fetched and processed automatically in priority order. The user does not specify a panel.
- **Dedup before processing** — a bug appearing in multiple panels is processed only once, attributed to the highest-priority panel it appeared in.
- **Three-tier sort drives everything** — panel tier, then bug priority, then panel order. This determines triage order AND assignment order (higher-priority bugs get first pick of lowest-loaded engineers).
- **Anomaly detection** — Normal/Minor bugs in Tier 1 panels are flagged. These panels should not contain low-priority bugs.
- **Priority-wave assignment** — bugs are assigned in three waves: Critical (always assigned, no capacity limits), Major (always assigned, even distribution), Normal/Minor (backfill to engineers with remaining capacity). This ensures high-priority bugs are never left unassigned.
- **Capacity mode drives limit enforcement** — read `Capacity mode` from the tracker header. In `baseline` mode, limits apply to RIT-assigned bugs only (pre-existing load is informational). In `total` mode, limits apply to total open Jira bugs from all sources. Default to `total` if the field is absent.
- **Soft limit = warning, hard limit = stop (Normal/Minor only)** — at soft limit, show ⚠️. At hard limit, show 🛑 and skip. These limits only gate Wave 3 (Normal/Minor). Critical and Major bugs are always assigned regardless of capacity.
- **All engineers at capacity** — for Normal/Minor, stop assigning and report remaining. Critical/Major are never blocked.
- **Expertise-weighted assignment** — within each wave, expertise match is preferred even with up to 2 more total bugs than a non-expert. Beyond a 2-bug difference, load takes priority.
- **Catch pass is report-only** — In Progress and All Open panels are scanned for Major/Critical outliers but no automatic action is taken.
- **Slack summary is optional** — only generated when user says yes. Includes all assigned bugs (full plate), no capacity numbers.
- **Automate the obvious, pause on judgment** — `triaged` labels are applied automatically. Priority, assignee, component transfer, and release blocker always require a proposal + confirmation.
- **Action 2 is the gate — always run, never skip** — even for Release Blockers panel. If not PIXAA, make zero changes and move on.
- **Summary ≠ Component** — flag mismatches between summary and Jira components field.
- **Release Blocker is mandatory for every PIXAA bug** — always set to Approved, Proposed, or Rejected. Use full rule set from `release_blocker_rules.md`. Skip for non-PIXAA bugs.
- **POST/ON_QA bugs: partial triage only** — never change status or assignee.
- **Batch proposals for efficiency** — group assignment proposals by component/engineer.
- **Bulk tracker write at end** — record all bugs in one write, not incrementally.
- **Always comment on status/priority changes** — use templates from `rit_manual.md`.
- **Preserve existing labels** — when adding `triaged`, keep all existing labels.
- **Respect PTO** — skip engineers marked PTO in roster.
- **Release Blockers panel** — bugs from this panel already have Release Blocker field set; skip assessment in Action 3 but still verify component in Action 2.
- **PR-closed reset** — same two cases as before: fully triaged (invisible to this skill, handled by `/rit-sweep`) and partially triaged (appears in JQL, detected in Action 1).
- **Sustaining label lag** — bugs created < 60 min ago with `arc:*` labels are auto-skipped and listed at end.
- **Paths use current working directory** — never hardcode paths.
- **If Jira API can't update a field** (screen config error), tell user to do it manually and continue.
- **All comments must be Red Hat Employee only** — every `addCommentToJiraIssue` call must include `commentVisibility: {"type": "group", "value": "Red Hat Employee"}`.
