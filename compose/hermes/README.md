# Hermes Agent

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is Nous Research's
self-improving AI agent. It works with any OpenAI-compatible LLM endpoint; this
setup uses free **Gemini** (primary) and **Groq** (fallback) keys, so it costs
nothing to run.

- Caddy URL: `https://myhermes.mydomain.com`
- Local direct: `http://localhost:9119` (dashboard)
- Start:
  ```bash
  docker compose -f compose/hermes/docker-compose.yml \
    --env-file .env_files/hermes.env up -d
  ```

## What runs

Two containers off the prebuilt `nousresearch/hermes-agent` image, sharing one
`data/hermes/data` volume (`/opt/data`: config, memory, skills, sessions):

- **gateway** (`hermes`) — the always-on agent runtime. Messaging platforms
  (Telegram, Discord, ...) stay off until you configure them.
- **dashboard** (`hermes-dashboard`) — the web UI on port 9119, reverse-proxied
  by Caddy.

The dashboard binds `0.0.0.0` so Caddy reaches it by name, so it **fails closed
unless an auth provider is set**. This kit uses the built-in Basic Auth
provider (single user, no sign-up) via `HERMES_DASHBOARD_BASIC_AUTH_*`.

## Model / backend

Non-secret model config lives in `data/hermes/data/config.yaml` and reads keys
from the env via `${VAR}` substitution. A working two-provider chain:

```yaml
model:
  provider: custom
  base_url: https://generativelanguage.googleapis.com/v1beta/openai/
  api_key: ${GEMINI_API_KEY}
  default: gemini-2.5-flash
  max_tokens: 8192            # Groq's free tier caps output; keep this modest
fallback_providers:
  - provider: custom
    model: llama-3.3-70b-versatile
    base_url: https://api.groq.com/openai/v1
    api_key: ${GROQ_API_KEY}
```

Gemini is primary because Groq's free tier returns HTTP 413 on the full agent
payload (large system prompt + bundled skills); Gemini's large context handles
it and Groq stays as a fast fallback. After editing config.yaml run
`docker exec hermes hermes config migrate` (stamps the schema version) and
restart.

## Login

Single user, no registration. Credentials come from the env vars, not sign-up.
The login form POSTs to `/auth/password-login` with `{provider:"basic",...}`.

## Setting / changing the dashboard password (hash, no plaintext at rest)

1. Compute the scrypt hash. The password is passed via an env var (so shell
   metacharacters aren't mangled) and piped through `sed` to double every `$` —
   **required**, because docker compose interpolates `$` in env-file values and
   scrypt hashes use `$` as separators:

   ```bash
   docker exec -e PW='YOUR_PASSWORD_HERE' -w /opt/hermes hermes-dashboard \
     .venv/bin/python -c "import os; from plugins.dashboard_auth.basic import hash_password; print(hash_password(os.environ['PW']))" \
     | sed 's/\$/$$/g'
   ```

2. Paste the (already `$$`-escaped) result into `.env_files/hermes.env`:

   ```
   HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
   HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH=scrypt$$16384$$8$$1$$...$$...
   #HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=   # leave removed — plaintext WINS over the hash if set
   HERMES_DASHBOARD_BASIC_AUTH_SECRET=<openssl rand -hex 32>
   ```

3. Apply. Use `up -d`, **not** `restart` — `docker compose restart` reuses the
   old container and won't re-read the env file:

   ```bash
   docker compose -f compose/hermes/docker-compose.yml \
     --env-file .env_files/hermes.env up -d dashboard
   ```

Verify the hash arrived intact (single `$`, not doubled) inside the container:

```bash
docker exec hermes-dashboard sh -c 'echo "$HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH"'
```

## Reaching it from the host itself

If your router has no NAT loopback, the host can't reach its own public IP. Pin
the name to loopback so the host talks to Caddy directly:

```bash
echo '127.0.0.1  myhermes.mydomain.com' | sudo tee -a /etc/hosts
```
