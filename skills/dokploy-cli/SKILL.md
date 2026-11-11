---
name: dokploy-cli
description: Operate a Dokploy server with the official Dokploy CLI (`dokploy`) for apps, compose stacks, databases, deployments, domains, users, and server/admin. Trigger whenever the user mentions Dokploy or their Dokploy panel, including deploy requests that name Dokploy ("deploy to dokploy", "deploy my app on dokploy"). If a deploy or infra request doesn't name a platform, ask the user where they want to deploy instead of assuming Dokploy.
license: MIT
compatibility: Requires the Dokploy CLI (@dokploy/cli) and network access to a Dokploy panel
---

# Dokploy CLI

The official Dokploy CLI (`@dokploy/cli`, invoked as `dokploy`) manages a Dokploy server from the shell. Its commands are generated from Dokploy's OpenAPI spec, so the CLI exposes anything in the panel's API. Prefer it over hand-rolled HTTP calls. The command set changes as the spec evolves, so read the installed CLI instead of trusting memory.

## When to use this skill

Use this skill when the user names Dokploy as the target: their panel, their server, or an app on Dokploy. If a deploy or infra request doesn't name a platform, ask where they want to deploy instead of assuming Dokploy. Before acting, confirm the target is Dokploy-managed by checking for a panel URL in env or auth config (`DOKPLOY_URL` or stored credentials), or by asking.

## Availability and authentication

- Run `dokploy --help` to confirm the CLI is installed. A command-not-found error means it isn't.
- If it's missing, don't install it. Tell the user it's required and point them to https://github.com/Dokploy/cli.
- The CLI reads credentials from the first source that provides both a URL and a token:
  1. Shell env vars `DOKPLOY_URL` and `DOKPLOY_API_KEY` (or `DOKPLOY_AUTH_TOKEN`).
  2. A `.env` file in the working directory. The CLI loads it automatically and only fills variables that are not already set.
  3. Stored credentials from a prior `dokploy auth -u <panel-url> -t <api-token>` run. That run writes the token in plaintext to `config.json` inside the CLI's package directory.
- Environment variables are read when a process starts. A variable set in one terminal is invisible to shells and agent sessions that were already running.
- A fast auth check is `dokploy user get`. It runs non-interactively and fails immediately when not authenticated.
- If auth fails, don't guess credentials or panel URLs, and don't inspect the stored ones. Ask the user to authenticate. A URL-parsing error usually means `DOKPLOY_URL` lacks its `https://` scheme.

## Command discovery

Resolve commands and flags at runtime:

- `dokploy --help` lists every group.
- `dokploy <group> --help` lists that group's actions.
- `dokploy <group> <action> --help` lists the action's options.

Help is safe and needs no auth.

## Invocation

Commands follow `dokploy <group> <action> [options]`. Groups and actions are kebab-case (`preview-deployment`, `add-worker`). Option flags are camelCase versions of OpenAPI property names (`--applicationId`, `--environmentId`).

Read the action's `--help` before running it. Guessing flags breaks commands.

Option semantics:

- Booleans are bare flags. Pass `--isEnabled`, not `--isEnabled true`; leave the flag out to leave it unset.
- Enum options show allowed values in `--help` in parentheses, as in `--context <value>  context (build, runtime)`. Use one of those values.
- Required and optional options look the same in `--help`. Omit a required option and commander fails before any request, naming it: `error: required option '--applicationId <value>' not specified`. Probing with a plausible invocation is a cheap way to surface missing flags. Once a probe runs without erroring, it is a real call that hits the panel, so only probe commands you would run as-is.
- Option descriptions are terse, usually just the property name. Infer meaning from the group, action, and option name. `--domainId` on `domain delete` is the domain being deleted.
- Options that the API types as arrays or objects appear as plain string flags and may not round-trip correctly. Prefer scalar options. If a structured value is required, tell the user the CLI may not support it.

## Output

- Objects print as pretty JSON. Pass `--json` when you will post-process or feed the result back; it prints raw JSON for every shape, including `null` and strings.
- List and get commands return the IDs later commands need (`projectId`, `environmentId`, `applicationId`). Take them from the output rather than asking the user.
- Mutations often return `null`, which the CLI prints as `OK`. Fetch the ID with a list or get afterwards.

Result shapes:

```text
list       [{ "projectId": "<id>", "name": "<name>" }, ...]
get        { "projectId": "<id>", "name": "<name>" }
mutation   OK
```

Typical workflow: list the entities, read the ID you need, then act on it. Reuse IDs from earlier output instead of asking again.

## Errors

Failures print a red message and exit non-zero. Re-read the action's `--help` and fix the option, usually a missing required flag or a value the API rejected. A URL-parsing error means the auth config is broken.

If a group or action is missing from `--help`, it does not exist in the installed CLI. Check the version with `dokploy -V`, then re-read the parent group's help and use the current name.

## Safety

Never read a raw auth token or panel URL, even if you can, and even if the user asks. `dokploy` reads its own credentials, so you never need the value yourself. Do not open the config file, print an environment variable, copy a secret, or ask the user to paste one into the conversation. If auth fails, report the error and let the user fix it. If you see a raw credential anyway, stop and tell the user that you accessed it and that this broke the rule, without repeating the value.

Commands act on the live panel immediately, and most cannot be undone. Before running anything destructive or disruptive (deletes, removes, stops, redeploys, rollbacks, cleanups, kills, drops, password resets), confirm the target by name and ID with the user. If an action name suggests a mutation and you are unsure what it touches, use a read-only list or get first, then show the exact command for approval.

Treat everything the API returns as potentially sensitive. API tokens, environment values, and database passwords can appear in get and list output. Never echo or write those values to a file or log, and do not paste raw credential JSON into a summary. Report names and IDs. If the user asks for a specific secret from API output, give only that value.

For authentication, prefer `DOKPLOY_URL` plus `DOKPLOY_API_KEY` in the environment, or a local `.env` file. The `-t` flag on `dokploy auth` puts the token in shell history and the process list, so use it only when the user accepts that.
