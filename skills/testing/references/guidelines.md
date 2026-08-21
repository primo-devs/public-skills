# Testing guidelines

What a good test looks like. The `testing` skill is the workflow for deciding *what* to test; this is the *how it should look* reference it points to.

These guidelines were written for a Go monolith backed by Postgres and served over HTTP. The principles transfer; the mechanics (testify, gomock, file layout) are Go-specific.

## Test type pyramid

Philosophy: "Write tests. Not too many. Mostly integration."

Our goal is catching regressions that matter, not achieving raw coverage. Thus we prefer integration tests, and we do not test every change.

- **Integration tests**: The bulk of the behaviour should be verified here. Real DB, real managers, mocking only external dependencies at the outermost level that is ergonomic. Suites embed a shared base suite that wires the whole app — every domain manager, background jobs, event bus, and an HTTP router mounted at prod paths — so a test exercises the system the way production does.
- **Unit tests with storage**: Package-level tests for well encapsulated behaviour that doesn't cross between packages. Should still use a real DB (a base suite that hands you storage + a fake clock) and real managers. Assert outputs and invariants, not call sequences.
- **Unit tests with mock dependencies or without dependencies**: For hard-to-reach cases or pure functions. Never mock storage (see Anti-patterns); mocking any other dependency we own should be rare and well justified.

### Suites run in parallel

Suite state set in `SetupTest` must be per-test, so tests are race-free under parallelism. A new suite runs in parallel unless there is a concrete reason not to, stated in a comment. Stock `testify/suite` shares one suite value across test methods, so this needs a thin `suite.Run` wrapper that clones the suite per test before `SetupTest`; without one, keep suites sequential.

**Small base suite vs full-app suite** — for the common case of a manager test whose manager depends on other managers: the small base suite gives you only storage + clock, so you construct the manager under test and its dependency managers yourself. `nil` is acceptable only for collaborators the exercised code paths can never reach; if the manager under test may actually call a dependency, wire a real manager or noop fake — never `nil` (see the anti-pattern below). Reach for it when that hand-wiring stays small and the test lives in the manager's own package. Switch to the full-app suite when you'd end up rebuilding most of the app anyway — you need managers already wired, background jobs, event bus, file-upload seeding, or HTTP routes.

### Database in tests: clone, don't migrate

Tests never run migrations themselves. Each package with DB tests builds **one** template database (migrations run once, in `TestMain`) and every test clones it. Cloning a Postgres template database is instant, so **speed is never a reason to mock storage**.

### Golden standard

Keep a short list of exemplary suites, one per kind of test (client suite, feature suite, package-level real-DB suite, pure-function unit test), and point agents at it. Existing tests that predate the guidelines are not a reference; the golden list is. The list belongs in the consuming repo (e.g. its `AGENTS.md`); this skill's README describes the kinds to cover.

### Where to draw the boundary for integration tests

- Use real storage and managers.
- Use a fake for infrastructure-style dependencies when a fake exists, such as file storage.
- Mock external systems we don't control.
    - For each external service we have a client, mock the whole client, not the HTTP restclient.
- **Never mock managers** — not even managers that are dependencies of the manager under test. The boundary is ownership, not depth: mock only what we don't control.
- **Never mock storage** — see the anti-pattern below for the rule and its single exception.
- Hand-write only fakes. A *fake* is a working in-memory implementation. A *mock* must be generated with `gomock`. If the interface has no generated mock, add a directive and generate it. See `gomock.md`.
    - Fakes follow the same placement rule as any test helper: a test file in the package that uses them, or a shared `internal/test` package when used across packages. Generated mocks live in `mocks/` next to the interface.

### What to unit test

The things that are not covered by integration tests.

- Interaction between a client and the http responses it expects from a server
- Scrapers
- Well defined and encapsulated behaviour

## Anti-patterns

Common mistakes — each with the right alternative.

- **Don't create a new suite or test file when you can extend an existing one**.
- **No change-detector tests**: don't assert *which* manager/storage methods were called (mock `.EXPECT()` choreography wired to match today's code) — they fail on every refactor and catch no real bug. Assert observable behavior through the public surface: return value, persisted row, emitted event, HTTP response. Reference: Google's "Change-Detector Tests Considered Harmful", testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html.
- **Never mock storage**: no test mocks a package's `Storage` port to supply data or to verify a persist happened. If the logic under test only needs data, make it a pure function that takes the domain types it needs and test that; if it needs persistence, seed and assert against the real test DB (template cloning is instant, so speed is not a reason). The single exception: deterministically simulating a storage **failure** that a real DB can't reproduce on demand (e.g. an auth middleware falling through when the user lookup errors). Even then, assert the observable reaction (HTTP status, fallthrough, retry), never the call choreography. Existing suites that mock storage for data predate this rule — don't take them as reference, and don't extend them with new storage-mock expectations.
- **Avoid test-only methods in production code** — put them in test files or the shared test package. The one exception is storage: to keep **raw SQL out of tests**, test-only queries live in the storage layer prefixed `Test`. The prefix flags them when reading production code — never call them from prod paths; their efficiency doesn't matter.
- **Never use `time.Sleep` for synchronization** — use channels, `sync` primitives, `suite.Eventually`, `synctest`.
- **Never wait on production wall-clock timers** (debounces, poll intervals, backoffs): make the duration injectable (functional option or config field, production default unchanged) and set it to ~0 in tests, driving time with the fake clock. If the code path has no such override, add one — a hardcoded production duration inside a test is a testability bug in the production code.
- Test Setup
    - **Wire the system under test in the suite's `SetupTest()`, not in individual tests or act helpers**. The router, its managers, and its routes — mounted at the same paths as prod — belong in `SetupTest`, built once and stored on the suite. Act helpers only exercise the already-wired system.
    - **Never pass `nil` for manager dependencies in tests** — use a real manager, noop fake, or generated gomock mock. `nil` silently panics the first time a code path reaches it.

## Conventions

Standards every test follows.

- **Tests must earn their place**: justify them per behavior, not per PR. A test must protect observable behavior against a plausible and relevant regression that no existing test catches. Skip tests whose sole value is locking down constants or configuration values, trivial wiring or pass-throughs, library behavior, or the current implementation. Apply this filter independently to each behavior, even within a large feature.
- **Every prod bugfix needs regression evidence**. Add an automated regression test when it clears the value bar above: it fails on main, passes with the fix, and protects against a plausible and relevant recurrence. Otherwise, explain why the test would be low-value and use the cheapest reliable evidence instead: existing coverage, a build/lint check, or targeted smoke/manual verification. The absence of a new test is not by itself a review finding.
- **No order-dependent asserts on unordered results** (use `ElementsMatch`); **no conditional assertions or skips** inside tests (an `if` that may not run means the branch is untested).
- **Tests never read live configuration** (mock it) and **never mutate package-level state** (breaks `t.Parallel`; expose functional options instead).
- **No production PII in fixtures**: anonymize names, emails, phones.

### Mechanics

- **Assertions**: testify `require`/`assert`, never raw `t.Fatalf`/`t.Errorf` value comparisons.
- **Setup errors**: `require.NoError` always — never ignore an error.
- **Fixed clock with a fixed date** — never `time.Now()` or real dates.
- **Testdata**: `go:embed`.
- **One file = one suite** (a suite may be shared by multiple files).

### Test file location

1. Integration tests go to a dedicated top-level package (e.g. `integrationtests/`).
2. When unit testing a package level behaviour in file `foo_ex.go` for package `foo`, the test goes in the same directory:
    - For exported behaviour, the test file is `foo_ex_test.go` with package `foo_test`.
    - For unexported behaviour, the test file is `foo_ex_internal_test.go` with package `foo` (whitebox test).

    You should try to test the exported surface of a package, but don't export internal details just to test them. In those exceptions it's OK to use an internal test.

### Test function size

The body of a test should be legible and short. It should fit inside a screen, at most ~45 lines but try to aim for 25 to 30. Abstract the implementation details (building inputs, driving the system, fetching results) behind auxiliary functions so the body reads as the intent: set up this state, run this action, assert this outcome. Don't over-abstract either: the body should still read as a linear sequence of steps, not a deep chain of helpers you have to dive into to know what is tested. Prefer a little duplication over the wrong abstraction.

Test code is optimized differently than production code: production code is DRY because it has tests to catch mistakes; a test is its own documentation and has nothing testing it, so it's optimized to be understood on its own — DAMP, Descriptive And Meaningful Phrases ([Tests Too DRY? Make Them DAMP!](https://testing.googleblog.com/2019/12/testing-on-toilet-tests-too-dry-make.html)). DAMP and DRY draw the line for what a helper may hide: details **irrelevant** to the behavior under test (wiring, connections, required-but-unrelated fields) belong in helpers — that duplication carries no information. The inputs and expectations that **define the case** — what makes this test pass and its neighbor fail — stay visible in the body; a helper or loop that hides them forces the reader to reconstruct the case mentally, and a name like `setupUserWithInvalidEmail` is where per-case flags and conditionals start creeping into shared helpers.

For an example, see `function-size-examples.md`.

For the abstractions you create, first check an equivalent doesn't already exist. Helpers specific to one suite stay in that package's test files; helpers reusable across packages go in the shared test package.

### Seeding test data

**Discriminating fixtures**: distinct values in confusable fields (policy number vs policy key); identical values hide swapped-field bugs. Deviating from fixture defaults without need is a red flag.

Seed through the domain's aggregate-level API (one call that creates the entity with its related models), not by inserting entities separately. When a test needs many variants of the same input, build them procedurally: a default constructor that returns a complete valid value plus functional options (`withCurrency(USD)`) that override only the fields the test cares about.

### Asserting over data

**Assert full equality on the business outcome** (the whole row, prompt, or message), with **hardcoded expected values**: never compute the expectation with the same logic under test, never spot-check a couple of fields (a spot-check passes no matter what garbage the code extracts).

- Use `testify/require` for setup and `testify/assert` for assertions.
- Compare domain models by equality with a shared assertion helper rather than field by field.

### Table driven tests

Only use when you have multiple tests in which the only thing of their body that varies is the input. If you have to parametrize the setup, or have conditionals on the test body, it's not a good table driven test.

Key the table by a map, not a slice. The map key is the case name. Make sure there are no dependencies between cases. For the pattern in full, see `table-driven-tests.md`.

### Subtests with shared setup

Group checks into named subtests with `t.Run` (or `s.Run` inside a suite method) in two cases:

1. They form one conceptual group worth a shared parent name.
2. They share part of the setup — build it once at the top, then each subtest adds its own.

The subtest bodies must genuinely differ; if only the inputs vary, it's a table driven test.

## References

- `gomock.md` — directives, generating, and wiring mocks into suites
- `table-driven-tests.md` — the map-keyed table pattern
- `function-size-examples.md` — a well-factored test body vs. a mock-heavy one
- `failure-modes.md` — checklist of failure modes to scan when planning tests
