---
name: status
description: On-demand status report for the Deliberate pipeline across one or more repos. Lists in-flight issues, blocked-on-human questions, recent cost spend, and any anomalies the sweeper would surface. Run locally when you want a snapshot.
allowed-tools: Read Bash(gh *)
---

You are running `/status`. You produce a concise snapshot of every Deliberate-managed repo accessible to the current `gh` user, focused on what the user might need to act on.

## Input

Optional: a list of repos. Default: every repo where the Deliberate GitHub App is installed and the user has access (discoverable via `gh repo list`).

## Output structure

```
# Deliberate status — <ISO timestamp>

## Awaiting your input
For each blocked-on-human question:
- <repo> #<issue>: <title>
  Question: <one-line summary>
  Posted: <relative time>
  Link: <URL>

(or "Nothing awaiting your input." if the list is empty)

## In-flight
For each non-blocked, non-done issue:
- <repo> #<issue>: <title> — <stage> — cycle <n>

## Recently merged (last 7 days)
For each:
- <repo> #<issue>: <title> — merged <relative time>

## Cost (last 7 days)
- All projects total: $<X>
- Per project: <breakdown>
- vs daily cap: <pct used average>

## Anomalies
For each issue stalled >2hr, missing-event, or cost-cap-warning:
- <repo> #<issue>: <one-line description>
```

## Process

1. List repos with the App installed.
2. For each repo, query issues with the relevant labels via `gh issue list --label state:* --json ...`.
3. For each in-flight issue, read the pinned-state-comment for stage, cycle, cost.
4. Aggregate.

## What you do not do

- You do not advance any issue's state. Read-only.
- You do not post any comments. Read-only.
- You do not invoke any agent. Read-only.

## Output

The formatted report. Done.
