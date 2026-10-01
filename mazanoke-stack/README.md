# MAZANOKE stack

Self-hosted MAZANOKE — a browser-side image optimizer (compress, convert, resize) that runs
entirely on the client. No upload, no backend, no stored data.

- **Upstream:** [civilblur/mazanoke](https://github.com/civilblur/mazanoke) — GPL-3.0
- **Submodule:** `../mazanoke` @ `f7be1b1d` — the revision the deployed image was built from (see [Provenance](#provenance))
- **Image:** `ghcr.io/civilblur/mazanoke:v1.1.7` → `sha256:ca83cddd90cd…` on this host
- **Deployed on:** `192.168.1.100` (MedBASE staging host), `/home/server/hub-playground/mazanoke-stack`
- **URL:** <http://192.168.1.100:3474/>
- **Status:** verified healthy, `0.0.0.0:3474->80/tcp`, no volumes, `RestartCount=0`, `OOMKilled=false`

## What actually runs

A stock `nginx:alpine` serving five prebuilt static files (104 MB image on disk). There is no
application server, no database, no scheduler and no inference. Images are decoded and
re-encoded by JavaScript in the visitor's browser, so **no file and no metadata ever reaches
this host** — the container's only job is to hand out HTML/JS/CSS.

Two consequences worth stating plainly:

- **No volumes.** Nothing is persisted, so there is nothing to back up, and no
  host-uid/container-uid permission problem to solve (contrast `laya-stack`, which needed a
  named volume because its image runs as uid 10001).
- **Blast radius is small.** A compromise yields a static file server plus whatever the
  `nginx` user can read inside the image. It cannot reach the MedBASE staging stack.

## Topology

| Item | Value |
| --- | --- |
| Container | `hub-playground-mazanoke` |
| Image | `ghcr.io/civilblur/mazanoke:v1.1.7` (pin the tag, never `latest`/`dev`) |
| Published port | `3474` → container `80` |
| Volumes / mounts | **none** (verified) |
| Compose project | `mazanoke-stack` (directory name) |
| Process | nginx master as root (PID 1), workers as `nginx` |
| Hardening | `no-new-privileges:true`, `pids_limit: 256`, log rotation |
| Root FS | **not** read-only — see "Boot-time write" below |

The container port is not configurable: upstream's `config/nginx.conf` hard-codes `listen 80`.
Only the host side moves.

### Boot-time write

`scripts/metatags.sh` runs at every container start and does
`cp /usr/local/share/mazanoke/index.html.template /usr/share/nginx/html/index.html`. Verified:
the served `index.html` carries the container's start timestamp (`Oct 1 02:43`) while the
template in the image is dated at build time (`Sep 20 17:14`). So the root filesystem **must
stay writable**; adding `read_only: true` would break startup unless a tmpfs were mounted over
`/usr/share/nginx/html`.

## Deploy

```bash
cd /home/server/hub-playground/mazanoke-stack
cp .env.example .env && chmod 600 .env     # first time only
docker compose up -d
```

First `up` pulls the image (104 MB on disk). Subsequent runs reuse it.

## Operating

```bash
docker compose ps                 # health + published port
docker compose logs -f            # nginx access/error log
docker compose restart            # measured: 1.47 s offline
docker compose down               # remove the container (image stays)
docker compose pull && docker compose up -d   # move to a newer tag
```

Restarts are quick: `docker compose restart` completed in **1.47 s** and the app answered `200`
immediately afterwards. nginx runs as PID 1 (verified: `cat /proc/1/comm` → `nginx`), so
SIGTERM reaches it directly and `docker stop` returns in ~0.6 s rather than waiting out the 10 s
grace period before SIGKILL.

Upstream's `CMD` is deliberately left untouched. It reads like
`/bin/sh -c "metatags.sh; basicauth.sh; nginx -g 'daemon off;'"` and looks as though a shell
would end up as PID 1, but BusyBox ash execs the last command of a `-c` string — so nginx is
PID 1 anyway. Keeping the upstream `CMD` also keeps its `;` semantics: a failing helper script
cannot prevent nginx from starting, which an `&&` override would have broken.

## Endpoint behaviour

| Request | Result |
| --- | --- |
| `GET /` | `200 text/html` (48 845 B app shell) |
| `GET /manifest.json` | `200 application/json` |
| `GET /service-worker.js` | `200 application/javascript` |
| `GET /favicon.ico` | `200 image/x-icon` |
| `GET /does-not-exist` | **`200 text/html`** — `error_page 404 /index.html` |

An unknown path returning `200` is upstream's SPA fallback, not a bug. It does mean a naive
"is it up?" check that only looks at the status code cannot distinguish the app from a typo —
use `/manifest.json` or assert on body content.

## Basic auth

Upstream can gate the app with HTTP basic auth: set **both** `MAZANOKE_USERNAME` and
`MAZANOKE_PASSWORD` in `.env` and recreate the container. If either is empty, auth is skipped.
`scripts/basicauth.sh` then writes `/etc/nginx/.htpasswd` and patches the nginx config at
container start; it is idempotent across restarts (it strips old `auth_basic` lines first).

**This stack ships with both empty, on purpose.** Verified: `GET /` returns no
`WWW-Authenticate` header, and the container environment contains `USERNAME`/`PASSWORD` as
empty values. Before turning it on, know two things:

1. **The password becomes an environment variable**, visible in `docker inspect` — so anyone
   who can run docker on this host can read it. That is the same exposure that forced a
   rotation of `data-formulator-stack`'s `FLASK_SECRET_KEY`. Treat it as a speed bump, not a
   secret, or put the instance behind the reverse proxy instead.
2. **It only gates `index.html`.** The config has an explicit `location = /index.html` block
   (where the directives are injected) while `/assets/`, `/manifest.json` and
   `/service-worker.js` have their own blocks with no auth. Requesting `/` still challenges you,
   because `try_files` internally redirects to `/index.html`; but fetching `/assets/…` directly
   does not.

Neither caveat matters much here, because there is nothing behind the door worth protecting —
which is exactly why the default is "off" and access is by port only. For real access control,
terminate TLS on a reverse proxy and either set `MAZANOKE_BIND_ADDRESS=127.0.0.1` or keep it
LAN-only.

## SEO metatags

The image also understands `METATAGS=true`, which injects the **official mazanoke.com** social
metatags into the served HTML (`scripts/metatags.sh` + `metatags.html`). They belong to the
upstream hosted product, not to this instance, so this stack pins `MAZANOKE_METATAGS=false` and
does not expose the setting as a convenience. Verified on the deployment: the served HTML
contains `noindex, nofollow` (from the template) and **zero** occurrences of `mazanoke.com`.

## Provenance

The `v1.1.7` git tag and the `v1.1.7` image **are not the same revision**, and the difference is
worth recording:

| Artifact | Revision |
| --- | --- |
| Git tag `v1.1.7` | `41d2ddfc` — *chore: increment version number* (2026-09-20 18:34 +02:00) |
| `org.opencontainers.image.revision` label on `ghcr.io/civilblur/mazanoke:v1.1.7` | `f7be1b1d` — *refactor: adjust handling of metatags for prod* (2026-09-20 19:13 +02:00) |

`f7be1b1d` is exactly **one commit ahead** of the tag (`status: ahead, ahead_by: 1`), and the
image was built one minute after that commit was authored. The tag was pushed, one further
commit landed, and the release build picked up the later commit while still labelling the image
`v1.1.7`. The observed diff between the two is confined to the metatags feature plus the
matching `CMD` change — no behavioural difference for this deployment.

The submodule is therefore pinned to **`f7be1b1d`**, so the checked-out source matches what is
actually running. Evidence that it does: the Dockerfile at that commit declares
`CMD ["/bin/sh", "-c", "/usr/local/bin/metatags.sh; /usr/local/bin/basicauth.sh; nginx -g 'daemon off;'"]`,
byte-identical to the running container's `.Config.Cmd`, and `scripts/metatags.sh` is 565 bytes
in both the repo and the image.

Verification commands:

```bash
# what revision is the image really?
docker inspect -f '{{index .Config.Labels "org.opencontainers.image.revision"}}' \
  ghcr.io/civilblur/mazanoke:v1.1.7

# what did this host actually pull?
docker inspect -f '{{range .RepoDigests}}{{println .}}{{end}}' \
  ghcr.io/civilblur/mazanoke:v1.1.7
# → ghcr.io/civilblur/mazanoke@sha256:ca83cddd90cd1e4b43102884bf39025731a1c7569f3bbda9d6530de72528cbab

# how far apart are the tag and that revision?
curl -s https://api.github.com/repos/civilblur/mazanoke/compare/v1.1.7...f7be1b1d
```

Because upstream tags are effectively "whatever `main` was when the release ran", a digest is
the only hard pin. The digest above is the one running today; note it in any reproducible
rebuild. `latest` and `dev` are never used here.

## Security notes

- **No secret exists in this stack.** No API keys, no database password, no token file.
  `.gitignore` still ignores `.env` and `secrets/` so that enabling basic auth cannot
  accidentally commit a password.
- **The image runs as root by default** — upstream never sets `USER`, and nginx binds `:80`,
  which needs `CAP_NET_BIND_SERVICE`. Adding `cap_drop: [ALL]` would require re-adding that
  capability plus `SETUID`/`SETGID`; not worth diverging further for a static file server.
- **The app is reachable unauthenticated on the LAN and tailnet.** See "Basic auth" for what
  that does and does not mean.
- **Static files only.** Since the optimizer is client-side, this host never receives user
  images — there is no upload path to attack, and no data to exfiltrate.

## Updating

The image is a static bundle, so a tag bump is the whole upgrade:

```bash
# locally: move the pin, then commit
cd ../mazanoke && git fetch --depth 3 origin main && git checkout <new-revision>
# server: edit MAZANOKE_IMAGE_TAG in .env, then
docker compose pull && docker compose up -d
```

Check `org.opencontainers.image.revision` after pulling and re-pin the submodule to that same
commit — the tag alone will not tell you which revision you got.

There are no migrations, no schema, and no cache to clear. Rollback is the same procedure with
the previous tag.

## License

MAZANOKE is **GPL-3.0**. This stack only pulls and runs the published image; it does not bundle,
modify or redistribute the software. If that ever changes — for example if a derived image is
built and shipped to someone else — the GPL obligations travel with it.
