# Dash patches to adk-go

This is `google.golang.org/adk/v2` **v2.5.0**, the version Dash requires, plus
one commit. Dash pins it with

    replace google.golang.org/adk/v2 => github.com/dashlytix/adk-go/v2 v2.5.0-dash.1

so every Dash package still imports `google.golang.org/adk/v2/...`.

## The patch

`internal/llminternal/stream_aggregator.go`: Gemini 3 ends a streamed text
answer with an empty chunk carrying only the thought signature. The aggregator
appended it as a part with a signature and no data, and Vertex rejects any
later request that replays that part beside inline document data (400
INVALID_ARGUMENT). The signature is now attached to the buffered text part.
The `gemini-3.1-pro-preview` `TestParallelFunctionCalls` recording is
re-recorded, because its expected response encoded the bare part.

    go test ./internal/llminternal/

## Upstream

The same commit against upstream `main` is on branch
`fix/stream-aggregator-text-thought-signature`. Drop this fork and the
`replace` line once a release contains it.
