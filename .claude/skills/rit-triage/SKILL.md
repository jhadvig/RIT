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

6. **Assignment distribution** — Read the "Assignment Distribution" table from the tracker. Parse each engineer's current bug count (from "X of Y" format) and assigned keys. Load the soft and hard limit values from the header line.

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
- Per-engineer load: name → "X of Y"

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

Spawned via the Agent tool. Receives triaged-but-unassigned bugs in sort order, plus the current assignment distribution (engineer → count, keys) and roster (name, Jira Account ID, expertise area).

**Step 1: Build candidate list**
- All engineers not at hard limit (`HARD_LIMIT`), sorted by current bug count ascending.
- Expertise area as tiebreaker when counts are equal.

**Step 2: Propose assignments**
- Group bugs by component where possible.
- Propose batches: "Console bugs (3): assign OCPBUGS-X to Jackson Lee, OCPBUGS-Y to Jon Jackson, OCPBUGS-Z to Jakub Hadvig?"
- Wait for user confirmation. User may adjust individual assignments.

**Step 3: Enforce capacity limits**
- **Under soft limit** (count < `SOFT_LIMIT`): assign normally.
- **At or above soft limit** (count >= `SOFT_LIMIT` and count < `HARD_LIMIT`): assign, but show warning: "⚠️ [Engineer] at [N] of [SOFT_LIMIT] — soft limit reached."
- **At hard limit** (count >= `HARD_LIMIT`): show 🛑, skip this engineer, pick next-lowest-loaded.
- **All engineers at hard limit**: stop assigning. Report: "All engineers at capacity. N bugs remain unassigned."

**Step 4: Execute**
- On confirmation: update assignee via Jira.
- Return to parent: bug key, assigned engineer, new count.

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

1. Rewrite "Assignment Distribution" with updated counts ("X of Y" format), keys, bandwidth summary, and capacity indicators (⚠️/🛑).
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

| Engineer | Bugs | Keys |
|----------|------|------|
| Name | X of 6 | OCPBUGS-XXXXX, OCPBUGS-YYYYY, ... |
| Name | 6 of 6 ⚠️ | OCPBUGS-XXXXX, ... |
| Name | 8 of 6 🛑 | OCPBUGS-XXXXX, ... |

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

The **Assignment Distribution** table must include **all engineers** from the roster (even those with 0 bugs) and a **Keys** column listing every bug key assigned, comma-separated. The **Bugs** column uses "X of Y" format where Y is `SOFT_LIMIT`. Add ⚠️ when X >= Y (soft limit reached). Add 🛑 when X >= `HARD_LIMIT`.

The **bandwidth summary** line uses: capacity = `(total_bugs / (engineer_count * SOFT_LIMIT)) * 100`.

---

## Important Notes

- **No panel argument** — all panels are fetched and processed automatically in priority order. The user does not specify a panel.
- **Dedup before processing** — a bug appearing in multiple panels is processed only once, attributed to the highest-priority panel it appeared in.
- **Three-tier sort drives everything** — panel tier, then bug priority, then panel order. This determines triage order AND assignment order (higher-priority bugs get first pick of lowest-loaded engineers).
- **Anomaly detection** — Normal/Minor bugs in Tier 1 panels are flagged. These panels should not contain low-priority bugs.
- **Soft limit = warning, hard limit = stop** — at soft limit (6), show ⚠️ and continue assigning. At hard limit (8), show 🛑 and skip to next engineer. Track in Assignment Distribution.
- **All engineers at capacity** — if every engineer is at hard limit, stop assigning and report remaining unassigned bugs.
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
