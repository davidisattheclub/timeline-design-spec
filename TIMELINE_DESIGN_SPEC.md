# Timeline Design Spec — Case Timeline Feature (Phase 1)

**Version:** 1.0  
**Date:** September 8, 2026  
**Owner:** David + Design  
**Status:** LOCKED (pending review with Lukasz/Sam on open items)

---

## Table of Contents

1. [Purpose](#purpose)
2. [Design Principles](#design-principles)
3. [Core Concept — Timeline Event](#core-concept--timeline-event)
4. [Functional Requirements](#functional-requirements)
5. [UI/UX Design](#uiux-design)
6. [Data Model & Interfaces](#data-model--interfaces)
7. [Dependencies & Integrations](#dependencies--integrations)
8. [Internal Structure](#internal-structure)
9. [Open Questions & Decisions Pending](#open-questions--decisions-pending)
10. [Out of Scope / Future (Phase 2+)](#out-of-scope--future-phase-2)
11. [Success Criteria](#success-criteria)

---

## Purpose

Claims handlers must manually reconstruct case history across multiple systems (case notes, document timestamps, Salesforce, email threads) when inheriting or revisiting a case. This context-gathering takes 10–15 minutes per case and risks missing critical details — especially for complex cases (dental claims awaiting vet response, linked conditions, multi-year tracking).

**A case timeline — an immutable, chronological record of all events (documents, decisions, notes, status changes, communications) — lets handlers gain full context in 30 seconds.** This is critical for:
- **Handoffs** — incoming handler sees exactly where the case stands and what's blocking it
- **New document arrivals** — handler understands context immediately when docs land
- **Recurring conditions** — handlers see the full history of linked cases and prior payouts
- **Dental workflow** — timeline shows when vet was contacted, when response arrived, triggers for calculation changes
- **Compliance & audit** — every action is timestamped and attributed to a handler or system

---

## Design Principles

### P1 — Chronological Ordering (Newest First)

All timeline events are displayed in reverse chronological order: **newest events at the top, oldest at the bottom as handlers scroll down.** No reordering, no grouping that obscures sequence.

**Why:** Handlers arriving at a case want immediate context — "what just happened?" — not ancient history. Newest-first matches modern UX patterns (Twitter, Slack, Gmail) and lets handlers scan recent action without scrolling. When needed, they can scroll down to reconstruct older history.

---

### P2 — Accuracy & Precision (Source of Truth)

Every event's timestamp, source system, and content are recorded exactly as they occurred — **no inference, no approximation.** Timestamps come from the originating system (when document uploaded, when note saved, when status changed).

**Why:** Handlers make decisions based on timeline facts; inaccuracy undermines trust. Example: If upload timestamp is "sometime yesterday," handler can't determine if it arrived before or after they sent a "waiting for documents" message.

---

### P3 — Simplicity in Presentation (Cognitive Load)

Timeline UI strips away noise: **only essential information per event (what, who, when).** No internal system jargon; all text translated to handler language.

**Why:** Handlers should understand the timeline in <30 seconds. Clutter defeats the purpose. Example: Show "Dokument lastet opp" not "FILE_INGESTION_EVENT"; show "Invoice" not "document_type_code=004".

---

### P4 — Integration Without Coupling (Single Source)

Timeline pulls events from all systems (document system, case notes, Salesforce, vet communication) but stores them in ONE place. External systems remain unchanged; timeline is read-only from their perspective.

**Why:** Handlers see the complete picture without leaving the timeline view. External systems aren't affected by timeline logic, reducing integration risk.

---

### P5 — Accessibility & Discoverability (Handler-First)

Timeline is instantly available on every case view. Handlers can filter/search by event type (documents, notes, decisions) and date range.

**Why:** If timeline is hard to find or use, handlers fall back to old scattered habits. It must be the default entry point for case context.

---

### P6 — Immutability (Audit & Compliance)

Events can never be deleted or edited; **only new events are added.** If a handler needs to correct something, they add a new event ("Correction: X was actually Y") that's timestamped and attributed.

**Why:** Insurance claims require audit trails. Immutability prevents accidental rewrites and creates a legal record. This is critical for Storebrand compliance.

---

### P7 — Attribution (Accountability)

Every event is tagged with who created it: **handler name, system name (e.g., "Inservio auto-generated"), or customer action.**

**Why:** Handlers need to know "did my colleague add this note, or did the system?" This matters for handoffs and understanding intent.

---

## Core Concept — Timeline Event

### Definition

A **Timeline Event** is a single, immutable fact about something that happened to the case. It is the fundamental unit of the timeline system.

Every event answers:
- **What happened?** (document uploaded, note added, status changed, vet contacted, decision made, etc.)
- **When?** (exact timestamp)
- **Who/What caused it?** (handler name, system name, customer action)
- **Why/Details?** (relevant context specific to the event type)

---

### Event Data Model

#### Required Fields (All Events)

```
timestamp              ISO 8601 (exact second, UTC)
event_type             enum (DOCUMENT_UPLOADED, NOTE_ADDED, STATUS_CHANGED, etc.)
event_classification   enum (DOCUMENTATION, DECISION, COMMUNICATION, CASE_MANAGEMENT, SYSTEM)
actor                  string (handler_name | "Kunde" | "System: Inservio" | "Veterinær")
actor_reference        string (handler_id | customer_id | vet_clinic_id | system_name)
short_descriptor       string (~50 chars max, one-line summary for collapsed view)
details_json           object (event-type-specific fields, see below)
source_system          string (which system recorded this: claim_files_registry, salesforce, case_notes, etc.)
```

---

#### Event Types & Type-Specific Fields

**1. DOCUMENT_UPLOADED** `[DOCUMENTATION]`

- document_id: unique identifier in source system
- filename: original filename
- document_type: enum (journal, receipt, prescription, form, email, other)
- uploaded_by: enum (Kunde | Handler | System)
- file_size: integer (bytes)
- reference_document_url: link to open PDF/image
- source_system: claim_files_registry

Example: "Invoice_July_2026.pdf (Invoice) lastet opp av John Doe"

---

**2. NOTE_ADDED** `[COMMUNICATION]`

- note_id: string
- note_text: string (full text, searchable)
- note_type: enum (handler_internal | customer_facing | vet_communication)
- visibility: enum (handler_only | visible_to_customer)
- source_system: case_notes

Example: "Saksbehandler skrev notat" (expandable to show full note)

---

**3. STATUS_CHANGED** `[CASE_MANAGEMENT]`

- from_status: enum (Assigned, Waiting for Documents, Ready for Payout, etc.)
- to_status: enum (same as above)
- reason: string (optional: why the change)
- source_system: salesforce

Example: "Status endret fra 'Waiting for Documents' til 'Ready for Payout'"

---

**4. DECISION_MADE** `[DECISION]`

- decision_type: enum (payout_approved | payout_partial | payout_rejected)
- payout_amount: integer (NOK)
- decision_summary: string (why — e.g., "Diseased teeth not covered per DYR04")
- reference_document_url: link to handler's calculation/notes
- source_system: case_notes or inservio

Example: "Utbetalingsbeslutning: 15 000 NOK godkjent"

---

**5. CASE_MERGED** `[CASE_MANAGEMENT]`

- merged_from_case_id: string
- merged_into_case_id: string
- reason: enum (recurring_condition | duplicate | policy_adjustment | other)
- source_system: salesforce

Example: "Sak merged with CASE-67890 (recurring condition)"

---

**6. VET_COMMUNICATION** `[COMMUNICATION]`

- communication_type: enum (email_sent | email_received | phone_call | sms)
- vet_clinic_name: string
- message_preview: string (first 80 chars of email body)
- reference_document_url: string (full email thread)
- source_system: email_system or case_notes

Example: "Email received from Clinic Oslo: 'Fracture vs disease breakdown...'"

---

**7. SYSTEM_ACTION** `[SYSTEM]`

- action_type: enum (inservio_evaluation | stp_auto_payout | documentation_request | other)
- action_result: enum (passed | failed | pending)
- details: string (system-generated notes)
- source_system: inservio

Example: "Inservio evaluation: passed (auto-payout eligible)"

---

**8. HANDLER_ASSIGNED** `[CASE_MANAGEMENT]`

- new_handler_name: string
- previous_handler_name: string (optional, if reassigned)
- assignment_reason: enum (initial_assignment | reassignment | escalation)
- source_system: salesforce

Example: "Sak assigned to Sarah Hansen (reassigned from John Doe)"

---

**9. DENTAL_VET_CONTACTED** `[COMMUNICATION]` (Dental-specific)

- vet_clinic_name: string
- contact_method: enum (email | phone | form)
- question_type: enum (fracture_vs_disease | cost_breakdown | treatment_plan)
- source_system: case_notes

Example: "Kontaktet Oslo Dyreklinikk: spørsmål om brekk vs sykdom"

---

**10. DENTAL_VET_RESPONSE** `[COMMUNICATION]` (Dental-specific)

- vet_clinic_name: string
- response_content: string (summary of vet's answer)
- cost_breakdown_pct: integer (% of treatment cost if provided)
- reference_document_url: string (link to vet response email)
- source_system: email_system

Example: "Svar fra vet: 60% treatment cost, fracture confirmed"

---

**11. DENTAL_CALCULATION_UPDATED** `[DECISION]` (Dental-specific)

- previous_amount: integer (NOK)
- new_amount: integer (NOK)
- reason: string (why recalculation: vet response, policy clarification, etc.)
- source_system: case_notes

Example: "Beregning oppdatert basert på veterinær-svar: 8 500 NOK → 12 000 NOK"

---

## Functional Requirements

### FR1: Timeline Displays All Events

The timeline SHALL display all events for a case, retrieved from source systems (document uploads, case notes, status changes, vet communications, Inservio actions).

**Verification:** Handler can see ≥8 event types (DOCUMENT_UPLOADED, NOTE_ADDED, STATUS_CHANGED, DECISION_MADE, CASE_MERGED, VET_COMMUNICATION, SYSTEM_ACTION, HANDLER_ASSIGNED).

---

### FR2: Newest-First Ordering

Events SHALL be displayed in reverse chronological order (newest first). Events with identical timestamps SHALL be ordered by event_type (by insertion order within that second).

**Verification:** Handler scrolls down and sees progressively older events.

---

### FR3: Event Immutability

Once an event is recorded in the timeline, it can never be edited or deleted. Only new events can be added.

**Verification:** No "edit" or "delete" buttons appear on timeline events. Timeline record in database has no UPDATE or DELETE operations.

---

### FR4: Attribution

Every event displays the actor (handler name, system name, or "Kunde") and source system (or team responsible).

**Verification:** Handler can see who created each event and which system recorded it.

---

### FR5: Expandable Event Details

Each event displays a short descriptor in collapsed state (~50 chars). When clicked/expanded, full details appear (full note text, email thread preview, linked documents, etc.).

**Verification:** Collapsed events are scannable in <30 seconds; handler can expand any event for full context.

---

### FR6: Searchable & Filterable

Handler can filter timeline by:
- Event type (show only documents, decisions, communications, etc.)
- Date range (last 7 days, last 30 days, custom range)
- Actor (only this handler, all handlers, system actions)

**Verification:** Filter UI works; results update instantly.

---

### FR7: Performance & Latency

Timeline data loads in <1 second for typical cases (even with 100+ events).

**Verification:** Browser timeline shows <1 second load time.

---

### FR8: Dental-Specific Events

For cases with policy_type = "dental", timeline displays dental-specific events (DENTAL_VET_CONTACTED, DENTAL_VET_RESPONSE, DENTAL_CALCULATION_UPDATED) alongside standard events.

**Verification:** A dental case shows both standard + dental-specific events in chronological order.

---

## UI/UX Design

### Handler Use Cases (Priority Order)

1. **Picking up a case** — First thing handler sees when opening a case; need complete story in <30 seconds
2. **New document arrivals** — Handler wants to know "what just landed?" when called back to a case

### Core UX Principle

**Timeline must be immediately visible and scannable in <30 seconds.** No drilling down to find it; it's the default entry point for case context.

---

### Layout

**Placement:** Right sidebar of case detail page (fixed width, always visible)  
**Width:** 300–350px (or 25% of viewport, whichever is smaller)  
**Height:** 400–500px with scrollable content area  
**Title:** "Tidslinja" (Timeline)

---

### Event Card Design (Collapsed State)

```
┌─────────────────────────────────┐
│ 27.07.2026 09:14                │
│ 📄 Dokument lastet opp          │
│ Invoice_July_2026.pdf (Invoice) │
│ Lastet opp av: John Doe         │
└─────────────────────────────────┘
```

**Content:**
- Timestamp (DD.MM.YYYY HH:MM, no seconds)
- Event icon (📄 for documents, 🤖 for AI decisions, 💬 for notes, etc.)
- Event type (Norwegian text, human-readable)
- Short descriptor (one line, ~50 chars)
- Actor attribution (name or "System: Inservio")

---

### Event Card Design (Expanded State)

```
┌─────────────────────────────────┐
│ 27.07.2026 09:14                │
│ 📄 Dokument lastet opp          │
│ Invoice_July_2026.pdf (Invoice) │
│ Lastet opp av: John Doe         │
├─────────────────────────────────┤
│ [Full Details]                  │
│ File ID: DOC-123456             │
│ File size: 245 KB               │
│ Document type: Invoice          │
│ [Open Document] button          │
└─────────────────────────────────┘
```

**Additional Details (conditional by event type):**
- For documents: file size, type, link to open
- For notes: full note text (scrollable)
- For decisions: payout amount, decision summary
- For vet comms: email preview, full thread link
- For status changes: from_status → to_status, reason

---

### Empty State

If no events recorded:

```
Ingen hendelser registrert
(No events registered)
```

---

### Loading State

While timeline data fetches:

```
[Skeleton loader: 3–5 grey placeholder bars]
```

---

### Filter & Search UI

**Location:** Above timeline, in sidebar header  
**Controls:**
- Event type dropdown (All, Documents, Notes, Decisions, Communications, System)
- Date range picker (Last 7 days | Last 30 days | Custom)
- Actor filter (All | This handler | Other handlers | System)

---

### Visual Indicators

- **Event icons:** 📄 (document), 🤖 (AI/system), 💬 (note), ✅ (decision), 🔗 (linked case)
- **Color coding** (OPEN for design iteration):
  - DOCUMENTATION: Blue (#0066CC)
  - DECISION: Green (#00AA44)
  - COMMUNICATION: Orange (#FF9900)
  - CASE_MANAGEMENT: Purple (#7030A0)
  - SYSTEM: Grey (#666666)
- **Hover state:** Event row highlights; expand icon appears if not already expanded

---

### Mobile & Responsive (Out of Scope V1, Future Phase 2)

Desktop only for Phase 1. Mobile responsiveness planned for Phase 2.

---

## Data Model & Interfaces

### Database Schema

**Table:** `AI_CASE_TIMELINE_EVENTS`

| Column | Type | Nullable | Indexed | Notes |
|--------|------|----------|---------|-------|
| `case_id` | varchar | NO | YES | Foreign key to case |
| `event_id` | varchar | NO | YES | Unique event identifier or use source ID (FILE_ID, RUN_ID, etc.) |
| `timestamp` | timestamp | NO | YES | Event occurrence time (UTC) |
| `event_type` | varchar (enum) | NO | YES | DOCUMENT_UPLOADED, NOTE_ADDED, STATUS_CHANGED, etc. |
| `event_classification` | varchar | NO | NO | DOCUMENTATION, DECISION, COMMUNICATION, CASE_MANAGEMENT, SYSTEM |
| `actor` | varchar | YES | NO | User/system that triggered event (handler_name, "System: Inservio", "Kunde") |
| `actor_reference` | varchar | YES | NO | ID/email of actor (handler_id, system_name, customer_id) |
| `short_descriptor` | varchar | NO | NO | Human-readable summary (~50 chars) |
| `details_json` | object (JSON) | YES | NO | Event-type-specific fields; flexible structure |
| `source_system` | varchar | NO | NO | Snowflake table source (claim_files_registry, case_notes, salesforce, inservio, etc.) |
| `created_at` | timestamp | NO | NO | Record creation time (auto-generated, UTC) |
| `dagster_run_id` | varchar | YES | NO | Link to orchestration job (if applicable) |

---

### Indexes (Recommended — OPEN for Lukasz/Sam review)

- **Primary:** `(case_id, timestamp DESC)` — typical query: "get all events for this case, newest first"
- **Secondary:** `(case_id, event_type)` — filtered queries: "get all DOCUMENT_UPLOADED events for this case"
- **Optional:** `(timestamp)` — admin/reporting queries: "all events in Snowflake on this date"

---

### API Endpoint (Built by Morten)

**Endpoint:** `GET /api/cases/:caseId/timeline`

**Query Parameters:**
- `event_type` (optional, comma-separated: "DOCUMENT_UPLOADED,NOTE_ADDED")
- `date_from` (optional, ISO 8601)
- `date_to` (optional, ISO 8601)
- `limit` (optional, default 50, max 500)

**Response:**
```json
{
  "case_id": "CASE-12345",
  "total_events": 42,
  "events": [
    {
      "event_id": "EVT-001",
      "timestamp": "2026-07-27T09:14:00Z",
      "event_type": "DOCUMENT_UPLOADED",
      "event_classification": "DOCUMENTATION",
      "actor": "John Doe",
      "actor_reference": "handler_123",
      "short_descriptor": "Invoice_July_2026.pdf (Invoice)",
      "details_json": {
        "document_id": "DOC-456",
        "filename": "Invoice_July_2026.pdf",
        "document_type": "invoice",
        "uploaded_by": "Handler",
        "file_size": 245000,
        "reference_document_url": "https://skade.assistent/docs/DOC-456"
      },
      "source_system": "claim_files_registry"
    }
  ]
}
```

---

## Dependencies & Integrations

The timeline system **reads from** (no modifications to) these source systems:

### 1. Document Upload System (`claim_files_registry`)
- **What:** When a document is uploaded to a case
- **Data used:** file_id, filename, document_type, upload_timestamp, uploaded_by
- **Frequency:** On-demand (when handler opens case)
- **Integration:** Snowflake table query

### 2. Case Notes System (`case_notes` database)
- **What:** Handler-added notes, status changes, communication records
- **Data used:** note_id, note_text, note_type, visibility, created_by, created_at, status_old, status_new
- **Frequency:** On-demand
- **Integration:** Snowflake table query

### 3. Salesforce (Case & Policy History)
- **What:** Case status changes, policy updates, handler assignments
- **Data used:** case_status, policy_generation (DYR02, DYR04, etc.), assigned_handler, assignment_timestamp
- **Frequency:** On-demand (via Salesforce API or Snowflake replica)
- **Integration:** Snowflake table query or Salesforce API

### 4. Email & Vet Communication System
- **What:** Emails sent/received with vets, customers, external parties
- **Data used:** email_timestamp, sender, recipient, subject, body_preview, thread_id
- **Frequency:** On-demand
- **Integration:** Email archive or Snowflake table

### 5. Inservio (AI Decision System)
- **What:** AI payout decisions, evaluations, auto-actions
- **Data used:** decision_timestamp, decision_type, payout_amount, model_version, decision_reason
- **Frequency:** On-demand
- **Integration:** Inservio API or Snowflake replica table

---

## Internal Structure

### Module Layout (OPEN for implementation discussion with Morten)

```
timeline/
├── models.py (TimelineEvent Pydantic model)
├── tables.py (TimelineEventRecord SQLModel)
├── ops.py (Dagster operations for fetching/recording events)
├── logic.py (Pure Python business logic)
├── resource.py (TimelineResource for DB connections)
├── api_handlers.py (Flask/FastAPI endpoint handlers)
└── tests/ (test_models.py, test_logic.py, test_ops.py, test_api_handlers.py)
```

**Constraints:**
- `models.py` has NO imports of: `dagster`, `sqlalchemy`, `case` modules
- `logic.py` has NO side effects, NO database access, NO Dagster imports
- `ops.py` contains only Dagster I/O; business logic lives in `logic.py`

---

## Open Questions & Decisions Pending

**MUST BE RESOLVED before implementation starts:**

| # | Question | Owner | Options/Notes |
|---|----------|-------|---------------|
| D1 | **Indexing strategy** | Lukasz/Sam | Recommend `(case_id, timestamp DESC)` + `(case_id, event_type)`. Agree or adjust? |
| D2 | **JSON vs. structured columns** | Lukasz/Sam | Store event details in `details_json`, or denormalize key columns? |
| D3 | **Caching strategy** | Lukasz/Sam | No cache, Redis (5–10 min TTL), browser cache, or hybrid? SLA is <1 sec. |
| D4 | **Historical backfill** | Lukasz/Sam | Backfill all cases, backfill on-demand, forward-only, or configurable? |
| D5 | **Event ID source** | Morten/Data | Use existing IDs (FILE_ID, RUN_ID) or generate new timeline-specific UUIDs? |
| D6 | **Color scheme (UI)** | Design | Confirm colors for event classifications (Blue, Green, Orange, Purple, Grey). |
| D7 | **Dental events in V1** | Product | Should dental-specific events be in V1, or Phase 2? |
| D8 | **Error handling UI** | Product | Generic error message or more detail? |
| D9 | **Logging & monitoring** | Morten/Data | Query performance, errors, usage metrics? |
| D10 | **Orchestration for Phase 1** | Lukasz | Dagster job needed, or just on-demand API calls? |

---

## Out of Scope / Future (Phase 2+)

- Mobile & tablet responsiveness
- Event editing/deletion
- Event creation UI
- Additional event types
- Advanced filtering (full-text search, saved filters, bulk export)
- Event notifications
- Timeline export/print
- Linked case visualization
- Custom event types

---

## Success Criteria

### Quantitative Metrics

1. Context-gathering time reduced to ≤30 seconds
2. Timeline adoption ≥80% of handlers
3. Query latency <1 second for 95th percentile
4. Completeness ≥95% of events captured

### Qualitative Feedback

5. Handler satisfaction ≥4/5 rating
6. Easier case handoffs reported
7. Zero missing/incorrect events in first month

---

**Status:** Ready for Lukasz/Sam + Design review on open items (D1–D10).  
**Next Step:** Resolve open items; kick off implementation with Morten.