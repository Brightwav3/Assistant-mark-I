## Status: Mark I complete — proof of concept

Assistant-mark-I is a completed meta-repository proof of concept. It proves on
real Windows hardware that the native Gemini Live path, interruption, memory,
bounded safe tools, and independent-core composition can work together.

This is a proof of the architecture, not a production-ready full-duplex
assistant. Mark II focuses on intelligence, delegation, serious tools, and
mature half-duplex interaction. Native full duplex, semantic timing, overlap,
backchannels, and continuous listening belong to Mark III.

# Assistant mark I

[![Integration](https://github.com/Brightwav3/Assistant-mark-I/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/Brightwav3/Assistant-mark-I/actions/workflows/ci.yml)
[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-4c1d95)](https://polyformproject.org/licenses/noncommercial/1.0.0/)
[![Architecture: git submodules](https://img.shields.io/badge/Architecture-git%20submodules-6f42c1)](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

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

## The full-duplex problem

The native realtime path is intentionally a foundation for full-duplex, not a
full-duplex assistant yet. [full-duplex-attempts](https://github.com/Brightwav3/full-duplex-attempts)
records the experiments, criteria, and limitations behind that distinction.

`Assistant-mark-I` currently runs a native bidirectional audio session through
Gemini Live. That solves the media plumbing, but Gemini Live is still a
generation-2, turn-based model: it waits for the user to stop before responding
and cannot backchannel while the user is speaking. This is a model limitation,
not a missing state transition or a silence timer that can be tuned away. True
full-duplex requires a [generation-3 model](https://github.com/Brightwav3/full-duplex-attempts/blob/main/docs/what-full-duplex-requires.md)
that can continuously process both directions and decide when to speak.

The current architecture still has one major internal gap that is independent of
model availability, while the tool path is now bounded and verified:

- **Bounded native realtime tool path.** `RealtimeSessionConfig` declares the
  active safe catalogue and `RealtimeSpeechEvent` carries correlated tool
  requests. The Tool System validates arguments, applies policy, executes the
  Host Tools catalogue, and returns the result to the native session. The
  default read-only path is hardware-verified; side-effecting tools such as
  `open_app` remain explicit opt-ins.
- **No shared acoustic signal path.** Scribe Core owns capture and Realtime Core
  owns playback, deliberately as independent repositories. Acoustic echo
  cancellation needs both signals on one timeline, so the current decomposition
  gives AEC nowhere to live while the microphone stays open during playback.

This is why the problem is not one missing feature. The model sets the ceiling;
the application still has to provide delegation, cancellation, and an audio
boundary that can handle echo. The intended outcome is to solve the internal gaps
now and keep the provider contract stable so a real generation-3 model can be
added later without rebuilding the assistant around it.

See [the Assistant mark I attempt](https://github.com/Brightwav3/full-duplex-attempts/blob/main/docs/attempts/assistant-mark-i.md)
for the dated criteria and three walls, and [the full-duplex attempts repository](https://github.com/Brightwav3/full-duplex-attempts)
for the comparison with the cascade attempt.

### Native runtime path

<p align="center">
  <img src="https://raw.githubusercontent.com/Brightwav3/Assistant-mark-I/99c002b6098db76b4aab349d39e039a6d213dc1e/docs/images/mark-i-runtime-path.png" width="960" alt="Assistant Mark I native runtime path" />
</p>

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
| [`assistant-runtime`](./assistant-runtime) | cross-core composition, interaction lifecycle, realtime tools, and operator diagnostics | **Mark I complete** — native realtime slice verified on hardware; modular path deferred |
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
| ❌ | Interaction Core | Coordination of conversational flow between the speech subsystems, if it turns out to be needed. |
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

The first usable slice runs today, and it has been used: on 2026-08-13 the native
path was verified end to end on real hardware — a double clap activated the
assistant, a Gemini Live session started, it answered out loud, interrupting it
stopped playback immediately, it wrote a summary of the conversation to SQLite,
the interaction timed out on its own, and after a restart it still knew what had
been said.

The model can also act. In the Mark I default composition, Gemini discovers the
safe read-only `get_time`, `calculate`, `uptime`, and `system_status` tools;
Tool System validates the arguments, consults policy, applies its guards, and
returns the result to the realtime session. The default tool path was exercised
on hardware with time, system-status, and multi-step calculation requests.
Side-effecting tools such as `open_app` remain explicit opt-ins. A denied tool
reaches nothing, a hallucinated argument is rejected before any launch, an
approval flag invented by the model is an undeclared argument rather than a
permission, and content returned from outside is labelled as data rather than
instruction. Every one of those is a test, not an intention.

Every core is also verified automatically on each push, and the meta-repository
builds them all and re-runs the composed slice.

Not yet verified on hardware: the modular Scribe → Intelligence → Voice path.
The native realtime tool loop is hardware-verified for the safe default
catalogue; the explicit `open_app` process-launch probe remains separate.

Each core repository documents its own verified state and known limitations in its
`README.md` and `PROGRESS.md`.

## License

[PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/).
See [LICENSE](./LICENSE).

Use is permitted for noncommercial purposes only — personal use, research,
education, and public-benefit work. Any use by or for a business requires a
separate license. This applies to the meta-repository; each core is a separate
repository and carries its own terms.
