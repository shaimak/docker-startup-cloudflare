# vnstat — Bandwidth Accounting

Tracks how much data the host's WAN interface has moved (today / this month /
all-time). Useful on a metered or capped home connection. CLI only, not exposed
publicly.

- **Image:** `vergoh/vnstat:latest`, container `vnstat`
- **Networking:** `network_mode: host` — required so it can read the real host
  interface counters from `/proc/net/dev`
- **Storage:** `data/vnstat` (the database persists across restarts and reboots)

**Before you start**

- **Requires:** nothing in this stack. vnstat runs on `network_mode: host` and
  never joins `proxy_network`, so it does not need Caddy.
- **Run every command below from the repo root** (the folder holding `compose/`
  and `.env_files/`), not from this folder. All paths are relative to it.

---

## Setup

1. Find your WAN interface name on the host:

```bash
ip -o link show | awk -F': ' '{print $2}'   # e.g. enp1s0, eth0
```

2. The compose file mounts `../../data/vnstat` and runs the image. Optionally set
   `VNSTAT_TZ` in an env file (defaults to `Etc/UTC`). No secrets needed, so you
   can start it without `--env-file`:

```bash
docker compose -f compose/vnstat/docker-compose.yml up -d
```

vnstat auto-detects interfaces. It needs a few minutes of data before numbers
appear.

---

## Reading the numbers

Replace `enp1s0` with your interface:

```bash
docker exec vnstat vnstat -i enp1s0        # summary: today / this month / all-time
docker exec vnstat vnstat -i enp1s0 -d     # per-day
docker exec vnstat vnstat -i enp1s0 -m     # per-month
docker exec vnstat vnstat -i enp1s0 -l     # live rate
```
