# Requirements decision

The first architecture choice was settled on 2026-10-13. The requirement is about agent work, not human
operations: a useful investigation should take as few context-consuming reads and tool calls as possible.

## Recorded priorities

1. **Context and time efficiency for the AI agent.** A single query should return a compact diagnosis,
   grouped failures, relevant timeline, and pointers to large payloads. The agent must not read raw logs
   line by line merely to understand what happened.
2. **Remote agent access.** Logs from capable projects should reach the server and be queryable remotely.
   The server is the default operational access point.
3. **Minimal integration effort.** Connecting another project should require a small adapter or wrapper,
   not a new logging design.

## Constraints and preferences

- No owned domain is available or expected.
- No additional paid services are acceptable.
- The existing server is available and is the preferred place for remote access.
- Logs may contain arbitrary payloads, including long text, images, and files.
- Old payloads should be compacted, summarized, deduplicated, compressed, or deleted when necessary.
- Final retention policy is intentionally undecided; it may combine age and disk-size limits.
- Notifications to the owner by Telegram and/or email are desirable.
- Existing VPN services, ports, firewall rules, and resource limits remain protected.

## Consequence

The proposal should be a server-centered, agent-oriented log store with a query layer that produces compact
tables and deterministic aggregates first and exposes raw content only on demand. SSH should remain the
private/admin transport. The supplied network research shows that authenticated HTTPS on a free port with
the `38.244.152.2.sslip.io` hostname is feasible for browser runtimes and cloud agents that cannot use SSH;
it must be treated as a separate, restricted gateway rather than as direct database access. Clients that
cannot connect directly should have a durable export/spool path. LLM processing should remain optional, not
a requirement for ingestion or analysis.
