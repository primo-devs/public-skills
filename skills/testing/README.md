# testing

Workflow for deciding *what* to test (equivalence classes, "which edit breaks this test", mutation check) plus the guidelines for what a good test looks like: mostly integration, never mock storage, DAMP over DRY, short bodies, table-driven only when the body is identical.

Written for a Go + Postgres monolith served over HTTP. The workflow and guidelines transfer to other stacks; the mechanics (testify, gomock, file layout) are Go-specific.

## Layout

- `SKILL.md` — the workflow: read → plan (partition inputs, justify each test) → write → run → break the code to verify → self-check.
- `references/guidelines.md` — what a good test looks like: test pyramid, where to mock, anti-patterns, conventions.
- `references/failure-modes.md` — checklist to scan when planning failure cases.
- `references/function-size-examples.md` — the same test before and after the guidelines.
- `references/table-driven-tests.md` — the map-keyed table pattern.
- `references/gomock.md` — generating and wiring mocks.

## Golden standard suites

The skill tells agents to verify any existing test they take as reference against a short list of exemplary suites — existing tests that predate the guidelines are not a reference. Keep one exemplar per kind of test in your repo and point the skill at them (e.g. from your `AGENTS.md`):

| Kind of test | Exemplar |
| --- | --- |
| Client suite — behavior gated by a client's settings or input formats, seeded through the production entry point | `integrationtests/<client>_test.go` |
| Feature suite — behavior that isn't specific to a client, organized around the feature | `integrationtests/<feature>_test.go` |
| Package-level real-DB test — well-encapsulated behavior inside one package, real storage, no HTTP | `internal/<pkg>/<behavior>_test.go` |
| Pure function unit test — no dependencies, table-driven over the input space | `pkg/<pkg>/<pkg>_test.go` |

Replace the paths with your own; the kinds are what matter.
