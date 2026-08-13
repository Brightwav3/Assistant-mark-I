# Realtime Tools and Audio Boundary Implementation Plan

> **Status: COMPLETE.** This plan is retained as the implementation record.
> The production default exposes only safe read-only tools; side-effecting
> tools remain explicit opt-ins.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Connect the existing Tool System to the native realtime path through provider-neutral contracts, consistent 20 ms microphone frames, bounded latency traces, and a safe read-only default catalogue without enabling side-effecting host tools by default.

**Architecture:** Realtime Core owns only provider-neutral tool declarations, tool requests/results, Gemini translation, fake-provider behavior, and timestamped events. Assistant Runtime owns the `RealtimeToolExecutor`, Tool System bridge, safe default catalogue wiring, input frameization, pending-audio bound, orchestration, cancellation, and aggregate traces. The root meta-repository records the verified child commits and does not modify `tool-system` or `host-tools`.

**Tech Stack:** TypeScript, Node.js 22+, `node:test`, `tsx`, `@google/genai`, existing `realtime-core`, `assistant-runtime`, and `tool-system` public APIs.

**Spec:** `docs/superpowers/specs/2026-08-13-realtime-tools-audio-boundary-design.md`

## Global Constraints

- Realtime Core does not import `tool-system`, `host-tools`, or any model contract.
- `createAssistantRuntime()` does not construct or enable side-effecting `open_app` unless a caller explicitly injects a `RealtimeToolExecutor`; it does expose the safe read-only Host Tools catalogue by default.
- The public input format is 16 kHz, mono, PCM16, 20 ms frames; one frame is exactly 320 samples.
- The pre-connect queue stores complete frames only and retains the newest 500 ms, at most 25 frames.
- Gemini-specific `functionDeclarations`, `toolCall`, and `functionResponses` fields remain inside `speech-system/realtime core/src/gemini.ts`.
- Tool validation, policy, guards, broker access, taint handling, and outcome classification remain inside `tool-system`.
- Tool arguments, API keys, and secrets are never copied into traces.
- No native audio backend, AEC, DSP, WebRTC, Interaction Core, playback-controller consolidation, or automatic session reopening is added.
- Tests remain offline and deterministic; credentialed hardware verification is documented separately and is never reported as passed without a real run.

## File Map

### Realtime Core

- Modify `speech-system/realtime core/src/contracts.ts` for the public tool contracts, immutable input format, event timestamps, and structured tool-result errors.
- Modify `speech-system/realtime core/src/gemini.ts` for Gemini declaration translation, tool-call events, private call-name correlation, and function responses.
- Modify `speech-system/realtime core/src/fake.ts` for deterministic tool-call/response turns and timestamped events.
- Modify `speech-system/realtime core/src/index.ts` to expose the input-format constant and public contracts through the existing package entry point.
- Modify `speech-system/realtime core/tests/gemini.test.ts` and `tests/runtime.test.ts` for contract, timestamp, declaration, response, fake-tool, and interruption coverage.

### Assistant Runtime

- Modify `assistant-runtime/src/contracts.ts` for the injectable `RealtimeToolExecutor` contract.
- Create `assistant-runtime/src/realtime-audio.ts` for pure 320-sample PCM frameization and the stable microphone stream ID.
- Modify `assistant-runtime/src/adapters.ts` for frameized input, the 500 ms pending bound, executor discovery/execution, cancellation, and aggregate traces.
- Modify `assistant-runtime/src/tool-bridge.ts` for `ToolSystemRealtimeToolExecutor` and standard JSON Schema translation while reusing Tool System outcome rendering.
- Modify `assistant-runtime/src/composition.ts` to pass an optional executor while keeping side-effecting host capabilities out of the default composition.
- Modify `assistant-runtime/src/index.ts` to expose the new runtime contract and adapter.
- Create `assistant-runtime/tests/realtime-audio.test.ts` and `tests/realtime-tools.test.ts`; extend `tests/audio-routing.test.ts`, `tests/session-lifecycle.test.ts`, and `tests/tool-bridge.test.ts` for regression and integration coverage.
- Create `assistant-runtime/tests/probe-realtime-tool.ts` as an explicitly injected, credentialed/manual-only Calculator probe.

### Documentation and root

- Modify each child repository's `README.md`, `ARCHITECTURE.md`, `WORKPLAN.md`, and `PROGRESS.md` only where the new boundary changes the documented contract.
- Create `assistant-runtime/docs/realtime-tools-smoke-test.md` with the explicit Calculator injection procedure and secret-handling rules.
- Modify `assistant-runtime/docs/hardware-smoke-test.md` to link the tool smoke procedure without claiming it was run.
- Update the root submodule pointers only after both child repositories pass their full `npm run verify` commands.

---

### Task 1: Realtime Core contracts and timestamp foundation

**Files:**
- Modify: `speech-system/realtime core/src/contracts.ts`
- Modify: `speech-system/realtime core/src/index.ts`
- Test: `speech-system/realtime core/tests/runtime.test.ts`

**Interfaces:**
- Produce `REALTIME_INPUT_FORMAT: Readonly<AudioFormat>` with `{ sampleRate: 16_000, channels: 1, sampleFormat: "pcm_s16le", frameDurationMs: 20 }`.
- Produce `RealtimeToolDeclaration`, `RealtimeToolResult`, `RealtimeSessionConfig.tools`, and `RealtimeSpeechSession.sendToolResult(result)`.
- Require `timestampMs: number` on every `RealtimeSpeechEvent` variant.
- Add `TOOL_CALL_UNKNOWN` to `RealtimeErrorCode` for locally rejected result IDs.

- [x] **Step 1: Write the failing timestamp and format tests**

Add tests that connect `FakeRealtimeSpeechProvider`, collect its lifecycle/input/output events, assert `Object.isFrozen(REALTIME_INPUT_FORMAT)`, assert the exact 16 kHz/mono/PCM16/20 ms values, and assert every collected event has a finite numeric `timestampMs`.

```ts
import { FakeRealtimeSpeechProvider, REALTIME_INPUT_FORMAT, RealtimeCore, type AudioFrame, type RealtimeSpeechEvent } from "../src/index.js";

test("exports one immutable 20 ms realtime input format and timestamps every event", async () => {
  assert.deepEqual(REALTIME_INPUT_FORMAT, {
    sampleRate: 16_000,
    channels: 1,
    sampleFormat: "pcm_s16le",
    frameDurationMs: 20,
  });
  assert.equal(Object.isFrozen(REALTIME_INPUT_FORMAT), true);
  const frame: AudioFrame = { streamId: "mic", timestampMs: Date.now(), format: REALTIME_INPUT_FORMAT, data: new Int16Array(320) };
  const session = await new RealtimeCore(new FakeRealtimeSpeechProvider()).connect({ provider: "fake", inputFormat: REALTIME_INPUT_FORMAT });
  const events = collect(session.events(), 4);
  await session.sendAudio(frame);
  const seen = await events;
  assert.ok(seen.every((event: RealtimeSpeechEvent) => Number.isFinite(event.timestampMs)));
  await session.close();
});
```

- [x] **Step 2: Run the focused test and verify the expected RED failure**

Run from `C:\Users\Sajmon\Jarvis\speech-system\realtime core`:

```powershell
npm test -- tests/runtime.test.ts
```

Expected: the test fails because `REALTIME_INPUT_FORMAT` and event timestamps are not yet present.

- [x] **Step 3: Implement the minimal public contract change**

In `contracts.ts`, add the readonly input constant, the two tool interfaces, optional `tools` on `RealtimeSessionConfig`, `sendToolResult` on `RealtimeSpeechSession`, `timestampMs` on every union member, and `TOOL_CALL_UNKNOWN`. Keep Gemini field names out of this file. Export the contract and constant through the existing `index.ts` wildcard.

- [x] **Step 4: Run the focused test and the existing realtime suite**

```powershell
npm test -- tests/runtime.test.ts
npm test
```

Expected: the new test passes; any remaining failures identify fake/Gemini/custom-session timestamp and method updates for the next task rather than a contract ambiguity.

- [x] **Step 5: Commit the contract foundation in the speech-system repository**

```powershell
git add -- "realtime core/src/contracts.ts" "realtime core/src/index.ts" "realtime core/tests/runtime.test.ts"
git commit -m "feat: define timestamped realtime tool contracts"
```

### Task 2: Gemini translation and deterministic fake provider

**Files:**
- Modify: `speech-system/realtime core/src/gemini.ts`
- Modify: `speech-system/realtime core/src/fake.ts`
- Test: `speech-system/realtime core/tests/gemini.test.ts`
- Test: `speech-system/realtime core/tests/runtime.test.ts`

**Interfaces:**
- `createGeminiSessionForTest(config)` returns the existing session/message seam plus `providerConfig` and `sentToolResponses` inspection values.
- `FakeRealtimeSpeechProvider` accepts `{ deferAudio?: boolean; toolCall?: { tool: string; arguments: Record<string, unknown> } }` and exposes received tool results for assertions.
- Gemini config maps each declaration to `{ name, description, parametersJsonSchema }` under `tools: [{ functionDeclarations: [...] }]`.
- A provider `toolCall.functionCalls[]` emits one neutral `tool.requested` event per call, correlating `FunctionCall.id` as `callId` and retaining the function name privately.
- `sendToolResult` sends `{ functionResponses: [{ id, name, response: { result: content } | { error: content } }] }` and rejects unknown IDs with `RealtimeError` code `TOOL_CALL_UNKNOWN`.

- [x] **Step 1: Write failing Gemini declaration/call/response tests**

Add tests that pass a tool declaration with lowercase standard JSON Schema, inspect the harness provider config for `tools[0].functionDeclarations[0].parametersJsonSchema`, feed `{ toolCall: { functionCalls: [{ id: "call-1", name: "open_app", args: { app: "calculator" } }] } }`, assert the neutral event and timestamp, call `sendToolResult` once for success and once for error, and assert exact response payloads. Add an unknown call ID assertion for `TOOL_CALL_UNKNOWN`.

```ts
const tool = { name: "open_app", description: "Open an app.", inputSchema: { type: "object", properties: { app: { type: "string" } }, required: ["app"] } };
const { session, receiveProviderMessage, providerConfig, sentToolResponses } = createGeminiSessionForTest({ ...config, tools: [tool] });
assert.deepEqual((providerConfig as any).tools[0].functionDeclarations[0].parametersJsonSchema, tool.inputSchema);
const seen = collect(session.events(), 2);
receiveProviderMessage({ toolCall: { functionCalls: [{ id: "call-1", name: "open_app", args: { app: "calculator" } }] } });
assert.equal((await seen)[1]?.type, "tool.requested");
await session.sendToolResult({ callId: "call-1", content: "Opened calculator." });
assert.deepEqual(sentToolResponses[0], { functionResponses: [{ id: "call-1", name: "open_app", response: { result: "Opened calculator." } }] });
```

- [x] **Step 2: Run the focused Gemini tests and verify RED**

```powershell
npm test -- tests/gemini.test.ts
```

Expected: declaration inspection, `tool.requested`, and `sendToolResult` assertions fail because the adapter currently ignores tool messages and has no response method.

- [x] **Step 3: Implement Gemini config and private call correlation**

Add a provider-local `toGeminiLiveConfig` helper in `gemini.ts`. Extend the private session with `Map<string, string>` for pending call IDs, parse `toolCall` before ordinary server content, and push timestamped neutral events without exposing SDK types. Generate a private UUID only if a provider call omits an ID; reject a missing function name as a provider failure rather than inventing a tool name. Remove a call from the map only after sending its response.

- [x] **Step 4: Add the fake provider's deterministic tool turn**

When the fake receives non-empty text and `options.toolCall` exists, emit a timestamped `tool.requested` event instead of audio for that turn. `sendToolResult` records the result and then emits the normal fake audio completion path, so Assistant Runtime tests can assert both execution and response without a network.

- [x] **Step 5: Update all realtime event construction with timestamps and run GREEN**

Use `Date.now()` at event creation for lifecycle, transcript, interruption, tool, and output events. Run:

```powershell
npm test -- tests/gemini.test.ts
npm test -- tests/runtime.test.ts
npm run verify
```

Expected: all realtime tests pass, including existing stale-output/interruption tests, with no Gemini API key.

- [x] **Step 6: Commit the provider and fake implementation**

```powershell
git add -- "realtime core/src/gemini.ts" "realtime core/src/fake.ts" "realtime core/tests/gemini.test.ts" "realtime core/tests/runtime.test.ts"
git commit -m "feat: translate realtime tool calls for Gemini and fake provider"
```

### Task 3: Assistant Runtime executor contract and PCM frameizer

**Files:**
- Modify: `assistant-runtime/src/contracts.ts`
- Create: `assistant-runtime/src/realtime-audio.ts`
- Modify: `assistant-runtime/src/index.ts`
- Create: `assistant-runtime/tests/realtime-audio.test.ts`

**Interfaces:**
- `RealtimeToolExecutor.discover(): Promise<RealtimeToolDeclaration[]>` returns provider-neutral declarations.
- `RealtimeToolExecutor.execute(input: { callId: string; tool: string; arguments: Record<string, unknown>; signal?: AbortSignal }): Promise<{ content: string; isError?: boolean }>` executes one call.
- `PcmInputFrameizer.push(data: Int16Array): Int16Array[]` returns only complete 320-sample copies and carries a partial tail.
- `PcmInputFrameizer.reset(): void` discards an incomplete tail.
- `REALTIME_MICROPHONE_STREAM_ID` remains `"windows-default-microphone"`.

- [x] **Step 1: Write failing frameizer tests**

Create tests for a 1,600-sample/100 ms chunk producing five independent 320-sample frames, a 100-sample tail carried into the next push, and input-array mutation after `push` not changing emitted frames.

```ts
test("splits a 100 ms capture chunk into five 20 ms frames", () => {
  const frameizer = new PcmInputFrameizer(320);
  const frames = frameizer.push(new Int16Array(1_600).fill(7));
  assert.equal(frames.length, 5);
  assert.deepEqual(frames.map((frame) => frame.length), [320, 320, 320, 320, 320]);
});

test("carries partial samples and copies the source", () => {
  const frameizer = new PcmInputFrameizer(320);
  const source = new Int16Array(400).fill(3);
  const first = frameizer.push(source);
  source.fill(9);
  const second = frameizer.push(new Int16Array(240).fill(4));
  assert.equal(first.length, 1);
  assert.equal(second.length, 1);
  assert.equal(second[0]?.[0], 3);
  assert.equal(second[0]?.[319], 4);
});
```

- [x] **Step 2: Run the focused tests and verify RED**

```powershell
npm test -- tests/realtime-audio.test.ts
```

Expected: module/class-not-found failure because the frameizer does not yet exist.

- [x] **Step 3: Implement the minimal frameizer and executor interface**

Use one internal `Int16Array` remainder. On each push, copy the input, join it with the remainder, slice complete frames, and retain the final partial slice. Do not emit a frame with a misleading duration. Add the executor interface to the public assistant contracts and export it through `src/index.ts`.

- [x] **Step 4: Run the focused tests and package typecheck**

```powershell
npm test -- tests/realtime-audio.test.ts
npm run typecheck
```

Expected: frameizer tests pass; existing custom realtime sessions may now fail typecheck until they implement `sendToolResult` in Task 5.

- [x] **Step 5: Commit the assistant contract and frameizer**

```powershell
git add src/contracts.ts src/realtime-audio.ts src/index.ts tests/realtime-audio.test.ts
git commit -m "feat: add realtime executor contract and pcm frameizer"
```

### Task 4: Tool System-backed realtime executor

**Files:**
- Modify: `assistant-runtime/src/tool-bridge.ts`
- Modify: `assistant-runtime/tests/tool-bridge.test.ts`
- Create: `assistant-runtime/tests/realtime-tools.test.ts`

**Interfaces:**
- `ToolSystemRealtimeToolExecutor` implements the exact `RealtimeToolExecutor` interface from Task 3.
- Discovery converts Tool System parameter declarations to lowercase standard JSON Schema (`object`, `string`, `integer`, `number`, `boolean`) and removes context-bound parameters from `required`.
- Execution calls `ToolRuntime.execute({ tool, args, requestId: callId }, signal)` and renders the returned Tool System outcome with the existing taint/error wording. Only `error` outcomes set `isError: true`.

- [x] **Step 1: Write failing bridge tests**

Extend the existing real Tool System fixture with a discovery assertion for lowercase JSON Schema and an execution assertion that a catalogued `open_app` call launches the stubbed broker only through `ToolRuntime`. Add a cancellation test with a registry handler that waits for `signal.aborted` and assert the returned result is an error without a broker launch.

```ts
test("realtime executor discovers and executes through Tool System", async () => {
  const { runtime, launched } = toolSystem();
  await runtime.start();
  const executor = new ToolSystemRealtimeToolExecutor(runtime);
  assert.deepEqual((await executor.discover())[0]?.inputSchema, {
    type: "object",
    properties: { app: { type: "string", description: "Application to open.", enum: ["browser", "editor"] } },
    required: ["app"],
  });
  const result = await executor.execute({ callId: "call-1", tool: "open_app", arguments: { app: "browser" } });
  assert.equal(result.isError, undefined);
  assert.equal(result.content, "Opened browser.");
  assert.deepEqual(launched, [{ executable: "firefox", args: [] }]);
});
```

- [x] **Step 2: Run the focused tests and verify RED**

```powershell
npm test -- tests/tool-bridge.test.ts tests/realtime-tools.test.ts
```

Expected: the new executor import or class is missing.

- [x] **Step 3: Implement schema conversion and outcome reuse**

Keep the current Intelligence Core schema conversion unchanged. Add a separate `toRealtimeInputSchema` with lowercase standard JSON Schema, factor the existing `describe` function into a shared local outcome renderer, and add `ToolSystemRealtimeToolExecutor`. Do not import any Tool System code into Realtime Core and do not add a second policy/validation path.

- [x] **Step 4: Run bridge, cancellation, and existing tool tests**

```powershell
npm test -- tests/tool-bridge.test.ts tests/realtime-tools.test.ts
```

Expected: all existing Intelligence-to-Tool-System tests and new realtime bridge tests pass.

- [x] **Step 5: Commit the bridge**

```powershell
git add src/tool-bridge.ts tests/tool-bridge.test.ts tests/realtime-tools.test.ts
git commit -m "feat: bridge realtime tool execution through tool system"
```

### Task 5: RealtimeCoreAdapter orchestration, framing, cancellation, and metrics

**Files:**
- Modify: `assistant-runtime/src/adapters.ts`
- Modify: `assistant-runtime/src/composition.ts`
- Modify: `assistant-runtime/tests/audio-routing.test.ts`
- Modify: `assistant-runtime/tests/session-lifecycle.test.ts`
- Modify: `assistant-runtime/tests/realtime-tools.test.ts`

**Interfaces:**
- Preserve existing `RealtimeCoreAdapter(core, config, trace?, onSpeechEvent?)` call sites and add one optional final `RealtimeToolExecutor` parameter.
- `open(input)` accepts the existing `interactionId`, `signal`, and `onActivity` fields; all are optional at the adapter boundary for existing probes/tests.
- `sendMicrophonePcm(data)` queues complete 320-sample frames in order and resolves after its enqueued frames are handed to the session or recorded as input failures.
- Optional executor discovery is awaited before `RealtimeCore.connect`; its declarations replace `config.tools` for that connection.
- Aggregate trace shapes are bounded and include correlation fields without raw arguments:
  - `realtime.input.metrics`: `framesSent`, `framesDropped`, `bufferedMs`;
  - `realtime.tool.metrics`: `requested`, `completed`, `failed`, `cancelled`, `callId` for the current transition;
  - `realtime.playback.metrics`: `firstChunkAt`, `bytesWritten`, `chunksWritten`, `abortRequested`, `abortCompleted`, `durationMs`.

- [x] **Step 1: Write failing adapter tests for frameization and pending bound**

Extend the delayed test session to record incoming `AudioFrame`s and add assertions that a 1,600-sample capture chunk creates five 320-sample frames with `frameDurationMs: 20`. Push 30 complete frames while `connect` is blocked and assert only the newest 25 are flushed, the first five are dropped, and a trace reports `framesDropped: 5` and `bufferedMs: 500` before flush.

- [x] **Step 2: Run the focused adapter tests and verify RED**

```powershell
npm test -- tests/audio-routing.test.ts tests/session-lifecycle.test.ts
```

Expected: the current adapter sends one 100 ms frame and does not report drops, so the new assertions fail.

- [x] **Step 3: Implement ordered frameization and the bounded pending queue**

Use `PcmInputFrameizer` for arbitrary capture chunks. Keep a single promise chain for input sends so Windows capture callbacks cannot reorder frames. Store complete frames only while `opening` is true; when the queue exceeds 25, shift the oldest frame and increment the aggregate drop counter. Use `REALTIME_INPUT_FORMAT` for every `AudioFrame`, and reset the frameizer on failed open/close.

- [x] **Step 4: Run the frameization tests and preserve lifecycle behavior**

```powershell
npm test -- tests/audio-routing.test.ts tests/session-lifecycle.test.ts
```

Expected: the new 20 ms and queue assertions pass; provider-closed sessions still stop accepting input without unhandled rejections.

- [x] **Step 5: Write failing adapter tests for discovery, execution, and cancellation**

Add a fake-provider test that injects `ToolSystemRealtimeToolExecutor`, captures the config passed to `connect`, emits the deterministic Calculator request, and asserts the Tool System broker receives `{ executable: "calc.exe", args: [] }` and the fake session records a successful tool result. Add an executor-failure test and an abort-while-session-closed test; both must avoid host launch and avoid a late `sendToolResult` call.

```ts
const executor: RealtimeToolExecutor = {
  async discover() { return [{ name: "open_app", description: "Open Calculator.", inputSchema: { type: "object" } }]; },
  async execute(input) { return { content: `completed:${input.tool}`, isError: false }; },
};
```

- [x] **Step 6: Run the tool tests and verify RED**

```powershell
npm test -- tests/realtime-tools.test.ts
```

Expected: the current adapter neither discovers tools nor handles `tool.requested` events.

- [x] **Step 7: Implement discovery, tool loop, and cancellation**

Augment the resolved session config with executor discovery. In the event consumer, handle `tool.requested` by passing the interaction `AbortSignal` to `execute`. Send the returned content with the original `callId` only if the interaction is not cancelled and the session is still active. On executor failure, send a redacted error result while the session is open; on cancellation or a closed session, record a cancelled transition and skip the late result. Never log `event.arguments`.

- [x] **Step 8: Add aggregate playback/input/tool metrics and keep activity event-first**

Trace event timestamps for every provider event. Count input frames at the send boundary, output bytes/chunks at playback handling, first-output timestamp from the first chunk, and abort requested/completed around interruption. Count tool requested/completed/failed/cancelled by `callId`. Keep amplitude activity as fallback and call `onActivity` from provider `input.speech_started`, final input transcript, output start, and tool request/completion events.

- [x] **Step 9: Inject the executor through composition without a default side-effecting host tool**

Add `realtimeToolExecutor?: RealtimeToolExecutor` to `AssistantCompositionOptions` and pass it as the final adapter constructor argument. Keep side-effecting host capabilities out of the default composition; the default path may install the safe read-only Host Tools catalogue.

- [x] **Step 10: Run the complete Assistant Runtime suite**

```powershell
npm run typecheck
npm test
npm run build
```

Expected: all existing lifecycle, playback, memory, composition, and Tool System tests pass together with the new realtime-tool tests.

- [x] **Step 11: Commit the adapter and composition integration**

```powershell
git add src/adapters.ts src/composition.ts src/contracts.ts src/realtime-audio.ts src/index.ts tests
git commit -m "feat: execute injected tools in realtime adapter"
```

### Task 6: Explicit manual probe and documentation

**Files:**
- Create: `assistant-runtime/tests/probe-realtime-tool.ts`
- Create: `assistant-runtime/docs/realtime-tools-smoke-test.md`
- Modify: `assistant-runtime/docs/hardware-smoke-test.md`
- Modify: `assistant-runtime/README.md`
- Modify: `assistant-runtime/ARCHITECTURE.md`
- Modify: `assistant-runtime/WORKPLAN.md`
- Modify: `assistant-runtime/PROGRESS.md`
- Modify: `speech-system/realtime core/README.md`
- Modify: `speech-system/realtime core/ARCHITECTURE.md`
- Modify: `speech-system/realtime core/WORKPLAN.md`
- Modify: `speech-system/realtime core/PROGRESS.md`

**Interfaces:**
- The manual probe explicitly constructs `ToolRuntime` with `openAppDeclaration({ calculator: "calc.exe" })`, `AllowlistPolicy({ allow: ["open_app"] })`, and `AllowlistProcessBroker({ executables: ["calc.exe"] })`.
- The probe injects `new ToolSystemRealtimeToolExecutor(runtime)` into `createAssistantRuntime` and never stores a key or creates the executor in production defaults.
- Documentation distinguishes offline verification from credentialed/hardware verification and records no unverified PASS claims.

- [x] **Step 1: Create the explicit Calculator probe**

Use only `GEMINI_API_KEY` from the current process environment. Start the Tool Runtime, create the explicit Calculator catalog and allowlisted broker, pass the executor through `createAssistantRuntime(..., { realtimeToolExecutor })`, start the runtime, print redacted JSON traces, and stop cleanly on `SIGINT`/`SIGTERM`.

- [x] **Step 2: Document the manual command and expected evidence**

Document prerequisites, `npm run build`, `GEMINI_API_KEY` process-scoped setup, `npx tsx tests/probe-realtime-tool.ts`, the double-clap/start sequence, the spoken phrase “Open Calculator”, expected `realtime.tool.requested`/`completed` traces, Calculator launch, and the spoken response. State explicitly that the probe is not part of offline tests and has not been run unless a real hardware run is recorded.

- [x] **Step 3: Update architecture/workplans/progress with the verified boundary**

Describe the provider-neutral declaration/request/result flow, the injected executor, 20 ms frame contract, 500 ms pending bound, timestamped aggregate traces, and unchanged non-goals. Update progress only with commands actually run in this implementation.

- [x] **Step 4: Run final child-repository verification**

From `speech-system/realtime core`:

```powershell
npm run verify
```

From `assistant-runtime`:

```powershell
npm install
npm run verify
```

Expected: both exit with code 0; the output reports zero test failures and builds the updated `dist` artifacts. Inspect `git status --short` in each child and confirm only intended source/tests/docs/lockfile changes exist.

- [x] **Step 5: Commit documentation in each child repository**

```powershell
# speech-system repository
git add -- "realtime core/README.md" "realtime core/ARCHITECTURE.md" "realtime core/WORKPLAN.md" "realtime core/PROGRESS.md"
git commit -m "docs: record realtime tools boundary and verification"

# assistant-runtime repository
git add README.md ARCHITECTURE.md WORKPLAN.md PROGRESS.md docs tests/probe-realtime-tool.ts
git commit -m "docs: record injected realtime tool smoke path"
```

- [x] **Step 6: Advance and verify root submodule pointers**

```powershell
cd C:\Users\Sajmon\Jarvis
git status --short
git diff --submodule=log -- speech-system assistant-runtime
git add speech-system assistant-runtime
git commit -m "chore: advance realtime tools child repositories"
```

The root commit is valid only if it points to the child commits whose full verification output was captured immediately beforehand. Do not modify `tool-system` or `host-tools`.

## Final audit checklist

- [x] Realtime Core contains no imports from `tool-system` or `host-tools`.
- [x] The default `createAssistantRuntime()` path advertises only `get_time`, `calculate`, `uptime`, and `system_status`; side-effecting tools remain opt-in.
- [x] Gemini fields are confined to `gemini.ts`; public contracts remain provider-neutral.
- [x] Every realtime event has a numeric timestamp.
- [x] Every microphone frame sent by `RealtimeCoreAdapter` is exactly 320 samples and marked 20 ms.
- [x] Pending input never exceeds 25 complete frames and reports drops.
- [x] Tool execution is performed by `ToolRuntime`, including validation, policy, guards, broker, cancellation, and outcome rendering.
- [x] Tool arguments and secrets are absent from traces.
- [x] Offline typecheck, tests, and builds pass in both child repositories.
- [x] Manual Calculator probe is documented and its run status is reported separately from offline verification.
- [x] Root submodule pointers are committed only after child verification.

## Execution choice

The user explicitly requested immediate implementation, so execute this plan inline with `superpowers:executing-plans`, using the TDD red-green cycle in each task and stopping only for a genuine failing verification or external hardware/credential requirement.
