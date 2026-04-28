# Part 3: Element Web

## Goal

Element Web accessible via browser, connected to your Synapse instance, able to register and authenticate users. An admin account created.

## What this involves

Element Web is a React SPA served by nginx. The Docker image (`vectorim/element-web`) bundles the compiled JS and a default `config.json`. We override `config.json` to point Element at our Synapse instance via an `envsubst` template. The config file is read at page load by the client-side JS — nginx doesn't proxy to Synapse, it only serves static files. All Matrix API calls go directly from the user's browser to Synapse.

## Steps

### 3a: Create a GitHub repo

Create a repo `element-web-config` with four files:

**`config.json.template`:**
```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://${SYNAPSE_DOMAIN}",
            "server_name": "${SYNAPSE_DOMAIN}"
        }
    },
    "brand": "Level3 Chat",
    "default_theme": "dark",
    "room_directory": {
        "servers": ["${SYNAPSE_DOMAIN}"]
    }
}
```

**`entrypoint.sh`:**
```bash
#!/bin/sh
envsubst < /app/config.json.template > /app/config.json
exec nginx -g "daemon off;"
```

**`Dockerfile`:**
```dockerfile
FROM vectorim/element-web:latest
USER root
RUN apk add --no-cache gettext
COPY config.json.template /app/config.json.template
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

**`.env.example`:**
```
SYNAPSE_DOMAIN=synapse-production-xxxx.up.railway.app
```

### 3b: Deploy on Railway

1. **"+ New"** → **"GitHub Repo"** → select `element-web-config`
2. Railway detects the Dockerfile, builds, and deploys
3. Rename the service to `element-web`
4. Go to **"Variables"** tab → add:

| Variable | Value |
|----------|-------|
| `SYNAPSE_DOMAIN` | Your Synapse domain from Part 2e |

### 3c: Generate a public domain

1. **"Settings"** → **"Networking"** → **"Generate Domain"** with port 8080
2. Record the domain (e.g., `element-web-production-xxxx.up.railway.app`)
3. Go back to the **Synapse service** → **"Variables"** → set `ELEMENT_DOMAIN` to this domain

### 3d: Create an admin account

Accounts created via Element's registration are regular users with no admin privileges. You need a server admin account. This must be done via Synapse's CLI, not via Element.

1. SSH into the Synapse container:
   ```bash
   railway link
   # Select your project → environment → synapse service
   railway ssh
   ```
2. Run:
   ```bash
   register_new_matrix_user -c /data/homeserver.yaml -u admin -p YOUR_PASSWORD -a http://localhost:8008
   ```
   Replace `YOUR_PASSWORD` with a strong password. The `-a` flag grants server admin privileges.
3. If username `admin` is taken, pick a different name (e.g., `sysadmin`).

### 3e: Handle the E2E encryption verification prompt

When you log into Element, it may show a prompt asking you to "verify your device" (in German: "Bestätige deine Identität"). This is Element's end-to-end encryption cross-signing flow. Since this is your first session, there's nothing to verify against.

Click **"Bestätigung nicht möglich?"** (Can't confirm?) or **"Skip"** to dismiss it.

**Important for bridging:** mautrix-discord works with unencrypted rooms. When you create rooms to bridge, ensure encryption is **off**. Public rooms have encryption disabled by default.

## ✅ Checkpoint

**Test 1:** Load your Element domain in a browser. Element's login UI renders.

**Test 2:** Log in with the admin account you created in 3d. You see the main chat interface.

**Test 3:** Verify admin status — the admin account was created with `-a` flag via CLI (not via Element registration).

```
curl -s -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  "https://YOUR-SYNAPSE-DOMAIN/_synapse/admin/v2/users/@admin:YOUR-SYNAPSE-DOMAIN" | python3 -m json.tool
```

**If you can log in as admin → move to Part 4.**

## ❌ Troubleshooting

- **"Cannot reach homeserver"** → `SYNAPSE_DOMAIN` variable is wrong or missing on the element-web service. Check the Variables tab.
- **Blank page / doesn't load** → Check Railway build logs. Verify filenames: `Dockerfile` (capital D), `config.json.template`, `entrypoint.sh`.
- **Registration returns 403** → `ENABLE_REGISTRATION` is not `true` on the Synapse service.
- **`M_LIMIT_EXCEEDED` on login** → Synapse is rate-limiting. Wait 5-10 minutes.
- **`register_new_matrix_user` says "User ID already taken"** → Pick a different username.
- **"Verify your device" prompt** → Click "Can't confirm?" / "Skip". See step 3e.