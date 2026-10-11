# Vaultwarden — Password Vault

A self-hosted server that speaks the Bitwarden protocol, so the official
Bitwarden browser extension, desktop app and mobile apps all talk to it. Your
passwords live on your disk instead of someone else's.

- **Image:** `docker.io/vaultwarden/server:1.37.1-alpine`, container `vaultwarden-app`
- **Public host (example):** `mypass.mydomain.com` → `vaultwarden-app:80`
- **Local port:** `127.0.0.1:18791` (loopback only)
- **Storage:** `data/vaultwarden` — SQLite db, `rsa_key.pem`, attachments, sends
- **Networks:** `proxy_network` only. One container, no database service, so
  there is no internal network here.
- **Behind SSO:** no, deliberately. See below.

**Before you start**

- **Requires:** [caddy](../caddy/README.md) (owns `proxy_network`) already up.
  Authentik is *not* required, and should *not* be put in front of this one.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.

---

## Why this service is NOT behind Authentik forward-auth

Every other web app in this kit can sit behind Authentik: either it speaks OIDC
itself (Outline, Guacamole) or you wrap it in the Caddy forward-auth block from
the [Caddyfile](../caddy/Caddyfile). Vaultwarden is the exception. Do not wrap
it.

Forward-auth works by bouncing an unauthenticated request to a login page in a
browser. The Bitwarden **clients** cannot do that. The browser extension and the
mobile apps are API consumers: they POST to `/identity/connect/token` and call
`/api/...` with a bearer token, and they expect JSON back. Put forward-auth in
front and those calls get a `302` to the Authentik login page instead. The
extension has nowhere to render a login form and no way to carry the resulting
session cookie, so it just fails.

The failure is nastier than a clean outage, because the one client that *is* a
browser keeps working:

| Client | Behind forward-auth |
|---|---|
| Web vault in a browser | works — you log into Authentik, then into the vault |
| Browser extension | broken — cannot sync or unlock |
| Android / iOS app | broken — login and sync fail |
| Desktop app / CLI | broken — same reason |

So you test it in your browser, see a working vault, and only discover the
damage when your phone can no longer autofill. A password manager that only
works in one browser is not a password manager.

**What it uses instead.** Vaultwarden has real authentication of its own, and
this stack leans on it:

- `SIGNUPS_ALLOWED=false` and `INVITATIONS_ALLOWED=false` — the vault is closed.
  Nobody can create an account, so the login page is not an open door.
- The `/admin` panel is gated by `ADMIN_TOKEN` (long random value, or an Argon2
  hash of one).
- Your master password never leaves the client. Vaultwarden only ever sees an
  encrypted blob, which is the whole point of the design and a good reason not
  to bolt another layer onto it.
- Turn on TOTP two-factor in the web vault for your own account. That is the
  layer worth adding here, and unlike forward-auth every client supports it.

If you want the vault reachable only from home, restrict it at the network
layer instead of the auth layer — bind it to the LAN, or filter by source IP in
the Caddy site block. Both leave the API paths intact.

---

## Setup

```bash
# 1. Env file
cp compose/vaultwarden/.env.example .env_files/vaultwarden.env
```

Fill in `.env_files/vaultwarden.env`:

- `DOMAIN` = `https://mypass.mydomain.com` — the exact public URL, with
  `https://`. Vaultwarden derives attachment links, WebAuthn origins and the
  notifications websocket URL from it, so a wrong value breaks 2FA and file
  downloads in ways that look unrelated.
- `ADMIN_TOKEN` = `openssl rand -base64 48`. This is the only credential in the
  file. It stays in `.env_files/` and is never committed.
- `IP_HEADER=X-Forwarded-For` — leave it. Explained below.

```bash
# 2. Start
docker compose -f compose/vaultwarden/docker-compose.yml \
  --env-file .env_files/vaultwarden.env up -d

# 3. Add the Caddy site block, then recreate Caddy
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
```

Caddyfile:

```caddy
mypass.mydomain.com {
    encode zstd gzip
    reverse_proxy vaultwarden-app:80
}
```

Plain `reverse_proxy`, no `route` block, no `forward_auth`.

**First account.** With signups disabled you cannot register through the web
page. Two ways in:

- Temporarily set `SIGNUPS_ALLOWED=true`, recreate the container, register your
  account, set it back to `false`, recreate again.
- Or open `https://mypass.mydomain.com/admin`, enter the `ADMIN_TOKEN`, and
  invite yourself from there (needs `INVITATIONS_ALLOWED=true` plus SMTP, so the
  first route is usually simpler).

---

## Why `IP_HEADER=X-Forwarded-For`

Vaultwarden reads the client IP from `X-Real-IP` by default. Caddy does not send
that header; it sends `X-Forwarded-For`. So out of the box, behind this proxy,
every request appears to come from the same address — the Docker bridge gateway
(`172.18.0.1`).

Two things break:

- **Logs are useless.** Every failed login is attributed to the gateway, so you
  cannot tell a typo from someone probing your vault.
- **Rate limiting pools everyone together.** The login throttle counts failures
  per IP. With one apparent IP, unrelated clients share a bucket: a burst from
  one device throttles every other device, and a real attacker is
  indistinguishable from your own phone retrying.

Setting `IP_HEADER=X-Forwarded-For` points Vaultwarden at the header Caddy
actually sets, and both behaviours become correct. (Guacamole in this kit needs
the same fix for the same reason — see its `REMOTE_IP_VALVE_ENABLED` note.)

---

## `data/vaultwarden/rsa_key.pem` — preserve it or log everyone out

`data/vaultwarden/` holds the SQLite database (`db.sqlite3`), your attachments,
your sends, and one small file that is easy to overlook: **`rsa_key.pem`**.

That key signs the JWTs Vaultwarden issues to clients. Every logged-in
extension, phone and desktop app holds a token signed by it. If the key changes,
every existing token fails validation at once: all clients are logged out and
have to re-authenticate, and clients that were relying on a cached session get
sync errors first. Vaultwarden silently generates a fresh key on startup when it
does not find one, so losing the file does not throw an error — it just
invalidates every session.

This matters most during a **migration**. Copy the whole `data/vaultwarden/`
directory, not just the database:

```bash
# stop first, so SQLite is not mid-write
docker compose -f compose/vaultwarden/docker-compose.yml \
  --env-file .env_files/vaultwarden.env down

# copy the entire folder, permissions and all
rsync -aH data/vaultwarden/ newhost:/path/to/apps/data/vaultwarden/

# on the new host, confirm the key came across BEFORE starting
ls -l data/vaultwarden/rsa_key.pem
```

Same rule for backups: a backup that only captures `db.sqlite3` restores your
passwords but logs out every device. Back up the directory.

---

## Notes

- **Pinned version.** The image is pinned to `1.37.1-alpine` rather than
  `latest`, so a `docker compose pull` cannot move the vault to a new schema
  without you choosing to. Bump it deliberately, and back up `data/vaultwarden/`
  before you do — Vaultwarden migrates the SQLite schema on first start and there
  is no downgrade path.
- **Local port is loopback.** `127.0.0.1:18791` is reachable from the host only,
  useful for checking the container is alive without going through DNS and TLS:
  `curl -I http://localhost:18791`.
- **Do not front it with a second proxy.** If another reverse proxy already
  handles this hostname, retire that route first. Two proxies mangling
  `X-Forwarded-For` puts you back in the shared-rate-limit hole above.

```bash
docker compose -f compose/vaultwarden/docker-compose.yml \
  --env-file .env_files/vaultwarden.env up -d      # start / apply
docker logs -f vaultwarden-app                     # logs
```
