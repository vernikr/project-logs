# Active project investigation

**Date:** 2026-10-13
**Sources:** the portfolio summary, local repositories, and their checked-in project guidance.
**Purpose:** identify the smallest common logging contract without forcing one runtime on unrelated
projects.

## Portfolio shape

The portfolio spans at least these runtime families:

- Node.js/Python CLIs and automation jobs;
- Chrome MV3 extensions with short-lived service workers and browser storage;
- Figma plugins with no network access;
- React/Vite libraries and browser applications;
- shell-based research workflows;
- systemd-managed server jobs.

The first active repositories in the portfolio summary are not all separate local folders. The local
paths below record the matches used for this investigation:

| Portfolio project | Local path or status | Runtime | Logging relevance |
|---|---|---|---|
| `size-report` | `size-report/` | Node.js CLI/package | stdout/stderr and machine-readable command results |
| `figma-safe-resets` | `figma/safe-resets/` | Figma plugin | offline, bounded in-memory journal and manual export |
| `yt-summary` | `yt-summary/` | Chrome MV3 + TypeScript | structured event journal across browser contexts |
| `alfa-ui-kit` | `vivica/alfa-ui-kit/` | React/Vite library | build/test runtime; no production event stream expected |
| `biblio-mcp` | not a separate local folder; integration documented in `research-workspace/` | Node.js MCP | operation ledger for agent-visible tool calls |
| `research-workspace` | `research-workspace/` | shell/Python/Node CLI workspace | structured JSONL, redaction, rotation, reports |
| `size-report-sandbox` | not found locally | Node.js/GitHub | inspect when it becomes an adoption target |
| `selfspy` | `selfspy/` | macOS/Python | desktop-local data and privacy-sensitive events |
| `tldr-digest` | `tldr-digest/` | Python + systemd | server stdout/stderr currently collected by journald |

## Findings by project

### `size-report`

**Observed entry points:** `bin/size.js`, `src/cli.js`, Git hooks, and CI profiles in `package.json`.

**Current behavior:** there is no persistent logger. User-facing results go to stdout; diagnostics,
warnings, and errors go to stderr. Several commands have JSON/data modes. Sensor reports are local
ignored artifacts, and hook state is kept under the Git directory.

**Integration constraint:** the stdout contract must not be polluted because other agents and CI consume
the machine-readable modes. A shared layer should be an optional process/run adapter that records command,
run ID, exit code, duration, stdout/stderr references, and generated artifact paths. It should not turn
the CLI into a network-dependent application.

**Useful checks:** `pnpm run verify:fast`, `pnpm run verify`, and `pnpm test`.

### `figma-safe-resets`

**Observed entry points:** `src/manifest.json`, `src/code.js`, and `src/ui.html`. The manifest explicitly
sets `networkAccess.allowedDomains` to `none`.

**Current behavior:** the plugin has a substantial in-memory journal with levels, timestamps, elapsed
time, scan context, phase profiling, cache information, and a bird's-eye summary. It is visible in the
Figma console and can be copied or downloaded from the UI. The journal is bounded and reports truncation.

**Integration constraint:** this project cannot depend on a live collector. The common layer needs a
local-only sink and a deliberate export/import path. The event schema must preserve document/page/scan
context and explicitly mark truncation. Names and layer IDs are potentially sensitive and need the same
redaction policy as server-bound events.

**Useful checks:** `pnpm run test:all` and the focused fingerprint test. The fingerprint test makes log
shape a compatibility surface.

### `yt-summary`

**Observed entry points:** background service worker, YouTube content scripts, popup/options/diagnostics
pages, and `scripts/push-journal.mjs`.

**Current behavior:** this is the most developed browser-side structured journal. Entries have an ID,
timestamp, level, context, registered event name, and scalar `params`/`details`. Events are persisted in
`chrome.storage.local`, bounded to a fixed number of entries, echoed to the console, and shown through a
diagnostics page. A downloaded NDJSON snapshot can be pushed to an orphan `runtime/logs` branch.

**Integration constraint:** MV3 service workers sleep and multiple browser contexts can write at once.
The common adapter must survive restarts, serialize writes, keep an explicit `run_id`/session, and leave
the current local journal useful when upload is unavailable. Secret-bearing provider parameters and
subtitles must never be sent as event fields.

**Useful checks:** `pnpm test:diag`, `pnpm verify`, and the documented journal push dry run.

### `alfa-ui-kit`

**Observed entry points:** Vite library build, Storybook, Vitest, and Playwright visual checks in
`vivica/alfa-ui-kit/package.json`.

**Current behavior:** it is a reusable UI library, not a long-running product process. Development and
test tooling produces logs, but the library itself should not emit a central runtime stream for consumers
by default.

**Integration constraint:** treat this as a consumer-facing library boundary, not a first logging source.
If diagnostics are needed later, they should be opt-in and supplied by the host application. Build and
visual-test evidence may be represented as CI/process events by an outer runner.

### `biblio-mcp` integration

The repository is not present as a separate local checkout, but `research-workspace/docs/biblio-mcp.md`
documents its use. Biblio operations are already tracked in a run-local metadata-only JSONL ledger with
operation ID, operation kind, tool name, outcome, result count/bytes where applicable, checksum status,
and elapsed time. Search text, titles, catalog IDs, local paths, raw responses, and free-form errors are
kept out of that ledger.

**Integration constraint:** this is a useful model for privacy-preserving agent/tool telemetry. The common
system should distinguish safe operational metadata from payload content and should support a project-local
ledger even when no central collector is available.

### `research-workspace`

**Observed entry points:** `bin/rw`, `bin/study.sh`, `bin/hpr.sh`, `bin/browser-fetch.sh`, and
`bin/doctor.sh`.

**Current behavior:** `bin/log.sh` writes both JSONL and human-readable lines. Entries include timestamp,
session, level, component, event, PID, and fields. It supports log levels, redaction, size-based rotation,
HTTP error classification, command duration/exit status, retry context, and summary reports. Logs are
local and ignored by Git. Browser and external-call operations are expected to use this logger.

**Integration constraint:** this is the strongest local starting point for the shared event envelope. It
still needs explicit cross-project fields such as `project`, `runtime`, `run_id`, `operation_id`, and
typed values rather than all fields being strings. Its file sink should remain valid even if a future
central sink is down.

**Useful checks:** `bash tests/run_all.sh`, `bash bin/doctor.sh --json --offline`, and
`bash bin/log-report.sh --since 24`.

### `tldr-digest`

**Observed entry points:** `src/main.py`, `src/config.py`, the browser-agent service, and the server
deployment scripts.

**Current behavior:** Python's standard logging writes human-readable, colored, emoji-enhanced messages
to stdout/stderr. The systemd service sends them to journald. The pipeline logs stages, counts, durations,
provider fallbacks, HTTP failures, model cooldowns, and mail errors. URLs are partially masked, but the
format is not a stable machine-readable event stream.

**Integration constraint:** journald is already the natural production sink and should not be replaced by
a custom always-on daemon on a 1 vCPU/1 GB server. The first server adapter should either emit structured
JSON to stdout or translate standard logging records while retaining a readable console view. Correlation
must cross the digest service, Browser Agent, providers, and manual remote runs. A single article failure
must remain an item-level failure rather than stopping the whole run.

**Useful checks:** the offline Python test suite and the documented systemd/journalctl status probes.

## Cross-project implications

### Common event envelope candidates

The observed projects converge on these concepts:

- `timestamp` with an explicit timezone/precision;
- `project`, `runtime`, `environment`, and `component`;
- `run_id` or `session_id`, plus an `operation_id` for nested work;
- stable `event` name and `level`;
- `outcome` (`started`, `ok`, `partial`, `failed`, `cancelled`, or equivalent);
- duration and retry/attempt information;
- typed safe fields for counts, statuses, provider/model labels, and bounded error classes;
- redaction status and an explicit payload policy;
- optional parent operation ID for cross-process/cross-service correlation.

The contract should not require human-readable prose in every record. Renderers can produce console text,
reports, or agent-oriented query results from the same event.

### Sink classes

At least four sink classes are needed:

1. **Local file or runtime sink** for JSONL and offline investigation.
2. **Console/stderr sink** that preserves existing CLI and service behavior.
3. **Browser/plugin storage/export sink** for restricted runtimes.
4. **Remote collector sink** for the server-accessible agent view.

Sinks should be interchangeable and best-effort by default. A collector outage must not make a Figma scan,
CLI report, or scheduled digest fail unless a project explicitly asks for delivery to be a hard guarantee.

### Important unknowns before architecture selection

- Whether the server is the canonical source of truth or only a searchable replica.
- Whether remote agents may access logs over a private network only, SSH, or a separately authenticated HTTPS
  endpoint.
- How long logs should remain available and whether raw event payloads may leave the workstation.
- Whether delivery should be push, pull, or a hybrid with a local queue.
- Whether human-facing summaries are required in v1 or only agent query primitives.
