# dokploy-cli

Agent skill for operating a Dokploy server through the official `dokploy` CLI.

The CLI's commands are auto-generated from Dokploy's OpenAPI spec, so the skill discovers them at runtime via `dokploy --help` instead of hardcoding command lists. Full instructions live in [SKILL.md](SKILL.md). They cover authentication and auth failures, option semantics (bare boolean flags, enums in `--help`, required-option fail-fast), output conventions (`--json`, `OK` for null results), and safe handling of destructive commands and secrets.

For installation and auth requirements, see the repo [README](../../README.md).
