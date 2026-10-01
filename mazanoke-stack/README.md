# MAZANOKE stack

Self-hosted MAZANOKE — a browser-side image optimizer (compress, convert, resize) that
runs entirely on the client. No upload, no backend, no stored data.

- **Upstream:** [civilblur/mazanoke](https://github.com/civilblur/mazanoke) — GPL-3.0
- **Submodule:** `../mazanoke`, pinned to the `v1.1.7` release (`41d2ddfc`)
- **Image:** `ghcr.io/civilblur/mazanoke:v1.1.7` (upstream publishes multi-arch images — nothing is built here)
- **Deployed on:** `192.168.1.100` (MedBASE staging host), directory `/home/server/hub-playground/mazanoke-stack`
- **URL:** <http://192.168.1.100:3474/>

## What actually runs

A stock `nginx:alpine` serving five prebuilt static files. There is no application
server, no database, no scheduler and no inference. Images are decoded and re-encoded
by JavaScript in the visitor's browser, so **no file and no metadata ever reaches this
host** — the container's only job is to hand out HTML/JS/CSS.

Two consequences worth stating plainly:

- **No volumes.** Nothing is persisted, so there is nothing to back up, and no
  host-uid/container-uid permission problem to solve (contrast `laya-stack`, which needed
  a named volume because its image runs as uid 10001).
- **Blast radius is small.** A compromise of this container yields a static file server
  plus whatever the `nginx` user can read inside the image. It cannot touch the MedBASE
  staging stack.

## Topology

| Item | Value |
| --- | --- |
| Container | `hub-playground-mazanoke` |
| Image | `ghcr.io/civilblur/mazanoke:v1.1.7` (pin the tag, not `latest`) |
| Published port | `3474` → container `80` |
| Volumes | none |
| Compose project | `mazanoke-stack` (directory name) |
| Process | nginx master as root, workers as `nginx` |

The container port is not configurable: upstream's `config/nginx.conf` hard-codes
`listen 80`. Only the host side moves.

## Deploy

```bash
cd /home/server/hub-playground/mazanoke-stack
cp .env.example .env && chmod 600 .env     # first time only
docker compose up -d
```

First `up` pulls the image (~50 MB compressed). Subsequent runs reuse it.

## Operating

```bash
docker compose ps                 # health + published port
docker compose logs -f            # nginx access/error log
docker compose restart            # clean stop is fast (see below)
docker compose down               # remove the container (image stays)
docker compose pull && docker compose up -d   # move to a newer tag
```

`docker compose restart` returns in well under a second because the container runs
nginx as PID 1 (see the `command:` note in `compose.yaml`). With upstream's own
`CMD` the stop path signals a shell instead, and Docker has to wait out its full
10 s grace period before the SIGKILL.

## Basic auth

Upstream can gate the app with HTTP basic auth: set **both** `MAZANOKE_USERNAME` and
`MAZANOKE_PASSWORD` in `.env` and recreate the container. If either is empty, auth is
skipped. Nothing else is required — `scripts/basicauth.sh` writes `/etc/nginx/.htpasswd`
and patches the nginx config at container start.

**This stack ships with both empty, on purpose.** Before you turn it on, know two things:

1. **The password becomes an environment variable.** It is visible in `docker inspect`,
   so anyone who can run docker on this host can read it. The same exposure that forced
   a rotation of `data-formulator-stack`'s `FLASK_SECRET_KEY`. Treat it as a speed bump,
   not a secret, or put the instance behind the reverse proxy instead.
2. **It only gates `index.html`.** `basicauth.sh` inserts `auth_basic` into the
   `location = /index.html` block alone; `/assets/*`, `/manifest.json` and
   `/service-worker.js` stay publicly readable. Requesting `/` still challenges you,
   because `try_files` internally redirects to `/index.html`. So the app shell is
   protected while its files are not.

Neither caveat matters much here, because there is nothing behind the door worth
protecting — which is exactly why the default is "off" and access is by port only.

If you want real access control, terminate TLS on a reverse proxy and either
`MAZANOKE_BIND_ADDRESS=127.0.0.1` the container or keep it LAN-only.

## Security notes

- **No secret exists in this stack.** No API keys, no database password, no token file.
  `mazanoke-stack/.gitignore` still ignores `.env` and `secrets/` so that enabling basic
  auth cannot accidentally commit a password.
- **The image runs as root by default** — upstream never sets `USER`, and nginx binds
  `:80`, which needs `CAP_NET_BIND_SERVICE`. Adding `cap_drop: [ALL]` would require
  re-adding the capability plus `SETUID`/`SETGID`; it was not worth diverging further for
  a static file server. `no-new-privileges:true` and `pids_limit: 256` are set.
- **`latest` is deliberately avoided.** `v1.1.7` is a release tag; upstream also
  publishes `dev`, which should never be deployed here.
- **Verify what you pulled** if you care about supply chain: the `v1.1.7` index resolves
  to the amd64 manifest `sha256:145256153b33cc30c86405d5b6c86278bc16bafd73014e8bdafa337a7a340985`.
- **The app is reachable unauthenticated on the LAN and tailnet.** See "Basic auth" for
  what that does and does not mean.

## Updating

The image is a static bundle, so a tag bump is the whole upgrade:

```bash
# locally: pin the new release, then commit
cd ../mazanoke && git fetch --depth 1 origin tag v1.1.8 && git checkout v1.1.8
# server: edit MAZANOKE_IMAGE_TAG in .env, then
docker compose pull && docker compose up -d
```

There are no migrations, no schema, and no cache to clear. Rollback is the same
procedure with the previous tag.

## License

MAZANOKE is **GPL-3.0**. This stack only pulls and runs the published image; it does not
bundle, modify or redistribute the software. If that ever changes — for example if a
derived image is built and shipped to someone else — the GPL obligations travel with it.
