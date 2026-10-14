# Bootstrap worklog: unified project logs

## Goal

Create a safe planning workspace for a common logging architecture across personal projects and prepare the
first business/operational decision needed before selecting infrastructure.

## Work completed

- Read the portfolio inventory in `mother/projects-overview.md`.
- Inspected the first active local projects and their runtime guidance, including `size-report`,
  `figma/safe-resets`, `yt-summary`, `vivica/alfa-ui-kit`, `research-workspace`, and `tldr-digest`.
- Recorded project-specific logging behavior and integration constraints in
  `docs/research/active-projects.md`.
- Read the supplied server specification and copied only redacted capacity and safety constraints into
  `docs/constraints/server-profile.md`.
- Created the documentation-only project scaffold and secret-safe `.gitignore`.
- Recorded the answer and expanded it into explicit priorities: agent context/time efficiency, remote
  access, and minimal integration effort.
- Compared no-domain/no-new-cost access and storage options, then recommended an SSH-backed MCP/CLI,
  SQLite index, and compressed content-addressed artifacts.
- Added the measured pilot plan, retention/compaction model, notification direction, acceptance criteria,
  risks, and public source links in `docs/decision/architecture-proposal.md`.
- Revised the proposal after requirements discussion: removed mandatory LLM/prose generation, made
  TSV-like output and read-only SQL first-class, added project views over `fields_json`, documented SSH
  credential handling and resource estimates, and reordered the pilot projects.
- Reviewed the second external critique and the supplied network-access research. Accepted the useful
  corrections around browser coverage, stdio-MCP stdout safety, fleet overview, SQLite concurrency,
  artifact boundaries, and compact output. Kept SQLite as the current recommendation but left time
  partitioning and VictoriaLogs as benchmark-driven decisions.
- Updated the architecture to support a possible authenticated HTTPS gateway on
  `38.244.152.2.sslip.io:8443` while retaining SSH for private/admin access. No public listener or server
  configuration was changed.
- Performed a read-only live server spot-check: ports `80` and `8443` were unused, `443`/`20724`/`2096`
  and `51820/udp` remained occupied by existing services, UFW was inactive, available memory was about
  564 MiB, journal usage was 110 MiB, the `tldr-digest` timer was active, and no Caddy/nginx/Tailscale/
  Cloudflare/fail2ban/rsyslog service was active. These are dated observations, not permanent assumptions.
- Recorded owner decisions: raw-artifact access through public HTTPS remains undecided; independent backup is
  deferred; direct Figma UI-iframe delivery through HTTPS is part of the pilot with export fallback.
- Added the useful low-overhead ideas from the earlier proposal: a small cross-language logger surface,
  per-project event catalogues, one-step `logs` commands and `fields` discovery, `doctor` end-to-end probes,
  systemd `OnFailure` fast alerts plus periodic absence rules, and compact TSV/hybrid output modes.

## Checks

| Command | Working directory | Result |
|---|---|---|
| `git diff --cached --check` | `project-logs/` | passed for the initial commit |
| `git diff --check` | `project-logs/` | passed after the requirements update |
| `git diff --check` | `project-logs/` | passed after external-review update |

No application checks exist yet because this phase contains documentation only. No server changes were
made and no credentials were copied.

## Next step

Review `docs/decision/architecture-proposal.md`. Development is intentionally not started yet. Remaining
business decisions are whether raw artifacts may be read through public HTTPS and whether independent
periodic backup is mandatory.
