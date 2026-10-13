# Agent guide

## Purpose

This repository is the design and implementation home for a shared logging access layer. Keep the first
working slice small, testable, and compatible with the existing projects rather than introducing a new
framework by default.

## Documentation

- `README.md` is the short project map.
- `docs/research/active-projects.md` records observed source behavior and should be updated when the
  inventory changes.
- `docs/constraints/server-profile.md` contains only redacted infrastructure constraints.
- `docs/decision/` contains unresolved product decisions and accepted architecture records.
- `docs/worklog/` contains task-specific setup notes; archive closed tasks under
  `docs/worklog/archive/`.

## Safety

- Never commit API keys, SSH keys, passwords, cookies, raw production logs, or unredacted server config.
- Do not change VPN services, firewall rules, reserved ports, or unrelated projects as part of this
  repository's work.
- Do not treat a successful local serialization test as proof that remote delivery works; test the full
  boundary when an adapter is added.

## Checks

The repository is currently documentation-only. Until implementation begins, the meaningful checks are:

```bash
git diff --check
```

Once code is introduced, add the project's actual lint, typecheck, unit, and integration commands here
before claiming them as checks.

## Commits

Use Conventional Commits. Keep one logical change per commit and update evergreen documentation with
functional changes.
