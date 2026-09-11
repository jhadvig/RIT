---
name: rit-start
description: Start-of-week RIT setup — select this week's RIT lead and team from the People Directory, create the week's tracker file with roster and empty triage tables
argument-hint: "[YYYY-MM-DD] — optional week start date (Monday); defaults to the most recent Monday"
allowed-tools: Read, Edit, Write, mcp__plugin_jira_atlassian__searchJiraIssuesUsingJql
---

# RIT Start

Start-of-week skill. Reads the full PIXAA roster from the People Directory in `rit_manual.md`, selects the RIT lead (which determines the pods on rotation), allows excluding pods or individuals, then creates the tracker file.

## Usage

```
/rit-start
/rit-start 2026-07-14
```

## Instructions

### Step 1: Load roster

Read `rit_manual.md` from the **current working directory**. Extract:

1. **Non-Engineering Leads table** from the "People Directory" section: name, email, Jira Account ID, pods for every row.
2. **Engineering Roster table** from the "People Directory" section: name, email, Jira Account ID, area of expertise, pod for every row.
3. **Bug count limits** — Read the "Shared Constants" section of `rit_manual.md` and extract `SOFT_LIMIT` and `HARD_LIMIT` values.

If the People Directory section is missing, stop and tell the user to add it.

### Step 2: Confirm week start date

If a date was passed as an argument, confirm with the user:
> "Setting up RIT rotation for week of YYYY-MM-DD. Correct? (yes/no)"

If no date was passed, ask:
> "Week start date? (YYYY-MM-DD, default: \<most recent Monday\>)"

Calculate the most recent Monday: if today is Monday, use today; otherwise use the most recent past Monday.

### Step 3: Select RIT Lead

Display the Non-Engineering Leads as a numbered list:
```
Non-Engineering Leads:
1. Christopher Moore — cmoore@redhat.com (Green Koala, Green Wolf)
2. Gavin Bell — gabell@redhat.com (Brown Adder, Brown Hornet)
...
```

Ask: "Who is the RIT Lead this week? (number)"

Record the selected lead's name, email, Jira Account ID, and pods.

### Step 4: Select pods on rotation

The selected lead manages one or more pods. List them:

If the lead has **one pod**: auto-select it and confirm:
> "Pod on rotation: Green Koala (5 engineers). Correct? (yes/no)"

If the lead has **multiple pods**: list them with engineer counts and ask which to include:
```
Gavin Bell manages:
1. Brown Adder (5 engineers)
2. Brown Hornet (5 engineers)

Include all pods? (yes / comma-separated numbers to include)
```

This determines the initial team — all engineers from the selected pod(s).

### Step 5: Review team and exclude individuals

Show the full team from the selected pod(s):
```
Team for this rotation (10 engineers):

--- Puce Vole ---
1. Nolan Brubaker — CAPI / Cluster Infrastructure
2. Rashmi Gottipati — OLM
3. Jon Jackson — Console
4. Jackson Lee — Console
5. Suhani Mehta — Hive

--- Papaya Goat ---
6. Pedro Almeida — CAPI testing
7. Robert Luby — Console
8. David Simansky — Serverless
9. Pranshu Srivastava — In-cluster Monitoring / C/MAPI
10. Matej Vasek — Serverless

Anyone unavailable this week (PTO, leave, other)? (comma-separated numbers, or 'no' if everyone is available)
```

For each person marked unavailable, ask for the reason:
> "Reason for [Name]? (PTO / other — briefly describe)"

- Engineers marked PTO appear in the tracker Engineering table with "PTO" in Notes so `/rit-triage` skips them.
- Engineers marked with a custom reason (e.g. "parental leave", "training") appear with that reason in Notes.
- Engineers excluded entirely (not PTO, just not participating) are removed from the team altogether and do not appear in the tracker.

Ask: "Should they be marked as unavailable (still listed in tracker, skipped by triage) or excluded entirely (not listed)?"

### Step 6: Query initial workload

For each available engineer (not PTO/excluded), run a Jira query to count their total open bugs: `assignee = "<accountId>" AND project = OCPBUGS AND statusCategory != done`. Store the count as `total_open` — this populates the "Total Open" column in the tracker so the first triage run starts with an accurate capacity picture.

### Step 6b: Confirm capacity mode

After querying workloads, ask:

> "Capacity mode for this rotation?
> - **baseline** (recommended): current open bugs are informational only — soft/hard limits apply to RIT-assigned bugs only. Use when engineers carry pre-existing load from prior rotations.
> - **total**: soft/hard limits apply to each engineer's total open bug count (RIT-assigned + pre-existing).
> (baseline/total, default: baseline)"

Record the chosen mode. It will be stored in the tracker and read by `/rit-triage` each run.

### Step 7: Create tracker file

Create `triaged_bugs_YYYY-MM-DD.md` in the **current working directory** using the week start date.

If a file with that name already exists, ask: "triaged_bugs_YYYY-MM-DD.md already exists. Overwrite? (yes/no)"

File contents:

```markdown
# RIT Triage Tracker — Week of YYYY-MM-DD (Pod Name(s))

## RIT Lead

| Role | Name | Email | Jira Account ID |
|------|------|-------|-----------------|
| RIT Lead | Name | email | jira-id |

## Engineering

| Name | Email | Jira Account ID | Area of expertise | Notes |
|------|-------|-----------------|-------------------|-------|
| Name | email | jira-id | expertise |  |

## Assignment Distribution

Soft limit: 6 | Hard limit: 8 | Capacity mode: baseline

Team bandwidth: X% (0/M RIT capacity) | A at limit | B over | C available

| Engineer | Total Open (baseline) | RIT Assigned | Keys |
|----------|-----------------------|--------------|------|
| Name | X | 0 |  |

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

- Sort engineers alphabetically by last name in the Engineering table.
- List **only available engineers** (not PTO/unavailable) in the Assignment Distribution table, each starting at 0 bugs, sorted alphabetically by last name.
- Include PTO/unavailable engineers in the Engineering table (so they're visible in the roster) but **not** in the Assignment Distribution table (so `/rit-triage` doesn't assign to them).

### Step 8: Confirm

Show a final summary:
```
RIT rotation set up for week of YYYY-MM-DD (Pod Name(s))

RIT Lead: Name
Available (N): Name, Name, Name, ...
Unavailable (M): Name (PTO), Name (parental leave), ...

File created: triaged_bugs_YYYY-MM-DD.md
```

---

## Important Notes

- **Lead selection drives the team** — the selected lead's pods determine which engineers are on rotation. No manual cherry-picking from unrelated pods.
- **Do not modify the People Directory** — it is a static reference. Only the tracker file is written by this skill.
- **Do not modify `rit_manual.md`** — the "Current RIT Rotation" section in the manual is just a pointer telling people to run `/rit-start`. All roster data lives in the tracker file.
- **Paths use the current working directory** — never hardcode paths.
- **The tracker file is gitignored** — it is local only and not committed.
- **PTO/unavailable engineers appear in the Engineering table but not in Assignment Distribution** — `/rit-triage` reads the Assignment Distribution table to decide who gets bugs, so omitting them there is sufficient.
- **Total Open is queried from Jira at setup** — the initial workload snapshot gives the first triage run a reference picture. `/rit-triage` re-queries each run to stay current.
- **Capacity mode is set once per rotation** — `baseline` (default) counts only RIT-assigned bugs against limits; `total` counts all open bugs. The choice is stored in the tracker header line and read by `/rit-triage` each run. In baseline mode the "Total Open" column header reads "Total Open (baseline)" and no ⚠️/🛑 indicators are shown there.
- **Missing email or Jira Account ID (`—`)** — if a selected engineer or the chosen lead has `—` in either field, stop and ask the user to fill those values in the People Directory before creating the tracker. The tracker cannot be created with `—` in those columns because `/rit-triage` uses them for Jira assignment.
- **Soft/hard limits are read from Shared Constants** — do not hardcode the values in the tracker template. Read `SOFT_LIMIT` and `HARD_LIMIT` from `rit_manual.md` at runtime.
