# gomock

Reference for the boundary rule in `guidelines.md`: **never hand-write a mock — generate it with `gomock`.** (Fakes — working in-memory implementations — are hand-written; mocks are not.)

This covers the three things you do with a mock: add the directive, generate, and wire it into a test.

## 1. Add the `go:generate` directive

Mocks are generated from the **consumer-side interface** (the interface the consumer declares for a dependency). Put a `//go:generate` directive next to that interface, or grouped at the top of the file that owns it.

```go
//go:generate go tool mockgen -destination=mocks/mocks.go -package=enricher_mocks example.com/app/internal/enricher Storage,DistributionProvider,EmailEnqueuer
```

Anatomy:

- `go tool mockgen` — the repo's pinned mockgen (Go 1.24+ `tool` directive in `go.mod`, no separate install). On older Go, install `mockgen` and drop the `go tool` prefix.
- `-destination=mocks/mocks.go` — colocate generated mocks in a `mocks/` subpackage next to the interface. One `mocks.go` per package is the default; split into named files when interfaces come from different source packages.
- `-package=<pkg>_mocks` — the generated package name.
- Final args: the **import path** of the package that declares the interfaces, then a comma-separated list of interface names. Reuse one directive for every interface that lives in the same source package.

To mock a **third-party client** you don't own, point the directive at the external package and rename the generated type so it reads clearly at the call site:

```go
//go:generate go tool mockgen -destination=mocks/mp_payment_mocks.go -package=payment_mocks -mock_names=Client=MockPaymentClient github.com/mercadopago/sdk-go/pkg/payment Client
```

If the interface already has a directive, you don't add another — just regenerate.

## 2. Generate

```bash
go generate ./internal/{somepkg}/...
```

Commit the generated `mocks/` files alongside the interface change — they're checked in, and the generated mock must always match the current interface.

## 3. Use it in a test

Don't construct mocks inline in every test — that boilerplate (`NewController`, `NewMockX`, build the manager) repeats in each case and drifts. Put it in a **suite** so every test starts from a fresh controller and mocks, and each test only declares the expectations it cares about.

`gomock.NewController(s.T())` ties each mock's lifecycle to the test and auto-verifies that every `EXPECT()` was met when the test ends — no manual `defer ctrl.Finish()` needed.

```go
import (
    "example.com/app/internal/enricher/mocks"
    "github.com/stretchr/testify/suite"
    "go.uber.org/mock/gomock"
)

type PrioritizeSuite struct {
    suite.Suite

    ctrl     *gomock.Controller
    mockDist *mocks.MockDistributionProvider
}

func TestPrioritizeSuite(t *testing.T) {
    suite.Run(t, new(PrioritizeSuite))
}

func (s *PrioritizeSuite) SetupTest() {
    s.ctrl = gomock.NewController(s.T())
    s.mockDist = mocks.NewMockDistributionProvider(s.ctrl)
}

func (s *PrioritizeSuite) TestAboveThresholdEnrichedBelowFiltered() {
    // Only the expectations this test cares about. Match args with gomock.Any()
    // (or a concrete value), then Return.
    s.mockDist.EXPECT().GetDistribution(gomock.Any(), gomock.Any()).Return(arsDist, nil)

    mgr := NewManager(Config{...}, s.mockDist)

    result, err := mgr.prioritize(ctx, candidates)
    s.Require().NoError(err)
    // assert on result with s.Len / s.Equal / ...
}
```

Notes:

- **Match args** with `gomock.Any()` when the value is incidental, or a concrete value when it's the thing under test. Use `gomock.Eq(x)` for explicit equality.
- **Call count**: a bare `EXPECT()` expects exactly one call. Chain `.Times(n)`, `.AnyTimes()`, or `.MinTimes(n)` to relax that.
- **Don't over-specify.** Only set expectations for calls this test cares about. Every unmet `EXPECT()` fails the test, and every unexpected call fails too — that's the point, but it makes brittle tests if you mock plumbing you don't care about.
- **Parallel suites**: stock `testify/suite` shares one suite value across test methods, so per-test state in `SetupTest` races under `t.Parallel()`. If you run suites in parallel, use a wrapper that clones the suite per test before calling `SetupTest`.
- **DB-backed tests**: embed your real-DB base suite instead of `suite.Suite`; create the controller in your own `SetupTest` after calling the base `SetupTest()`. Remember the rule: mocks are for external systems, never for storage.
