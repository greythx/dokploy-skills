---
name: dokploy-cli
description: Operate a Dokploy server with the official Dokploy CLI (`dokploy`) — apps, compose stacks, databases, deployments, domains, users, server/admin. Trigger whenever the user mentions Dokploy or their Dokploy panel, including deploy requests that name Dokploy ("deploy to dokploy", "deploy my app on dokploy"). If a deploy or infra request doesn't name a platform, ask the user where they want to deploy instead of assuming Dokploy.
license: MIT
compatibility: Requires the Dokploy CLI (@dokploy/cli) and network access to a Dokploy panel
---

# Dokploy CLI

The official Dokploy CLI (`@dokploy/cli`, invoked as `dokploy`) manages a Dokploy server from the shell. Its commands are auto-generated from Dokploy's OpenAPI spec — anything reachable in the web panel is reachable here. Prefer it over hand-rolling HTTP calls to the panel API. The command set changes as the spec evolves, so treat the CLI itself as the source of truth.

## When to use this skill

Use this skill only when the user names Dokploy as the target: their panel, their server, or an app on Dokploy. When a deploy or infra request doesn't name a platform, ask the user where they want to deploy; don't assume Dokploy. Before acting, verify the target is Dokploy-managed — check for a panel URL in env/auth config (`DOKPLOY_URL` or stored credentials), or ask the user if it's unclear.

## Availability and authentication

- Confirm the CLI is installed with `dokploy --help`; a "command not found" error means it isn't.
- If it's missing: don't install it yourself. Tell the user it's required for the task and point them to https://github.com/Dokploy/cli
- The CLI authenticates from the first available of these (shell env wins over `.env`, which wins over stored config):
  1. Shell env vars `DOKPLOY_URL` and `DOKPLOY_API_KEY` (or `DOKPLOY_AUTH_TOKEN`)
  2. A `.env` file in the working directory with the same keys (loaded automatically)
  3. Stored credentials from a prior `dokploy auth -u <panel-url> -t <api-token>` run
- A fast auth check is `dokploy user get` — non-interactive, errors out immediately when not authenticated.
- If auth fails: don't guess credentials or panel URLs. Ask the user to authenticate. A URL-parsing error usually means `DOKPLOY_URL` is missing its scheme (`https://`).

## Command discovery — always query the CLI

Resolve everything at runtime:

- `dokploy --help` — every group
- `dokploy <group> --help` — the group's actions
- `dokploy <group> <action> --help` — the action's exact options

These help commands are safe and need no auth.

## Invocation

Everything follows `dokploy <group> <action> [options]`. Groups/actions are kebab-case (`preview-deployment`, `add-worker`); option flags are camelCase derived from OpenAPI property names (`--applicationId`, `--environmentId`, ...). Work help-first: read the action's `--help` before invoking it — guessing flags is the #1 cause of broken commands.

Option semantics you can rely on:

- Boolean options are bare flags — pass `--isEnabled`, not `--isEnabled true`; omit the flag to leave it unset.
- Enum options list their allowed values in `--help` in parentheses (`--context <value>  context (build, runtime)`); use one of those values.
- Required vs optional is NOT marked in `--help`. If you omit a required option, commander fails fast before anything hits the panel, naming the option: `error: required option '--applicationId <value>' not specified`. So probing with a best-guess invocation is a cheap way to surface missing flags — the error tells you exactly what's missing. A probe that runs without erroring is a real invocation that hits the panel, so only probe commands you'd be willing to run as-is.
- Option descriptions in `--help` are terse (just the property name), so infer an option's meaning from its group, action, and name (e.g. `--domainId` on `domain delete` is the domain being deleted).
- Options whose API type is an array or object (a list of domains, a JSON config, a metadata blob) are exposed as plain string flags and may not round-trip to the API correctly. Prefer endpoints that take scalar values; if a structured value is required, tell the user the CLI may not support it.

## Output

- Default output for objects is already pretty-printed JSON. Pass `--json` when you'll post-process or feed a result back — it emits raw JSON for every result shape, including `null` and strings.
- IDs returned by list/get commands (`projectId`, `environmentId`, `applicationId`, ...) are the values to feed into later commands — take them from the output rather than asking the user.
- Mutations often return nothing: the API responds with `null` and the CLI just prints `OK` — the ID, when you need it, comes from a list/get afterwards.
- Typical entity workflow: list → read the id from the output → act on that id. Chain ids across steps instead of re-asking.

## Errors

On failure the CLI prints a red error message and exits non-zero. Stay help-first — re-read the action's `--help`, fix the offending option (usually a missing required flag or a value the API rejected), and retry. A URL-parsing error means auth config is broken — see above.

## Safety

Commands act immediately on the live panel and are not undoable. Before running anything destructive or disruptive — deletes, removes, stops, redeploys/rollbacks, cleanups, kills, drops, password resets — confirm the exact target (name + id) with the user. When an action name suggests a mutation but you're unsure what it touches, use a read-only list/get first to identify the target, then present the exact command for approval. Fetching entities can return secrets (API tokens, env values, database passwords) — never echo or persist them; surface only what the user asked for.