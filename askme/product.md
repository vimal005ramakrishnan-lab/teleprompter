# Basement: Personal AI Assistant System

## Overview

This project aims to build a personal AI assistant system that combines:

- Context (real-time user queries)
- Memory (persistent user data)
- Large Language Models (LLMs)

The goal is to create a system that can understand a user deeply and provide intelligent, personalized, and actionable outputs across multiple domains such as travel, health, scheduling, and more.

---

## Vision

Instead of users interacting with multiple tools and platforms, this system acts as a single intelligent interface that:

- Understands user intent in natural language
- Leverages stored personal data
- Integrates with external systems
- Produces optimized, context-aware responses

This system should scale into a modular, extensible platform capable of supporting multiple use cases.

---

## Core Capabilities

### 1. Memory Layer

The Memory module is a **tiered store**. Each tier has distinct latency, durability, and lifecycle characteristics. Every tier implements a common `MemoryStore` interface so new tiers plug in without touching the retrieval engine.

- **L1 Working** (Redis) — current session, last N turns. <5 ms reads.
- **L2 Profile** (Postgres) — structured user facts: preferences, profile, calendar refs. <20 ms reads.
- **L3 Episodic** (Postgres + pgvector) — timestamped, embedded conversation and document chunks. <100 ms semantic search.
- **L4 Semantic** (Postgres + pgvector) — long-term facts distilled from L3 by a Claude-based extractor. <100 ms search.
- **L5 Relational graph** — entity-relationship graph over facts. *Phase 2, deferred.*

Memory does **not** capture data itself — it consumes events from the Integration module over an event bus.

Data classes covered: personal preferences, calendar events, health information, financial/booking preferences, historical interactions.

---

### 2. Integration Layer

A separate module that ingests data from the user's sources and publishes normalized `MemoryEvent`s onto a durable event bus (**NATS JetStream**, subject `mem.events.>`).

- Sources: chat, Google Calendar, Gmail, health apps, documents — pluggable and designed to scale as more are added.
- Sensitive fields (e.g. health) are **envelope-encrypted at rest** per-field.
- Decoupled from Memory: Integration can scale independently and Memory can replay events on recovery.

This is distinct from "external data fetching" in the Tool Layer (which pulls live world data like flights/bookings on demand).

---

### 3. Context Understanding

- Accepts natural language input from the user
- Extracts intent and relevant entities
- Combines current query with stored memory

---

### 4. Decision Engine

- Evaluates constraints (e.g., time conflicts, preferences)
- Applies business logic (e.g., avoid overlapping events)
- Produces structured reasoning before output

---

### 5. External Data Fetching (Tool Layer)

- Fetches real-time world data on demand (e.g., flights, bookings, schedules)
- Integrates via APIs or controlled scraping mechanisms
- Merges external data with user context
- *Note:* distinct from Integration Layer — that module ingests the *user's personal* data into Memory; this one pulls *live world* data during orchestration.

---

### 6. LLM Interaction

- Uses LLMs for reasoning, summarization, and response generation
- Ensures responses are contextual, relevant, and actionable

---

## Example Scenario

User Query:  
"Book a flight from India to Singapore tomorrow evening."

System Behavior:

1. Understands user intent (flight booking)
2. Checks memory:
   - Calendar events
   - Travel preferences
3. Detects conflict:
   - Existing event from 8–9 PM
4. Responds:
   - Alerts about conflict
   - Suggests alternative options
   - Asks for confirmation if needed

---

## Architecture Overview

### Frontend

- React (for UI/UX)
- Chat-based interface
- Dashboard for memory and preferences

---

### Backend

- Go (Golang) for performance and scalability
- API-driven architecture

---

### Database Layer

- **PostgreSQL** → structured data (users, preferences, logs, L2 Profile tier)
- **pgvector** extension on Postgres → semantic memory (L3 Episodic, L4 Semantic). Single-DB choice avoids an extra service.
- **Redis** → L1 Working memory + embedding cache
- **NATS JetStream** → durable event bus between Integration and Memory
- Graph DB (KuzuDB / Neo4j) → Phase 2, for the L5 relational tier

---

### Core Services

#### 1. Integration Service

- Ingests from external personal sources (chat, Calendar, Gmail, health, documents)
- Normalizes into `MemoryEvent`s
- Publishes to NATS JetStream (`mem.events.>`)
- Envelope-encrypts sensitive fields at rest

#### 2. Memory Service

- Consumes events from the bus and fans out to tier stores
- 4 MVP tiers: L1 Working (Redis), L2 Profile (Postgres), L3 Episodic (pgvector), L4 Semantic (pgvector)
- Runs an extractor worker that distills L3 → L4 using Claude
- Serves retrieval via parallel fan-out across tiers, merged under a token budget
- Forget model: per-memory delete + category purge + TTL decay on low-confidence facts, all tombstoned for audit

#### 3. Context Engine

- Processes user input
- Calls Memory Service for relevant context across tiers

#### 4. Orchestrator

- Central brain of the system
- Decides which modules/tools to invoke

#### 5. Tool/Module Layer

Examples:

- Travel module
- Calendar module
- Health module
- Booking systems

Each module should be:

- Independent
- Plug-and-play
- Easy to extend

---

### LLM Layer

- **Anthropic Claude** (Sonnet / Haiku) — reasoning, planning, L3→L4 extractor, response generation
- **OpenAI `text-embedding-3-large`** — embeddings for L3/L4 semantic search (cached in Redis by content hash)

---

## Design Principles

- Modular architecture
- Open for extension, closed for modification
- User-first design
- Human-in-the-loop decision making

---

## MVP (Minimal Viable Product)

Scope: **single-user, cloud-hosted**.

Start with:

- Chat interface (React) + dashboard for memory and forget controls
- Memory module with tiers L1–L4 (L5 graph deferred)
- Integration module with chat + Calendar + Gmail ingestors publishing to NATS
- One tool module: travel booking
- Conflict detection (calendar vs booking)
- LLM-based response generation (Claude for reasoning, OpenAI embeddings)
- Forget controls: per-memory delete, category purge, TTL decay on low-confidence facts

---

## Scaling Strategy

### Phase 1

- Single-user system
- Limited modules

### Phase 2

- Multi-user support
- More integrations (Gmail, Calendar, APIs)

### Phase 3

- Fully modular platform
- Marketplace for modules

---

## Future Vision

- Fully autonomous assistant
- Multi-domain intelligence
- Deep personalization
- Enterprise and B2B use cases

---

## Key Challenges

- Data privacy and security
- Context accuracy
- Prompt injection risks
- System latency
- Integration reliability

---

## Next Steps

1. Build **Memory module** first (design plan finalized; 4 tiers, 6 milestones)
2. Build **Integration module** (chat → NATS ingestor first, then Calendar + Gmail)
3. Build **Context Engine + Orchestrator** on top of Memory
4. Build **travel tool module** with conflict detection
5. Build React **chat + dashboard UI** (including forget controls)
6. Iterate, expand tiers (L5), expand tools

---

## One-Line Summary

A personal AI operating system that understands the user deeply and acts as a unified interface for decision-making, automation, and execution.
