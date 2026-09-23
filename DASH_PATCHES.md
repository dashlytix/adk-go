# Dash patches to adk-go

This is `google.golang.org/adk/v2` **v2.2.0**, the version `go.mod` already
required, copied here so one bug can be fixed. `go.mod` carries

    replace google.golang.org/adk/v2 => ./third_party/adk-go

so nothing imports a different path: every Dash package still imports
`google.golang.org/adk/v2/...`. Upstream's `LICENSE` (Apache 2.0) is kept as it
shipped; the release has no `NOTICE` file.

## The patch

One file: `internal/llminternal/stream_aggregator.go`, plus its tests in
`internal/llminternal/stream_aggregator_test.go`. Nothing else in the tree is
modified — `diff -rq` against the pristine module reports exactly those two
files.

Vertex streams a function call's arguments as `partialArgs` chunks, each
addressing one leaf with an RFC 9535 JSON Path (`$.tables[0]`,
`$.props.data[0].month`). `setValueByJSONPath` split that path on `.` alone and
wrote the last segment as a map key, so `$.tables[0]` became the literal key
`"tables[0]"` and `$.data[0].label` became `{"data[0]": {"label": …}}`;
`getValueFromPartialArg` walked the same split path to find the string a chunk
was continuing, so a string split across chunks inside an array restarted at
every chunk. Every Dash tool takes an array, so every streamed call arrived
corrupt, failed its own schema, and ended the turn in
`MALFORMED_FUNCTION_CALL`.

The fix parses the path into segments — `.name`, `['name']`, `["name"]`,
`[index]`, the selectors a normalized path is built from — writes through a
traversal that creates maps for names and grows `[]any` for indices, and uses
that same traversal for the string-continuation lookup. A path outside that
grammar (a wildcard, a slice, or Vertex's own value-less `$.tables[]`
array terminator) is dropped rather than written to an invented key. No regexp,
no new dependency.

Run the tests with:

    cd third_party/adk-go && go test ./internal/llminternal/

This directory is a separate Go module, so the repository's `go build ./...`,
`go test ./...` and `golangci-lint run ./...` do not descend into it.

## Upstream

The identical patch and tests are committed against current upstream `main`
(`5ff8228`, 22 September 2026, where the bug is unchanged) on branch
`fix/stream-aggregator-array-json-path`, with `PR_DESCRIPTION.md` beside it.
Not yet pushed.

`docs/deferred/streamed-function-call-arguments.md` is the follow-up that
deletes this directory and the `replace` line once the fix is in a release.

## What was pruned from the copy, and why that is safe

- `examples/` (1.3 MB) — standalone `package main` programs. Nothing imports
  them, and as a separate module they are never built by the Dash tree.
- `.github/` (32 KB) — upstream's own CI workflows. GitHub reads workflows only
  from the repository root, so these never ran here.

Everything else is kept, including all `*_test.go` files (so the patched
package's own tests run as upstream would run them) and
`cmd/launcher/web/webui/distr` (8.2 MB of embedded web-UI assets — Dash imports
`cmd/launcher/web/webui`, whose `go:embed` needs them). 14 MB, 488 Go files.
