# Basement — High Level Design

Companion to `product.md`. All diagrams are Mermaid — GitHub, VS Code, and most IDEs render them inline.

---

## 1. System context

Who talks to Basement, and which external services Basement talks to.

```mermaid
flowchart LR
    User([User])

    subgraph Basement["Basement (Personal AI OS)"]
        FE[React Frontend<br/>chat + dashboard]
        BE[Go Backend<br/>services]
    end

    subgraph External["External systems"]
        Claude[Anthropic Claude<br/>reasoning + extraction]
        OpenAI[OpenAI Embeddings<br/>text-embedding-3-large]
        GCal[Google Calendar]
        GMail[Gmail]
        Travel[Travel / Booking APIs]
        Health[Health apps<br/>future]
    end

    User <-->|chat, dashboard| FE
    FE <-->|REST / WebSocket| BE

    BE -->|prompt / extract| Claude
    BE -->|embed| OpenAI
    BE <-->|ingest| GCal
    BE <-->|ingest| GMail
    BE <-->|fetch live data| Travel
    BE <-->|ingest| Health
```

---

## 2. Component view

All services inside the backend, the event bus, and the data stores.

```mermaid
flowchart TB
    FE[React Frontend]

    subgraph Backend["Go Backend"]
        direction TB
        Int[Integration Service<br/>ingest + normalize + encrypt]
        Mem[Memory Service<br/>tiered store + retrieval]
        Ctx[Context Engine<br/>intent + entity extraction]
        Orc[Orchestrator<br/>decides modules/tools]
        Tool[Tool Layer<br/>travel, calendar, booking...]
    end

    Bus[[NATS JetStream<br/>mem.events.*]]

    subgraph Stores["Data stores"]
        Redis[(Redis<br/>L1 Working + embed cache)]
        PG[(PostgreSQL + pgvector<br/>L2 Profile, L3 Episodic, L4 Semantic, tombstones)]
        Graph[(Graph DB<br/>L5 — Phase 2)]:::phase2
    end

    subgraph LLMs["LLM providers"]
        Claude[Claude]
        OAI[OpenAI embeddings]
    end

    FE <-->|REST / WS| Orc

    GCal[Google Calendar]:::ext --> Int
    GMail[Gmail]:::ext --> Int
    Chat[Chat stream]:::ext --> Int

    Int -->|publish MemoryEvent| Bus
    Bus -->|consume| Mem

    Mem <--> Redis
    Mem <--> PG
    Mem -.Phase 2.-> Graph
    Mem -->|extract L3->L4| Claude
    Mem -->|embed| OAI

    Orc --> Ctx
    Ctx -->|retrieve context| Mem
    Orc --> Tool
    Tool -->|live world data| ExtAPIs[External APIs]:::ext
    Orc -->|reason + respond| Claude

    classDef phase2 stroke-dasharray: 5 5,fill:#f8f8f8
    classDef ext fill:#eef
```

---

## 3. Memory module — internal design

Zoom into the Memory Service. This is the part we're building first.

```mermaid
flowchart TB
    Bus[[NATS JetStream]]

    subgraph Memory["Memory Service"]
        direction TB

        subgraph Ingest["Ingest pipeline"]
            Consumer[Consumer<br/>durable subscriber]
            Router[Router<br/>by Kind / Sensitivity]
            Extractor[Extractor worker<br/>L3 → L4 via Claude]
            DLQ[[DLQ: mem.events.dlq]]
        end

        subgraph Tiers["Tier stores (MemoryStore interface)"]
            direction LR
            L1[L1 Working<br/>Redis]
            L2[L2 Profile<br/>Postgres]
            L3[L3 Episodic<br/>Postgres + pgvector]
            L4[L4 Semantic<br/>Postgres + pgvector]
            Tomb[Tombstones<br/>Postgres]
        end

        subgraph Serve["Serve"]
            Retrieval[Retrieval Engine<br/>parallel fan-out + merge<br/>+ token budget]
            Forget[Forget API<br/>by-id / category / TTL]
            GC[Nightly GC<br/>low-confidence TTL]
        end

        Embed[Embedding Client<br/>OpenAI + Redis cache]
    end

    Caller[Caller:<br/>Orchestrator / Context Engine]

    Bus --> Consumer --> Router
    Router -->|Fact| L2
    Router -->|Utterance / Document| Embed --> L3
    Router -->|CalendarItem| L2
    Router -->|CalendarItem| L3
    Router -. fail .-> DLQ

    L3 --> Extractor --> L4

    Caller -->|Retrieve| Retrieval
    Retrieval --> L1
    Retrieval --> L2
    Retrieval --> L3
    Retrieval --> L4

    Caller -->|Forget| Forget
    Forget --> L1
    Forget --> L2
    Forget --> L3
    Forget --> L4
    Forget --> Tomb

    GC --> L3
    GC --> L4
    GC --> Tomb
```

---

## 4. Ingestion sequence

From a raw signal at the edge all the way to a queryable semantic fact.

```mermaid
sequenceDiagram
    autonumber
    participant Src as External source<br/>(Gmail / Calendar / Chat)
    participant Int as Integration Service
    participant Bus as NATS JetStream
    participant Cons as Memory: Consumer
    participant Rtr as Memory: Router
    participant Emb as Embedding<br/>(OpenAI + cache)
    participant L3 as L3 Episodic<br/>(pgvector)
    participant Ext as Memory: Extractor
    participant Claude as Claude
    participant L4 as L4 Semantic<br/>(pgvector)

    Src->>Int: raw signal
    Int->>Int: normalize + classify Sensitivity<br/>+ envelope-encrypt if Health
    Int->>Bus: publish MemoryEvent
    Bus->>Cons: deliver (at-least-once)
    Cons->>Rtr: route by Kind
    Rtr->>Emb: embed(content)
    Emb-->>Rtr: vector
    Rtr->>L3: INSERT episodic row
    L3-->>Cons: ack
    Cons->>Bus: ack (commit offset)

    Note over Ext,L4: Async — runs on a separate consumer
    Ext->>L3: sample recent rows
    Ext->>Claude: extract (subject,predicate,object,confidence)
    Claude-->>Ext: structured facts
    Ext->>L4: upsert facts<br/>(merge / supersede / tombstone)
```

---

## 5. Retrieval sequence

How a user query becomes a contextual answer.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant FE as Frontend
    participant Orc as Orchestrator
    participant Ctx as Context Engine
    participant Mem as Memory: Retrieval
    participant Emb as Embedding
    participant L1 as L1 Working
    participant L2 as L2 Profile
    participant L3 as L3 Episodic
    participant L4 as L4 Semantic
    participant Claude

    User->>FE: "Book a flight to Singapore tomorrow evening"
    FE->>Orc: POST /chat
    Orc->>Ctx: parse(query)
    Ctx-->>Orc: intent=book_flight, entities={dest, time}
    Orc->>Mem: Retrieve(user_id, query, budget)
    Mem->>Emb: embed(query) [cache hit?]
    par Parallel fan-out
        Mem->>L1: last N turns
        Mem->>L2: full profile
        Mem->>L3: top-k episodic (cosine + recency)
        Mem->>L4: top-k facts + exact subject match
    end
    Mem->>Mem: merge + token-budget trim<br/>(priority: L2 > L1 > L4 > L3)
    Mem-->>Orc: MemoryContext
    Orc->>Orc: decide tools (travel + calendar)
    Orc->>Claude: prompt(query + context + tools)
    Claude-->>Orc: response (with tool calls)
    Orc-->>FE: streamed answer + conflict alert
    FE-->>User: "You have an event 8–9 PM tomorrow. Options: ..."
```

---

## 6. Forget flow

Per-memory delete with cascade to derived semantic facts.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant FE as Dashboard
    participant API as Forget API
    participant L3 as L3 Episodic
    participant L4 as L4 Semantic
    participant Tomb as Tombstones

    User->>FE: delete memory X
    FE->>API: DELETE /memory/X
    API->>Tomb: record {tier:L3, id:X, reason:user_delete}
    API->>L3: DELETE X
    API->>L4: SELECT facts WHERE X IN source_refs
    L4-->>API: [f1, f2]
    loop for each fact
        API->>L4: remove X from source_refs
        alt source_refs is now empty
            API->>L4: DELETE fact
            API->>Tomb: record {tier:L4, id:fact, reason:orphaned}
        end
    end
    API-->>FE: {deleted: 1, cascaded: 2}
```

Category purge and TTL decay follow the same tombstone-then-delete pattern; TTL runs on a nightly cron instead of an API call.

---

## 7. Data at rest

```mermaid
erDiagram
    USER_PROFILE {
        uuid user_id PK
        jsonb facts
        timestamp updated_at
    }
    EPISODIC_MEMORIES {
        uuid id PK
        uuid user_id FK
        text content
        vector embedding
        timestamp ts
        text source
        text source_ref
        text sensitivity
        timestamp received_at
    }
    SEMANTIC_FACTS {
        uuid id PK
        uuid user_id FK
        text subject
        text predicate
        text object
        float confidence
        vector embedding
        timestamp first_seen
        timestamp last_seen
        text_array source_refs
        text status
    }
    MEMORY_TOMBSTONES {
        uuid id PK
        uuid user_id FK
        text tier
        uuid original_id
        text reason
        timestamp deleted_at
    }

    USER_PROFILE ||--o{ EPISODIC_MEMORIES : has
    USER_PROFILE ||--o{ SEMANTIC_FACTS : has
    EPISODIC_MEMORIES ||--o{ SEMANTIC_FACTS : "source of (via source_refs)"
    EPISODIC_MEMORIES ||--o{ MEMORY_TOMBSTONES : "may produce"
    SEMANTIC_FACTS ||--o{ MEMORY_TOMBSTONES : "may produce"
```

---

## Legend

- **Solid arrow** — synchronous call or runtime dependency
- **Dashed arrow** — async / deferred / Phase-2
- **Double-bracket `[[ ]]`** — message bus / queue
- **Cylinder `(( ))`** — data store
- **Dashed border** — Phase 2 component (not in MVP)
