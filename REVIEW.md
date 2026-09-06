# Review: functional-option comparison defects

## Scope and evidence limits

Reviewed only the checked-out tree. The local Git log exposes a single snapshot
commit, not the introducing merge or subsequent correction. Consequently, this
review identifies defects present in the snapshot; it does not claim to establish
their historical attribution. No implementation files were changed.

## 1. Incorrect index in functional-option mismatch diagnostics

**Defective file:** `mock/mock.go`, `assertOpts`, lines 1290–1320.

The outer loop indexes the options with `i`. The inner loop at line 1316 also
declares `i`, this time indexing the option function's parameters. When parameter
values differ, lines 1318–1319 use that **parameter index** to access
`expectedNames` and `actualNames`, which contain one entry per **option**.

This is observable with ordinary single-parameter options: a mismatch in the
second option is described using the first option's name. With one option taking
two pointer parameters, a mismatch in its second parameter accesses
`expectedNames[1]` even though that slice has length one, causing an index-out-of-
range panic instead of returning the argument difference.

Minimal public-API reproduction (illustrative; not executed):

```go
func option(value int) func(*int, *int) {
    return func(first, second *int) { *second = value }
}

expected := mock.Arguments{mock.FunctionalOptions(option(1))}
actual := []interface{}{[]func(*int, *int){option(2)}}
_, differences := expected.Diff(actual) // Should report a mismatch, not panic.
```

Both closures originate from the same function, so the function-identity check
passes. The first parameter remains equal; the second differs and reaches the
out-of-range name lookup. `Arguments.Diff` delegates to `assertOpts` at line 1062
without a recovery around this branch. This affects mock matching and failure
reporting, not just an unused diagnostic helper.

**Coverage gap:** `mock/mock_test.go:1594–1676` tests functional-option matching
and mismatching, but the differing-value case at lines 1663–1676 uses only one
option and checks only that a panic occurs. It cannot distinguish a deliberate
unexpected-call panic from a panic inside mismatch formatting, nor detect an
incorrect option name. This is supporting test evidence, not a second defective
production file.

## 2. Zero-parameter options skip remaining comparisons

**Defective file:** `mock/mock.go`, `assertOpts`, lines 1302–1306.

Inside the option loop, `if ot.NumIn() == 0 { return }` returns from the entire
comparison with empty named results, rather than finishing only that option.
`Arguments.Diff:1062–1067` interprets these empty results as a successful match.
Thus equal option-list lengths and an identical first zero-parameter function
are sufficient to bypass checking every later option, including different
function identities.

Minimal public-API reproduction (illustrative; not executed):

```go
func first() {}
func wanted() {}
func unwanted() {}

expected := mock.Arguments{mock.FunctionalOptions(first, wanted)}
actual := []interface{}{[]func(){first, unwanted}}
_, differences := expected.Diff(actual) // Returns 0 despite the second mismatch.
```

This can silently accept an unexpected mock call. The implementation explicitly
handles zero-parameter functions, so the failure is in that handling, rather
than an inference about how pointer-parameter options should behave.

## Verification

Findings are based on source inspection and explicit control-flow traces above.
Attempted `GOPROXY=off GOSUMDB=off go test ./suite ./mock ./assert`, but the tool's
execution policy rejected the command before execution. No test success or
runtime reproduction is claimed. No network sources were consulted and no fix
was implemented.
