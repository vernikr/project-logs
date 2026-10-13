# Architecture proposal: server-centered agent log hub

**Status:** revised after requirements discussion; recommended for a measured pilot, not yet deployed
**Date:** 2026-10-14
**Primary objective:** let an AI agent inspect cross-project runtime evidence with as few remote reads and
as little repeated field markup as possible.

## Executive recommendation

Build a small self-hosted **log hub** on the existing server:

```text
project runtime
  → local structured event adapter or stdout/journald bridge
  → durable batch/spool
  → SSH upload to the server
  → SQLite event index + FTS5 search
  → compressed/deduplicated artifact store
  → compact tabular query command and MCP server over SSH stdio
  → bounded evidence, aggregates, and on-demand payload pointers
```

The first version should not expose a new public HTTP port. A cloud agent that already has SSH access can
start the remote MCP process with one SSH command. The server can accept uploads through SSH on the existing
port. This avoids a domain, TLS provisioning, firewall changes, and another internet-facing service.

The first implementation should be a small Python service/CLI using the standard library where practical:

- SQLite for the index, metadata, and full-text search;
- a content-addressed filesystem tree for text, images, and files;
- gzip from the standard library as the guaranteed compression fallback, with optional zstd only when it is
  already available and measured to be useful;
- systemd for the collector/compaction/notification jobs;
- journald as an input and operational fallback, not as the cross-project query interface.

This is deliberately smaller than Grafana/Loki/Elasticsearch and more specialized for the actual user:
one owner, AI-operated projects, low server capacity, no new paid services, and a need to minimize agent
context rather than maximize dashboard features.

## Why the priority changes the design

The system must optimize for **investigation cost**, not merely ingestion throughput. Returning 10,000 raw
lines over SSH would technically provide access but would waste context and force the agent to make more
tool calls. The query layer is therefore a compact evidence surface, not an expert system:

- it accepts ordinary filters such as project, time range, run ID, operation ID, component, level, and
  event;
- it returns bounded rows and deterministic aggregates, not a project-specific prose diagnosis;
- repeated events can be grouped by explicit fields or a normalized technical signature;
- a timeline is a selectable table of events, not a generated narrative;
- large payloads are represented by IDs, MIME/size/hash, previews, and lifecycle state;
- raw records and artifacts are available through deliberate follow-up operations;
- the default representation is compact TSV-like text, while JSON remains available when structured output
  is more useful.

The server may store everything, but the default agent response must not return everything. No LLM is needed
for ingestion, querying, grouping, compression, or retention.

## Proposed components

### 1. Client contract and adapter

Each project emits versioned NDJSON events. The adapter may be:

- a tiny library for Python/Node projects;
- a shell wrapper for command execution;
- a parser around existing stdout/stderr or journald records;
- a file/export adapter for browser and plugin runtimes.

The adapter owns correlation IDs and local durability. It should be possible to add a project without
replacing its current logger or changing its user-facing output. The event contract must tolerate new
project fields: an agent can add more detailed logging during a bug investigation without first changing
the central schema.

### 2. Upload/spool path

The default workstation path is one bounded spool plus one SSH batch upload:

```text
project → ~/.local/state/project-logs/spool/<batch>.ndjson[.gz]
       → ssh agent@server logctl ingest --stdin
```

If the server is unavailable, the project continues locally and the spool retries later. Upload is
idempotent by event ID and content hash, so retrying a batch does not duplicate events or attachments.

The application does not contain an SSH private key. It invokes the normal SSH client, which obtains
credentials from the machine's `ssh-agent`, `~/.ssh/config`, or an externally configured secret in the
agent environment. Private keys must never be stored in this repository, embedded in project code, passed
as command-line values, or written to logs. The server may use separate restricted credentials for ingest
and read access.

Server-local projects can write directly to the hub or use journald ingestion. Restricted runtimes keep
their local export behavior until a safe transport exists:

- Figma remains offline and exports a journal;
- Chrome extension snapshots can continue through the existing `runtime/logs` mechanism, with a later
  server pull/bridge;
- a browser or plugin must never be made dependent on the collector just to complete its main work.

### 3. Event index

SQLite stores searchable metadata and selected compact text, not every large binary inline. The initial
logical fields are:

| Field | Purpose |
|---|---|
| `event_id` | idempotency and exact lookup |
| `ts` | UTC event time |
| `project`, `runtime`, `environment` | cross-project filtering |
| `component`, `event`, `level` | stable event classification |
| `run_id`, `operation_id`, `parent_id` | investigation correlation |
| `outcome`, `attempt`, `duration_ms` | lifecycle and performance evidence |
| `schema_version`, `project_schema_version` | evolution of the common and project contracts |
| `fields_json` | typed project-specific fields that may change frequently |
| `content_ref`, `content_kind`, `content_bytes`, `content_hash` | optional payload pointer |
| `payload_state` | raw, compressed, extracted, deduplicated, deleted |
| `ingested_at`, `expires_at` | lifecycle and retention decisions |

SQLite FTS5 indexes event names, messages or error text supplied by the project, selected string fields,
and extracted text previews. It does not need to index raw images or arbitrary binary files.

The shared table is the source of truth. It is not split into one physical database per project. Each
project can have a logical namespace through a view, for example:

```sql
CREATE VIEW gemini_mcp_events AS
SELECT
  event_id, ts, run_id, operation_id, parent_id,
  component, event, level, outcome,
  json_extract(fields_json, '$.provider') AS provider,
  json_extract(fields_json, '$.model') AS model,
  json_extract(fields_json, '$.http_status') AS http_status,
  fields_json
FROM events
WHERE project = 'gemini-mcp';
```

New fields are accepted and preserved immediately in `fields_json`. When a field becomes stable and useful
for repeated queries, it can be exposed through the project's view and indexed without rewriting old
events. This gives agents a familiar flat table while preserving cross-project queries through `events`.

`fields_json` is a storage and ingestion convenience, not the default agent-facing representation. JSON
keys are paid for once per stored event and are not repeated in the default TSV-like response. An agent can
request the raw JSON when it needs an uncommon field, or select a field through a project view when it is
used often.

The planned project views follow the pilot's actual variety rather than forcing one business schema:

| Project | Useful view fields beyond the common envelope |
|---|---|
| `tldr-digest` | `stage`, `provider`, `model`, `source_url`, `http_status`, `attempt` |
| `gemini-mcp` | `tool`, `provider`, `model`, `request_id`, `quota_class`, `http_status` |
| `gemini-router` | `route`, `candidate`, `parent_operation_id`, `fallback_reason`, `selected` |
| `size-report` | `command`, `mode`, `exit_code`, `git_sha`, `artifact_id` |
| `biblio-mcp` | `operation_kind`, `tool`, `results_count`, `download_bytes`, `checksum_status` |
| `figma-safe-resets` | `scan_id`, `page_id`, `phase`, `nodes_scanned`, `dropped_tail` |
| `browser-agent` / `browser-agent-mcp` | `session_id`, `action`, `url_host`, `browser_status`, `artifact_id` |

These are examples of fields worth exposing, not a fixed schema. Logging can add a new field immediately to
`fields_json`; the view is updated only when repeated agent queries show that a flat column is worthwhile.

### 4. Artifact store

Large content is stored outside SQLite using a content hash:

```text
/opt/apps/project-logs/data/artifacts/ab/cd/<sha256>.<kind>[.gz]
```

The event keeps the reference, original byte count, MIME type, hash, and lifecycle state. This gives three
useful properties without buying object storage:

1. repeated screenshots/files are stored once;
2. the database remains small and quick to query;
3. raw content can be compacted or deleted without losing the event timeline.

For text payloads, the hub should retain a bounded preview and, where useful, deterministic extracted text
or repeated-content statistics. For images and arbitrary files, it should retain metadata and, where local
tools support it, a thumbnail or OCR/text extraction. A semantic description is optional and must not be
required for ingestion or querying. The hub must not require an LLM call for every payload: that would add
quota usage, latency, and another failure path.

### 5. Retention and compaction

Use two independent limits instead of choosing only time or only size:

- **age limit** for raw payloads and verbose event details;
- **disk reserve/size limit** to protect the VPN server and keep space for compaction and rollback.

Recommended lifecycle:

```text
hot: raw event + payload
  → warm: compressed/deduplicated payload + preview + metadata
  → compact: event metadata + error signature + counts + content hash
  → expired: delete payload; retain only the compact event if policy allows
```

The first pilot should make these limits configurable and report what it would delete before enabling
automatic deletion. A sensible starting policy is to preserve compact event metadata much longer than raw
payloads, with raw content governed by both age and free-disk reserve. The exact days and bytes are a user
decision after measuring real growth.

When content is compacted, the record must say what happened. The agent should never mistake “payload was
deleted” for “no payload existed”. Semantic summaries, if ever added, are an optional artifact type rather
than a prerequisite for the compact history.

### 6. Agent-first query surface

The primary operation should be a deterministic **evidence bundle**, not a project-specific diagnosis. A
better name than `investigate` for the first version is `inspect-run`: it selects the standard facts around
a run or operation and leaves interpretation to the agent.

```text
inspect-run(
  project?, run_id?, operation_id?, since?, until?,
  component?, level?, event?, limit?, detail_budget?
)
```

The default response should be compact text with one header per table and no repeated field names:

```text
RUN
# columns: project run_id started duration_ms outcome events
tldr-digest run-001 2026-10-14T15:00:00Z 434000 partial 184

ERRORS
# columns: count component event level http_status first_seen last_seen
4 articles fetch_failed error 403 2026-10-14T15:01:10Z 2026-10-14T15:02:03Z
1 llm request_failed error 429 2026-10-14T15:03:18Z 2026-10-14T15:03:18Z

TIMELINE
# columns: ts component event outcome
2026-10-14T15:00:00Z pipeline run_started
2026-10-14T15:03:20Z llm fallback_selected ok
2026-10-14T15:07:14Z pipeline run_finished partial

# rows: 5
# truncated: false
```

The wire format is still MCP/JSON-RPC, but the tool result does not need to be a deeply nested JSON object.
TSV-like output is the default because column names appear once, repeated rows are compact, and the format is
still straightforward for an agent to parse. JSON is an explicit alternative for callers that need typed
machine processing:

```text
logctl inspect-run ... --format json
logctl sql ... --format json
```

`inspect-run` may include deterministic aggregates such as counts, first/last timestamps, durations,
explicitly logged outcomes, and grouped technical error signatures. It must not compose an `answer` sentence
for every project. It must not infer `partial` merely because some errors exist when the project did not log
an outcome.

SQL is a first-class interface, not an escape hatch:

```text
logctl sql --format tsv --max-rows 100 \
  "SELECT ts, component, event, level, outcome
   FROM events
   WHERE project = 'tldr-digest' AND run_id = 'run-001'
   ORDER BY ts"
```

The MCP surface should expose `query_sql(sql, max_rows, max_bytes)` alongside the standard operations. The
initial SQL guardrails are read-only `SELECT`, `WITH`, and `EXPLAIN`; automatic row/byte limits; no BLOBs in
ordinary rows; and explicit truncation markers. A separate artifact operation returns selected content:

```text
get_artifact(artifact_id, max_bytes?, start?, format?)
```

The first version should not generate `next_queries`. The response should instead expose direct identifiers
and mechanical continuation information:

```text
# related_operation: route-001
# related_artifact: artifact-77
# continuation: same filters, offset=100
```

This leaves the agent free to choose the next step. Later, repeated real usage can show whether predefined
continuations save more calls than they cost in context.

The MCP server should run over SSH stdio rather than a public HTTP endpoint in v1:

```text
cloud agent MCP client
  → SSH command
  → /opt/apps/project-logs/bin/logctl mcp
  → SQLite/artifacts
```

This is also easy to test with the CLI and does not depend on a DNS name. If a future agent platform cannot
launch an SSH-backed MCP process, the same `logctl inspect-run --format tsv` command can be called through a
single remote execution step.

The following standard operations remain useful:

- `search` for exact structured or FTS filtering;
- `get_event` for one event;
- `get_artifact` for a selected payload with byte and line limits;
- `tail` for a narrow recent window;
- `schema` for common fields and project views;
- `health` for collector, spool, disk, and retention state.

None of these operations requires an LLM. A future optional operation may ask a configured external model to
summarize one selected text artifact, but it is outside the ingestion and query path and must tolerate quota
exhaustion.

### LLM policy

The required path is fully deterministic. It uses SQLite, FTS5, SQL aggregation, event correlation,
technical error grouping, bounded previews, filesystem compression, deduplication, and retention rules.
There is no automatic Gemini or Gemma call when logs arrive, when `inspect-run` runs, or when old data is
compacted.

An optional later operation may process one explicitly selected text artifact with a configured Gemini Flash
Lite-class model or another available provider. Such a call is outside the core query path, is not required
for correctness, and must fail softly when quota is unavailable. A local Gemma is not part of the initial
resource plan because its memory and latency depend on model size and quantization.

### 7. Notifications

Notifications should be a separate consumer of normalized events, not part of the write path. A rule engine
can match:

- a new failed run;
- repeated failures above a threshold;
- a critical service becoming unhealthy;
- a rate-limit/authentication/bot-wall class;
- a spool backlog or low disk condition;
- a deployment health check failure.

The recommended first channel is **Telegram Bot API**: it is free, needs no domain, and only requires a bot
token plus the owner's chat ID. The notifier should send a compact alert with project, event class, run ID,
counts, first/last timestamps, and a short command for investigation. It should deduplicate and apply a
cooldown so one incident does not produce dozens of messages.

Email is a later optional channel through an SMTP account the user already controls. It should not be
implemented as a new mail service or make the server depend on a paid provider. Telegram and SMTP
credentials remain server-private and are never event fields.

## Why not start with a standard log platform?

| Option | Strength | Why it is not the first choice |
|---|---|---|
| Journald only | already installed; excellent for systemd | weak cross-project indexing and compact agent-oriented evidence; workstation data is awkward |
| Git branches/files | almost no infrastructure; works for browser exports | poor time/query semantics, slow raw reads, no bounded artifact lifecycle, weak deduplication |
| Loki + Grafana | familiar log search and dashboards | extra services and operational surface; dashboards do not solve context-efficient agent answers |
| VictoriaLogs | lightweight log database, ingestion/query integrations, retention support | good candidate if volume grows, but still needs a compact query/evidence layer and separate artifact policy |
| Elasticsearch/OpenSearch/ClickHouse | powerful search and analytics | disproportionate CPU/RAM/disk and maintenance for a 1 vCPU/1 GB server |
| SQLite + filesystem artifacts | tiny, local, fast, free, easy to back up and customize | requires us to implement bounded tabular output and compaction logic |

VictoriaLogs should remain a measured fallback, not a default. Its official documentation confirms support
for multiple ingestion paths and HTTP/query interfaces, so it can replace the SQLite event index later if
real volume or query requirements exceed the custom tabular-evidence layer. It does not remove the need for
a compact agent interface or a separate binary-artifact policy.

## Access and security model

The absence of a domain is not a blocker for the recommended private path:

- SSH remains the transport and authentication boundary;
- the collector and MCP server bind locally or are launched on demand over SSH;
- no new public port or firewall change is required;
- read access should use a restricted server account and a forced command/profile where practical;
- write access should be separated from query access if different project credentials are introduced;
- event payloads may contain secrets by user choice, so the raw-artifact operation must be more restricted
  than the compact investigation operation;
- local event IDs and content hashes prevent accidental duplication but are not access controls.

Do not use the existing VPN or proxy ports for the log service. A future public HTTPS option is technically
possible, including an IP-address certificate path, but it is not needed for v1 and would compete with the
server's reserved port layout. A private SSH-backed MCP path is simpler and safer.

## Deployment shape

The server-side project should fit the existing deployment conventions:

```text
/opt/apps/project-logs/
├── .venv/                  # only if optional Python dependencies are introduced
├── bin/logctl              # CLI and MCP entry point
├── src/                    # collector, query, compaction, notifier
├── config.toml             # non-secret policy and paths
├── .env                    # Telegram/SMTP secrets, chmod 600, never Git
└── data/
    ├── index.sqlite3
    ├── artifacts/
    ├── spool/
    └── reports/
```

Initial units can be minimal:

- `project-logs-compactor.service` plus timer;
- `project-logs-notifier.service` or a timer that reads the event cursor;
- optional `project-logs-ingest.service` only if journald/stdin ingestion needs a long-lived process;
- no always-running HTTP server in v1.

Each unit needs explicit memory and disk safeguards. The system's existing VPN remains the priority. The
collector must fail closed on low disk and preserve a health event rather than consuming the remaining
space.

## Resource estimate before implementation

This is a planning estimate, not a benchmark. It assumes streaming reads/writes and that a large artifact is
never loaded into memory in full.

| Process | Expected idle RSS | Expected peak for one bounded request |
|---|---:|---:|
| SQLite CLI/query process | 0–5 MB | 20–80 MB |
| SSH-launched MCP process | 15–40 MB | 40–80 MB |
| ingest process | 15–40 MB | 50–120 MB |
| compactor | 20–50 MB | 80–150 MB |
| notifier | 15–35 MB | 30–60 MB |

There need not be a permanently running HTTP server. The normal remote query can launch one short-lived
process over SSH. The target for the log hub is under roughly 50–80 MB during ordinary operation and under
roughly 128–192 MB for one bounded task, excluding existing services such as `tldr-digest` and
`browser-agent`. These values must be measured on the real server before changing memory limits.

An LLM is deliberately excluded from this estimate. Gemini Flash Lite would mainly add API latency and
quota use rather than server RSS. A locally hosted Gemma could require hundreds of megabytes or more,
depending on model and quantization, so it must not be a prerequisite for the hub.

## First vertical slice

Do not onboard all projects at once. Prove the value with the two easiest and most representative paths:

1. `tldr-digest` on the server: convert its existing Python logging records or a wrapper into structured
   events while retaining readable journald output.
2. `gemini-mcp` and `gemini-router`: verify nested operations, provider attempts, quota/fallback events, and
   cross-project correlation.
3. `size-report`: capture external CLI/process runs without changing stdout or stderr contracts.
4. `biblio-mcp`: capture metadata-only tool operations and large-file references.
5. `figma-safe-resets`: accept an offline exported journal without making the plugin network-dependent.
6. `browser-agent` and `browser-agent-mcp`: capture long-lived browser sessions, actions, screenshots/files,
   and MCP transport boundaries.

The slice should include:

- event validation and idempotent ingestion;
- one SQLite index, project view, and FTS5 query;
- one compressed/deduplicated text artifact;
- the `inspect-run` response with TSV-like tables, deterministic aggregates, and a bounded timeline;
- one read-only SQL query over a project view;
- one SSH-backed MCP/CLI invocation;
- compaction in dry-run mode;
- a Telegram test notification using a placeholder/configured secret only when the user supplies it.

`yt-summary` and `research-workspace` remain useful later as already-structured export adapters. The UI kit
is not an early runtime-log source.

## Acceptance criteria for the pilot

The pilot is useful only if all of these are demonstrated with real project-shaped fixtures:

1. One remote agent operation returns bounded tabular evidence for a failed run without a project-specific prose generator.
2. The response remains small when the underlying run contains thousands of events or a large text/file
   payload.
3. A follow-up can fetch one selected artifact without reopening the whole log.
4. Replaying the same upload produces no duplicate event or artifact.
5. A server outage leaves a local batch that can be retried later.
6. Compaction dry-run reports candidates, expected bytes, and the exact state transition without deleting.
7. The server refuses ingestion or pauses retention work before disk safety limits are crossed.
8. The existing `tldr-digest` behavior and `research-workspace` checks still pass.
9. Telegram notification deduplication prevents repeated alerts for the same incident.
10. SSH/MCP access works without a domain, new public port, or modification of protected VPN services.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Raw logs contain credentials | keep raw access restricted; never put credentials in project docs; add optional redaction and alerting later |
| Arbitrary files fill 20 GB | content hashes, age + disk limits, low-disk fail-closed mode, compaction dry-run |
| Compaction or grouping hides important detail | retain event IDs, hashes, previews, and explicit payload states; fetch raw item on demand |
| SSH access is unavailable to an agent | keep CLI/API shape stable; later add a private tunnel or carefully reviewed HTTPS option |
| SQLite becomes a bottleneck | measure first; partition/rotate by time or migrate the index to VictoriaLogs only with evidence |
| Notifications become noisy | normalized signatures, cooldown, grouping, and a health report |
| Per-project adapters drift | version the event envelope, provide conformance fixtures, and keep adapters thin |

## Open decisions after the pilot

- exact raw/compact retention days and disk reserve;
- whether compact event metadata is retained indefinitely;
- whether images receive OCR/thumbnails automatically or only on demand;
- whether `yt-summary` continues using GitHub as a transport or gains a workstation-side uploader;
- whether cloud-agent environments can reliably launch SSH-backed MCP servers;
- whether the measured event volume justifies VictoriaLogs or another index.

## Sources checked

The following public documentation was checked on 2026-10-13 for current feasibility, not as a commitment
to use those products:

- [VictoriaLogs documentation](https://docs.victoriametrics.com/victorialogs/)
- [VictoriaLogs data ingestion](https://docs.victoriametrics.com/victorialogs/data-ingestion/)
- [VictoriaLogs querying](https://docs.victoriametrics.com/victorialogs/querying/)
- [VictoriaLogs FAQ](https://docs.victoriametrics.com/victorialogs/faq/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Tailscale Funnel documentation](https://tailscale.com/docs/features/tailscale-funnel)
- [Let's Encrypt IP address certificates](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability)

The local project evidence and server limits are recorded in `docs/research/active-projects.md` and
`docs/constraints/server-profile.md`.
