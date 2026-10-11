# docker-startup-cloudflare

A self-hosted Docker Compose stack, reverse-proxied by Caddy and published to the
internet through a **Cloudflare Tunnel**, with single sign-on. Use it two ways:

1. **A starter kit** to stand up your own home server: SSO, a wiki, remote
   desktop in the browser, a password vault, and bandwidth accounting.
2. **A reference layout** to replicate an existing stack onto a new machine.

The tunnel is the point of this variant: it reaches the internet with **no
public IP and no port forwarding**, so it works behind **CGNAT** or a **dynamic
IP** where port forwarding is impossible. If your connection does give you a real
public IP and open ports, the port-forwarding sibling repo
[docker-startup-kit](https://github.com/shaimak/docker-startup-kit) uses Caddy
with Let's Encrypt directly instead.

Every service is one folder. Secrets never live in the repo. Bring services up
one at a time behind a single reverse proxy.

I run this on **Debian**, but the structure works out of the box on any Linux.

---

## Who this is for

- Anyone who wants to run their own services instead of renting SaaS.
- People who care about privacy and owning their own data.
- Homelab learners who want a real, working stack to study and extend.
- Small teams and businesses. The same layout scales to a powerful machine with
  an account for every employee (see **Enterprise-ready** below).

You do not need to be a sysadmin. You need one always-on machine, a domain, and
a willingness to copy-paste commands.

---

## Why self-host

- **You own the data.** It lives on your disk, not on someone else's server.
- **It is always within physical reach.** If something breaks you can walk over
  to the machine. If you stop paying a SaaS bill, nobody locks you out.
- **Privacy.** Your passwords, notes, and desktop sessions never leave your
  hardware.
- **It is a good exercise.** You learn networking, DNS, reverse proxies, TLS,
  containers, and SSO by running the real thing.

## Why Docker

- **Lift and shift.** The whole stack is portable. To move to a new machine you
  copy three things: this repo, your `data/` folder, and your `.env_files/`
  folder. Bring the containers up on the new box and everything is exactly where
  you left it. No reinstalling, no reconfiguring.
- **Reproducible.** Each service is pinned in a compose file. The same command
  brings up the same stack every time.
- **Isolated.** Every service and its database run in their own containers on
  their own network. One service cannot see another's internals.
- **Easy to set up.** No compiling, no dependency hell. `docker compose up -d`
  and the service is running.

## Single sign-on with Authentik

[Authentik](compose/authentik/README.md) is the identity provider for the stack.
Log in once and every connected service recognises you. No separate username and
password per app.

- One account for the wiki, remote desktop, and anything else you put behind it.
- Apps that speak OIDC (Outline, Guacamole) federate to Authentik directly.
- Apps that have no login of their own can still sit behind Authentik using
  Caddy forward-auth (template in the [Caddyfile](compose/caddy/Caddyfile)).

## Auth: OIDC vs forward auth

Two very different ways a service in this stack can end up "behind Authentik".
Which one you pick shapes what happens on every request, what the app knows
about the user, and what breaks if you get it wrong.

### OIDC (the app speaks OpenID Connect itself)

The app has real auth built in. You register it as an OIDC provider in
Authentik, paste the client ID and secret into the app's env file, and the app
grows a "Sign in with Authentik" button. Clicking it redirects the browser to
`mysso.mydomain.com`, the user logs in there, Authentik redirects back with an
authorization code, the app trades it for an ID token and access token, and
from that point on the app owns the session and knows exactly who the user is.

Examples in this repo: [Outline](compose/outline/README.md) and
[Guacamole](compose/guacamole/README.md) both federate this way.

The important part: the app has a real per-user identity. It can attribute a
wiki edit to Alice, apply a role only Bob is allowed to hold, and log Alice out
without touching anyone else. Tokens are scoped and revocable, and there is no
"just set a header" back door — the app validated the signature on the token
itself.

### Forward auth (the app has no auth; Caddy asks Authentik on every request)

The app is oblivious. Caddy sits in front and, on every incoming request, calls
Authentik's embedded outpost at `/outpost.goauthentik.io/auth/caddy` to ask
"should this request go through?". Authentik answers based on the browser's
session cookie:

- **No valid session** → outpost returns 401. Caddy translates that into a 302
  bouncing the browser to `mysso.mydomain.com` to log in. On success the
  browser lands back on the original URL, now with a valid session, and the
  next request passes.
- **Valid session** → outpost returns 200, and Caddy adds a batch of
  `X-authentik-*` headers (username, email, groups) to the upstream request
  before proxying it to the app. The app just sees a request that arrived with
  a name attached and trusts it.

The template block is in the [Caddyfile](compose/caddy/Caddyfile).

### When to pick which

Prefer OIDC when the app supports it. Real identity inside the app, scoped
tokens, no header trust, cleaner logout. This is the default for anything in
this stack that offers it.

Use forward auth when the app has no login of its own and is only ever reached
in a browser. It is the way to bolt an auth wall onto a static site, a
dashboard with no user model, or a legacy tool with hardcoded credentials you
would rather nobody see.

### Safety, and the sharp edge on forward auth

OIDC is the safer of the two when you have the choice. Tokens are scoped and
can be revoked centrally, and the app never trusts a plain HTTP header for
identity — a spoofed header cannot forge a signed JWT.

Forward auth is fine, but the wall only holds if every request actually goes
through Caddy. It is easy to accidentally leak a bypass in the compose file:

```yaml
# leaky: the container port is published on the host, so
# http://<host-ip>:12345 reaches the app WITHOUT going through Caddy,
# and therefore without going through Authentik.
services:
  myapp:
    ports:
      - "12345:3000"
```

Anyone on the LAN or on a shared Docker network can hit that port directly and
the auth check never runs. Two clean fixes:

```yaml
# option A: bind the published port to loopback only. Useful for local curl,
# still not reachable from the LAN.
ports:
  - "127.0.0.1:12345:3000"

# option B (preferred): drop the ports block entirely. Caddy reaches the
# container by name over proxy_network. Nothing is published on the host.
# ports: []
```

The app is still on `proxy_network`, Caddy still routes to it as
`reverse_proxy myapp:3000`, and there is now no way to reach it without going
through the auth wall.

### The Vaultwarden trap

Do not put forward auth in front of Vaultwarden, and by extension do not put
it in front of any service that has non-browser clients — mobile apps, browser
extensions, desktop clients, webhooks, RSS readers, anything that talks to
`/api` directly. Those clients cannot follow a 302 to a login page and cannot
carry a session cookie back. The web UI keeps working, so the outage looks
fine in a browser while every mobile and extension client silently fails.

[compose/vaultwarden/README.md](compose/vaultwarden/README.md) has the long
version and the table of which clients break.

### Setting up a new forward-auth app in Authentik

Three steps, in this order. Miss step 3 and the outpost returns errors instead
of redirecting to the login page.

1. **Proxy Provider.** *Applications → Providers → Create → Proxy Provider*.
   Set the authorization flow, then choose mode **"Forward auth (single
   application)"** and set **External host** to the full public URL of the
   app, e.g. `https://myapp.mydomain.com`. The external host must match
   exactly, scheme included — a mismatch here is why the login loop breaks.
2. **Application.** *Applications → Applications → Create*. Give it a name
   and slug, and set the provider to the one you just made.
3. **Add to the embedded outpost.** *Applications → Outposts → edit
   `authentik Embedded Outpost` → add the new application to it*. The
   embedded outpost is what Caddy actually calls at
   `/outpost.goauthentik.io/auth/caddy`; if the application is not listed on
   it, the outpost has no rules for that host and every request fails.

Then add the forward-auth block for the new host to the
[Caddyfile](compose/caddy/Caddyfile) and force-recreate Caddy.

## Enterprise-ready

This is the same pattern a company would use. Run it on a powerful machine,
create an Authentik account for each employee, and every internal tool is behind
one login. Add or remove a person in one place and their access to every service
follows. Data stays on hardware you control.

---

## What's in the box

| Service | What it does | Public host (example) | Behind SSO |
|---|---|---|---|
| [caddy](compose/caddy/README.md) | Reverse proxy + automatic HTTPS. Owns the shared network. | (routes all the below) | n/a |
| [authentik](compose/authentik/README.md) | Identity provider / single sign-on. | `mysso.mydomain.com` | is the IdP |
| [outline](compose/outline/README.md) | Team wiki / knowledge base. | `mywiki.mydomain.com` | via OIDC |
| [guacamole](compose/guacamole/README.md) | Clientless remote desktop (RDP/VNC/SSH in the browser). | `myremote.mydomain.com` | via OIDC |
| [vaultwarden](compose/vaultwarden/README.md) | Bitwarden-compatible password vault. | `mypass.mydomain.com` | no — [on purpose](compose/vaultwarden/README.md) |
| [cloudflared](compose/cloudflared/README.md) | Cloudflare Tunnel connector. Exposes the stack with no public IP and no port forwarding. | n/a | n/a |
| [vnstat](compose/vnstat/README.md) | Bandwidth accounting for the host WAN interface. | n/a (CLI) | n/a |

---

## Requirements

- **One machine, ideally on 24/7.** Anything from a spare laptop to a mini PC to
  a server works. It stays on so your services stay reachable. I run Debian; any
  Linux is fine.
- **Docker Engine + Docker Compose v2** (`docker compose`, not the old
  `docker-compose`).
- **A domain**, with DNS hosted on **Cloudflare**. You can register anywhere
  (Porkbun, Namecheap, Cloudflare itself) and then point the domain's
  nameservers at Cloudflare. The registrar does not change; only the
  nameservers move. DNS is free on Cloudflare.
- **A free Cloudflare account**, for the tunnel (Zero Trust is free for personal
  use).
- **No port forwarding and no public IP required.** This is the whole point of
  the Cloudflare Tunnel variant. `cloudflared` makes an outbound connection, so
  it works behind **CGNAT** and a **dynamic IP**, where port forwarding is
  impossible. There is no `ddclient` and no router configuration.

---

## Folder structure

Everything lives under one parent folder. I use `~/apps/`. You can rename it or
move the whole set anywhere.

```
apps/
├── compose/<service>/
│   ├── docker-compose.yml   # the stack. Header comment has the local + public URL + start command.
│   ├── .env.example         # committed template. No real secrets.
│   ├── README.md            # per-service setup + gotchas.
│   └── Caddyfile            # caddy only: the reverse-proxy routing table.
│
├── data/<service>/...       # all persistent data (databases, files). Bind-mounted as ../../data/...
│
└── .env_files/<service>.env # your REAL secrets, one file per service.
```

**About the env files.** Normally Docker reads a `.env` file sitting next to the
compose file. I do it differently: I keep all the real env files together in one
`.env_files/` folder and point each service at its file with `--env-file`. Two
reasons:

- All my secrets are in one place, easy to back up and easy to keep private.
- The per-service folders stay clean and safe to share. `.env_files/` is
  git-ignored, so secrets never land in the repo. That is also why this folder
  is **not** included here: it holds my actual credentials.

**What you do when cloning this repo:** create your own `.env_files/` folder,
copy each service's `.env.example` (or `.env.sample`) into it as
`<service>.env`, and fill in your own values.

```bash
mkdir -p .env_files
cp compose/outline/.env.example .env_files/outline.env
# then edit .env_files/outline.env with your own secrets
```

If you prefer the conventional layout, you can instead drop a `.env` next to each
compose file and leave off `--env-file`. Both work.

`data/` and `.env_files/` are both in [.gitignore](.gitignore). They live on the
host and are never committed.

---

## Generating secrets

Several services need a strong password or secret key. Generate them with
`openssl` in the terminal:

```bash
openssl rand -hex 32     # 64-char secret. Use for app secret keys.
openssl rand -hex 24     # 48-char secret. Good for database passwords.
```

- `openssl rand -hex N` outputs N bytes as hex (characters `0-9` and `a-f`), so
  the result is `2 x N` characters long.
- **Use hex, not base64, for database passwords.** base64 can include `/` and
  `+`, which some databases and connection strings mishandle. Hex is always safe.

Each service README lists exactly which values to generate.

---

## Setup order

Bring the pieces up in this order. Each service's own README has the detailed
steps (env values, first-run, creating your account, wiring SSO).

1. **[caddy](compose/caddy/README.md)**: start it first. It creates the shared
   `proxy_network` that every other service joins.
2. **[authentik](compose/authentik/README.md)**: start it next, then open its
   URL and create your admin account. This is your single sign-on. Every other
   service authenticates against it.
3. **The services you want**: [outline](compose/outline/README.md),
   [guacamole](compose/guacamole/README.md),
   [vaultwarden](compose/vaultwarden/README.md), and so on. For the SSO ones you
   create an OIDC provider in Authentik (the per-service README shows how), paste
   the credentials into that service's env file, and start it.
4. **[cloudflared](compose/cloudflared/README.md)**: the Cloudflare Tunnel that
   publishes the stack. Set it up once (create the tunnel, point a wildcard
   hostname at Caddy) and every service above becomes reachable from the
   internet. No ports, no public IP.
5. **Supporting pieces**: [vnstat](compose/vnstat/README.md) for bandwidth, any
   time.

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/shaimak/docker-startup-cloudflare.git
cd docker-startup-cloudflare

# 2. Create the host-side folders the compose files expect
mkdir -p .env_files data

# 3. Bring up Caddy FIRST. It creates the shared proxy_network everything joins.
docker compose -f compose/caddy/docker-compose.yml up -d

# 4. For each service: copy the env template, fill in secrets (see "Generating
#    secrets"), add a Caddy site block, then start it.
cp compose/authentik/.env.example .env_files/authentik.env
#    ...edit .env_files/authentik.env ...
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env pull
docker compose -f compose/authentik/docker-compose.yml \
  --env-file .env_files/authentik.env up -d

# 5. Add the site block to compose/caddy/Caddyfile, then RECREATE Caddy
#    (a plain reload misses the change, see "Things to keep in mind").
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
```

---

## How the networking fits together

- **Caddy owns one external network, `proxy_network`.** Its compose file creates
  it. Every service that needs a public URL joins `proxy_network` so Caddy can
  reach the container **by name** (e.g. `reverse_proxy outline-app:3000`).
- Multi-container services (anything with its own Postgres/Redis) also get a
  private `<service>_internal_network` so the database is not exposed to the
  proxy or to other services.
- Bring Caddy up before anything else, since it owns `proxy_network`. Everything
  else marks that network `external: true`. `cloudflared` joins `proxy_network`
  too, so it can reach `caddy` by name.
- **TLS is split in two.** Cloudflare terminates the public HTTPS at its edge
  (free certificates, managed for you). On the private side, Caddy serves an
  internal self-signed certificate to `cloudflared` (`local_certs` in the
  Caddyfile). There is no Let's Encrypt and no inbound port 443, because nothing
  connects to this box from the internet directly. See
  [compose/cloudflared/README.md](compose/cloudflared/README.md).

---

## Conventions

- **Run every command from the repo root** — the folder holding `compose/`,
  `data/`, and `.env_files/`. Every path in every README is relative to it
  (`-f compose/<svc>/docker-compose.yml`, `--env-file .env_files/<svc>.env`).
  Running from inside a service folder fails: the `-f` path won't resolve, and
  the `../../data/...` bind mounts in the compose files point at the repo root.
- **Two paths on every start command, and they are different things.** `-f`
  picks the compose file; `--env-file` supplies the secrets. Omit `--env-file`
  and the service starts with blank values, or refuses to start where the
  compose file marks a variable required (`${VAR:?...}`).
- **Secrets live only in `.env_files/`.** Never in a compose folder, never in
  `data/`, never committed.
- **`data/` holds live persistent volumes.** Back it up. Do not wipe it.
- **Single sign-on is optional per service.** Outline and Guacamole federate to
  Authentik over OIDC. Most other apps can sit behind Authentik forward-auth
  using the template block in the [Caddyfile](compose/caddy/Caddyfile).
- **Forward-auth only suits apps you reach in a browser.** Anything with a native
  or mobile client that calls an API directly cannot follow the login redirect,
  so wrapping it breaks those clients while the web UI keeps working — which
  makes it easy to miss. [vaultwarden](compose/vaultwarden/README.md) is the
  example in this kit, and is left on its own auth for exactly that reason.

---

## Things to keep in mind

### Reloading Caddy does not pick up a Caddyfile edit

Editing the Caddyfile and running `caddy reload` reports *"config is unchanged"*
and keeps serving the old config. The Caddyfile is a single-file bind mount, and
most editors save by atomic-rename (new inode), so the running container keeps
the original. **Force-recreate instead:**

```bash
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy
# verify the container actually sees the change:
docker exec caddy_reverse_proxy grep <newdomain> /etc/caddy/Caddyfile
```

### If you keep `data/` on an extra drive, `nofail` is not optional

Plenty of home servers put the big volumes on a second disk or an external USB
drive and bind-mount it in. The moment that drive is in `/etc/fstab`, it becomes
part of boot: systemd waits for it, and a mount that fails takes
`local-fs.target` down with it. The machine then drops to an **emergency shell**
instead of finishing boot. No Docker, no Caddy, no SSH — every service is
offline, and the only way in is a keyboard and monitor physically attached to the
box. A drive that is unplugged, renamed, slow to spin up, or replaced is enough
to trigger it.

Add `nofail` and a bounded timeout to every non-root mount:

```fstab
# /etc/fstab — nofail: boot even if the drive is missing
#              x-systemd.device-timeout: stop waiting after Ns, do not hang
UUID=xxxx-xxxx  /mnt/storage  ext4  defaults,nofail,x-systemd.device-timeout=10s  0  2
```

Test it before you rely on it: `sudo systemctl daemon-reload && sudo mount -a`,
then reboot **once** with the drive physically disconnected and confirm the host
still comes up and answers SSH. Containers whose bind mounts are missing will
fail to start, which is the outcome you want — a couple of dead services beats an
unreachable machine.

---

## License

[MIT](LICENSE).
