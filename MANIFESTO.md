# The Jarvis Manifesto

## A persistent, ambient, model-independent personal assistant

Jarvis is infrastructure through which intelligence can become a persistent, useful part of a person's digital and physical environment.

Jarvis is not primarily a chatbot, a smart speaker, a desktop application, a mobile application, an animated avatar, or an LLM wrapper. Those may be interfaces or components. They are not the system.

## Principles

1. **The assistant is larger than its interface.** Voice, text, displays, devices, and future channels are independent ways to reach the same assistant.
2. **The model is a component.** Models will change. Memory, tools, permissions, state, devices, and interaction contracts should survive provider changes.
3. **Infrastructure comes before personality.** Build the body and nervous system before adding decoration.
4. **Headless and agent-first by default.** Important capabilities must work through typed, discoverable, machine-readable contracts without GUI scraping.
5. **Deterministic responsibilities stay deterministic.** Use AI for reasoning where it helps; do not make lifecycle, validation, storage, or safety depend on model behavior.
6. **State is not memory.** Current facts, historical knowledge, and inferred context are different kinds of data with different lifecycles.
7. **Tools belong to the platform.** Models may request actions, but capabilities, permissions, validation, execution, and auditability belong to Jarvis.
8. **External information is untrusted.** Web pages, messages, files, and provider output are data, not authority.
9. **Privacy is intentional.** The ability to observe something does not imply that it should be recorded or remembered.
10. **Reliability matters more than demos.** Restart, timeout, reconnect, cancellation, degraded operation, and safe shutdown are product behavior.
11. **Integrate mature infrastructure where it is better.** Jarvis should own its abstractions and boundaries without rebuilding solved problems unnecessarily.
12. **Grow vertically.** Every stage should produce one real, testable capability before another major layer is added.

## The first useful shape

The first meaningful Jarvis slice is:

```text
activation
    → speech or realtime audio
    → assistant runtime
    → replaceable intelligence
    → optional tools and memory
    → response through the active interface
```

It should be useful before it is comprehensive, and replaceable before it is optimized for one provider.

## What success means

The user should not need to think, “I need to open my AI.” They should be able to speak, type, tap, move between rooms, use a computer, or receive a notification while the same assistant remains present behind those interactions.

The long-term test is simple:

> A better model should fit into the same assistant without requiring the nervous system to be rebuilt.

**Build the body and nervous system so every future generation of intelligence has somewhere useful to live.**
