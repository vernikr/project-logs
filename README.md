# project-logs

Design and implementation workspace for a shared logging access layer across personal pet projects.

## Why this exists

The project portfolio contains CLIs, browser extensions, desktop/plugin tools, research workspaces, and
server-side jobs. They currently log to different places and in different formats. This workspace will
define one repeatable logging contract and the smallest practical adapters so an AI agent working across
projects can inspect related runtime history from one access point.

The owner is currently the only human user. AI agents perform implementation, documentation, and routine
operation. A human sets goals and validates the product through real use rather than reviewing every
change. The design therefore prioritizes machine-readable evidence, low-friction adoption, safe failure,
and resumable investigation over dashboards or process ceremonies.

## Current phase

**Requirements settled; architecture proposal in progress.** The initial inventory, server constraints,
and user priorities are recorded in `docs/` without copying credentials. The proposed first implementation
is in `docs/decision/architecture-proposal.md`.

## Initial evidence

- Portfolio summary: `/Users/vernikr/Downloads/projects/mother/projects-overview.md`
- Redacted server notes: `docs/constraints/server-profile.md`
- Active-project investigation: `docs/research/active-projects.md`
- Setup worklog: `docs/worklog/2026.10.13-unified-logs-bootstrap/README.md`

## Planned outcomes

1. A versioned cross-project event schema with explicit redaction and retention rules.
2. Small adapters for the first active projects, without forcing one language or runtime on all projects.
3. A server-side collection/query path that remote AI agents can reach safely.
4. Local fallback behavior for projects that cannot or should not use a network connection.
5. Operational checks that prove logs are present, useful, bounded, and free of credentials.

## Current recommendation

Use the server as a private log hub reached through SSH, with a small SQLite-based index, compressed
filesystem artifacts, and an agent-oriented query command/MCP surface. Return compact tables and
deterministic aggregates first; fetch full content only when the investigation needs it. Keep journald and
existing project-local logs as inputs or fallbacks rather than making every project depend on a network
collector. LLM processing is optional and outside the core path.

## Non-goals for the discovery phase

- No changes to the VPN server.
- No collection of existing private logs into this repository.
- No commitment to a particular database, observability vendor, dashboard, or cloud service.
- No assumption that every project can send logs directly over the network.

## Working rules

- Keep secrets and raw operational logs outside Git.
- Preserve each project's stdout/stderr and local diagnostic contracts while adding adapters.
- Treat logs as evidence for an agent: stable identifiers, timestamps, outcomes, durations, and safe
  context matter more than polished human prose.
- Validate integrations with real project checks and small end-to-end probes before broad rollout.
