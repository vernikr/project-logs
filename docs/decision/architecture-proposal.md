# Architecture proposal: server-centered agent log hub

**Status:** recommended for a measured pilot, not yet deployed
**Date:** 2026-10-13
**Primary objective:** let an AI agent answer “what happened and what should I inspect next?” with one
compact remote query instead of several raw-log reads.

## Executive recommendation

Build a small self-hosted **log hub** on the existing server:

```text
project runtime
  → local structured event adapter or stdout/journald bridge
  → durable batch/spool
  → SSH upload to the server
  → SQLite event index + FTS5 search
  → compressed/deduplicated artifact store
  → agent-first query command and MCP server over SSH stdio
  → compact diagnosis, timeline, clusters, and on-demand payload pointers
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
tool calls. The query layer therefore becomes a first-class product surface:

- one request accepts a project, time range, run ID, error class, or natural-language-like search phrase;
- the response begins with a short diagnosis and counts;
- repeated failures are grouped by normalized signature;
- a timeline contains only the most informative events;
- large payloads are represented by IDs, MIME/size/hash, text previews, and generated summaries;
- the response includes explicit “no evidence found” and “payload omitted due to budget” states;
- raw records are available through a second, deliberate operation when needed.

The server may store everything, but the default agent response must not return everything.

## Proposed components

### 1. Client contract and adapter

Each project emits versioned NDJSON events. The adapter may be:

- a tiny library for Python/Node projects;
- a shell wrapper for command execution;
- a parser around existing stdout/stderr or journald records;
- a file/export adapter for browser and plugin runtimes.

The adapter owns correlation IDs and local durability. It should be possible to add a project without
replacing its current logger or changing its user-facing output.

### 2. Upload/spool path

The default workstation path is one bounded spool plus one SSH batch upload:

```text
project → ~/.local/state/project-logs/spool/<batch>.ndjson[.gz]
       → ssh agent@server logctl ingest --stdin
```

If the server is unavailable, the project continues locally and the spool retries later. Upload is
idempotent by event ID and content hash, so retrying a batch does not duplicate events or attachments.

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
| `summary` | short safe-to-return explanation |
| `fields_json` | typed structured fields |
| `content_ref`, `content_kind`, `content_bytes`, `content_hash` | optional payload pointer |
| `payload_state` | raw, compressed, summarized, deduplicated, deleted |
| `ingested_at`, `expires_at` | lifecycle and retention decisions |

SQLite FTS5 indexes event names, summaries, error signatures, selected string fields, and extracted text
previews. It does not need to index raw images or arbitrary binary files.

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

For text payloads, the hub should retain a bounded preview and generate a compact summary at ingest or
compaction time. For images and arbitrary files, it should retain metadata and, where local tools support
it, a thumbnail, OCR/text extraction, or a short description. It must not require an LLM call for every
payload: that would add cost, latency, and another failure path. Summaries can be generated on demand or
only for items selected by an investigation.

### 5. Retention and compaction

Use two independent limits instead of choosing only time or only size:

- **age limit** for raw payloads and verbose event details;
- **disk reserve/size limit** to protect the VPN server and keep space for compaction and rollback.

Recommended lifecycle:

```text
hot: raw event + payload
  → warm: compressed/deduplicated payload + preview + metadata
  → compact: summary + error signature + counts + content hash
  → expired: delete payload; retain only the compact event if policy allows
```

The first pilot should make these limits configurable and report what it would delete before enabling
automatic deletion. A sensible starting policy is to preserve compact event metadata much longer than raw
payloads, with raw content governed by both age and free-disk reserve. The exact days and bytes are a user
decision after measuring real growth.

When content is compacted, the record must say what happened. The agent should never mistake “payload was
deleted” for “no payload existed”.

### 6. Agent-first query surface

The primary interface should be a single high-level operation, exposed both as a CLI and an MCP tool:

```text
investigate(
  project?, run_id?, since?, until?, query?, level?, limit?, detail_budget?
)
```

The response should be a small JSON object with this order:

1. `answer` — a few sentences derived from evidence, not a raw log dump;
2. `scope` — filters, time range, projects, and event count;
3. `status` — ok, failed, partial, or no_evidence;
4. `error_clusters` — normalized signature, count, first/last seen, affected runs;
5. `timeline` — bounded high-signal events around failures and phase boundaries;
6. `artifacts` — IDs, types, sizes, previews, summaries, and lifecycle states;
7. `next_queries` — at most a few targeted follow-ups when ambiguity remains.

Default budgets should be conservative and configurable. The tool can support a `detail_budget` or
`include_raw` flag so an agent must explicitly spend context on large records.

Lower-level operations are still needed, but should not be the normal first step:

- `search` for exact structured filtering;
- `get_event` for one event;
- `get_artifact` for a selected payload, with byte and line limits;
- `tail` for a narrow recent window;
- `health` for collector, spool, disk, and retention state.

The MCP server should run over SSH stdio rather than a public HTTP endpoint in v1:

```text
cloud agent MCP client
  → SSH command
  → /opt/apps/project-logs/bin/logctl mcp
  → SQLite/artifacts
```

This is also easy to test with the CLI and does not depend on a DNS name. If a future agent platform cannot
launch an SSH-backed MCP process, the same `logctl investigate --json` command can be called through a
single remote execution step.

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
| Journald only | already installed; excellent for systemd | weak cross-project indexing and agent-oriented summaries; workstation data is awkward |
| Git branches/files | almost no infrastructure; works for browser exports | poor time/query semantics, slow raw reads, no bounded artifact lifecycle, weak deduplication |
| Loki + Grafana | familiar log search and dashboards | extra services and operational surface; dashboards do not solve context-efficient agent answers |
| VictoriaLogs | lightweight log database, ingestion/query integrations, retention support | good candidate if volume grows, but still needs a custom agent summary layer and separate artifact policy |
| Elasticsearch/OpenSearch/ClickHouse | powerful search and analytics | disproportionate CPU/RAM/disk and maintenance for a 1 vCPU/1 GB server |
| SQLite + filesystem artifacts | tiny, local, fast, free, easy to back up and customize | requires us to implement the query summary and compaction logic |

VictoriaLogs should remain a measured fallback, not a default. Its official documentation confirms support
for multiple ingestion paths and HTTP/query interfaces, so it can replace the SQLite event index later if
real volume or query requirements exceed the custom store. It does not remove the need for an agent-first
response layer or a separate binary-artifact policy.

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

## First vertical slice

Do not onboard all projects at once. Prove the value with the two easiest and most representative paths:

1. `tldr-digest` on the server: convert its existing Python logging records or a wrapper into structured
   events while retaining readable journald output.
2. `research-workspace` from the workstation: upload its existing JSONL through one SSH batch command and
   query it remotely.

The slice should include:

- event validation and idempotent ingestion;
- one SQLite index and FTS5 query;
- one compressed/deduplicated text artifact;
- the `investigate` response with summary, error clusters, and bounded timeline;
- one SSH-backed MCP/CLI invocation;
- compaction in dry-run mode;
- a Telegram test notification using a placeholder/configured secret only when the user supplies it.

`yt-summary` should be the next adapter because it already has a structured journal and an export/push
workflow. `size-report` should be added as an outer process/run adapter. Figma should remain an explicit
export adapter, not a network client. The UI kit is not an early runtime-log source.

## Acceptance criteria for the pilot

The pilot is useful only if all of these are demonstrated with real project-shaped fixtures:

1. One remote agent operation returns a diagnosis and bounded evidence for a failed run.
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
| Summary generation loses the important detail | retain event IDs, hashes, previews, and explicit payload states; fetch raw item on demand |
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
