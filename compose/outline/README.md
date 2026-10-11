# Outline — Team Wiki

A self-hosted wiki / knowledge base. Logs in through Authentik over OIDC, so it
has no separate user list.

- **Images:** `outlinewiki/outline` + `postgres:16-alpine` + `redis:alpine`
- **Containers:** `outline-app`, `outline-db`, `outline-redis`
- **Public host (example):** `mywiki.mydomain.com` → `outline-app:3000`
- **Storage:** local disk (`data/outline/storage`), no S3 needed
- **Networks:** `outline_internal_network` (db/redis) + `proxy_network` (app)

**Before you start**

- **Requires:** [caddy](../caddy/README.md) (owns `proxy_network`) and
  [authentik](../authentik/README.md), both already up.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.

---

## Setup

```bash
# 1. Env file
cp compose/outline/.env.example .env_files/outline.env
```

Fill in `.env_files/outline.env`:

- `OUTLINE_URL` = `https://mywiki.mydomain.com` (exact public URL)
- `OUTLINE_SECRET_KEY` = `openssl rand -hex 32`
- `OUTLINE_UTILS_SECRET` = `openssl rand -hex 32`
- `OUTLINE_DB_PASS` = `openssl rand -hex 24` (hex, so the DB URL never contains `/` or `+`)
- OIDC values (`OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, the three URIs) — see next
  section.

```bash
# 2. Start
docker compose -f compose/outline/docker-compose.yml \
  --env-file .env_files/outline.env up -d

# 3. Add the Caddy site block, then recreate Caddy
```

Caddyfile:

```caddy
mywiki.mydomain.com {
    encode zstd gzip
    reverse_proxy outline-app:3000
}
```

---

## OIDC provider (in Authentik)

1. In Authentik, create an **OAuth2/OpenID Provider**:
   - Redirect URI: `https://mywiki.mydomain.com/auth/oidc.callback`
   - Signing key: any certificate
2. Create an **Application** pointing at that provider. Note its slug.
3. Copy the credentials into `outline.env`:

```ini
OIDC_CLIENT_ID=<from Authentik>
OIDC_CLIENT_SECRET=<from Authentik>
OIDC_AUTH_URI=https://mysso.mydomain.com/application/o/authorize/
OIDC_TOKEN_URI=http://authentik-server:9000/application/o/token/
OIDC_USERINFO_URI=http://authentik-server:9000/application/o/userinfo/
OIDC_DISPLAY_NAME=SSO Login
```

Note the split between the three URIs, and that only the first one is `https`:

- `OIDC_AUTH_URI` is the only one the **browser** visits, so it has to be the
  public HTTPS host.
- `OIDC_TOKEN_URI` and `OIDC_USERINFO_URI` are **back-channel** calls that
  `outline-app` makes server-to-server, from inside the container. Those go
  direct to the Authentik container on `proxy_network`. Plain `http` is fine —
  it is container-to-container traffic that Caddy never sees.

**Why this matters (real failure):** if the back-channel URIs point at your
public HTTPS host, the container has to exit your network and come back in by
public IP. That only works if your router supports **NAT hairpin** (also called
NAT loopback). Plenty of routers don't. Move the box to such a network and
Outline login breaks in a very specific way: Authentik accepts your username and
password, then Outline errors out on the callback, because the token exchange it
runs behind the scenes can't reach the public URL. Nothing in the browser hints
at the cause.

Guacamole in this kit survives that same network, because its one server-side
call (`OPENID_JWKS_ENDPOINT`) is already pointed at `authentik-server:9000` for
exactly this reason. If you add another SSO app, check every URL it fetches
**server-side** and give it the internal address.

4. Recreate `outline-app`. The login page now shows the SSO button.

---

## Notes

- `FORCE_HTTPS=true` is set because Outline sits behind Caddy. Internal
  plain-HTTP calls to `outline-app:3000` get redirected — always reach Outline
  through the public HTTPS URL.
- First user to log in via OIDC becomes the admin.
- Back up `data/outline/` (Postgres + uploaded files) to preserve the wiki.

```bash
docker compose -f compose/outline/docker-compose.yml \
  --env-file .env_files/outline.env up -d      # start / apply
docker logs -f outline-app                     # logs
```
