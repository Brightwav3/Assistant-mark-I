# Assistant mark I

Infrastructure for a persistent, ambient, model-independent personal assistant.

This is the meta-repository. It contains the manifesto and every core linked as a
git submodule. Each core remains a fully independent repository with its own
history, workplan, and issues.

## On the name "Jarvis"

**Jarvis is a working name, not the product name.**

It appears throughout the manifesto, the architecture documents, and the agent
instructions. It is used deliberately, for one reason: it is a single word that
communicates the entire intent of the project without a paragraph of explanation.
Saying "I'm building Jarvis" ends the question. Saying "I'm building headless,
provider-independent assistant infrastructure" starts three more.

That convenience is the whole justification. It is not a commitment.

Two consequences follow, and both are intentional:

1. **The name never reached the code.** Every package is named for what it does —
   `core-runtime`, `memory-core`, `state-core`, `intelligence-core`,
   `device-network`, `activation-core`, `realtime-core`, `scribe-core`,
   `voice-core`. Directory names match their packages one-to-one. No component
   carries the working name.
2. **Renaming stays cheap forever.** Because the name lives only in prose, the
   final product name can be chosen at any point without touching a single
   module, import, or contract.

"Jarvis" will not be the shipped name — it belongs to Marvel/Disney. The manifesto
keeps it because a manifesto is written for a human reader who needs to know what
kind of thing is being built, and the name does that job in one word.

## What this is

Read [the manifesto](./MANIFESTO.md) first. The short version:

> A better model should fit into the same assistant without requiring the nervous
> system to be rebuilt.

The assistant is larger than its interface. The model is a component. Voice, text,
displays, and devices are independent ways to reach the same system. Lifecycle,
validation, storage, permissions, and safety stay deterministic and belong to the
platform, not to whatever model is currently plugged in.

## Repository map

Every directory below is a git submodule pointing at a standalone repository.

| Core | Owns | Status |
| --- | --- | --- |
| [`core-runtime`](./core-runtime) | lifecycle, configuration, events, component registry, local API, health | foundation complete |
| [`activation-core`](./activation-core) | activation providers and detection events | v0.1 complete |
| [`speech-system`](./speech-system) | Scribe Core, Voice Core, Realtime Core | component foundations complete |
| [`intelligence-core`](./intelligence-core) | model gateway, context, action, production boundaries | core complete |
| [`memory-core`](./memory-core) | deliberate durable memory and retrieval | v0.1 complete |
| [`state-core`](./state-core) | current state, freshness, revisions, subscriptions | v0.1 complete |
| [`device-network`](./device-network) | protocol, registry, WebSocket transport, liveness, commands | v0.1 complete |
| [`tool-system`](./tool-system) | the tool contract, brokered execution, policy enforcement point | v0.1 complete |
| [`host-tools`](./host-tools) | the capability catalogue declared against that contract | v0.1 complete |
| [`assistant-runtime`](./assistant-runtime) | cross-core composition and interaction lifecycle | usable v0.1, hardening in progress |
| [`activation-gemini-bridge`](./activation-gemini-bridge) | temporary activation-to-realtime bridge | temporary |

The cores have **zero imports between each other**. The only component that knows
about more than one core is `assistant-runtime`, which composes them behind typed
adapters and hosts them as components inside `core-runtime`.

## Planned structure

The eleven repositories above are the beginning, not the shape. Below is the full set
of cores the system is planned to consist of, and what each one owns. Cores are
added one at a time — each must produce a real, testable capability before the next
major layer begins.

### Cores

| | Core | What it owns |
| --- | --- | --- |
| ✅ | Brain Core | Lifecycle, component registry, health, the base runtime and orchestration. |
| ✅ | Activation Core | Activation by wake phrase, clap, external trigger, and similar signals. |
| ✅ | Intelligence Core | Model gateway, context, action loop, routing, and model-independent inference. |
| ✅ | Memory Core | Long-term structured memory with provenance, confidence, and update/forget semantics. |
| ✅ | State Core | Current state of the system and the world: devices, active interactions, freshness, TTL, snapshots. |
| ✅ | Scribe Core | Audio and microphone input → STT → transcript. |
| ✅ | Voice Core | Text → TTS → audio playback. |
| ✅ | Realtime Core | Persistent native-audio sessions of the Gemini Live kind, audio ⇄ model. |
| ❌ | Interaction Core | Both directions of audio at once: the played signal, the captured signal, one shared timeline. Echo cancellation is its first identified responsibility; conversational flow between the speech subsystems follows from owning the same ground. |
| ❌ | Event Core | Central cross-system event infrastructure. |
| ❌ | Context Core | Broader environmental and user context across systems. |
| ❌ | Security Core | Authority, permissions, trust boundaries, and policy. |
| ❌ | Task Core | Long-running, persistent work independent of any single conversation. |
| ❌ | Automation Core | Deterministic trigger → conditions → action workflows with no AI involved. |
| ❌ | Presence Core | Where the user is, with confidence and room-level presence. |

### Beyond the cores

| | Component | What it owns |
| --- | --- | --- |
| ✅ | Device Network | Communication with devices and future room satellites. |
| ✅ | Assistant Runtime | Composes every independent core into one running assistant. |
| ✅ | Tool System | The tool contract: declaration, validation, guards, brokered execution, policy enforcement point. |
| ✅ | Host Tools | The capability catalogue: what the assistant can actually do on a machine. |
| ❌ | Display System | Structured visual output. |
| ❌ | Home Bridge | Integration with Home Assistant and smart-home infrastructure. |
| ❌ | Apple Bridge | Calendar, Mail, Contacts, Reminders, and related services. |
| ❌ | Internet Gateway | A separate internet-facing trust zone. |
| ❌ | Room Satellite | A physical microphone, speaker, display, and sensor endpoint. |
| ❌ | Control Center | Administration, diagnostics, and configuration. |

Reliability is deliberately **not** a core, because it is a property of every core
rather than a place in the system: failures degrade predictably instead of
collapsing. Security is the exception — authority, permissions, and trust
boundaries are enforceable only if something owns them, so Security Core is a core.

The list has no end state by design. The goal is not to build everything that can be
imagined — it is to avoid making any of it unnecessarily difficult later.

## Why submodules

A submodule reference is a pinned commit, not a copy. That gives three properties
worth the small amount of ceremony:

1. Each core stays clonable, buildable, and releasable on its own.
2. One commit in this repository records a combination of eleven repositories that is known
   to work together.
3. There is exactly one source of truth per core. Nothing is duplicated, so nothing
   can drift.

## Getting started

```bash
git clone --recurse-submodules https://github.com/Brightwav3/Assistant-mark-I.git
```

`--recurse-submodules` is required; without it the core directories are empty.

Every core publishes its public entry from `dist/`, so the cores are built before
the composition can resolve them:

```bash
for dir in core-runtime activation-core intelligence-core memory-core state-core tool-system host-tools            "speech-system/realtime core" "speech-system/scribe core" "speech-system/voice core"; do
  (cd "$dir" && npm install && npm run build)
done
```

Then verify the assembled slice:

```bash
cd assistant-runtime && npm install && npm run verify
```

## Current state

The first usable slice runs today, and it has been used: on 2026-08-12 the native
path was verified end to end on real hardware — a double clap activated the
assistant, a Gemini Live session started, it answered out loud, interrupting it
stopped playback immediately, it wrote a summary of the conversation to SQLite,
the interaction timed out on its own, and after a restart it still knew what had
been said.

The model can also act. Gemini requests a tool, Tool System validates the
arguments, consults policy, applies its guards, and launches the process through
a broker that accepts an argv array and has no shell entry point. A denied tool
reaches nothing, a hallucinated argument is rejected before any launch, an
approval flag invented by the model is an undeclared argument rather than a
permission, and content returned from outside is labelled as data rather than
instruction. Every one of those is a test, not an intention.

Every core is also verified automatically on each push, and the meta-repository
builds them all and re-runs the composed slice.

Not yet verified on hardware: the modular Scribe → Intelligence → Voice path,
and the tool loop above. Every hardware result came from the native realtime
path; the tool loop is verified end to end against a scripted provider.

Each core repository documents its own verified state and known limitations in its
`README.md` and `PROGRESS.md`.

## What comes next

Three pieces of work are planned, in this order. Each is listed with the evidence
that motivated it rather than as an intention, because the reason a thing is next
is more useful to a later reader than the fact that it is.

All three are instances of one structural decision, described first because the
three items below are hard to judge without it.

### The organising principle: two paths, not one

The assistant carries two kinds of work that have nothing in common except that
they happen during the same conversation.

**The media path** moves audio. Microphone in, speech out, uninterrupted. Its
only obligation is that every frame arrives on time, because a delay here is not
a slow response — it is an audible gap, a stutter, a word cut in half. Nothing on
this path may be allowed to wait for anything.

**The application path** does everything else. Choosing and running a capability,
consulting a stronger model, writing to memory, enforcing policy. This work is
allowed to be slow, because slowness here costs a pause in *content* rather than a
break in *sound*.

The decision is to keep these separate and to let the second one be slow without
the first one noticing:

```text
  ┌──────────┐   audio in    ┌──────────────┐   audio    ┌────────────────┐
  │   user   │ ────────────▶ │   Realtime   │ ─────────▶ │  realtime      │
  │          │ ◀──────────── │     Core     │ ◀───────── │  model session │
  └──────────┘   audio out   └──────┬───────┘            └────────────────┘
        MEDIA PATH — must never wait │  ▲
                                     │  │ result, whenever it is ready
              tool / delegation      │  │
                     request         ▼  │
                             ┌──────────┴─────────┐
                             │  Assistant Runtime │   composition, correlation
                             └──────────┬─────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
            ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
            │ Tool System + │   │ Intelligence  │   │ Memory / State│
            │  Host Tools   │   │     Core      │   │     Cores     │
            └───────────────┘   └───────────────┘   └───────────────┘
                    APPLICATION PATH — allowed to be slow
```

**What this buys.** Capabilities can be added, policy can be tightened, a slower
and better model can be consulted, and none of it changes the code responsible for
keeping audio moving. The live path stays small and stays boring — which is the
only way it stays reliable. It also means the failure modes stay separate: a
capability that times out produces an assistant that says it could not do
something, not an assistant that stops talking mid-sentence.

**The rule that enforces it.** A request leaving the media path is *dispatched,
not awaited*. The session continues consuming and producing audio while the work
runs; the result is delivered through the session whenever it exists, correlated
by an identifier rather than by position in a sequence. If the result never
arrives, the guard that owns the timeout produces an outcome, and the outcome
travels the same way any other outcome does.

This is the property worth testing directly rather than assuming: audio frames
must keep flowing while a deliberately slow capability runs. That test is the
difference between having adopted the architecture and having drawn it.

**Where this system already stands.** Every part of the shape exists. Realtime
Core is the media path and holds a live session. Assistant Runtime is the only
component that may compose across cores. Tool System, Host Tools, Intelligence
Core, Memory Core, and State Core are the application path, and each is
independently verified.

What does not exist is the connection between them. The two paths are currently
*alternatives* — either the native realtime session runs, or the modular
transcribe-reason-speak pipeline runs — rather than *layers*, where the first
carries the conversation and the second does the thinking underneath it. The tool
loop is complete but reachable only from the path that is not the one running on
hardware.

So the work below is not three unrelated improvements. It is one join, made three
times: capabilities joined to the live session (§1), a speech contract able to
hold real providers on either path (§2), and a component owning both directions of
audio so the media path can survive hearing itself (§3).

### 1. The tool loop reaches what actually runs

**The gap.** The model can act, and separately the assistant can hold a live
conversation, but those are two different paths. The native realtime path bypasses
Intelligence Core, and Intelligence Core is where tools live. Everything in Tool
System and Host Tools is therefore unreachable from the thing that runs on
hardware. Concretely, `RealtimeSessionConfig` has no field for tool declarations
and `RealtimeSpeechEvent` has no variant for a tool being requested, so a
capability has neither a way in nor a way out of a live session.

**The shape.** A live session declares the catalogue, the model requests a tool
mid-conversation, the request leaves the session as an event, Tool System
validates and executes it under policy, and the result returns through
`sendToolResult`. The session's `sendText` already exists, so the return path is
half built.

This is the first place the two-path rule above becomes code: execution is
dispatched from the session, never awaited inside it, and the outcome is delivered
back through the session when it exists. Concurrent requests are tracked by
identifier and cancelled together when the session closes, so a capability cannot
outlive the conversation that asked for it.

**Why not two models.** A well-known realtime architecture routes the voice model
through a second, stronger text model that owns the tools. That second hop exists
because the voice model reasons poorly. Gemini Live can call functions directly,
so the first version is one hop: realtime session → Tool System → Host Tools.

Delegation to Intelligence Core is then added later as *another entry in the
catalogue* rather than as a second pipeline. A `delegate_to_reasoner` capability
returning a `continuation` gives the same architecture — the assistant says it
will look into something and answers when the answer exists — while passing
through the same policy, the same guards, and the same trace as every other
capability. A separate delegation path would have needed all three built twice.

**Boundary.** Realtime Core must not import Tool System. Its tool declaration
type is its own minimal shape; the translation between the two vocabularies
belongs in `assistant-runtime`, alongside the bridge that already performs
exactly this translation for the modular path.

### 2. A speech-to-text contract that can hold a second provider

**The gap.** The modular Scribe → Intelligence → Voice path has never run on
hardware, and the reason is narrow: Scribe Core has only a fake transcriber.
Intelligence Core is real and Voice Core is real. Speech-to-text is the one
missing link.

**Why the interface comes first.** `SttProvider` is what makes a model
replaceable, and it currently has only fake implementations, so it has never been
tested as an abstraction against anything real. Its request carries a complete
array of audio frames, which fits batch transcription — upload a finished
recording, receive a transcript — but cannot express *"I am still speaking, here
is more audio"*. Streaming providers are session-shaped: open, push frames,
receive partial and final results, close.

| Provider shape | Example | Fits the contract today |
| --- | --- | --- |
| Batch upload | Whisper API, whisper.cpp | yes |
| Push-based streaming | Voxtral realtime, Deepgram, Azure | no — only by buffering the whole utterance |

Two of four, and the two that do not fit are the ones worth having for live
conversation. Writing the first real adapter against the present contract would
mean bending it to fit and then treating the bent shape as settled.

So the contract becomes session-shaped first. Batch providers wrap into a session
trivially — buffer on push, transcribe on end, emit one final result. The reverse
is not possible, which is why the direction of the fix is not a matter of taste.

**Replaceability is proven with two, not one.** A second adapter of the opposite
shape is part of this work rather than a later nicety. Until the contract has held
two genuinely different providers, its neutrality is a claim.

**A useful trait to copy.** A published realtime transcription client validates
every field it accepts from its provider — error codes, request identifiers —
before letting them travel further, and reports whether a disconnect is
recoverable rather than leaving the caller to guess. Both are principles this
system already applies elsewhere: external content is untrusted, and retryability
is derived rather than assumed. Speech is simply a boundary where neither has been
applied yet.

### 3. Echo, and the component it implies

**The gap.** There is no echo cancellation anywhere in this system. Audio is
captured raw from the platform and played back through a separate path, and
nothing subtracts the second from the first. The realtime path has been verified
on hardware regardless, which means the machine it ran on happened to hide the
problem — headphones, or a microphone with cancellation in its own driver.
Neither is a property of the system.

**Why the obvious fix is a trap.** Muting the microphone while the assistant
speaks solves echo in an hour and destroys interruption, which is already
verified working. Trading a demonstrated capability for a workaround is the wrong
direction.

**What it actually needs.** Cancelling echo requires the played signal and the
captured signal in one place, aligned on a shared timeline. Capture lives in
Scribe Core, playback lives in Realtime Core, and the cores have zero imports
between them by design. There is currently nowhere for this to live.

That makes echo the first requirement that genuinely demands a component owning
**both directions of audio**, which is why Interaction Core is described above in
those terms. It began as a speculative entry — coordination of conversational
flow, if it turned out to be needed. A live session hearing itself speak is not
speculative, so the core now has a concrete first responsibility instead of a
hypothetical one. It is not started yet, because the two items above
deliver capability the system does not have, while this one repairs a condition
that headphones currently mask.

### Deliberately not planned

Ideas encountered while reading how large realtime systems are built, and left
alone on purpose. Each solves a problem this system does not have.

| Not doing | Why not |
| --- | --- |
| Custom transport handshake optimisation | Saves network round trips at session start. One machine, one user, localhost. |
| Model-instance handoff and live context compaction | Keeps multi-hour sessions alive across instance transitions at scale. |
| Shadow-testing a new path against production traffic | There is no production traffic to shadow. |
| Rewriting hot paths in a faster language | Frame delivery smoothness has not been measured, let alone found wanting. |
| Speculative and authoritative views of the transcript | The split is already owned correctly — State Core holds what is current, Memory Core holds what was said. |

## License

[PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/).
See [LICENSE](./LICENSE).

Use is permitted for noncommercial purposes only — personal use, research,
education, and public-benefit work. Any use by or for a business requires a
separate license. This applies to the meta-repository; each core is a separate
repository and carries its own terms.
