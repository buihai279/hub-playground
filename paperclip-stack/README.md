# paperclip-stack

Deploy stack for [Paperclip](https://github.com/paperclipai/paperclip), the
open-source control plane for teams of AI agents. The upstream source is
vendored in the [`../paperclip`](../paperclip) git submodule; this directory
holds the compose files that run it.

## Topology

- One container (`hub-playground-paperclip`) on the published
  `ghcr.io/paperclipai/paperclip:latest` image (the upstream `production`
  target).
- Paperclip's **embedded PostgreSQL** — no external database required.
- Host port `3100` → container `3100` (server + UI).
- Persistent state bind-mounted at `./data` → `/paperclip` (embedded Postgres
  data, uploads, local secrets key, agent workspaces).
- Access model: `authenticated` / `private` (sign-in required, LAN only).

## Runbook

```sh
cd paperclip-stack
cp .env.example .env
# generate the two required secrets
sed -i'' -e "s|^BETTER_AUTH_SECRET=.*|BETTER_AUTH_SECRET=$(openssl rand -hex 32)|" \
          -e "s|^PAPERCLIP_TOOL_ACTION_SIGNING_SECRET=.*|PAPERCLIP_TOOL_ACTION_SIGNING_SECRET=$(openssl rand -hex 32)|" .env
# point the public URL at the host you will browse from
sed -i'' "s|^PAPERCLIP_PUBLIC_URL=.*|PAPERCLIP_PUBLIC_URL=http://192.168.1.100:3100|" .env

mkdir -p data
docker compose up -d
docker compose ps
docker compose logs -f paperclip
```

First boot runs the database migrations before the server starts listening, so
give it a minute. When `docker compose ps` reports the container healthy,
open `PAPERCLIP_PUBLIC_URL`.

`./data` must be writable by uid/gid **1000** — the container drops to that
user (`node`) and the entrypoint will `chown` the mount to `node:node` on
startup.

## Claiming the instance

`authenticated/private` installs let the first admin be claimed from the
browser: open the instance, sign in (or create an account) and choose
**Claim this instance** on the setup screen.

## Optional: enable agent runs

Set any of `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY` in
`.env` and restart. Without a provider key the app still runs; agents cannot
execute work.

## Building from source

```sh
docker compose -f compose.yaml -f compose.build.yaml up -d --build
```

This builds `../paperclip` (the submodule) with the upstream Dockerfile
`production` target. It compiles the native Rust runner and installs five
agent CLI toolchains, so it is slow and image-heavy — prefer the published
image unless you need local changes.

## Operating

```sh
docker compose ps
docker compose logs -f paperclip
docker compose pull && docker compose up -d     # update
docker compose down                             # stop (keeps ./data)
```

`docker compose down -v` and deleting `./data` are destructive: they discard
the embedded database and all agent state.
