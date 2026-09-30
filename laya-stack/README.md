# laya-stack

Deployment for [NandhaKishorM/laya](https://github.com/NandhaKishorM/laya), added to
this repository as the `laya` submodule (Apache-2.0, pinned to `v0.3.22`).

Laya is a non-autoregressive "System 1" decision engine: you send a state plus a
set of typed questions (`choice`, `score`, `noul`) and get answers back in one
forward pass. It generates no text, so there is nothing to parse and nothing to
hallucinate. A router picks the checkpoint per request — `laya` (ModernBERT-large,
English), `laya-multilingual` (mmBERT-base, 100+ languages), and
`laya-typed-decisions` (the fine-tuned variant).

## What this stack adds over upstream

There is no published container image, so the image is built on the host from the
submodule. Upstream's `compose.yaml` runs a one-shot SDK quickstart and
`compose.http.yaml` adds the HTTP server; this stack keeps the serving half and
changes three things for a shared host:

| Upstream default | Here | Why |
| --- | --- | --- |
| Publishes `127.0.0.1` | `0.0.0.0` | Reachable from the LAN. The API is authenticated, so this is safe. |
| `LAYA_API_KEY` in the environment | `LAYA_API_KEY_FILE` | An environment variable is readable by anyone in the `docker` group through `docker inspect`. The entrypoint loads the file and drops the `_FILE` variable before exec'ing the server. |
| `model-cache` volume | Explicitly named volume | Keeps the ~1.5 GB of checkpoints addressable across rebuilds. |

## Topology

- **`laya-serve`** — the upstream FastAPI/uvicorn server inside the image, started
  as `laya-serve`. Serves `GET /health`, `POST /v1/systemone` and
  `POST /v1/systemone/batch` (capped at 64 states).
- **Host port** — `8000` (configurable via `LAYA_PORT`). The container binds the
  same port, so the two sides cannot drift apart.
- **`hub-playground-laya-model-cache`** — named volume mounted at
  `/home/laya/.cache/huggingface`, holding the downloaded checkpoints.
- **`./secrets/laya_api_key`** — mounted read-only at `/run/secrets/laya_api_key`.
- **Device** — `cpu`. This host has no NVIDIA runtime and no `/dev/nvidia*` node,
  so the CUDA build args are not used.

> The image runs as uid **10001** (`laya`), which is why the model cache is a named
> volume rather than a bind mount: creating a host directory owned by that uid
> would need sudo, which this host does not grant.

## Runbook

```sh
cd laya-stack
cp .env.example .env                 # non-secret settings only

mkdir -p secrets                     # 700: only you and root can list it
head -c 32 /dev/urandom | od -An -tx1 | tr -d ' \n' > secrets/laya_api_key
printf '\n' >> secrets/laya_api_key  # the entrypoint strips whitespace anyway
chmod 644 secrets/laya_api_key       # the container reads it as uid 10001
chmod 700 secrets

docker compose up -d --build
docker compose ps                    # expect (healthy)
```

The first build pulls `python:3.11-slim-bookworm`, installs a CPU `torch` wheel and
installs `laya[serve]` — several minutes, and a couple of GB of image. Rebuilds
reuse the layer cache unless `pyproject.toml` or `laya/` changed.

## Warming the models

`LAYA_PRELOAD=0` (the default here) means each checkpoint is downloaded and built
on **first use**, not at boot. Upstream ships `LAYA_PRELOAD=1` but warns that it
makes the container download the whole family on first boot, so this stack warms
the two checkpoints the router actually picks, deliberately and observably:

```sh
KEY=$(cat secrets/laya_api_key)

# English checkpoint (~843 MB)
time curl -s localhost:8000/v1/systemone \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"state":{"body":"billed twice, refund please"},"questions":{"dept":{"type":"choice","instructions":"which team?","criteria":{"billing":"refunds","tech":"bugs"}}}}'

# Multilingual checkpoint (~644 MB + a 34 MB tokenizer)
time curl -s localhost:8000/v1/systemone \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{"state":{"body":"Tôi bị tính phí hai lần, xin hoàn tiền"},"questions":{"dept":{"type":"choice","instructions":"Bộ phận nào?","criteria":{"billing":"hoàn tiền","tech":"lỗi hệ thống"}}}}'
```

The first call per checkpoint blocks while it downloads; watch progress with
`docker compose logs -f`. Everything after that is served from the cache volume.

## Calling the API

```sh
KEY=$(cat laya-stack/secrets/laya_api_key)
curl -s http://192.168.1.100:8000/v1/systemone \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{
    "state": {"body": "I was charged twice for my subscription. Please refund the duplicate charge."},
    "questions": {
      "department": {"type": "choice", "instructions": "Which department should handle this request?",
                     "criteria": {"billing": "invoices, payments, refunds",
                                  "technical": "bugs, outages, system errors",
                                  "sales": "pricing, new contracts"}},
      "urgency": {"type": "score", "instructions": "How urgent is this request?",
                  "criteria": ["not urgent", "needs attention soon", "critical deadline or blocking issue"]},
      "refund_requested": {"type": "noul", "instructions": "Does the user explicitly request a refund?"}
    }
  }'
```

Batch several states in one call with `POST /v1/systemone/batch` and a `states`
array. The wire format is identical to TypeSafe's hosted Jev API, so a Jev client
only needs its `baseUrl` repointed.

Question types:

- **`choice`** — pick one label from `criteria`. Upstream caps 100 options per
  question at the HTTP layer.
- **`score`** — pick a level from an ordered list.
- **`noul`** — probability that the answer is yes.

## Authentication

`LAYA_API_KEY_FILE` points at `./secrets/laya_api_key`; the entrypoint moves it into
`LAYA_API_KEY` and unsets the `_FILE` variable before starting the server, so the
key never appears in the container's environment and never appears in
`docker inspect`.

With the key set, requests must carry `Authorization: Bearer <key>` and anything
else gets `401`. Keep `secrets/` at mode 700 — the file itself is 644 only so that
the container's uid 10001 can read it, and directory permissions on the host are
what keep it away from other local users.

If the key is ever exposed, rotate it and restart:

```sh
head -c 32 /dev/urandom | od -An -tx1 | tr -d ' \n' > secrets/laya_api_key
chmod 644 secrets/laya_api_key
docker compose up -d --force-recreate
```

## Operating

```sh
docker compose ps                     # health + uptime
docker compose logs -f                # model downloads, request lines
docker stats --no-stream hub-playground-laya    # CPU/RAM
docker volume inspect hub-playground-laya-model-cache
```

- **Health** — `GET /health` also reports the device each checkpoint really runs on
  and how often it fell back to CPU.
- **Memory** — the router keeps two checkpoints resident by default (~1.5 GB of
  weights, more once loaded as fp32, plus the torch runtime), which is exactly the
  number automatic routing picks between. `LAYA_MAX_LOADED=1` caps it at one, but on
  CPU that costs **20-23 s per request** while the evicted checkpoint rebuilds
  (upstream measured this in #172), so only set it if the box is genuinely out of
  room. Narrowing with `LAYA_MODELS=multilingual` is the cheaper lever.
- **One inference at a time** — the server runs predictions on a single-worker
  executor, so requests queue rather than run in parallel. `/health` stays responsive
  while a prediction is in flight.
- **Threads** — `LAYA_THREADS` / `OMP_NUM_THREADS` are 4 of this host's 10 cores,
  leaving room for the MedBASE staging stack.
- **Busy responses** — the server returns `503` with `Retry-After` when it has no
  checkpoint slot free.

## Updating

```sh
cd laya && git fetch --depth 1 origin tag vX.Y.Z && git checkout vX.Y.Z && cd ..
# bump LAYA_IMAGE_TAG in .env, then:
cd laya-stack && docker compose up -d --build
```

The model cache survives rebuilds. An upgrade that changes the Hub revision can be
pinned with `LAYA_REVISION`; `HF_HUB_OFFLINE=1` makes the server refuse Hub lookups
and serve only what is already cached.

## Limits worth knowing

- **CPU only here.** Upstream's latency figures (tens of milliseconds) are measured
  on GPUs. On this host expect substantially more per request; measure with the
  warm-up calls above rather than trusting the README numbers.
- **English is the strong checkpoint.** The multilingual one is ~2x faster but less
  accurate on non-Latin scripts; the English checkpoint degrades badly there.
- **`confidence` is not Jev's confidence.** It is `1 - normalised entropy`; gate on
  `answer_confidence` instead.
