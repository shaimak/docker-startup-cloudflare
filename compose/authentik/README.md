# Authentik — Single Sign-On (Identity Provider)

The identity provider for the stack. Other apps (Outline, Guacamole, anything
behind forward-auth) federate to it over OIDC. Nothing else needs its own user
database.

- **Images:** `ghcr.io/goauthentik/server` (server + worker) + `postgres:16-alpine`
- **Containers:** `authentik-server`, `authentik-worker`, `authentik-db`
- **Public host (example):** `mysso.mydomain.com` → `authentik-server:9000`
- **Networks:** `authentik_internal_network` (db) + `proxy_network` (server)

> **No Redis.** Authentik 2025.10 dropped it — caching, background tasks,
> outpost sessions and websockets all run on PostgreSQL now. Guides that still
> show a `redis` container and `AUTHENTIK_REDIS__HOST` are pre-2025.10; on
> current versions both are simply ignored.

**Before you start**

- **Requires:** [caddy](../caddy/README.md) already up — it owns `proxy_network`.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.

---

## Setup

```bash
# 1. Env file
cp compose/authentik/.env.example .env_files/authentik.env

# 2. Fill in secrets:
#    AUTHENTIK_SECRET_KEY  ->  openssl rand -hex 32
#    PG_PASS               ->  openssl rand -hex 24
#    AUTHENTIK_URL         ->  https://mysso.mydomain.com  (your public URL — used for redirects)

# 3. Start the stack
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env up -d

# 4. Add the Caddy site block (below), then recreate Caddy
```

Caddyfile:

```caddy
mysso.mydomain.com {
    encode zstd gzip
    reverse_proxy authentik-server:9000
}
```

---

## First run

1. Browse to `https://mysso.mydomain.com/if/flow/initial-setup/`.
2. Set the `akadmin` password. This is your break-glass admin — keep it safe.
3. From here, create Providers + Applications for each app you want to protect.

### Optional: skip the setup page

If you would rather not click through the web form — scripted installs, or you
just want the admin to exist the moment the container is up — set these in
`.env_files/authentik.env` **before the first `up -d`**:

```ini
AUTHENTIK_BOOTSTRAP_PASSWORD=<a strong password you choose>
AUTHENTIK_BOOTSTRAP_EMAIL=you@mydomain.com
```

Authentik seeds the `akadmin` account with them and you can log in at
`https://mysso.mydomain.com/` immediately. Three things to know:

- **First boot only.** They are read once, against an empty database. Setting
  them later, or changing them after the fact, does nothing — the account
  already exists. To change the password after that, use the Authentik UI.
- **Blank them out afterwards.** Once you are logged in, clear both lines. A
  plaintext admin password sitting in an env file is easy to forget about.
- **Never commit them.** `.env_files/` is gitignored for this reason. The
  template in `.env.example` ships commented out and empty.

---

## Wiring another app to it

- **OIDC apps (Outline, etc.):** create an *OAuth2/OpenID Provider* + *Application*
  in Authentik, then paste the client id/secret and the authorize/token/userinfo
  URLs into that app's env file. See [outline](../outline/README.md).
- **Apps with no auth of their own:** protect them with Caddy **forward-auth**.
  Create a *Proxy Provider* (mode `forward_single`, external host = the app's
  URL) + Application, attach it to the **embedded outpost**, then use the
  forward-auth template block in the [Caddyfile](../caddy/Caddyfile).

The standard Authentik endpoints for OIDC apps:

```
authorize:  https://mysso.mydomain.com/application/o/authorize/
token:      https://mysso.mydomain.com/application/o/token/
userinfo:   https://mysso.mydomain.com/application/o/userinfo/
jwks:       https://mysso.mydomain.com/application/o/<app-slug>/jwks/
issuer:     https://mysso.mydomain.com/application/o/<app-slug>/
```

> Server-to-server calls (like a JWKS fetch from another container) can target
> `http://authentik-server:9000/...` internally to avoid NAT-hairpin issues,
> while the browser-facing URLs stay on the public `mysso.mydomain.com`.

---

## Day-to-day

```bash
EF=.env_files/authentik.env
docker compose -f compose/authentik/docker-compose.yml --env-file $EF up -d   # start / apply
docker logs -f authentik-server                                               # logs

# Scripting Authentik objects (providers/apps) from the API shell:
docker exec authentik-server ak shell
# ak shell chokes on piped multi-line input — write a script to a file and exec() it:
#   docker exec authentik-server ak shell -c "exec(open('/tmp/script.py').read())"
```

> Back up `data/authentik/` (Postgres + file storage + certs) to preserve users,
> providers, and signing keys across a rebuild.

### Upgrading

Move **one minor train at a time** — 2026.2 → 2026.5, not 2026.2 → 2027.x — and
never downgrade. Each release runs forward-only schema migrations, so a rollback
leaves the database ahead of the binary. Bump `AUTHENTIK_TAG`, `up -d`, then
watch `docker logs -f authentik-worker` until migrations finish before you
declare it done.

Two things that look like breakage but are not:

- After a version jump, the embedded outpost can serve **404s on forward-auth
  protected apps** for the first minute while it re-registers. Wait it out
  before you start editing providers.
- File storage moved from `/media` to `/data` in 2025.12. If you are coming from
  an older copy of this file, copy the old `media/` contents into
  `authentik-data/` rather than leaving the `/media` mount in place.
