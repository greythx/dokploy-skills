# Dokploy Skills

[Agent skills](https://agentskills.io) for operating a Dokploy server through the official `dokploy` CLI. Commands are auto-generated from Dokploy's OpenAPI spec and change over time, so each skill discovers them at runtime instead of hardcoding command lists.

## Skills

- `dokploy-cli` operates any Dokploy server: projects, applications, compose stacks, databases, deployments, domains, environments, users, and server/admin tasks.

## Requirements

- [Dokploy CLI](https://github.com/Dokploy/cli): `npm install -g @dokploy/cli`
- Auth to your panel, one of:
  - `dokploy auth -u <panel-url> -t <api-token>` stores the credentials for future runs
  - `DOKPLOY_URL` and `DOKPLOY_API_KEY` (or `DOKPLOY_AUTH_TOKEN`) shell environment variables
  - A `.env` file in the working directory with the same keys (loaded automatically)

Precedence when multiple are set: shell env vars win over `.env`, which wins over stored `auth` credentials.

## Install

```bash
# Install all skills
npx skills add greythx/dokploy-skills

# Install a single skill
npx skills add greythx/dokploy-skills --skill dokploy-cli

# Preview what the repo contains without installing
npx skills add greythx/dokploy-skills --list
```

## License

[MIT](./LICENSE)