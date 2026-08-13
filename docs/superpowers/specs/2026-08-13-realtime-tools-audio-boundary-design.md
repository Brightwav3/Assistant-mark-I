# Realtime Tools and Audio Boundary Design

**Status:** Design for review

**Date:** 2026-08-13

## Goal

Add the smallest Mark I vertical connection between the existing Tool System
and the native realtime conversation path while making microphone framing and
latency diagnostics consistent. Realtime Core remains provider-independent,
Assistant Runtime remains the only cross-core coordinator, and no host tool is
enabled by default merely because this integration exists.

## Scope

1. Add a provider-neutral realtime tool declaration, tool request event, and
   tool result submission method.
2. Translate those contracts inside the Gemini adapter using the Live API's
   function declaration, tool call, and function response messages.
3. Let `RealtimeCoreAdapter` discover and execute tools through an injected
   runtime-owned `RealtimeToolExecutor`.
4. Add a Tool System-backed executor adapter without duplicating validation,
   policy, guards, broker access, or outcome classification.
5. Standardize microphone input at 16 kHz, mono, PCM16, 20 ms frames and bound
   pre-connect audio to the newest 500 ms.
6. Add timestamps and aggregated input, tool, and playback trace metrics.
7. Add deterministic fake-provider and Tool System integration tests plus a
   documented, explicitly injected real-device Calculator smoke path.

## Non-goals

- Realtime Core does not import `tool-system`, `host-tools`, or any model
  contract.
- `createAssistantRuntime()` does not silently construct or enable an
  `open_app` capability. A caller must inject a `RealtimeToolExecutor`.
- No native audio backend, `DuplexAudioDevice`, AEC, DSP stack, WebRTC,
  Interaction Core, or playback-controller consolidation is part of this
  change.
- No automatic session reopening after provider closure.

## Repository ownership

### `speech-system/realtime core`

Owns public realtime contracts, the Gemini protocol translation, the fake
provider, and provider-neutral event timestamps. It stores the tool name for
each provider call so `sendToolResult({ callId, content, isError })` can create
the provider-specific response without exposing Gemini types.

The public contract uses a standard JSON Schema object in
`RealtimeToolDeclaration.inputSchema`. The adapter passes it to Gemini as
`parametersJsonSchema`; no Gemini field appears in `contracts.ts`.

### `assistant-runtime`

Owns `RealtimeToolExecutor`, tool discovery/execution orchestration, audio
frameization, bounded pending input, activity policy, and trace aggregation.
`ToolSystemRealtimeToolExecutor` maps the existing `ToolRuntime` declarations
and reports back its typed outcomes. All validation, policy, guards, broker
access, taint handling, and error classification remain in Tool System.

### Root meta-repository

Owns this cross-repository design and records the two submodule pointer changes
after the child repositories are verified. `tool-system` and `host-tools` are
not modified by this design.

## Public contracts

Realtime Core adds the following shapes:

```ts
export interface RealtimeToolDeclaration {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

export interface RealtimeToolResult {
  callId: string;
  content: string;
  isError?: boolean;
}
```

`RealtimeSessionConfig.tools?: RealtimeToolDeclaration[]` supplies the
declarations at connection time. `RealtimeSpeechSession` adds:

```ts
sendToolResult(result: RealtimeToolResult): Promise<void>;
```

Every `RealtimeSpeechEvent` carries `timestampMs: number`. The new event is:

```ts
{
  type: 'tool.requested';
  sessionId: string;
  callId: string;
  tool: string;
  arguments: Record<string, unknown>;
  timestampMs: number;
}
```

The existing event names and interruption authority rules remain unchanged.

Assistant Runtime adds:

```ts
export interface RealtimeToolExecutor {
  discover(): Promise<RealtimeToolDeclaration[]>;
  execute(input: {
    callId: string;
    tool: string;
    arguments: Record<string, unknown>;
    signal?: AbortSignal;
  }): Promise<{ content: string; isError?: boolean }>;
}
```

`RealtimeCoreAdapter` accepts this executor optionally. When present, its
discovery result becomes the realtime connection's tool declaration list.

## Data flow

```text
ToolRuntime
    -> ToolSystemRealtimeToolExecutor.discover()
    -> RealtimeToolDeclaration[]
    -> RealtimeCoreAdapter
    -> RealtimeSessionConfig.tools
    -> Gemini functionDeclarations

Gemini toolCall.functionCalls
    -> GeminiLiveSession
    -> RealtimeSpeechEvent(tool.requested)
    -> RealtimeCoreAdapter
    -> ToolSystemRealtimeToolExecutor.execute()
    -> RealtimeSpeechSession.sendToolResult()
    -> Gemini functionResponses
```

For each request the adapter passes the interaction `AbortSignal`. If the
interaction is cancelled, it does not send a late result to a closed session.
If discovery or execution fails, the adapter records a structured trace and
sends an error result when the session is still open, allowing the model to
recover without a hanging function call.

The Gemini adapter maps `FunctionCall.id` to `callId` and retains the call's
function name privately. A successful result becomes
`response: { result: content }`; an error becomes `response: { error: content }`.
Unknown result IDs are rejected locally with a structured realtime error.

## Audio framing and activity

Realtime Core exports one immutable input format constant:

```ts
{
  sampleRate: 16_000,
  channels: 1,
  sampleFormat: 'pcm_s16le',
  frameDurationMs: 20,
}
```

`RealtimeCoreAdapter.sendMicrophonePcm()` accepts arbitrary capture chunks but
splits and carries them into 320-sample frames before sending. The pending
pre-connect queue stores complete 20 ms frames only, keeps the newest 500 ms,
and drops the oldest frames when the bound is exceeded. Dropped-frame counts
and buffered duration are reported in aggregated trace entries.

Runtime activity is driven first by provider-neutral events: input speech
start, final input transcript, output audio start, tool request, and completed
tool execution. Raw amplitude remains a fallback only for providers that do not
produce input speech activity; it is not the primary VAD.

## Observability

All lifecycle, transcript, audio, interruption, tool, and session events carry
the event creation timestamp. The runtime emits bounded aggregate metrics
instead of logging every PCM frame:

- input frames sent, dropped, and currently buffered milliseconds;
- first output chunk, bytes written, chunks written, abort requested/completed,
  and playback duration;
- tool requested, completed, failed, and cancelled with call correlation.

Argument values and API secrets remain redacted. Tool result content is traced
only through the existing Tool System outcome rendering rules.

## Testing strategy

### Realtime Core

- Verify the immutable 20 ms input format and timestamp presence.
- Verify Gemini declarations become function declarations with JSON Schema.
- Feed a provider-shaped tool call to the deterministic Gemini harness and
  assert the neutral `tool.requested` event.
- Submit success and error results and assert the exact Gemini response shape.
- Verify fake sessions support a deterministic tool-call/response turn.
- Preserve existing interruption and stale-output tests.

### Assistant Runtime

- Verify a 100 ms capture chunk becomes five 20 ms `AudioFrame`s.
- Verify the pending queue keeps at most 500 ms and reports dropped frames.
- Verify `ToolSystemRealtimeToolExecutor` discovers declarations and executes
  through a real `ToolRuntime` with a stubbed process broker.
- Verify a fake realtime tool request opens the catalogued Calculator through
  the Tool System and returns the result to the session.
- Verify cancellation and executor failure do not launch a host process or
  write a result into a closed session.
- Verify aggregated playback and input traces preserve correlation IDs.

### Manual hardware smoke

The repository will document a probe that injects a Tool Runtime whose
`open_app` catalog maps `calculator` to the local Calculator executable and
whose process broker is explicitly allowlisted. The operator starts it only
with `GEMINI_API_KEY` in the current process environment, says “Open
Calculator”, and verifies the process launch plus the spoken response. This
path is not part of the offline test suite and is not enabled by default.

## Definition of done for this design

- Both child repositories expose the contracts above without cross-core imports
  from Realtime Core into Tool System.
- Gemini-specific function-calling fields remain inside `gemini.ts`.
- The adapter executes tools through the existing Tool System and propagates
  cancellation.
- Input frames, pending audio, timestamps, and metrics are covered by tests.
- Offline verification passes for both child repositories.
- The manual hardware procedure is documented without storing credentials.
- Root submodule pointers reference the verified child commits.
