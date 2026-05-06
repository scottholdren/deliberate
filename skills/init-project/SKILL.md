---
name: init-project
description: Inception flow for a new Deliberate-managed project. Runs gstack /office-hours, optional plan reviews, then dispatches the ticket-groomer to create an umbrella issue + child issues. Run locally; the rest of the pipeline runs in GitHub Actions.
disable-model-invocation: true
allowed-tools: Read Write Bash(gstack *) Bash(gh *) Bash(git *) Task
---

You are running `/init-project` for a new project. This is the only phase that runs in the user's local terminal; everything after the umbrella is approved runs in GitHub Actions.

## Preconditions

1. The user is in a git repository connected to a GitHub remote.
2. `gstack` is installed and accessible on the path.
3. `gh` is installed and authenticated.
4. The Deliberate GitHub App is installed on the repo (verify via `gh api /repos/<owner>/<name>/installation` or by checking the App's presence; surface any setup steps clearly if missing).

If any precondition fails, stop and tell the user exactly what to fix. Do not proceed.

## Phase 1: Office hours

Invoke `gstack office-hours`. This is conversational; the user participates. The output is a `DESIGN.md` (or similar — whatever gstack writes).

If `DESIGN.md` already exists from a prior run, ask the user whether to:
- Use the existing file as-is
- Re-run office-hours fresh
- Have you read the existing file and ask the user a few targeted refinement questions

## Phase 2: Plan reviews (optional, recommended)

Ask the user whether to run plan reviews. Default: yes for first project, ask for subsequent. The reviews are:

1. `gstack plan-ceo-review` — scope, ambition, what to cut, what to expand. Conversational; user input expected.
2. `gstack plan-eng-review` — architecture, edge cases, test coverage. Conversational.

After each review, the user may have edited `DESIGN.md`. Re-read it before proceeding.

If the user opts to skip plan reviews, proceed but log the decision.

## Phase 3: Ticket grooming

Invoke the `ticket-groomer` subagent via Task. Pass it:
- The path to the design document.
- The target repo (`<owner>/<name>`).

The groomer creates child issues and an umbrella issue, then returns URLs.

## Phase 4: Hand off

Print a final summary:
```
Inception complete.

Design: <path>
Umbrella: <umbrella URL>
Children: <count>

Next: review the umbrella. Tick all boxes and close it to start the pipeline.
The pipeline runs in GitHub Actions; you do not need to keep this terminal open.
```

## What you do not do

- You do not skip the human gate. Even if the design is brilliant, the user reviews the umbrella before any code is written.
- You do not start the pipeline. The pipeline starts when the user closes the umbrella with all boxes ticked.
- You do not invent acceptance criteria the design doesn't support. Surface ambiguities to the user during office-hours, not silently in the groomer.

## Failure modes

- gstack not installed → tell the user how to install gstack and exit.
- GitHub App not installed → tell the user how to install the App on the repo and exit.
- Office-hours produces a thin design → ask the user to spend more time on it before grooming.
