# Guacamole — Remote Desktop in the Browser

Apache Guacamole, a clientless remote-desktop gateway (RDP/VNC/SSH in the
browser), fronted by Authentik SSO. Used here to reach the host's own desktop
over RDP, but it can target any machine you can route to.

- **Public hosts (example):** `myremote.mydomain.com` (primary) + `myotherdomain.com` (optional anonymous front-end)
- **Auths via:** Authentik OIDC (SSO-first); the built-in `guacadmin` stays as a local break-glass admin
- **Reaches:** in this setup, the host's own desktop (GNOME Remote Desktop / RDP) at `host.docker.internal:3389`

Three containers: `guacamole-app` (Tomcat UI + auth), `guacamole-guacd` (the
proxy daemon that speaks RDP/VNC/SSH), `guacamole-db` (Postgres). App is on
`proxy_network` + `guacamole_internal_network`; the others stay internal.

**Before you start**

- **Requires:** [caddy](../caddy/README.md) (owns `proxy_network`) and
  [authentik](../authentik/README.md), both already up.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.

---

## Architecture

```
Browser ──HTTPS──> Caddy ──> guacamole-app (Tomcat UI + auth)
                                 │  ├── guacamole-db (Postgres: users, connections, history)
                                 │  └── guacd (proxy daemon; speaks RDP/VNC/SSH)
                                 │         └──RDP──> host.docker.internal:3389 (a desktop)
                                 └── OIDC ──> Authentik (mysso.mydomain.com)
```

---

## Directory layout

```
compose/guacamole/
    docker-compose.yml      # the stack
    .env.sample             # committed template (no secrets)
    README.md               # this file
.env_files/guacamole.env    # YOUR real secrets — never commit
data/guacamole/
    postgres/               # DB data (persistent)
    init/initdb.sql         # schema, auto-loaded on first DB boot only
    drive/                  # (optional) shared-drive redirection
    record/                 # (optional) session recordings
```

---

## Install from scratch

```bash
# 1. Data dirs
mkdir -p data/guacamole/{postgres,init,drive,record}

# 2. Generate the Postgres schema (only consumed on first DB boot)
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql \
  > data/guacamole/init/initdb.sql

# 3. Env file
cp compose/guacamole/.env.sample .env_files/guacamole.env
#    Generate PG_PASS with: openssl rand -hex 24
#    Fill the OPENID_* values after you create the Authentik provider (below).

# 4. Add the Caddy site block (below) and recreate Caddy

# 5. Start
docker compose -f compose/guacamole/docker-compose.yml \
  --env-file .env_files/guacamole.env up -d
```

First login is `guacadmin` / `guacadmin`. **Change it immediately** (top-right →
Settings → Preferences).

Caddyfile:

```caddy
myremote.mydomain.com {
    encode zstd gzip
    reverse_proxy guacamole-app:8080
}
```

---

## `.env_files/guacamole.env`

```ini
PG_USER=guacamole
PG_DB=guacamole
PG_PASS=<openssl rand -hex 24>

# --- Authentik SSO (OpenID Connect) ---
OPENID_AUTHORIZATION_ENDPOINT=https://mysso.mydomain.com/application/o/authorize/
# JWKS is fetched server-side -> internal address avoids NAT hairpin
OPENID_JWKS_ENDPOINT=http://authentik-server:9000/application/o/guacamole/jwks/
OPENID_ISSUER=https://mysso.mydomain.com/application/o/guacamole/
OPENID_CLIENT_ID=<from Authentik>
OPENID_REDIRECT_URI=https://myremote.mydomain.com/
OPENID_USERNAME_CLAIM_TYPE=preferred_username   # = the Authentik username field
OPENID_SCOPE=openid email profile
EXTENSION_PRIORITY=openid          # redirect straight to Authentik (SSO-first)

# --- Behind Caddy: trust X-Forwarded-For so the ban tracker sees real client IPs ---
REMOTE_IP_VALVE_ENABLED=true

# --- Relax the brute-force ban (defaults 5/300s self-lock on the OIDC nonce resubmit) ---
BAN_MAX_INVALID_ATTEMPTS=20
BAN_ADDRESS_DURATION=30
```

Guacamole uses the OIDC **implicit** flow, so there is no client secret — the
provider is a **public** client.

---

## Authentik OIDC provider

Create a **public** OAuth2/OpenID provider (implicit flow, RS256 signing key)
plus an Application with slug `guacamole`. `ak shell` is interactive and chokes
on piped multi-line blocks, so write the script to a file and `exec()` it:

```python
# ak_create_guac.py  ->  docker cp into authentik-server, then:
#   docker exec authentik-server ak shell -c "exec(open('/tmp/ak_create_guac.py').read())"
from authentik.flows.models import Flow
from authentik.providers.oauth2.models import OAuth2Provider, ScopeMapping, RedirectURI, RedirectURIMatchingMode
from authentik.crypto.models import CertificateKeyPair
from authentik.core.models import Application

auth_flow  = Flow.objects.get(slug="default-provider-authorization-implicit-consent")
inval_flow = Flow.objects.get(slug="default-provider-invalidation-flow")
cert       = CertificateKeyPair.objects.get(name="authentik Self-signed Certificate")
scopes     = list(ScopeMapping.objects.filter(scope_name__in=["openid","email","profile"]))

p = OAuth2Provider.objects.create(
    name="Guacamole", client_type="public",
    authorization_flow=auth_flow, invalidation_flow=inval_flow,
    signing_key=cert, include_claims_in_id_token=True,
)
p.redirect_uris = [
    RedirectURI(RedirectURIMatchingMode.STRICT, "https://myremote.mydomain.com/"),
]
p.save(); p.property_mappings.set(scopes); p.save()
Application.objects.create(name="Guacamole", slug="guacamole", provider=p)
print("client_id =", p.client_id)   # -> goes into OPENID_CLIENT_ID
```

Put the printed `client_id` into `OPENID_CLIENT_ID` and recreate `guacamole-app`.

---

## Seed a connection + SSO admin (psql)

SSO users are just-in-time: Authentik authenticates them, but they land with
**zero** permissions. Pre-create your user as an admin, and add a connection to
whatever machine you want to reach.

```sql
-- An RDP connection to the host's own desktop
INSERT INTO guacamole_connection (connection_name, protocol) VALUES ('My Desktop','rdp');
-- parameters (connection_id from the row above):
--   hostname=host.docker.internal  port=3389  username=<host-user>  password=<RDP pw>
--   security=nla  ignore-cert=true  width=1920  height=1080  dpi=96
--   color-depth=16
--   disable-paste=true   (block clipboard upload into remote — see quirk 6)
--   (NO resize-method — see quirk 2)

-- SSO admin: create a USER entity matching your Authentik username, a
-- guacamole_user row (random password; auth is via SSO), then grant ADMINISTER
-- (+ CREATE_* perms) and READ on the connection. Leave guacadmin intact.
```

> If you target GNOME Remote Desktop, the RDP credential is a separate keyring
> credential, not the login password. On the host user's session:
> `grdctl --show-credentials status` (via `XDG_RUNTIME_DIR` + `DBUS_SESSION_BUS_ADDRESS`).

---

## Day-to-day

```bash
EF=.env_files/guacamole.env
C="docker compose -f compose/guacamole/docker-compose.yml --env-file $EF"

$C up -d                       # start / apply config changes
$C restart guacamole           # restart app only
$C down                        # stop (keeps data)
docker logs -f guacamole-app   # app / auth logs
docker logs -f guacamole-guacd # RDP connection logs
```

> Connection-parameter changes apply on the **next connect** — no restart
> needed. Changing `OPENID_*` / `BAN_*` env vars **does** need `up -d` (recreate).

---

## Quirks & gotchas (the ones that actually bite)

These were hit against a GNOME Remote Desktop target on GNOME 43. If you target a
different machine, the RDP-specific ones may not apply, but the SSO/ban ones do.

1. **GNOME Remote Desktop requires NLA.** Plain TLS gets
   `HYBRID_REQUIRED_BY_SERVER`. Use `security=nla` + `ignore-cert=true`.

2. **`resize-method=display-update` breaks the connection.** GNOME Remote Desktop
   mishandles dynamic monitor resize and forcibly disconnects right after keymap
   load (NLA succeeds first, so it looks like auth success then drop). **Fix: use
   a fixed resolution and remove `resize-method`.** Guacamole scales the fixed
   desktop to the browser instead of live resize-to-window.

3. **Connection works, then dies after idle.** GNOME Remote Desktop shares the
   *active* session; when it locks/blanks on idle there is nothing to share, so
   RDP is refused until it is unlocked at the console. It also refuses to serve a
   **locked** session over RDP. If you rely on RDP, disable the lock on the host
   user (screen relies on SSO + the RDP credential for security instead):
   ```bash
   export XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
   gsettings set org.gnome.desktop.screensaver lock-enabled false
   gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
   gsettings set org.gnome.desktop.session idle-delay 0
   ```
   If you want the lock back **and** unlock-via-browser, use VNC instead of RDP
   (x11vnc mirroring the session) — VNC streams the lock screen and passes
   keystrokes.

4. **SSO-first needs `EXTENSION_PRIORITY=openid`.** Otherwise the `jdbc`
   (database) extension sorts before `openid` by filename and Guacamole shows the
   local username/password form instead of redirecting to Authentik.

5. **Behind a proxy, set `REMOTE_IP_VALVE_ENABLED=true`.** Without it every
   request looks like it comes from Caddy's container IP, so the brute-force ban
   pools *all* failures onto one address and locks everyone out at once.

6. **The brute-force ban can self-lock on logout.** The OIDC implicit flow
   benignly re-submits the id_token once per logout ("invalid/old nonce"), which
   counts as a failed login; the defaults (5 attempts / 300s) ban quickly.
   Relaxed here to `BAN_MAX_INVALID_ATTEMPTS=20` / `BAN_ADDRESS_DURATION=30`. The
   ban extension is always-on (tune only, can't disable). Recreating
   `guacamole-app` clears the in-memory ban instantly. Connection/tunnel errors
   do **not** count toward this login ban.

7. **Logout doesn't return to the Authentik login.** Guacamole's openid extension
   has no RP-initiated logout, and Authentik silently re-issues a token while its
   own session cookie is alive. True logout = log out of Authentik itself, or
   shorten the Authentik session duration.

8. **SSO accounts are just-in-time, not imported.** Authentik users are
   recognized by username on login but start with zero permissions. Pre-create
   your admin (see the psql section). `OPENID_USERNAME_CLAIM_TYPE=preferred_username`
   maps to the Authentik *username*, which may differ from the login email.

9. **JWKS over the internal network.** `OPENID_JWKS_ENDPOINT` points at
   `authentik-server:9000` (server-to-server) to avoid NAT-hairpin issues; the
   browser-facing authorize URL stays the public `mysso.mydomain.com`. Token `iss`
   still matches `OPENID_ISSUER` (the public URL).

10. **The Guacamole on-screen menu and dialogs look tiny — that is fine.** They
    are drawn by the browser as an HTML overlay, so their size follows the client
    browser zoom / DPI, not the RDP resolution.

---

## Optional: anonymous front-end

You can serve the same app at a second, neutral domain so the URL you share has
no descriptive name in it. It is front-end only; auth still runs through
Authentik.

- Add a second Caddy site block (`myotherdomain.com → guacamole-app:8080`).
- Set `OPENID_REDIRECT_URI` to the neutral domain and whitelist **both** redirect
  URIs on the Authentik provider. Guacamole has a single redirect URI, so logins
  always land on the neutral domain.
- Give the neutral domain its **own** Cloudflare zone and wildcard tunnel
  hostname pointing back to this Caddy (not a CNAME to your main domain, which
  would leak the name). See [cloudflared](../cloudflared/README.md).

---

## Break-glass (Authentik down → use local `guacadmin`)

Comment out `EXTENSION_PRIORITY` and recreate, so the local login form returns:

```bash
sed -i 's/^EXTENSION_PRIORITY=/#EXTENSION_PRIORITY=/' .env_files/guacamole.env
docker compose -f compose/guacamole/docker-compose.yml \
  --env-file .env_files/guacamole.env up -d
# ...log in as guacadmin, then revert the line and `up -d` again.
```

---

## Backup

The only stateful piece is the database:

```bash
PGPW=$(grep '^PG_PASS' .env_files/guacamole.env | cut -d= -f2)
docker exec -e PGPASSWORD="$PGPW" guacamole-db pg_dump -U guacamole guacamole > guacamole-backup.sql
```

`data/guacamole/init/initdb.sql` only runs on a *fresh* DB (empty data dir); it
is ignored once the DB exists.
