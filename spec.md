# OpenBrain Specification
Version: 0.1
Status: Draft
Author: Eric Paul
Date: 2026-03-02

---

# 1. Overview

OpenBrain is a personal LLM work ledger and memory system designed to track prompts, sessions, tool calls, artifacts, and decisions across AI providers (Claude, Codex, OpenAI, local models, etc.).

It provides:

- Durable storage of AI interactions
- Structured retrieval (search + embeddings)
- MCP access for agentic workflows
- Internet-accessible API
- Cost and token telemetry
- Cross-provider session continuity

Primary goal:

Prevent repeating work. Preserve context. Enable agentic continuity.

---

# 2. Design Principles

1. Store everything as events.
2. Keep raw data immutable.
3. Add derived data (summaries, embeddings) separately.
4. Be local-first but internet-accessible.
5. Secure by default.
6. Minimal surface area for MCP.
7. Structured outputs only.

---

# 3. High-Level Architecture

Components:

- PostgreSQL (source of truth)
- OpenBrain API (.NET minimal API)
- MCP Server (thin wrapper over API)
- Search layer (Postgres FTS initially)
- Optional Embedding layer (pgvector)
- Secure tunnel (Cloudflare Tunnel recommended)

Logical Flow:

Agent or LLM
  -> MCP tools
  -> OpenBrain API
  -> Postgres
  -> Search or retrieval
  -> Returned context

---

# 4. Core Concepts

## 4.1 Session

A logical container for a conversation or workflow.

Examples:
- "Jenkins Pipeline Refactor"
- "AI Gateway Design"
- "VehicleLead RabbitMQ Implementation"

Sessions are project-scoped and time-bound.

## 4.2 Message

A single interaction unit.

Roles:
- system
- user
- assistant
- tool

Messages are immutable once written.

## 4.3 Tool Call

Structured record of an agent invoking a tool.

Includes:
- tool name
- arguments
- results
- timestamps

## 4.4 Artifact

External or derived output associated with a session.

Examples:
- generated file
- SQL script
- design doc
- API spec
- commit hash
- diff summary

---

# 5. Data Model (Postgres)

## 5.1 Table: Session

Fields:

- SessionId UUID PK
- Title TEXT
- Project TEXT
- Repo TEXT
- Branch TEXT
- Tags TEXT[]
- CreatedAt TIMESTAMP
- UpdatedAt TIMESTAMP
- IsArchived BOOLEAN DEFAULT false

Indexes:
- Project
- Repo
- CreatedAt

---

## 5.2 Table: Message

Fields:

- MessageId UUID PK
- SessionId UUID FK
- Role TEXT
- Content TEXT
- Provider TEXT
- Model TEXT
- RequestId TEXT
- TokenIn INT
- TokenOut INT
- CostUsd NUMERIC(10,6)
- LatencyMs INT
- CreatedAt TIMESTAMP

Indexes:
- SessionId
- Provider
- Model
- CreatedAt

Optional:
- ContentVector VECTOR (if pgvector enabled)
- ContentTSVector TSVECTOR (for full-text search)

---

## 5.3 Table: ToolCall

Fields:

- ToolCallId UUID PK
- SessionId UUID FK
- ToolName TEXT
- ArgumentsJson JSONB
- ResultJson JSONB
- CreatedAt TIMESTAMP

Indexes:
- SessionId
- ToolName

---

## 5.4 Table: Artifact

Fields:

- ArtifactId UUID PK
- SessionId UUID FK
- Type TEXT
- PathOrUrl TEXT
- Hash TEXT
- MetadataJson JSONB
- CreatedAt TIMESTAMP

Indexes:
- SessionId
- Type

---

# 6. API Surface (HTTP)

Base URL:

https://openbrain.yourdomain.com

Authentication:

- API Key (header: X-API-Key)
- Later: JWT or Cloudflare Access

---

## 6.1 Create Session

POST /sessions

Body:

{
  "title": "Jenkins Groovy Refactor",
  "project": "LeadManagement",
  "repo": "leadmanagement",
  "branch": "feature/refactor",
  "tags": ["jenkins", "groovy"]
}

Response:

{
  "sessionId": "uuid"
}

---

## 6.2 Append Message

POST /sessions/{sessionId}/messages

Body:

{
  "role": "assistant",
  "content": "Refactored pipeline to use global vars.",
  "provider": "anthropic",
  "model": "claude-4.1",
  "tokenIn": 512,
  "tokenOut": 240,
  "costUsd": 0.0042
}

---

## 6.3 Append Tool Call

POST /sessions/{sessionId}/toolcalls

Body:

{
  "toolName": "git.diff",
  "argumentsJson": {...},
  "resultJson": {...}
}

---

## 6.4 Search

GET /search?q=jenkins+yaml&project=LeadManagement&limit=10

Response:

[
  {
    "sessionId": "...",
    "snippet": "... structured config variable issue ...",
    "createdAt": "..."
  }
]

---

## 6.5 Get Session

GET /sessions/{sessionId}?limit=100

Returns ordered message history.

---

# 7. MCP Tool Surface

Tools exposed via MCP:

- create_session
- append_message
- append_tool_call
- search
- get_session
- summarize_session (optional)

All MCP tools must return structured JSON only.

---

# 8. Search Strategy

Phase 1:
- Postgres full-text search
- Weighted ranking
- Snippet extraction

Phase 2:
- pgvector embeddings
- Cosine similarity search
- Hybrid search (FTS + vector)

---

# 9. Security

- HTTPS required
- API key required
- Rate limiting
- Audit logging
- Secrets stored outside repo
- Optional IP allowlist

Internet exposure via:
- Cloudflare Tunnel recommended

---

# 10. Deployment Strategy

Phase 1:
- Run on Mac mini
- Docker compose
- Local Postgres

Phase 2:
- Migrate to Unix server
- Reverse proxy or Cloudflare Tunnel
- Automated backups

---

# 11. Non-Goals (v0.1)

- No UI dashboard
- No multi-user RBAC
- No automatic IDE plugin
- No AI summarization pipeline
- No vector DB external dependency

---

# 12. Future Enhancements

- Multi-user support
- Project-scoped memory injection
- Automatic summarization of long sessions
- Diff tracking against Git
- Cost analytics dashboard
- Agent orchestration layer
- Knowledge graph layer
- Time-based recall weighting
- Secret redaction engine

---

# 13. Success Criteria

- Prompts never lost.
- Search returns relevant prior solutions.
- MCP agents can store and retrieve memory reliably.
- Works locally and over internet.
- Minimal operational overhead.

---

# 14. Guiding Philosophy

OpenBrain is not a chatbot.

It is an append-only, queryable memory substrate for agentic engineering.

It should feel like Git for AI work.
