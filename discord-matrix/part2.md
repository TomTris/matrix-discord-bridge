# Part 2: Synapse

## Goal

Synapse running on a public HTTPS endpoint, responding to Matrix Client-Server API requests, with the Discord bridge registered as an Application Service.

## What this involves

Synapse ships as a Docker image (`matrixdotorg/synapse`). We wrap it in a custom Dockerfile that injects configuration via environment variables at startup (using `envsubst`). The server name becomes the domain component of all Matrix IDs (`@user:domain`) and is immutable after first use — the event DAG cryptographically signs events with this identity.

Synapse also needs a registration file for the Discord bridge (mautrix-discord). This file tells Synapse: "there's an Application Service at this URL, authenticated with these tokens, that manages users matching these patterns." Both Synapse and mautrix-discord must share the same `as_token` and `hs_token` values.

## Steps

### 2a: Generate shared secrets

On your Mac terminal, generate three random strings:

```bash
openssl rand -hex 32    # → use as MACAROON_SECRET_KEY
openssl rand -hex 32    # → use as FORM_SECRET
openssl rand -hex 32    # → use as AS_TOKEN
openssl rand -hex 32    # → use as HS_TOKEN
openssl rand -hex 32    # → use as REGISTRATION_SHARED_SECRET

```

Save all four. `AS_TOKEN` and `HS_TOKEN` are shared between Synapse and mautrix-discord. `MACAROON_SECRET_KEY` and `FORM_SECRET` are Synapse-internal secrets for token signing and form CSRF protection.

### 2b: Create a GitHub repo for Synapse

Create a repo `synapse-config` with four files:

**`homeserver.yaml.template`:**
```yaml
## Server ##
server_name: "${SYNAPSE_SERVER_NAME}"
pid_file: /data/homeserver.pid
web_client_location: "https://${ELEMENT_DOMAIN}"
public_baseurl: "https://${SYNAPSE_SERVER_NAME}"
suppress_key_server_warning: true

## Listeners ##
listeners:
  - bind_addresses:
    - 0.0.0.0
    port: 8008
    resources:
    - compress: false
      names:
      - client
      - federation
    tls: false
    type: http
    x_forwarded: true

## Database ##
database:
  name: psycopg2
  allow_unsafe_locale: true
  args:
    host: "${POSTGRES_HOST}"
    port: ${POSTGRES_PORT}
    database: "${POSTGRES_DB}"
    user: "${POSTGRES_USER}"
    password: "${POSTGRES_PASSWORD}"
    cp_min: 5
    cp_max: 10

## Logging ##
log_config: "/data/log.config"

## Media ##
media_store_path: /data/media_store
max_upload_size: 50M

## Registration ##
enable_registration: ${ENABLE_REGISTRATION}
enable_registration_without_verification: ${ENABLE_REGISTRATION}

## Signing Keys ##
signing_key_path: "/data/signing.key"
trusted_key_servers:
  - server_name: "matrix.org"

## Rate Limiting ##
rc_login:
  account:
    per_second: 1
    burst_count: 20
  failed_attempts:
    per_second: 1
    burst_count: 20

## Application Services ##
app_service_config_files:
  - /data/discord-registration.yaml

## Misc ##
report_stats: false
macaroon_secret_key: "${MACAROON_SECRET_KEY}"
form_secret: "${FORM_SECRET}"
registration_shared_secret: "${REGISTRATION_SHARED_SECRET}"
registration_requires_token: ${REGISTRATION_REQUIRES_TOKEN}
```

Key points:
- `bind_addresses: 0.0.0.0` — accepts connections from Railway's reverse proxy (which runs in a separate container, not on localhost)
- `allow_unsafe_locale: true` — Railway's managed PostgreSQL uses `en_US.utf8` instead of locale `C`. Safe to bypass for a private homeserver without federation.
- `rc_login` — raised rate limits to prevent lockout during bridge startup
- `app_service_config_files` — tells Synapse to load the Discord bridge registration

**`discord-registration.yaml.template`:**
```yaml
id: discord
as_token: "${AS_TOKEN}"
hs_token: "${HS_TOKEN}"
namespaces:
  users:
    - exclusive: true
      regex: "^@discord_.*:${SYNAPSE_SERVER_NAME}$"
    - exclusive: true
      regex: "^@discordbot:${SYNAPSE_SERVER_NAME}$"
  aliases:
    - exclusive: true
      regex: "^#discord_.*:${SYNAPSE_SERVER_NAME}$"
url: "${BRIDGE_URL}"
sender_localpart: discordbot
rate_limited: false
de.sorunome.msc2409.push_ephemeral: true
push_ephemeral: true
```

This tells Synapse:
- The bridge's bot user is `@discordbot:domain`
- Users matching `@discord_*:domain` are ghost users managed by the bridge (one per Discord user)
- Aliases matching `#discord_*:domain` are managed by the bridge
- Push events to the bridge at `BRIDGE_URL` (Railway internal networking)
- `as_token` / `hs_token` — mutual authentication between Synapse and the bridge

**`entrypoint.sh`:**
```bash
#!/bin/sh
set -e

# Generate configs from templates
envsubst < /conf/homeserver.yaml.template > /data/homeserver.yaml
envsubst < /conf/discord-registration.yaml.template > /data/discord-registration.yaml

# Generate signing key if it doesn't exist (first run only)
if [ ! -f /data/signing.key ]; then
    echo "Generating signing key..."
    python -m synapse.app.homeserver \
        --config-path /data/homeserver.yaml \
        --generate-keys
fi

# Generate log config if it doesn't exist
if [ ! -f /data/log.config ]; then
    cat > /data/log.config << 'EOF'
version: 1
formatters:
  precise:
    format: '%(asctime)s - %(name)s - %(lineno)d - %(levelname)s - %(request)s - %(message)s'
handlers:
  console:
    class: logging.StreamHandler
    formatter: precise
root:
  level: INFO
  handlers: [console]
disable_existing_loggers: false
EOF
fi

# Start Synapse
exec python -m synapse.app.homeserver --config-path /data/homeserver.yaml
```

**`Dockerfile`:**
```dockerfile
FROM matrixdotorg/synapse:latest
RUN apt-get update && apt-get install -y gettext-base && rm -rf /var/lib/apt/lists/*
COPY homeserver.yaml.template /conf/homeserver.yaml.template
COPY discord-registration.yaml.template /conf/discord-registration.yaml.template
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

**`.env.example`:**
```
SYNAPSE_SERVER_NAME=synapse-production-xxxx.up.railway.app
ELEMENT_DOMAIN=element-web-production-xxxx.up.railway.app
POSTGRES_HOST=value-of-PGHOST
POSTGRES_PORT=5432
POSTGRES_DB=value-of-PGDATABASE
POSTGRES_USER=value-of-PGUSER
POSTGRES_PASSWORD=value-of-PGPASSWORD
ENABLE_REGISTRATION=true
MACAROON_SECRET_KEY=output-of-openssl-rand-hex-32
FORM_SECRET=output-of-openssl-rand-hex-32
AS_TOKEN=output-of-openssl-rand-hex-32
HS_TOKEN=output-of-openssl-rand-hex-32
BRIDGE_URL=http://mautrix-discord.railway.internal:29334
PORT=8008
REGISTRATION_SHARED_SECRET=output-of-openssl-rand-hex-32
REGISTRATION_REQUIRES_TOKEN=false
```

### 2c: Deploy on Railway

1. In Railway: **"+ New"** → **"GitHub Repo"** → select `synapse-config`
2. Rename the service to `synapse`

### 2d: Attach a persistent volume

Synapse stores signing keys, media cache, and generated configs on the filesystem. Without a persistent volume, this data is lost on every container restart.

1. On the project canvas, **right-click the Synapse service tile** (or click its `…` menu)
2. Select **"Attach Volume"**
3. Set the mount path to: `/data`
4. Save → the service will redeploy

It may/will fail. But don't worry, keep going.

### 2e: Generate a public domain

1. **"Settings"** → **"Networking"** → **"Generate Domain"**
2. Railway assigns a subdomain, e.g., `synapse-production-xxxx.up.railway.app` with port 8008
3. Record this. It becomes the `server_name` in Matrix — embedded in every user ID and event signature. **Cannot be changed without rebuilding all state from scratch.**

### 2f: Set environment variables

**"Variables"** tab → add all variables from `.env.example` with real values:

| Variable | Value |
|----------|-------|
| `SYNAPSE_SERVER_NAME` | Your Railway domain from 2e |
| `ELEMENT_DOMAIN` | (leave blank for now, fill in after Part 3) |
| `POSTGRES_HOST` | `PGHOST` value from Part 1 |
| `POSTGRES_PORT` | `5432` |
| `POSTGRES_DB` | `PGDATABASE` value from Part 1 |
| `POSTGRES_USER` | `PGUSER` value from Part 1 |
| `POSTGRES_PASSWORD` | `PGPASSWORD` value from Part 1 |
| `ENABLE_REGISTRATION` | `true` |
| `MACAROON_SECRET_KEY` | From step 2a |
| `FORM_SECRET` | From step 2a |
| `AS_TOKEN` | From step 2a |
| `HS_TOKEN` | From step 2a |
| `BRIDGE_URL` | `http://mautrix-discord.railway.internal:29334` |
| `PORT` | `8008` |
| `REGISTRATION_REQUIRES_TOKEN` | `false` |

Note: `BRIDGE_URL` points to a service that doesn't exist yet. Synapse will log a warning about not being able to reach it — that's expected until Part 5.

### 2g: redeploy

After setting environment variables, redeploy to set the container up.

## ✅ Checkpoint

Request the version endpoint from your terminal:
```bash
curl https://YOUR-SYNAPSE-DOMAIN.up.railway.app/_matrix/client/versions
```

Expected response (HTTP 200):
```json
{"versions":["r0.0.1","r0.1.0","r0.2.0",...],"unstable_features":{...}}
```

This confirms Synapse has started, connected to PostgreSQL, and is serving the Client-Server API.

**Verify in Railway logs:**
- No `IncorrectDatabaseSetup` errors
- No `yaml.scanner.ScannerError` errors
- Warning about `trusted_key_servers` → harmless, ignore
- Warning about unreachable Application Service → expected, the bridge isn't deployed yet

**If you get the JSON response → move to Part 3.**

