# Test function size: before and after

Example for the Test function size section in `guidelines.md` — see there for the principle.

Same behavior under test: a function that processes an uploaded policy file and writes it to the DB must drop rows whose currency isn't ARS.

## Before — mocked storage, change detector, DRY

```go
func (s *TestSuite) TestOnUploadShouldDropNonARSRows() {
	// Storage is mocked and the test is a change detector: rename InsertPolicy
	// to UpsertPolicy and it breaks, even though the behavior is the same.
	s.storage.EXPECT().
		InsertPolicy(gomock.Any(), s.matchPolicy("1", money.ARS)).
		Return(nil)
	s.storage.EXPECT().
		InsertPolicy(gomock.Any(), s.matchPolicy("2", money.USD)).
		Times(0)

	// Too DRY: how is the file built and uploaded? What are "1" and "2"?
	s.processFileWith(map[string]string{"1": "ARS", "2": "USD"})
}
```

## After — real DB, observable outcome, DAMP

```go
func (s *TestSuite) TestWhenProcessingFileShouldDropNonARSRows() {
	arsPolicyRow := newRow(withPolicyNumber("1"), withCurrency(money.ARS))
	usdPolicyRow := newRow(withPolicyNumber("2"), withCurrency(money.USD))
	file := s.newFile(arsPolicyRow, usdPolicyRow)

	err := s.PolicyProvider.ProcessFile(file)
	s.Require().NoError(err)

	_, err = s.PolicyProvider.GetPolicyByNumber(arsPolicyRow.PolicyNumber)
	s.NoError(err, "ARS policy should be processed")

	_, err = s.PolicyProvider.GetPolicyByNumber(usdPolicyRow.PolicyNumber)
	s.ErrorIs(err, ErrNotFound, "USD policy should not be processed")
}
```

What changed:

- **Storage is real.** The assertion is "the ARS policy exists and the USD one doesn't", which survives any refactor of how rows get persisted.
- **The case is visible.** `newRow(withPolicyNumber("1"), withCurrency(money.ARS))` states exactly what makes this row different from its neighbor; the default-row constructor hides every field that doesn't matter here.
- **Plumbing is hidden, intent isn't.** Building the file and running the processor are one line each; the body still reads as a linear arrange → act → assert.
