# Architecture Diagrams

## GCP Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SLACK BOT LLM PLATFORM                               │
│                            GCP Architecture                                  │
└─────────────────────────────────────────────────────────────────────────────┘

                                 ┌──────────────┐
                                 │    SLACK     │
                                 └──────┬───────┘
                                        │ Events API (HTTP)
                                        ▼
                              ┌──────────────────┐
                              │   CLOUD RUN      │
                              │ Message Handler  │
                              │ (min: 1 instance)│
                              └────────┬─────────┘
                                       │ Publish (no ordering key)
                                       ▼
                              ┌──────────────────┐
                              │    PUB/SUB       │
                              │  query-requests  │
                              │   (parallel)     │
                              └────────┬─────────┘
                                       │ Pull (parallel delivery)
                                       ▼
                              ┌──────────────────┐
                              │   CLOUD RUN      │
                              │    Workers       │
                              │ (auto-scales)    │
                              └────────┬─────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
              ▼                        ▼                        ▼
     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
     │  MEMORYSTORE    │     │   CLOUD SQL     │     │   VERTEX AI     │
     │    (Redis)      │     │  (PostgreSQL)   │     │  Matching Engine│
     │  Thread Cache   │     │ Source of Truth │     │  (Vector Search)│
     └─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 1. Customer Onboarding Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CUSTOMER ONBOARDING                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Admin Portal (Cloud Run)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. Register Company                                                         │
│     └─→ Cloud SQL: INSERT INTO tenants (name, plan, config)                │
│                                                                              │
│  2. OAuth with Slack                                                         │
│     └─→ Redirect to Slack → User authorizes → Receive tokens               │
│     └─→ Secret Manager: Store bot_token, signing_secret                    │
│     └─→ Cloud SQL: UPDATE tenants SET slack_workspace_id, status='active'  │
│                                                                              │
│  3. Configure Channels                                                       │
│     └─→ Fetch channels via Slack API                                        │
│     └─→ Cloud SQL: INSERT INTO channels (tenant_id, channel_id, enabled)   │
│                                                                              │
│  4. Upload Knowledge Sources                                                 │
│     └─→ Cloud Storage: Store raw documents                                  │
│     └─→ Trigger ingestion pipeline                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Knowledge Ingestion Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INGESTION PIPELINE                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Cloud Storage (document uploaded)
         │
         │ Cloud Storage Trigger
         ▼
┌──────────────────┐
│  Cloud Function  │  Ingestion Trigger
│  (or Cloud Run)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Pub/Sub       │  ingestion-jobs topic
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Cloud Run      │  Ingestion Worker
│   (Batch Job)    │
└────────┬─────────┘
         │
         ├──→ 1. Validate document (type, size, permissions)
         │
         ├──→ 2. Extract text (PDF, DOCX, etc.)
         │        └─→ Document AI (OCR for images/scans)
         │
         ├──→ 3. Chunk text (semantic chunking, ~500 tokens)
         │
         ├──→ 4. Generate embeddings (Vertex AI Embeddings API)
         │
         ├──→ 5. Store vectors
         │        └─→ Vertex AI Matching Engine (or AlloyDB + pgvector)
         │
         └──→ 6. Update metadata
                  └─→ Cloud SQL: INSERT INTO knowledge_chunks
```

---

## 3. Query Request Flow (Core)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QUERY REQUEST FLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘


STEP 1: MESSAGE HANDLER (Cloud Run)
─────────────────────────────────────────────────────────────────────────────

Slack Event (HTTP POST)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Message Handler Service                                                     │
│                                                                              │
│  async def handle_slack_event(request):                                     │
│      # 1. Validate Slack signature                                          │
│      verify_slack_signature(request)                                        │
│                                                                              │
│      # 2. Parse event                                                        │
│      event = parse_event(request)                                           │
│      tenant_id = lookup_tenant(event.team_id)                               │
│                                                                              │
│      # 3. Check if bot mentioned / DM / enabled channel                     │
│      if not should_respond(event, tenant_id):                               │
│          return Response(200)                                                │
│                                                                              │
│      # 4. Immediate Slack acknowledgment                                    │
│      post_to_slack(event.channel, event.thread_ts, "Looking into this 🔍") │
│                                                                              │
│      # 5. Create request record                                             │
│      request_id = create_request(tenant_id, event)  # Cloud SQL            │
│                                                                              │
│      # 6. Publish to Pub/Sub (NO ordering key = parallel)                  │
│      publish_to_pubsub("query-requests", {                                  │
│          "request_id": request_id,                                          │
│          "tenant_id": tenant_id,                                            │
│          "channel_id": event.channel,                                       │
│          "thread_ts": event.thread_ts,                                      │
│          "message_ts": event.ts,                                            │
│          "query": event.text,                                               │
│          "user_id": event.user                                              │
│      })                                                                      │
│                                                                              │
│      return Response(200)  # Must respond within 3 seconds                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼

STEP 2: PUB/SUB (Parallel Queue)
─────────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│  Pub/Sub: query-requests                                                     │
│                                                                              │
│  - No ordering keys (full parallel processing)                              │
│  - Ack deadline: 300s (5 min for LLM processing)                           │
│  - Dead letter topic after 5 retries                                        │
│  - Message retention: 7 days                                                │
│                                                                              │
│  25 messages → 25 delivered in parallel → 25 processed simultaneously      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼

STEP 3: WORKER (Cloud Run) - Orchestrator Logic
─────────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│  Worker Service (auto-scales based on Pub/Sub backlog)                      │
│                                                                              │
│  Configuration:                                                              │
│    min_instances: 1                                                          │
│    max_instances: 50                                                         │
│    concurrency: 50 (50 parallel messages per instance)                      │
│    timeout: 300s                                                             │
│                                                                              │
│  def process_message(message):                                              │
│      data = json.loads(message.data)                                        │
│      request_id = data["request_id"]                                        │
│      tenant_id = data["tenant_id"]                                          │
│      thread_ts = data["thread_ts"]                                          │
│      message_ts = data["message_ts"]                                        │
│      query = data["query"]                                                   │
│                                                                              │
│      # 1. Load tenant config (cached in memory)                             │
│      tenant = get_tenant_config(tenant_id)                                  │
│                                                                              │
│      # 2. Load thread context if follow-up (SLIDING WINDOW HERE)           │
│      is_followup = (thread_ts != message_ts)                                │
│      thread_context = load_thread_context(tenant_id, thread_ts)             │
│                                                                              │
│      # 3. Intent classification (fast mini-LLM)                             │
│      intent = classify_intent(query, tenant)                                │
│                                                                              │
│      # 4. RAG search                                                         │
│      rag_context = search_knowledge(query, intent, tenant)                  │
│                                                                              │
│      # 5. Generate answer (main LLM)                                        │
│      answer = generate_answer(query, thread_context, rag_context, tenant)   │
│                                                                              │
│      # 6. Update thread context cache                                       │
│      update_thread_context(tenant_id, thread_ts, query, answer)             │
│                                                                              │
│      # 7. Post answer to Slack                                              │
│      post_to_slack(data["channel_id"], thread_ts, answer)                   │
│                                                                              │
│      # 8. Mark complete                                                      │
│      mark_request_complete(request_id, answer)                              │
│                                                                              │
│      message.ack()                                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Thread Context & Sliding Window

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THREAD CONTEXT MANAGEMENT                                 │
│                    (Sliding Window in Worker)                                │
└─────────────────────────────────────────────────────────────────────────────┘


STORAGE:
─────────────────────────────────────────────────────────────────────────────

  ┌─────────────────────┐                    ┌─────────────────────┐
  │    MEMORYSTORE      │                    │     CLOUD SQL       │
  │      (Redis)        │                    │   (PostgreSQL)      │
  ├─────────────────────┤                    ├─────────────────────┤
  │                     │                    │                     │
  │  Key: thread:{t}:{s}│   Cache Miss       │  requests table     │
  │                     │ ─────────────────→ │  - tenant_id        │
  │  Value: {           │                    │  - thread_ts        │
  │    messages: [...]  │ ←───────────────── │  - query            │
  │  }                  │   Populate Cache   │  - answer           │
  │                     │                    │  - created_at       │
  │  TTL: 1 hour        │                    │                     │
  │                     │                    │                     │
  └─────────────────────┘                    └─────────────────────┘


SLIDING WINDOW LOGIC (in Worker):
─────────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  def load_thread_context(tenant_id, thread_ts):                             │
│      cache_key = f"thread:{tenant_id}:{thread_ts}"                          │
│                                                                              │
│      # Try Redis first                                                       │
│      cached = redis.get(cache_key)                                          │
│      if cached:                                                              │
│          messages = json.loads(cached)["messages"]                          │
│      else:                                                                   │
│          # Cache miss - query Cloud SQL                                     │
│          messages = db.query("""                                            │
│              SELECT query, answer FROM requests                             │
│              WHERE tenant_id = %s AND thread_ts = %s                        │
│              ORDER BY created_at                                            │
│          """, [tenant_id, thread_ts])                                       │
│          # Store in Redis for next time                                     │
│          redis.setex(cache_key, 3600, json.dumps({"messages": messages}))  │
│                                                                              │
│      # ═══════════════════════════════════════════════════════════════════ │
│      # SLIDING WINDOW - Keep last N messages or by token count             │
│      # ═══════════════════════════════════════════════════════════════════ │
│                                                                              │
│      MAX_CONTEXT_MESSAGES = 10                                              │
│      MAX_CONTEXT_TOKENS = 4000                                              │
│                                                                              │
│      # Option A: Simple - last N messages                                   │
│      if len(messages) > MAX_CONTEXT_MESSAGES:                               │
│          messages = messages[-MAX_CONTEXT_MESSAGES:]                        │
│                                                                              │
│      # Option B: Token-based - keep as many as fit                          │
│      total_tokens = 0                                                        │
│      windowed = []                                                           │
│      for msg in reversed(messages):                                         │
│          msg_tokens = count_tokens(msg["query"] + msg["answer"])            │
│          if total_tokens + msg_tokens > MAX_CONTEXT_TOKENS:                 │
│              break                                                           │
│          windowed.insert(0, msg)                                            │
│          total_tokens += msg_tokens                                          │
│                                                                              │
│      return windowed                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘


PROMPT BUILDING:
─────────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  def generate_answer(query, thread_context, rag_context, tenant):           │
│                                                                              │
│      prompt = f"""                                                          │
│      You are a helpful assistant for {tenant.name}.                         │
│                                                                              │
│      KNOWLEDGE BASE:                                                        │
│      {rag_context}                                                          │
│                                                                              │
│      CONVERSATION HISTORY:                                                  │
│      {format_thread_context(thread_context)}                                │
│                                                                              │
│      USER QUESTION:                                                         │
│      {query}                                                                │
│                                                                              │
│      Provide a helpful answer based on the knowledge base and context.      │
│      """                                                                    │
│                                                                              │
│      return call_llm(prompt, tenant.llm_config)                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Model (Cloud SQL)

```sql
-- Tenants (Companies)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slack_team_id   VARCHAR(50) UNIQUE,
    plan            VARCHAR(50) DEFAULT 'standard',
    config          JSONB,  -- LLM settings, features, limits
    status          VARCHAR(20) DEFAULT 'pending',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Channels
CREATE TABLE channels (
    id              UUID PRIMARY KEY,
    tenant_id       UUID REFERENCES tenants(id),
    slack_channel_id VARCHAR(50) NOT NULL,
    name            VARCHAR(255),
    enabled         BOOLEAN DEFAULT true,
    config          JSONB,  -- Channel-specific settings
    UNIQUE(tenant_id, slack_channel_id)
);

-- Requests (Query Trace + Thread Context Source)
CREATE TABLE requests (
    id              UUID PRIMARY KEY,
    tenant_id       UUID REFERENCES tenants(id),
    channel_id      UUID REFERENCES channels(id),
    thread_ts       VARCHAR(50),        -- Slack thread timestamp
    message_ts      VARCHAR(50),        -- Slack message timestamp
    user_id         VARCHAR(50),
    query           TEXT NOT NULL,
    intent          VARCHAR(100),
    rag_sources     JSONB,              -- Which docs were used
    answer          TEXT,
    status          VARCHAR(20) DEFAULT 'pending',
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    
    INDEX idx_thread (tenant_id, thread_ts)  -- For thread context queries
);

-- Knowledge Sources
CREATE TABLE knowledge_sources (
    id              UUID PRIMARY KEY,
    tenant_id       UUID REFERENCES tenants(id),
    name            VARCHAR(255),
    source_type     VARCHAR(50),        -- confluence, gdrive, upload
    gcs_path        VARCHAR(500),       -- Cloud Storage path
    status          VARCHAR(20),
    chunk_count     INTEGER,
    last_synced     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. GCP Services Summary

| Component | GCP Service | Purpose |
|-----------|-------------|---------|
| Message Handler | Cloud Run | Receive Slack webhooks, respond fast |
| Queue | Pub/Sub | Parallel message delivery, durability |
| Workers | Cloud Run | Process queries, auto-scale |
| Thread Cache | Memorystore (Redis) | Fast thread context lookup |
| Database | Cloud SQL (PostgreSQL) | Source of truth, metadata |
| Vector Search | Vertex AI Matching Engine | RAG semantic search |
| LLM | Vertex AI / External | Answer generation |
| Secrets | Secret Manager | API keys, tokens |
| Documents | Cloud Storage | Raw knowledge files |
| OCR | Document AI | Extract text from images/PDFs |
| Monitoring | Cloud Logging + Monitoring | Logs, metrics, alerts |

---

## 7. Scaling Characteristics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SCALING BEHAVIOR                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Load: 100 concurrent queries from different users/companies

  Pub/Sub: 100 messages ready
       │
       ▼
  Cloud Run Workers:
    - Auto-scales based on Pub/Sub backlog
    - Each instance handles 50 concurrent messages
    - 100 messages → 2 instances → ALL processed in ~3 seconds
       │
       ▼
  Result: ALL 100 users get answers in ~3 seconds (parallel)


Load: 1000 concurrent queries (spike)

  Pub/Sub: 1000 messages ready
       │
       ▼
  Cloud Run Workers:
    - Auto-scales to 20 instances (1000 / 50 = 20)
    - ALL 1000 processed in parallel
       │
       ▼
  Result: ALL 1000 users get answers in ~3-5 seconds


Key points:
  ✅ No ordering keys = no serialization
  ✅ Any worker handles any message
  ✅ Thread context from Redis (shared cache)
  ✅ Auto-scaling based on demand
  ✅ Per-message ack (no message loss)
```

---

## 8. Error Handling

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ERROR HANDLING                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Pub/Sub Retry + Dead Letter Queue:

  Message fails → nack() → Pub/Sub retries with exponential backoff
       │
       ├── Retry 1: after 10s
       ├── Retry 2: after 20s
       ├── Retry 3: after 40s
       ├── Retry 4: after 80s
       └── Retry 5: after 160s
              │
              ▼ (still failing)
       Dead Letter Topic: query-requests-dlq
              │
              ▼
       Alert + Manual review


Worker error handling:

  def process_message(message):
      try:
          # ... processing logic ...
          message.ack()
      except TransientError:
          message.nack()  # Retry via Pub/Sub
      except PermanentError:
          mark_failed(request_id, error)
          post_to_slack(channel, "Sorry, couldn't process. Team notified.")
          message.ack()  # Don't retry permanent failures
```

---

## Architecture Principles

1. **Parallel by default** - No ordering keys, all messages processed simultaneously
2. **Stateless workers** - Any worker handles any message, thread context in Redis
3. **Fast acknowledgment** - Slack gets response in <3s, processing happens async
4. **Sliding window** - Thread context limited by message count or tokens
5. **Auto-scaling** - Cloud Run scales workers based on Pub/Sub backlog
6. **Durability** - Pub/Sub guarantees delivery, DLQ for failures
7. **Multi-tenant** - Tenant isolation via tenant_id in all queries
