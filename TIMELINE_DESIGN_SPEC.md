# Case Timeline — Product & Design Workshop Brief

**Version:** 1.0

**Date:** September 8, 2026

**Owner:** David + Design

**Status:** Draft for product and design alignment

---

## Table of Contents

1. [Purpose](#purpose)
2. [Workshop Goals](#workshop-goals)
3. [Design Principles](#design-principles)
4. [Core Concept — Timeline Event](#core-concept--timeline-event)
5. [Handler Needs and Use Cases](#handler-needs-and-use-cases)
6. [Desired Experience](#desired-experience)
7. [Product and Data Considerations](#product-and-data-considerations)
8. [Dependencies and Alignment](#dependencies-and-alignment)
9. [Out of Scope](#out-of-scope)
10. [Success Criteria](#success-criteria)

---

## Purpose

Claims handlers must manually reconstruct case history across multiple systems, including case notes, documents, status history, Salesforce, and communication threads. When inheriting or revisiting a case, gathering that context takes 10–15 minutes and risks missing critical details, especially in complex cases such as dental claims awaiting a vet response, linked conditions, or multi-year histories.

**The case timeline should give handlers a trustworthy, chronological account of what happened so they can understand the current situation in 30 seconds.**

This matters most for:

- **Handoffs:** An incoming handler can see where the case stands and what is blocking progress.
- **New document arrivals:** A handler can understand what arrived and how it relates to prior activity.
- **Recurring conditions:** A handler can follow relevant history across linked cases and prior decisions.
- **Dental workflows:** A handler can see when a vet was contacted, when a response arrived, and what changed afterward.
- **Compliance and audit:** Actions are timestamped and attributed to a person, customer, or system.

## Workshop Goals

Use this brief to align on:

- the handler problem the timeline must solve;
- the product principles that should guide later design decisions;
- the minimum story each timeline event must communicate;
- the experience handlers should have when reviewing a case;
- the boundaries of the first useful release; and
- how we will know the timeline has succeeded.

The workshop should produce shared direction, not an implementation specification.

## Design Principles

### P1 — Chronological Ordering

Events appear in reverse chronological order, with the newest events first. The sequence must remain clear and must not be obscured by grouping.

**Why:** Handlers usually begin with “what just happened?” and then work backward when they need more history.

### P2 — Accuracy and Precision

The timeline reflects facts from their originating sources without inference or approximation. The timing, source, and meaning of an event must be trustworthy.

**Why:** Handlers make decisions based on sequence and context. Uncertain or misleading history undermines confidence in the whole timeline.

### P3 — Simplicity in Presentation

Each event emphasizes the essential story: what happened, when, and who or what caused it. Internal system language is translated into handler language, for example “Dokument lastet opp” rather than a technical event code.

**Why:** The timeline must reduce cognitive load and support a rapid scan.

### P4 — One Coherent Case Story

The timeline brings together relevant activity from multiple sources without forcing handlers to reconstruct the story system by system.

**Why:** The value comes from a coherent account of the case, not another isolated view.

### P5 — Handler-First Accessibility

The timeline is easy to find and useful when a handler opens a case. Finding recent activity and relevant history should require minimal effort.

**Why:** If the timeline is difficult to access or navigate, handlers will return to fragmented existing workflows.

### P6 — Immutability

Historical events are not rewritten or deleted. A correction is represented as a new, attributable event.

**Why:** Claims handling requires a dependable audit trail and a transparent record of how understanding changed over time.

### P7 — Attribution

Every event makes clear whether it came from a handler, customer, external party, or system.

**Why:** Handlers need to understand the origin and intent of an action, especially during handoffs.

## Core Concept — Timeline Event

A **timeline event** is a single, immutable fact about something that happened in the life of a case.

Every event must communicate:

- **What happened?** A concise, handler-friendly description of the action or change.
- **When did it happen?** The time recorded by the originating source.
- **Who or what caused it?** The responsible handler, customer, external party, or system.
- **Why does it matter?** Enough context to understand the event’s relevance and follow the underlying source when needed.

Events may represent documentation, communication, decisions, case-management changes, or system activity. Examples include a document arriving, a note being added, a status changing, a vet responding, a payout decision being made, or a handler being assigned.

The workshop should focus on whether these event categories tell a complete and understandable case story, rather than defining technical fields or event-specific payloads.

## Handler Needs and Use Cases

### Primary Use Cases

1. **Picking up a case:** The handler needs the complete story and current position in less than 30 seconds.
2. **Responding to new activity:** The handler needs to understand what just happened and how it relates to earlier events.
3. **Continuing another handler’s work:** The handler needs to identify completed actions, pending dependencies, and the reason for the current status.
4. **Reviewing a complex or recurring case:** The handler needs to trace decisions, documents, communications, and relevant linked history in sequence.
5. **Understanding a dental claim:** The handler needs to follow contact with the vet, the response, and any resulting change in assessment or calculation.
6. **Explaining or auditing a case:** The handler needs a reliable record of who or what acted, when, and on what basis.

### Core Product Needs

- Relevant case events are brought together in one chronological story.
- Recent activity is immediately apparent while older history remains available.
- Events remain easy to scan and use language familiar to handlers.
- Attribution and source are clear enough to establish trust.
- Supporting detail or source material can be reached when the summary is not sufficient.
- Common case histories remain quick and reliable to access, including cases with many events.
- Standard and domain-specific activity, such as dental communication, fits into the same understandable sequence.

## Desired Experience

When a handler opens a case, the timeline should answer three questions quickly:

1. **What is the current situation?**
2. **What happened most recently?**
3. **What should I inspect to understand or continue the case?**

The experience should feel calm, factual, and scannable. Summaries should use recognizable Norwegian handler language where appropriate, such as “Dokument lastet opp,” “Status endret,” “Svar fra veterinær,” and “Kunde.” More context should be available when needed without making every event visually dense.

The timeline should support orientation first and deeper investigation second. It should not require handlers to understand source-system terminology or infer relationships between disconnected records.

## Product and Data Considerations

The product depends on a consistent interpretation of an event across source systems. Alignment is needed on:

- which activities are important enough to appear in the case story;
- which source owns the factual time, attribution, and content of each activity;
- how corrections, duplicates, linked cases, and delayed source updates should be understood;
- what detail handlers need in the timeline versus in the originating record; and
- what level of completeness, freshness, and performance is necessary for handlers to trust the experience.

These considerations define the product contract for the timeline. Technical storage, interfaces, and delivery mechanisms should follow after that contract is agreed.

## Dependencies and Alignment

The timeline will rely on relevant categories of source information: documents, case notes, case and policy history, handler assignments, customer and external-party communications, decisions, and system-generated activity.

Product, design, claims operations, data, and source-system owners must align on which sources are authoritative, how their events should be described to handlers, and what constitutes a complete case story. The timeline should consume these sources without changing their underlying workflows or ownership.

## Out of Scope

The first release is not intended to provide:

- event creation, editing, or deletion;
- notifications or workflow automation;
- timeline export or printing;
- custom event types;
- saved searches, bulk actions, or other advanced discovery tools;
- a dedicated linked-case visualization; or
- mobile- and tablet-specific experiences.

These boundaries should be revisited only where they prevent the timeline from solving the core orientation and handoff problem.

## Success Criteria

### Quantitative Signals

1. Median context-gathering time is reduced from 10–15 minutes to 30 seconds or less.
2. At least 80% of handlers use the timeline as part of case review.
3. Timeline information is available in under one second for at least 95% of typical case views.
4. At least 95% of relevant case events are represented.

### Qualitative Signals

5. Handlers rate the timeline experience at least 4 out of 5.
6. Handlers report easier and more confident case handoffs.
7. Handlers trust the sequence, attribution, and meaning of events.
8. Early use reveals no recurring category of missing or incorrect event.

---

**Workshop outcome:** Shared agreement on the problem, goals, principles, desired handler experience, first-release boundaries, and success measures.

**Next step:** Capture the aligned product direction and identify any assumptions that need validation with handlers or source-system owners.
