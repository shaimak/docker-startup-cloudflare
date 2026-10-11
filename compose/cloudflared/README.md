# cloudflared — Cloudflare Tunnel

This service is what lets the whole stack be reachable from the internet **with
no public IP and no port forwarding**. It is the one real difference between this
repo and the port-forwarding starter kit.

`cloudflared` opens an **outbound** connection from your machine to Cloudflare's
edge and keeps it alive. Visitor traffic comes back down that connection. Your
router never has to accept an inbound connection, so none of the usual blockers
matter:

- **CGNAT** (your ISP hands you a private `10.x` / `100.x` WAN address and shares
  one public IP across many customers). Port forwarding is impossible under
  CGNAT. A tunnel works anyway.
- **Dynamic IP** that changes whenever the line reconnects. There is nothing to
  track, so no dynamic-DNS client is needed. This repo has no `ddclient`.
- **A locked-down router** you cannot forward ports on.

## How it fits together

```
Internet --HTTPS--> Cloudflare edge --tunnel--> cloudflared --https://caddy:443--> Caddy --> services
```

One wildcard rule (`*.yourdomain.com`) sends everything to Caddy, and Caddy does
all the per-hostname routing exactly as it would with port forwarding. Cloudflare
terminates the public TLS at its edge; Caddy serves an internal self-signed
certificate on the private network (see the `local_certs` note in
`compose/caddy/Caddyfile`).

## Prerequisites

1. Your domain's DNS is hosted on **Cloudflare** (move your nameservers at your
   registrar; the registrar does not change). Turn DNSSEC off before switching
   nameservers, or resolution breaks during the cutover.
2. The apex and any GitHub-Pages-style records stay as **DNS only (grey cloud)**.
   Only the tunnel records are proxied (orange).

## Setup

### 1. Create the tunnel

Cloudflare dashboard -> **Zero Trust -> Networks -> Tunnels -> Create a tunnel**
-> connector type **Cloudflared** -> name it (for example `homeserver`).

On the "Install connector" screen, **ignore the host install commands** — you run
cloudflared as the container in this folder. Copy only the **token** (the long
string after `--token`).

Put it in `../../.env_files/cloudflared.env`:

```
CLOUDFLARE_TUNNEL_TOKEN=eyJ...
```

### 2. Add the public hostname

In the tunnel config, add a public hostname / published application route:

- Subdomain: `*`
- Domain: `yourdomain.com`
- Path: *(blank)*
- Service type: **HTTPS**
- URL: `caddy:443`
- Additional application settings -> TLS:
  - **No TLS Verify: ON**
  - **Match SNI to Host: ON**

Both TLS toggles matter. See the gotchas below.

### 3. Add the wildcard DNS record by hand

Cloudflare does **not** auto-create DNS for *wildcard* tunnel hostnames (it warns
"this domain contains a wildcard, so no DNS record will be created"). Add it
yourself in the normal dashboard -> **DNS -> Records**:

- Type: **CNAME**
- Name: `*`
- Target: `<tunnel-id>.cfargotunnel.com`
- Proxy status: **Proxied (orange)**

The tunnel id is the `t` field inside the connector token (base64-decode the
middle segment), or it is shown in the tunnel's overview URL.

### 4. Start it

```
docker compose --env-file ../../.env_files/cloudflared.env up -d
docker logs -f cloudflared     # look for "Registered tunnel connection"
```

## The two gotchas

Both live in the public hostname's TLS settings and both will leave every service
returning **502** if wrong.

1. **No TLS Verify = ON.** Caddy presents a self-signed internal certificate.
   Without this, cloudflared refuses it.

2. **Match SNI to Host = ON.** This is the one that silently breaks everything.
   Without it, cloudflared uses the SNI `caddy` (from the `caddy:443` service
   URL). Caddy has no certificate for the name `caddy`, so the TLS handshake
   fails with `remote error: tls: internal error` and every hostname 502s. With
   it on, cloudflared sends the real request hostname (for example
   `mywiki.yourdomain.com`) as SNI, Caddy matches the site and serves its cert,
   and the handshake succeeds. Routing is by the HTTP Host header regardless, so
   SNI only affects the handshake.

## Verifying

Your own machine's resolver may cache the old nameservers after the switch, so a
plain curl to a subdomain can fail with `000` even when the service is healthy.
Test against Caddy directly to confirm the internal path and certificate:

```
curl -sk --resolve mywiki.yourdomain.com:443:<caddy-container-ip> https://mywiki.yourdomain.com/
```

## Known limits

- **Cloudflare's free plan caps request bodies at 100 MB.** Large file uploads
  (wiki attachments, password-vault file sends) will fail above that.
- Cloudflare terminates TLS at its edge, so it can see your traffic in the clear.
  End-to-end-encrypted apps (like a Bitwarden-compatible vault) keep their
  contents encrypted regardless, but the edge still sees metadata.
- Tunneled traffic takes an extra hop through Cloudflare's network, which adds a
  little latency to things like remote-desktop sessions.
