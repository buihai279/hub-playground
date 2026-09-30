# odoo-stack

Deploy stack for [Odoo 20](https://github.com/odoo/odoo) (community edition) —
ERP/CRM, accounting, website and e-commerce. The upstream source is vendored in
the [`../odoo`](../odoo) git submodule; this directory holds the compose files
that actually run it.

## Topology

| Service | Image | Host port | Data |
|---------|-------|-----------|------|
| `odoo`  | `odoo:20` | `0.0.0.0:8069 -> 8069` | named volume `odoo-stack_odoo-data` (`/var/lib/odoo`), `odoo-stack_odoo-addons` (`/mnt/extra-addons`) |
| `db`    | `postgres:17-alpine` | not published | named volume `odoo-stack_odoo-db` (`/var/lib/postgresql/data`) |

- Odoo 20 requires **PostgreSQL >= 16** (`MIN_PG_VERSION` in `odoo/release.py`);
  the stack pins PostgreSQL 17.
- Both data sets live in Docker **named volumes**, not bind mounts. The official
  images run as `uid 101` (`odoo`) and `uid 999` (`postgres`), which do not match
  a typical host user — named volumes let the images own their own data without
  host-side `chown`.
- `--workers=0` runs the threaded server (lower memory, fine for one instance).
  Demo data is disabled (`--without-demo=all`).
- The stack is fully isolated: its own compose project, network, database and
  volumes. It does not touch any other service on the host.

## Runbook

```sh
cd odoo-stack
cp .env.example .env

# generate the two required secrets
printf 'ODOO_PORT=8069\nPOSTGRES_PASSWORD=%s\nODOO_MASTER_PASSWORD=%s\n' \
  "$(openssl rand -hex 24)" "$(openssl rand -hex 16)" >> .env

docker compose up -d
docker compose ps
docker compose logs -f odoo
```

First run: open the database manager and create the initial database.

- URL: `http://<host>:8069/web/database/manager`
- Master password: `ODOO_MASTER_PASSWORD` from `.env`.
- Name the database, uncheck "Load demonstration data", set the admin email and
  password. This first database is what `/web/login` serves afterwards.

## Operating

```sh
docker compose ps                      # status
docker compose logs -f odoo            # follow logs
docker compose pull && docker compose up -d   # update images
docker compose stop                    # stop, keeps volumes
docker compose down                    # remove containers, keeps volumes
```

`docker compose down -v` is **destructive**: it deletes the Odoo filestore and
the whole PostgreSQL data directory.

### Backups

```sh
# database (repeat per tenant database)
docker compose exec db pg_dump -U odoo -Fc <dbname> > <dbname>.dump

# filestore (attachments, images stored outside the database)
docker run --rm -v odoo-stack_odoo-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/odoo-filestore.tgz -C /data .
```

### Adding custom addons

Put modules in the `odoo-stack_odoo-addons` volume (mounted at
`/mnt/extra-addons`), or add a bind mount in a local `compose.override.yaml`,
then `docker compose up -d` to restart Odoo.

## Secrets

| Variable | Purpose |
|----------|---------|
| `POSTGRES_PASSWORD` | password of the PostgreSQL `odoo` role; also handed to Odoo |
| `ODOO_MASTER_PASSWORD` | database-manager master password at `/web/database/manager` |

Both are required and have no defaults. The master password guards create /
duplicate / backup / drop of every database on the instance — if it is lost it
has to be reset by editing the running instance's config.

## Notes

- The `../odoo` submodule is a **shallow** (`--depth 1`) checkout of branch
  `20.0`. A full clone of `odoo/odoo` is >10 GB, so only the tip is vendored.
  `git submodule update --remote --depth 1 odoo` refreshes it.
- This is the community edition. Enterprise-only modules and the Odoo Cloud
  integrations are not available in `odoo:20`.
