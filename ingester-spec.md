# OpenBrain Ingestor Specification
Version: 0.1
Status: Draft
Author: Eric Paul
Date: 2026-03-02

---

# 1. Overview

OpenBrain Ingestor is a local agent that captures LLM tool activity (sessions, prompts, responses, tool calls, artifacts, and relevant metadata) from CLI-based assistants and related tools and asynchronously pushes normalized events into OpenBrain.

This is designed to avoid placing OpenBrain on the critical path of interactive workflows. Ingestor must not materially impact latency of Claude Code, Codex CLI, or other tooling.

Primary sources (initial):
- Claude Code (CLI)
- Codex CLI

Planned/possible sources (later):
- "pi agent" (if it produces local logs/transcripts)
- Google Gemini CLI/tools (if logs/transcripts exist or can be exported)
- Any future tools that emit transcripts or structured logs

---

# 2. Goals

- Asynchronously ingest prompts/sessions without slowing down the CLI tools.
- Normalize different tool log formats into a common OpenBrain event model.
- Provide reliable, idempotent ingestion with checkpoints and deduplication.
- Support Windows-first operation (with optional Unix deployment later).
- Include configurable redaction/scrubbing of sensitive information (API keys, secrets).
- Allow future extension via pluggable "source adapters".

---

# 3. Non-Goals (v0.1)

- Real-time proxying of LLM requests (gateway approach is out-of-scope for ingestor).
- Full IDE integrations (VS Code extension, etc.).
- Deep semantic interpretation of prompts or code beyond what is needed for metadata extraction.
- Perfect secret detection (v0.1 provides configurable redaction and best-effort scrubbing).

---

# 4. Operating Model

Ingestor runs as:
- A long-running background service (recommended), or
- An on-demand CLI command (manual runs), or
- A scheduled task (Windows Task Scheduler) for periodic ingestion.

Ingestor watches one or more source directories for:
- JSONL session transcripts
- log files
- export files

Ingestor reads new data incrementally, transforms it, optionally redacts it, then pushes it to OpenBrain API in batches.

Ingestor maintains local checkpoints to ensure:
- No double-ingestion
- Safe restarts
- Robust recovery after network outages

---

# 5. Deployment Targets

Primary: Windows
- Runs where Eric executes claude-code/codex tools.
- Uses Windows filesystem watchers or periodic scans.
- Stores checkpoints in a local folder (e.g., %APPDATA%\OpenBrain\Ingestor\).

Secondary: Unix (optional future)
- Same behavior with paths under ~/.config/openbrain/ingestor/ (or similar).

---

# 6. High-Level Architecture

Components:
- Source Adapters (per tool)
- File Watcher / Scanner
- Checkpoint Store
- Redaction Pipeline
- Normalizer (OpenBrain Event mapping)
- Batcher + Retry Queue
- OpenBrain API Client

Flow:
1. Detect new or updated transcript/log data.
2. Read only new lines/records since last checkpoint.
3. Parse source-specific record format.
4. Normalize into OpenBrain event(s).
5. Apply redaction rules (optional but recommended).
6. Send batch to OpenBrain API with idempotency keys.
7. Persist checkpoints only after successful ingest.

---

# 7. Inputs and Sources

## 7.1 Source: Codex CLI

Expected capabilities:
- Session transcripts stored locally (often JSONL).
- Operational logs stored locally.

Discovery strategy:
- Detect standard directories if present.
- Allow explicit configuration of log/session directories.

Possible default paths (subject to confirmation per installation):
- %USERPROFILE%\.codex\sessions\
- %USERPROFILE%\.codex\log\
- Optional override via config "CODEX_HOME" or equivalent.

Minimum data to capture:
- Session identifier (from filename/metadata)
- Timestamp
- Role (user/assistant/system/tool)
- Message content
- Model/provider info if present
- Token usage/cost if present
- Tool events if present

## 7.2 Source: Claude Code (claude-code)

Expected capabilities:
- Local transcripts/logs (often JSONL).
- May include tool events and file operations.

Discovery strategy:
- Detect standard directories if present.
- Allow explicit configuration of transcript/log directories.

Possible default paths (subject to confirmation per installation):
- %USERPROFILE%\.claude\
- %APPDATA%\Claude\ (if applicable)
- Optional override via env/config.

Minimum data to capture:
- Session identifier
- Timestamp
- Role
- Message content
- Model/provider info if present
- Tool events if present

## 7.3 Other Sources (Future Adapters)

- pi agent (if it produces logs)
- Gemini CLI or other Google tooling
- Any tool that can export a transcript (JSONL/JSON/Markdown)

Requirement:
- Each adapter must specify a file discovery mechanism and parsing logic.

---

# 8. Normalized Event Model

Ingestor converts all source records into one of:

1) SessionUpsert
2) MessageAppend
3) ToolCallAppend
4) ArtifactAppend
5) RunTelemetry (optional, for costs/latency if not captured on messages)

## 8.1 SessionUpsert

Fields:
- sessionExternalId (string, from source)
- title (string, derived or configured)
- project (string, optional)
- repo (string, optional)
- branch (string, optional)
- tags (string[])
- startedAt (timestamp, optional)
- updatedAt (timestamp, optional)
- source (string, e.g., "codex-cli", "claude-code")

## 8.2 MessageAppend

Fields:
- sessionExternalId
- messageExternalId (string, stable id if available)
- role (system|user|assistant|tool)
- content (string)
- createdAt (timestamp)
- provider (string, optional)
- model (string, optional)
- tokenIn (int, optional)
- tokenOut (int, optional)
- costUsd (decimal, optional)
- latencyMs (int, optional)
- raw (json, optional raw record)

## 8.3 ToolCallAppend

Fields:
- sessionExternalId
- toolCallExternalId (string)
- toolName (string)
- argumentsJson (json)
- resultJson (json)
- createdAt (timestamp)
- raw (json, optional)

## 8.4 ArtifactAppend

Fields:
- sessionExternalId
- artifactExternalId (string)
- type (file|snippet|link|diff|commit|other)
- pathOrUrl (string)
- hash (string, optional)
- metadataJson (json)
- createdAt (timestamp)

---

# 9. Idempotency and Deduplication

Ingestor must generate stable idempotency keys for each ingested record.

Recommended id scheme (per record):
- source: "codex-cli" or "claude-code"
- sourcePath: full path to transcript file
- recordOffset: line number or byte offset
- recordHash: hash of raw line (optional, for sanity)

IdempotencyKey format:
- "{source}|{sourcePath}|{recordOffset}"

OpenBrain API should enforce uniqueness on:
- (source, sourcePath, recordOffset)

Ingestor must tolerate:
- partial files (still being written)
- file rotation
- reprocessing after crash

---

# 10. Checkpointing

Checkpoint storage is local, per sourcePath.

Checkpoint fields:
- sourcePath
- lastProcessedOffset (line number or byte offset)
- lastProcessedTimestamp (optional)
- fileFingerprint (size + mtime; optional)
- lastSeenAt

Behavior:
- If fileFingerprint indicates truncation/rotation, reset safely:
  - Prefer "resume from last known line if present", else restart from 0.
- Checkpoints only advance after successful OpenBrain API ingestion.

Storage locations:
- Windows: %APPDATA%\OpenBrain\Ingestor\checkpoints.json (or sqlite)
- Unix: ~/.config/openbrain/ingestor/checkpoints.json (or sqlite)

---

# 11. Redaction and Sensitive Data Scrubbing

OpenBrain Ingestor must support optional redaction of sensitive data before sending to OpenBrain.

This should be configurable per environment and per source.

## 11.1 Redaction Modes

- off: no scrubbing
- basic: pattern-based masking
- aggressive: pattern-based + heuristics + file-based exclusions

Default for Internet-accessible OpenBrain:
- basic (at minimum)

## 11.2 Target Data to Redact (Examples)

- API keys (OpenAI, Anthropic, OpenRouter, Gemini, AWS keys)
- Bearer tokens
- Private keys / certificates (PEM blocks)
- Connection strings containing passwords
- Common secret env vars:
  - OPENAI_API_KEY
  - ANTHROPIC_API_KEY
  - OPENROUTER_API_KEY
  - GOOGLE_API_KEY
  - AWS_ACCESS_KEY_ID
  - AWS_SECRET_ACCESS_KEY
  - AZURE_* secrets

## 11.3 Redaction Strategy (v0.1)

- Regex-based replacements on message content and raw JSON.
- Configurable patterns list.
- Replacement format:
  - "[REDACTED]" or "[REDACTED:TYPE]"
- Record redaction metadata:
  - redactionApplied: true/false
  - redactionRulesVersion: string

Important:
- Do not attempt perfect detection in v0.1.
- Provide an extensible rule set.
- Provide a test mode that shows what would be redacted without uploading.

## 11.4 Exclusions

Ingestor should support exclusion rules to avoid uploading any content from:
- Certain directories (e.g., containing ".env", "secrets", "private")
- Certain filenames/extensions (e.g., ".pem", ".pfx", ".key")
- Certain message types (tool events involving file reads of secrets)

Exclusion behavior options:
- skip entirely
- upload metadata only (no content)
- upload redacted content

---

# 12. Transport and Reliability

## 12.1 OpenBrain API Contract (Batch Ingest)

Endpoint:
- POST /ingest/batch

Body:
{
  "clientId": "eric-windows-01",
  "source": "codex-cli",
  "events": [
    {
      "type": "MessageAppend",
      "idempotencyKey": "...",
      "payload": { ... }
    }
  ]
}

Response:
- 200 OK with per-event status
- Must indicate which idempotency keys succeeded/failed

## 12.2 Retry

- Network errors: retry with exponential backoff.
- Partial failures: retry only failed events.
- Maintain a small local "outbox" queue on disk for reliability.

Outbox:
- Stores unsent batches or individual events.
- Cleared after server ack.

---

# 13. Performance Requirements

- Ingestor must not block interactive CLI workflows.
- File watching and scanning must be low overhead.
- Default batch size:
  - 50 to 200 events per request (configurable)
- Default max upload interval:
  - 1 to 5 seconds when backlog exists (configurable)
- Backpressure:
  - If OpenBrain API unavailable, store events locally and pause upload.

---

# 14. Configuration

Config file: openbrain.ingestor.json

Example fields:
- openBrainApiBaseUrl
- apiKey
- clientId
- sources:
  - name
  - enabled
  - watchPaths[]
  - filePatterns[]
  - adapterType
- scanIntervalSeconds
- batchSize
- redaction:
  - mode
  - rulesFile
  - exclusions
- outbox:
  - path
  - maxSizeMb

Environment variables should override config:
- OPENBRAIN_API_URL
- OPENBRAIN_API_KEY
- OPENBRAIN_CLIENT_ID

---

# 15. Source Adapter Interface

Each adapter must implement:

- DiscoverFiles(config) -> list of candidate transcript/log files
- ReadNewRecords(file, checkpoint) -> list of raw records + new checkpoint
- ParseRecord(raw) -> normalized event(s)
- IdentifySession(raw) -> sessionExternalId
- IdentifyTimestamp(raw) -> createdAt
- IdentifyRole(raw) -> role
- Optional: IdentifyTokens/Cost/Latency

Adapters should be resilient to partial writes.

---

# 16. Observability

Ingestor logs:
- ingestion rate
- backlog size
- last successful upload time
- redaction applied counts
- parse errors (with safe truncation)
- file discovery counts

Optional:
- metrics endpoint (localhost) for counters and health.

Health states:
- Healthy: ingesting and uploading
- Degraded: ingesting but cannot upload (outbox growing)
- Error: cannot read logs or configuration invalid

---

# 17. Security Requirements

- OpenBrain API must require auth.
- Ingestor stores API keys securely (env var recommended).
- Ingestor should avoid logging raw secrets.
- Support "redaction test mode" before enabling uploads.
- Optionally support TLS pinning or known CA trust constraints.

---

# 18. Acceptance Criteria (v0.1)

1. Ingestor can run on Windows and ingest at least one Codex CLI transcript source.
2. Ingestor can ingest Claude Code transcripts if present and configured.
3. Ingestor is restart-safe and idempotent (no duplicates).
4. Ingestor supports redaction basic mode with configurable patterns.
5. Ingestor uploads batches to OpenBrain API with retries and local outbox.
6. Ingested sessions and messages are searchable in OpenBrain.

---

# 19. Open Questions (to resolve during implementation)

- Exact default transcript/log paths for claude-code and codex on Windows in your installs.
- Exact record format differences per tool version.
- Whether "pi agent" emits logs and where.
- Gemini tooling: which client you plan to use and what it logs.
- Redaction rules: initial pattern list and how aggressive to be.
- Whether to store raw records in OpenBrain, and if so how to encrypt/compress them.

---
