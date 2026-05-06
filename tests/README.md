# Tests

Two suites:

- `unit/` — unit tests for the orchestrator's state-comment parser, the trend-aware retry budget logic, the umbrella all-or-nothing checker, and any helper utilities that aren't pure prose.
- `e2e/` — full pipeline runs against fixture repos in `fixtures/`. CI seeds an issue, labels it `state:ready`, and asserts that the pipeline drives it to a green PR through the expected state transitions.

Test discipline is core to the project. Per [CONTRIBUTING.md](../CONTRIBUTING.md):

> We don't ship code we can't test. Adapters, agents, helpers, and skills all require an end-to-end test path before they ship.

## Status

Skeleton only. Test infrastructure lands in v0.1.

The first concrete e2e test will exercise the umbrella mechanics:

1. Seed a fixture repo with a `DESIGN.md`.
2. Invoke `ticket-groomer` directly (bypassing `/init-project`).
3. Assert: N child issues created, one umbrella issue, all labeled `state:proposed`.
4. Tick all umbrella boxes, close the umbrella.
5. Assert: every child gets `state:ready` within one event tick.
6. Tear down.

Fixture repos live under `tests/fixtures/<name>/` and are real git repos that the test harness clones into a workspace before each run.
