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
| [`assistant-runtime`](./assistant-runtime) | cross-core composition and interaction lifecycle | usable v0.1, hardening in progress |
| [`activation-gemini-bridge`](./activation-gemini-bridge) | temporary activation-to-realtime bridge | temporary, private |

The cores have **zero imports between each other**. The only component that knows
about more than one core is `assistant-runtime`, which composes them behind typed
adapters.

## Planned structure

The nine repositories above are the beginning, not the shape. Below is the full set
of cores the system is planned to consist of, and what each one owns. Cores are
added one at a time — each must produce a real, testable capability before the next
major layer begins.

### Built

| Core | What it is | Subsystems |
| --- | --- | --- |
| `core-runtime` | The nervous system. Component lifecycle, configuration, structured logging, local API, health aggregation, clean shutdown. Exists and runs with no AI present. | event bus, component registry, local JSON API, health, logger |
| `activation-core` | Decides when the assistant is being addressed. Provider-independent detection, emits activation events and PCM frames. | activation providers, Windows listener |
| `speech-system` | Everything between air and text, and back. | **Scribe Core** (wake word, VAD, streaming STT), **Voice Core** (TTS, controlled local playback), **Realtime Core** (persistent native-audio sessions, barge-in hard stop) |
| `intelligence-core` | Makes the model a replaceable component. Provider-neutral model contracts, context assembly, policy-gated tool loop, production routing. | model gateway, context assembly, action boundary, Gemini REST adapter |
| `memory-core` | Deliberate long-term knowledge. What is remembered is a decision, not a side effect. | durable store, retrieval, summaries |
| `state-core` | Current facts, not history. Provenance, freshness, confidence, revision protection, subscriptions. | snapshots, TTL freshness, subscriptions, context adapter |
| `device-network` | Physical endpoints as first-class citizens. Microphones, speakers, displays, and sensors exist independently of the core. | typed protocol, registry, WebSocket transport, authentication, liveness, simulator |
| `assistant-runtime` | The composition root. The only component that knows about more than one core. | interaction lifecycle, typed adapters, conversation memory |

### Planned

| Core | What it is |
| --- | --- |
| `interaction-core` | Turn-taking and conversation shape as an explicit subsystem of the speech system, plus full acoustic echo cancellation. Currently the largest gap in the voice path. |
| `agent-runtime` | The tool-calling loop. Context, model, tool, result, answer — with iteration limits, timeouts, cancellation, structured errors, and execution tracing. |
| `policy-core` | Enforceable capability boundaries. The model may request actions; this decides whether they happen. The model cannot bypass it. |
| `environment-core` | A bridge to mature smart-home infrastructure, so the assistant reasons over "this room" and "the front door" instead of raw entity identifiers. |
| `display-core` | Structured visual output. A spoken request can produce something to look at without opening an application. |
| `automation-core` | Deterministic triggers, conditions, and actions that run without any AI involvement. |
| `proactivity-core` | Turns selected events into candidate interruptions. The step from reactive software to an ambient assistant. |
| `model-router` | Routes each task by complexity, latency, privacy, availability, context size, and cost. Cheapest adequate intelligence by default, not the most expensive one. |
| `task-core` | Background work that outlives the conversation and the device it started on. |
| `tool-registry` | Digital capabilities behind stable, agent-native contracts — plus an SDK so new capabilities are plugins, not architectural changes. |
| `presence-core` | Multi-room presence and environmental context, so interaction follows the user while respecting privacy and uncertainty. |

Two concerns are deliberately **not** cores, because they are properties of every
core rather than a place in the system: **reliability** (failures degrade
predictably instead of collapsing) and **security** (authentication, network
isolation, secrets, tool permissions, prompt injection, audit logs, memory privacy,
device spoofing, microphone isolation).

The list has no end state by design. The goal is not to build everything that can be
imagined — it is to avoid making any of it unnecessarily difficult later.

## Why submodules

A submodule reference is a pinned commit, not a copy. That gives three properties
worth the small amount of ceremony:

1. Each core stays clonable, buildable, and releasable on its own.
2. One commit in this repository records a combination of nine cores that is known
   to work together.
3. There is exactly one source of truth per core. Nothing is duplicated, so nothing
   can drift.

## Getting started

```bash
git clone --recurse-submodules https://github.com/Brightwav3/Assistant-mark-I.git
```

`--recurse-submodules` is required; without it the core directories are empty.
One submodule (`activation-gemini-bridge`) is currently private and will be skipped
unless you have access.

To verify the assembled slice:

```bash
cd assistant-runtime && npm install && npm run verify
```

## Current state

The first usable slice runs today: double-clap activation, Gemini Live native audio,
microphone routing, barge-in cancellation, conversation summaries persisted to
SQLite that survive a restart, inactivity timeout, and safe shutdown. It is verified
by 15 passing tests plus typecheck and build.

Each core repository documents its own verified state and known limitations in its
`README.md` and `PROGRESS.md`.

## License

[PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/).
See [LICENSE](./LICENSE).

Use is permitted for noncommercial purposes only — personal use, research,
education, and public-benefit work. Any use by or for a business requires a
separate license. This applies to the meta-repository; each core is a separate
repository and carries its own terms.
