# data-formulator-stack

Deploy stack for [Data Formulator](https://github.com/microsoft/data-formulator)
(Microsoft, MIT) — an interactive AI-powered data analysis and visualisation
app. The upstream source is vendored in the
[`../data-formulator`](../data-formulator) git submodule; this directory holds
the compose file that runs it.

## Topology

- One container (`hub-playground-data-formulator`) built from the **upstream
  Dockerfile** in `../data-formulator`: a node 20 stage compiles the
  React/TypeScript frontend, a python 3.11 stage installs the Python package
  with that frontend bundled in.
- **No published image exists** upstream — there is no Docker Hub or GHCR
  package for Data Formulator, and the CI workflows only produce PyPI releases
  and desktop bundles. `docker compose up --build` is the supported path.
- No database service: workspace state is files on disk.
- Host port `5567` → container `5567` (server + UI).
- Persistent state bind-mounted at `./data` → `/home/appuser/.data_formulator`
  (uploaded files, parquet tables, session metadata).
- Access model: no login (anonymous), reachable from the LAN.

## Runbook

The build context is the submodule next to this directory, so both paths must
be present on the host.

```sh
cd data-formulator-stack
cp .env.example .env
# session-signing key — without it every restart invalidates all sessions
sed -i'' "s|^FLASK_SECRET_KEY=.*|FLASK_SECRET_KEY=$(openssl rand -hex 32)|" .env

mkdir -p data
docker compose up -d --build
docker compose ps
docker compose logs -f data-formulator
```

The first build pulls `node:20-slim` and `python:3.11-slim`, runs a yarn install
plus a Vite build, and then a pip install — expect several minutes and a ~2.4 GB
image. Later builds reuse the layer cache and only redo the stage whose inputs
changed.

### `./data` must exist before the first `up`

`./data` must be writable by uid **1000**: the image runs as `appuser` (uid
1000), which is the same uid as the host's `server` user — hence the `mkdir -p
data` in the runbook above. Do not skip it.

The container has no root phase, so nothing inside it can repair the mount for
you. If `./data` is missing, Docker creates it as `root:root` and the container
crash-loops with:

    PermissionError: [Errno 13] Permission denied: '/home/appuser/.data_formulator/sessions'

Recover by removing and recreating the (still empty) directory as your own user,
then starting again:

```sh
rmdir data && mkdir data && docker compose up -d
```

## Using it

Open `http://<host>:5567`. Upload a CSV/Excel/JSON file or connect a data
source, then describe the chart you want.

The app is inert without an LLM. Either set provider keys in `.env` (see
`.env.example`), or leave them unset and paste a key in the UI — the latter
requires `DISABLE_CUSTOM_MODELS=false`.

Sessions, workspaces and the credential vault are namespaced by an identity
resolved per request. This stack sets `HOST=0.0.0.0` so that identity is
`browser:<uuid>`, a UUID the browser keeps in local storage — each visitor gets
a separate namespace, at the cost of orphaning their data if they clear site
data.

Without `HOST=0.0.0.0` the app instead enters single-user mode (the upstream
code reads `HOST` at import time and defaults it to `127.0.0.1`) and hands
**every** visitor the same fixed `local:<os_user>` identity. That was the
behaviour of the first deploy here: `/api/app-config` reported
`"IS_LOCAL_MODE": true` and `"IDENTITY": {"type": "local", "id": "appuser"}`,
so all visitors shared one workspace.

## Security notes

This instance is **anonymous by default** (`AUTH_PROVIDER` unset). Anyone who
can reach the port can drive it, and it executes LLM-generated Python through
its sandbox. The sandbox is `local` — an in-process subprocess with audit hooks
that block file writes and dangerous operations — not a container boundary.

If more than one person can reach the port, consider the upstream
"multi-user anonymous" profile in `.env`:

```sh
DISABLE_DATA_CONNECTORS=true   # no MySQL/PostgreSQL/… connectors
DISABLE_CUSTOM_MODELS=true     # no user-supplied api_base (SSRF)
DISABLE_DISPLAY_KEYS=true      # never show API keys in the UI
```

`DISABLE_DATABASE=true` enables all three at once. Note this also removes the
ability for users to bring their own LLM key, so configure providers in `.env`
if you enable it. With custom models allowed, `DF_ALLOWED_API_BASES` restricts
which endpoints users may target.

Do **not** set `SANDBOX=docker`: it requires bind-mounting workspace paths into
child containers, which does not work when Data Formulator itself runs in a
container.

## Operating

```sh
docker compose ps
docker compose logs -f data-formulator
docker compose up -d --build          # rebuild after moving the submodule
docker compose down                   # stop (keeps ./data)
docker compose down --rmi local       # also drop the built image
```

`./data` holds every workspace. To back it up, stop the container and tar the
directory (parquet files are large — check `du -sh data` first). Deleting
`./data` discards all uploaded data and sessions.

## Updating

1. Move the submodule: `git -C ../data-formulator fetch --depth 1 && git -C
   ../data-formulator checkout <commit-or-tag>`.
2. Bump `DF_IMAGE_TAG` in `.env` so the old image stays identifiable.
3. `docker compose up -d --build`.

Keep the tag in step with the submodule commit: the image tag is the only record
of what the running container was built from.
