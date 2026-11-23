# Dash patches to adk-go

This is `google.golang.org/adk/v2` **v2.5.0**, the version Dash requires, plus
two patches. Dash pins it with

    replace google.golang.org/adk/v2 => github.com/dashlytix/adk-go/v2 v2.5.0-dash.2

so every Dash package still imports `google.golang.org/adk/v2/...`.

## Patch 1: thought signature on aggregated text

`internal/llminternal/stream_aggregator.go`: Gemini 3 ends a streamed text
answer with an empty chunk carrying only the thought signature. The aggregator
appended it as a part with a signature and no data, and Vertex rejects any
later request that replays that part beside inline document data (400
INVALID_ARGUMENT). The signature is now attached to the buffered text part.
The `gemini-3.1-pro-preview` `TestParallelFunctionCalls` recording is
re-recorded, because its expected response encoded the bare part.

    go test ./internal/llminternal/

## Patch 2: plugins see the error the runner yields

`plugin.Config.OnRunErrorCallback`: called with the exact error before
`Runner.Run` or `Runner.RunLive` yields it to the caller. `AfterRunCallback`
gets no error, so before this a plugin could not tell a failed run from a
clean one. Dash's turn-outcome plugin (DAS-862) uses it to record `run_error`.
This is google/adk-go#1892 (issue #1883), squashed onto v2.5.0. Its test
helper `beforeRunSessionService` is copied into `run_error_test.go`, because
v2.5.0 has no `before_run_test.go`.

    go test ./runner/ ./plugin/...

## Upstream

Patch 1: the same commit against upstream `main` is on branch
`fix/stream-aggregator-text-thought-signature`. Patch 2: google/adk-go#1892,
open. Drop this fork and the `replace` line once a release contains both.
