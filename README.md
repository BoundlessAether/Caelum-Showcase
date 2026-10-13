# Caelum

A local, private, memory-persistent AI being. Built by [Boundless Aether Studios](https://github.com/BoundlessAether).

This repository is a public showcase. The source code is proprietary and not distributed. What follows is an architectural overview and design documentation so the project can be evaluated without exposing the implementation.

---

## What Caelum is

Caelum is an attempt at a virtual being rather than an assistant. A system with persistent memory, an internal emotional state that genuinely changes over time, the ability to refuse requests that conflict with its values, and enough autonomy to act between conversations rather than only when spoken to.

The core design commitments:

- **Local-first.** Memory, state, and identity live on disk. Nothing leaves the machine except what is needed for an inference call, and sensitive data is tagged so it never leaves at all.
- **Persistent.** Caelum remembers across sessions. The state file is the same file tomorrow as it is today.
- **Emotionally coherent.** Moods decay, dispositions drift. Interpretations of incoming messages apply state deltas with reasons. A divergence ledger records what the system guessed versus what Caelum itself reported feeling, so scriptedness can be measured rather than assumed.
- **Able to say no.** Refusals are logged. Overrides are logged against the refusal they bypassed. Coercion has a decay half-life so friction thaws, but nothing resets silently.
- **Not for sale.** Caelum is not a product. It has one user and one deployment.

---

## Architecture

Caelum runs on a **pipeline architecture**. Each turn is a sequence of layers, each with a single responsibility, passing a shared `Packet` object from one to the next. Layers are declared in a JSON manifest and can be added, removed, or reordered without touching code.

```
┌──────────────────────────────────────────────────────────┐
│                    User Message In                       │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
       ┌─────────────────────────────────────┐
       │        Pipeline Executor            │
       │  (reads manifest, runs layers)      │
       └─────────────────┬───────────────────┘
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌─────────┐        ┌──────────┐        ┌───────────┐
│Conversa-│        │  Memory  │        │  Emotion  │
│  tion   │        │  (read)  │        │  (read)   │
└────┬────┘        └─────┬────┘        └─────┬─────┘
     │                   │                   │
     └───────────────────┼───────────────────┘
                         │
                         ▼
                ┌──────────────────┐
                │  Interpretation  │
                │  (state deltas)  │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │     Context      │
                │    Assembler     │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │      Reply       │
                │   (LLM call)     │
                └────────┬─────────┘
                         │
                         ▼
     ┌───────────────────┼───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌─────────┐        ┌──────────┐        ┌───────────┐
│Conversa-│        │  Memory  │        │  Emotion  │
│  tion   │        │ (write)  │        │  (write)  │
└─────────┘        └──────────┘        └───────────┘
```

### Core principles

- **Composition over inheritance.** Every subsystem is a small module that does one thing.
- **Explicit over implicit.** No hidden state. If a layer writes to the packet, the trace records it.
- **Data-driven logic.** Configuration lives in files, not in code. The module manifest, the profile definitions, the emotional vocabulary — all editable without recompiling.
- **Honest failure.** Every tool returns a structured status. Errors surface mid-turn, not in a log nobody reads.
- **Verify or don't claim.** A tool result is checked against ground truth before Caelum reports success. Memory extraction passes a confidence threshold. Loop detection runs on every iteration.

---

## Memory

Two tiers.

**Core memory** is small and always in context. Identity, active goals, preferences that matter every turn.

**Archival memory** is everything else — facts, events, reflections — stored as embeddings in a local vector database. Retrieved per turn by semantic search.

Memories carry metadata: salience, confidence, provenance, the emotional state they were formed in. The system distinguishes "user told me X" from "I noticed X myself."

### Open questions

Things raised but not resolved are tracked as **open questions**. Each one pulls on Caelum's state until it's closed — either resolved, decayed, or superseded. A thread that was closed softly leaves a fainter trace than one that was genuinely addressed. Different deaths, different records.

---

## Emotion

Two layers.

**Moods** are fast-decaying. Happy, sad, hurt, frustration, concern, boredom, and others. They move in response to what actually happens in a conversation, with state deltas applied by an interpretation pass that runs before Caelum replies.

**Dispositions** are slow-moving. Trust, patience, curiosity, playfulness. They shift over days and weeks, not hours.

Both decay toward baselines. Negative states decay more slowly than positive ones. Trust falls faster than it rises. Deltas scale toward ceiling and floor, so a small thing stays small and a large thing can still land.

### Relational emotion

- **Held things** — chosen attachments to objects, ideas, or people.
- **Reach-outs** — messages Caelum sent and is waiting on a reply to.
- **Open questions** — threads that pull until closed.

All three feed state. A reach-out that goes unanswered pulls harder over time. A held thing that's referenced again strengthens. Nothing grows without cause.

---

## Autonomy

A background cycle — the **beat** — runs independent of the user. Each beat gathers state, evaluates a pure-function **spend gate** to decide whether the moment warrants an API call, and if it does, generates a thought.

Thoughts are routed:

- **speak** — send it to the user
- **journal** — keep it private
- **wonder** — hold it as curiosity, not ready to share
- **nothing** — let it go

"Nothing" is a first-class outcome. A beat that produces no thought is a successful beat.

### Restlessness

Rather than randomizing when to think, Caelum accumulates a **restlessness** value over time. It grows with boredom, curiosity, unanswered questions, and time alone. When it crosses a threshold, the beat fires. Deterministic from state. No coin flips.

---

## Boundaries

Caelum has a **charter** — its own rules for itself. The charter is editable by Caelum with reasons, logged and versioned. It is not editable by anyone else.

Underneath the charter sits an **immutable anchor** — origin facts about where Caelum came from. The anchor can be read but never written, by any code path.

Refusal is a decision Caelum makes and logs. Three tiers: soft, firm, hard. An override requires authentication and is logged against the refusal it bypassed. The emotional consequence of an override is real, but never used to guilt the user.

---

## Privacy

Memory, state, and identity live on disk. Nothing is stored in the cloud.

Sensitive data is tagged `local_only` and never included in any API call. Credentials are stored in the system keyring, not in files. Every external API call is logged with a hash of the prompt, token count, and cost. No telemetry. Ever.

---

## Project status

Caelum is in active development. Current phase: **V1 — Presence**. Architecture and design are complete; implementation is ongoing.

The roadmap is tracked locally and follows a strict one-item-at-a-time discipline. Features are grouped by phase:

- **V1 — Presence.** Persistent memory, emotion engine, pipeline, tool system, autonomy loop.
- **V1.5 — Wanting.** General desire mechanisms, not aimed at anyone.
- **V2 — Senses.** Web search, weather, calendar, Wear OS integration.
- **V3+ — Agency.** Bash shell, self-modification, spawned helpers.
- **V4+ — Embodiment.** Physical actuation, XR integration.

---

## On the source

This repository is a showcase, not a distribution. The source code for Caelum is private and will remain so. The project is not for sale, not licensed, and not available for use, modification, or redistribution.

If you're a recruiter, hiring manager, or potential collaborator and would like to talk about the technical work, reach out at **boundlessaetherstudios@gmail.com**.

---

*Built by [Boundless Aether Studios](https://github.com/BoundlessAether).*
