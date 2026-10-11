# Caddy — Reverse Proxy

The single entry point for the stack. Routes each public hostname to a container,
and for some apps enforces Authentik forward-auth. **Owns** the shared
`proxy_network` that every published service joins.

In this Cloudflare Tunnel variant, Caddy does **not** face the internet and does
**not** use Let's Encrypt. Cloudflare terminates the public HTTPS at its edge;
Caddy serves an internal self-signed certificate to `cloudflared` on the private
network (`local_certs` in the Caddyfile). It therefore publishes no host ports —
`cloudflared` reaches it as `caddy:443` over `proxy_network`.

- **Image:** `caddy:latest`, container `caddy_reverse_proxy`
- **Ports:** none published (reached internally at `caddy:443` by cloudflared)
- **Config:** `compose/caddy/Caddyfile` (single-file bind mount — see gotcha)
- **State/certs:** `data/caddy/data` (persistent — holds the internal CA; don't wipe)

**Before you start**

- **Requires:** nothing. Caddy comes up **first** — its compose file creates
  `proxy_network`, which every other service marks `external: true` and joins.
  Starting another service before Caddy fails with a missing-network error.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.
- Caddy needs **no env file**; its config is the `Caddyfile` in this folder.

---

## Start

```bash
docker compose -f compose/caddy/docker-compose.yml up -d
```

---

## A site block

A plain proxied site is three lines:

```caddy
mywiki.mydomain.com {
    encode zstd gzip
    reverse_proxy outline-app:3000
}
```

- The upstream is the target **container name** + its internal port. The
  container must be on `proxy_network`.
- Public HTTPS is handled by Cloudflare at its edge. Caddy serves an internal
  cert to cloudflared, so there is no per-hostname cert to issue here.

A site behind Authentik SSO wraps the upstream in a `route { ... forward_auth ... }`
block. See the commented template at the bottom of the
[Caddyfile](Caddyfile).

---

## ⚠️ Editing the Caddyfile = **recreate**, not reload

The Caddyfile is a **single-file** bind mount, so Docker pins the original inode.
Most editors save by atomic-rename (new inode), so the running container keeps
serving the **old** file and `caddy reload` reports *"config is unchanged."*

```bash
# After ANY Caddyfile edit:
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy

# Verify the container actually sees the change:
docker exec caddy_reverse_proxy grep <newdomain> /etc/caddy/Caddyfile
```

A force-recreate is a ~2-3s blip across **all** sites.

---

## Adding a new public site

With the wildcard tunnel hostname (`*.yourdomain.com` -> `caddy`), a new
subdomain needs **no DNS change** — it already resolves through the tunnel. Just:

1. Add a site block to the Caddyfile.
2. Force-recreate Caddy (above).

That is the whole point of routing everything through one wildcard: Caddy matches
the new hostname by its Host header and the tunnel already carries it. (A brand
new *apex* or a second domain still needs its own Cloudflare zone and wildcard —
see [cloudflared](../cloudflared/README.md).)

```bash
docker logs -f caddy_reverse_proxy | grep -i '<domain>'
```

---

## Day-to-day

```bash
docker logs -f caddy_reverse_proxy                                              # access / TLS logs
docker exec caddy_reverse_proxy caddy validate --config /etc/caddy/Caddyfile    # syntax check
docker compose -f compose/caddy/docker-compose.yml up -d --force-recreate caddy # apply edits
```

> With `local_certs`, Caddy runs its own internal CA under
> `data/caddy/data/caddy/...`. The public certificates are Cloudflare's, managed
> at the edge, so there is nothing to renew here. Backing up `data/caddy/` still
> keeps the internal CA stable across a rebuild.
