---
name: testing
description: "Write tests for code. Use when: (1) user asks to write, add, or improve tests; (2) after implementing non-trivial features—propose which tests would add value before writing; (3) user asks to verify a bugfix with a regression test; (4) user wants to understand what tests a change needs."
---

# Writing Tests

This skill is the workflow for deciding *what* to test and turning that into a suite.
Read `references/guidelines.md` before planning tests: it defines what a good test looks like.

## Workflow

### 1. Read the Code

- Read the code being tested thoroughly
- Look at existing tests. Extend suites / test files before creating new ones.

Existing tests are not automatically a reference: verify any test you take as an example against the guidelines and the golden-standard suites the repo lists (see the Golden standard section of the guidelines); if they diverge, the guidelines win. Ask the user only when the intended behavior is still unclear.

### 2. Plan: What are we testing?

Think of partitioning the input/state domain into undistinguishable categories (think of equivalence classes), which the system under test treats as:

- Happy paths: which kinds of inputs are supported?

    For example: a list of 2 elements and 3 elements can be classified under the same "non empty list of more than one element", which should be a single input case for a system under test that handles lists.

- Failures: Scan `references/failure-modes.md` for common failure modes to consider. For each one think of a input/state category that triggers it.

**Balance black-box and white-box.** Start black-box: partition by the contract — what should happen for each kind of input. Then read the implementation to refine the partition: it reveals edge cases the contract view misses, and cases the code treats identically (collapse those into one). The implementation is a source of test ideas, not the spec — use branches to find cases, but assert observable behavior only, so a refactor that preserves behavior never breaks the test. **Litmus test: if one realistic code edit could break input A but not input B, they belong to different classes.** (This is the flip side of "combine tests for the same failure mode" below.)

For each potential test, write **one sentence: why does it earn its place?** Name the edit that would break it, the code path, and why that edit is realistic — *"if someone makes `parseFilters` drop the `status = active` clause, this fails."* "Increases coverage", "good practice", and "the function exists" don't count. If you can't write the sentence, the test doesn't belong.

Also, check that:

1. **Nothing already catches it** — otherwise extend that test or make it a table row, don't add a new function.
2. **We own it** — it tests our logic, not that a library works (but DO test how our code handles library/API failures).

### Choose the kind of test

Read the Test type pyramid section in `references/guidelines.md` for the definition of each one

- Integration tests
- Unit tests with real storage and managers
- Unit tests with mock dependencies or without dependencies

Never mock storage: if you're reaching for a storage mock to feed data, make the function pure — it takes the domain types it needs — or use the real test DB. The only exception (storage-failure simulation) is in the Anti-patterns section of `references/guidelines.md`.

### Build two lists

**Tests to write:**

| Test                         | Type        | Why it earns its place — the edit that breaks it, and why that's realistic                            |
| ---------------------------- | ----------- | ----------------------------------------------------------------------------------------------------- |
| Discount skips certain items | Integration | If `applyDiscount()`'s switch stops excluding flagged items (business rules change often), this fails |
| Handler rejects empty input  | Integration | If `validate()`'s nil check is removed, an empty form submission slips through                        |
| Retry stops on fatal error   | Unit        | If `shouldRetry()` returns true on a fatal code (API changes error format), the retry loop never exits |

- **Type** should be **integration** by default. Unit tests should be clearly justified.
- **Why it earns its place** names the *specific* branch that breaks the test, not the construct around it: one independently-breakable branch = one row. One failure mode = one test; if you can't write the sentence, the test is too vague.

**Tests to skip:**

| Skip                       | Why                                            |
| -------------------------- | ---------------------------------------------- |
| `IsActive()`               | One-liner getter; caller tests cover it        |
| JSON parsing               | Library behavior—we just call `json.Unmarshal` |
| Token signature validation | JWT library, not our code                      |

### Red flags your plan has too many tests

- **Testing library behavior** — "token expiry" = JWT library, "password hash" = bcrypt
- **Vague justifications** — "critical boundary", "error-prone", "must work"—what _specific_ bug in _our_ code?
- **Testing one-liners** — trivial getters, pass-through calls
- **Multiple tests for the same failure mode** — combine into one or use table-driven
- **Unit tests where integration would work** — the philosophy says "mostly integration"
- **Inconsistent reasoning** — skipping `HasScope()` as "just `slices.Contains`" but testing scope validation separately

### The filtering mindset

**Look for reasons NOT to write each test.** If you can't find a strong reason to skip it, it probably belongs. But if you're stretching to justify it, skip it. It must protect observable behavior against a plausible and relevant regression that no existing test catches. Not every regression is worth automating: skip tests whose sole value is locking down constants or configuration values, trivial wiring or pass-throughs, library behavior, or the current implementation. Apply this filter independently to each behavior, even within a large feature.

### Risk slices (complex flows only)

For large features, group tests into slices to maintain budget discipline:

- Happy path + invariants
- Validation failures
- Idempotency / duplicates
- External dependency failures
- Authorization

Budget 1–3 tests per slice. If total exceeds ~8 tests, revisit the skip table.

## 3. Write

- Follow existing patterns and helpers in the codebase (see `references/guidelines.md`)
- **Test behavior, not implementation** — tests shouldn't break when you refactor internals
- **Test doubles**: hand-write only fakes, mocks should be generated with `gomock` — add a directive to the interface and regenerate if one is missing (see `references/gomock.md`)
- Assert observable outcomes: return values, persisted state, emitted side effects
- Avoid: generated IDs in assertions, `time.Sleep`, order dependence between tests

**Implementation must match the table.** No extra tests that weren't justified. If you discover a new case while writing, go back and justify it in the table first.

## 4. Run

```bash
go test ./path/to/package -run TestName   # focused first
go test ./...                              # full suite at end
```

## 5. Verify (when in doubt)

**Temporarily break the code** to verify the test catches it.

For at least one new test, introduce the bug it's meant to catch and confirm it fails. A test that always passes is worthless.

## 6. Self-Check

Before presenting:

- Every written test appears in the plan table.
- Every skipped obvious case has a reason.
- No anti-pattern from the Anti-patterns section of `references/guidelines.md` is present.
- For critical branches, you can name the test that would fail if the branch were deleted.
- Boundary conditions are covered or explicitly skipped: empty, null, max, off-by-one.
- Unit tests are justified over integration tests.
- Targeted test command passed.
- If using gomock, generated mocks are updated.
