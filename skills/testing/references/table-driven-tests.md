# Table driven tests

Example for the Table driven tests section in `guidelines.md` — see there for when to use this and why a map over a slice.

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            assert.Equal(t, tc.want, got)
        })
    }
}
```

## Inside a testify suite

Table tests inside a suite method use `s.Run` instead of `t.Run`. This also covers tables whose shared arrange is expensive (uploads, mocks, consent): do it once before the loop, then iterate the cases.

The one case where a slice beats a map: cases that are an ordered progression (e.g. advancing a fake clock through the stages of a calendar and asserting what was sent at each one). Order is the point there, so a slice is correct.

## Anti-pattern: slice-keyed tables

Don't use the slice form for independent cases — the case name lives inside the struct, so a case can only be identified by reading its fields, and nothing stops two cases from sharing the same name.

```go
// AVOID: name is a struct field, order is significant, easy to duplicate names
tests := []struct {
    name  string
    input string
    sep   string
    want  []string
}{
    {name: "simple", input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
    {name: "no sep", input: "abc", sep: "/", want: []string{"abc"}},
}

for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) { ... })
}
```

Prefer the map form above: the key *is* the case name, duplicate literal keys are a compile error, and there's no `name` field to thread through the struct.
