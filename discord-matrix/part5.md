# Part 5: mautrix-discord

## Goal

mautrix-discord running and connected to both Synapse and Discord. Able to bridge rooms via `!discord bridge <channelID>` commands in Element, with relay mode enabled so all Element users' messages reach Discord.

## What this involves

mautrix-discord is a Matrix Application Service bridge. Unlike Matterbridge (a standalone relay), it registers directly into Synapse. This means:

- Synapse pushes Matrix events to mautrix-discord over HTTP
- mautrix-discord pushes events back to Synapse via the Client-Server API
- mautrix-discord connects to Discord via bot token + Gateway API
- Channel bridging is managed via chat commands in Element — no config files to edit, no redeployment needed

The bridge creates "ghost" users on Matrix for each Discord user (e.g., `@discord_123456:domain`). When a Discord user sends a message, it appears in Matrix as if that ghost user sent it — not as a generic "bridge-bot" message.

## Important: How v0.7.x handles configuration

The current version of mautrix-discord (v0.7.x) behaves differently from older versions:

- **Config rewriting:** When the bridge starts, it reads your minimal config template, merges it with its full default config, and **overwrites the config file** with the expanded version. This is expected — your template values (homeserver, database, tokens, permissions) are carried over.
- **Discord bot token is NOT set via config file.** Older versions had a `discord.token` field. v0.7.x removed this — you authenticate the bot via a chat command in Element after deployment (step 5f).

## Steps

### 5a: Create the bridge database (if not done in Part 1)

If you didn't create the `discord_bridge` database in Part 1b, do it now:

1. SSH into the Synapse container:
   ```bash
   railway link
   # Select synapse service
   railway ssh
   ```
2. Install psql and connect:
   ```bash
   apt-get update && apt-get install -y postgresql-client
   psql "postgres://PGUSER:PGPASSWORD@PGHOST:5432/PGDATABASE"
   ```
   Replace with your actual PostgreSQL values.
3. Create the database:
   ```sql
   CREATE DATABASE discord_bridge;
   \q
   ```

### ✅ Checkpoint 5a

In psql, run `\l` — you should see both your Synapse database and `discord_bridge` in the list.

---

### 5b: Create a GitHub repo for mautrix-discord

Create a repo `mautrix-discord-config` with four files:

**`config.yaml.template`:**
```yaml
homeserver:
  address: "https://${SYNAPSE_DOMAIN}"
  domain: "${SYNAPSE_DOMAIN}"
  software: standard

appservice:
  address: "http://0.0.0.0:${BRIDGE_PORT}"
  hostname: 0.0.0.0
  port: ${BRIDGE_PORT}
  database:
    type: postgres
    uri: "postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${BRIDGE_DB}?sslmode=disable"
  id: discord
  bot:
    username: discordbot
    displayname: Discord Bridge Bot
    avatar: "mxc://maunium.net/nIdEykemnwdisvHbpxflpDlC"
  as_token: "${AS_TOKEN}"
  hs_token: "${HS_TOKEN}"

bridge:
  command_prefix: "!discord"
  permissions:
    "*": relay
    "${SYNAPSE_DOMAIN}": user
    "@${BRIDGE_ADMIN}:${SYNAPSE_DOMAIN}": admin

logging:
  min_level: info
  writers:
    - type: stdout
      format: pretty-colored
```

Key points:
- `homeserver.address` — the public URL of your Synapse
- `appservice.database.uri` — connects to the `discord_bridge` database
- `appservice.as_token` / `hs_token` — must match the values in Synapse's registration file
- `bridge.permissions` — who can use the bridge:
  - `*` = everyone can relay messages
  - `SYNAPSE_DOMAIN` = all users on your homeserver can use bridge commands
  - `@BRIDGE_ADMIN:domain` = this user gets admin commands
- **No `discord.token` field** — v0.7.x removed this. You authenticate the bot via a chat command in step 5f.

**`registration.yaml.template`:**
```yaml
id: discord
as_token: "${AS_TOKEN}"
hs_token: "${HS_TOKEN}"
namespaces:
  users:
    - exclusive: true
      regex: "^@discord_.*:${SYNAPSE_DOMAIN}$"
    - exclusive: true
      regex: "^@discordbot:${SYNAPSE_DOMAIN}$"
  aliases:
    - exclusive: true
      regex: "^#discord_.*:${SYNAPSE_DOMAIN}$"
url: "${BRIDGE_URL}"
sender_localpart: discordbot
rate_limited: false
de.sorunome.msc2409.push_ephemeral: true
push_ephemeral: true
```

This file is also in the Synapse repo — both must render to identical content. The env vars ensure this.

**`entrypoint.sh`:**
```bash
#!/bin/sh
set -e

envsubst < /conf/config.yaml.template > /data/config.yaml
envsubst < /conf/registration.yaml.template > /data/registration.yaml

exec /usr/bin/mautrix-discord -c /data/config.yaml -r /data/registration.yaml
```

**`Dockerfile`:**
```dockerfile
FROM dock.mau.dev/mautrix/discord:latest
RUN apk add --no-cache gettext
COPY config.yaml.template /conf/config.yaml.template
COPY registration.yaml.template /conf/registration.yaml.template
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
RUN mkdir -p /data
ENTRYPOINT ["/entrypoint.sh"]
```

**`.env.example`:**
```
SYNAPSE_DOMAIN=synapse-production-xxxx.up.railway.app
DISCORD_BOT_TOKEN=your-discord-bot-token
POSTGRES_HOST=value-of-PGHOST
POSTGRES_PORT=5432
POSTGRES_USER=value-of-PGUSER
POSTGRES_PASSWORD=value-of-PGPASSWORD
BRIDGE_DB=discord_bridge
AS_TOKEN=same-value-as-synapse-service
HS_TOKEN=same-value-as-synapse-service
BRIDGE_URL=http://mautrix-discord.railway.internal:29334
BRIDGE_PORT=29334
BRIDGE_ADMIN=admin
PORT=29334
```

Note: `DISCORD_BOT_TOKEN` is listed here for your records but is **not used in the config template**. You'll provide it via a chat command in step 5f.

---

### 5c: Deploy on Railway

1. **"+ New"** → **"GitHub Repo"** → select `mautrix-discord-config`
2. Rename to `mautrix-discord`
3. Go to **"Variables"** tab → add:

| Variable | Value |
|----------|-------|
| `SYNAPSE_DOMAIN` | Your Synapse domain |
| `DISCORD_BOT_TOKEN` | Bot token from Part 4a (stored here for reference — not used in config template) |
| `POSTGRES_HOST` | Same `PGHOST` as Synapse |
| `POSTGRES_PORT` | `5432` |
| `POSTGRES_USER` | Same `PGUSER` as Synapse |
| `POSTGRES_PASSWORD` | Same `PGPASSWORD` as Synapse |
| `BRIDGE_DB` | `discord_bridge` |
| `AS_TOKEN` | Same value as Synapse service |
| `HS_TOKEN` | Same value as Synapse service |
| `BRIDGE_URL` | `http://mautrix-discord.railway.internal:29334` |
| `BRIDGE_PORT` | `29334` |
| `BRIDGE_ADMIN` | `admin` (or your admin username from Part 3d) |
| `PORT` | `29334` |

4. **No public domain needed** — Synapse reaches it via Railway's internal networking

**Critical:** `AS_TOKEN` and `HS_TOKEN` must be identical on both the Synapse and mautrix-discord services. If they don't match, Synapse will reject the bridge's requests.

---

### 5d: Verify mautrix-discord is running

#### ✅ Checkpoint: Check logs

In Railway, click the mautrix-discord service → **"Logs"** tab. Look for:

```
INF Bridge initialization complete, starting...
INF Starting HTTP listener address=0.0.0.0:29334
INF Bridge started!
```

At this point the bridge is running and registered with Synapse, but **not yet connected to Discord**. That happens in step 5f.

If you see errors about connecting to Synapse or the database, check the troubleshooting section.

---

### 5e: Verify the bridge bot exists in Element

1. Open Element → log in as admin
2. Start a new direct message
3. Search for `@discordbot:YOUR-SYNAPSE-DOMAIN` (use the full `@user:domain` format — partial search won't find it)
4. The Discord Bridge Bot user should appear

If it appears, Synapse has recognized the Application Service registration and created the bot user.

#### ✅ Checkpoint

The `@discordbot` user exists in Element.

---

### 5f: Connect the bridge to Discord

This step replaces the old `discord.token` config method. In v0.7.x, you authenticate the bot via a chat command.

1. In Element, open the **DM with the Discord Bridge Bot** (it may have sent you a welcome message like "Hello, I'm a Discord bridge bot")
2. In this DM, type:
   ```
   login-token bot YOUR_DISCORD_BOT_TOKEN
   ```
   Replace `YOUR_DISCORD_BOT_TOKEN` with the actual token from Part 4a.
3. The bot should respond with a success message.

> **Note:** In the DM management room, you don't need the `!discord` prefix — commands work directly.

#### ✅ Checkpoint

- The bot responds with a success/login confirmation in Element
```
Connecting to Discord as user ID 1497245507762782428
Successfully logged in as @Matrix Bridge
```

- In Discord, the `Matrix Bridge` bot shows as **online** in the member list

**If the bot is online in Discord → move to step 5g.**

---

### 5g: Bridge a channel

This is the payoff — bridging via a chat command, no config files.

1. In Element, create a new **public** room (e.g., `test`)
2. Invite `@discordbot:YOUR-SYNAPSE-DOMAIN` to the room
3. In Discord, right-click the channel you want to bridge → **"Copy Channel ID"** (requires Developer Mode from Part 4d)
4. In the Element room, type:
   ```
   !discord bridge <paste-channel-ID>
   ```
5. The bot should confirm the bridge is active

---

### 5h: Enable relay mode

By default, only the admin account (which authenticated the bot in step 5f) can send messages to Discord. Other Element users' messages won't go through. To fix this, enable **relay mode** in each bridged room.

In the **bridged Element room** (not the DM with the bot), type:

```
!discord set-relay --create
```

This creates a Discord webhook in the channel. Once active, **all** messages from Element users are relayed to Discord with their Matrix display names — not as the bot.

> **Do this in every room you bridge.** It's a per-room setting.

---

### 5i: Test both directions

#### ✅ Checkpoint

**Test 1 — Discord → Matrix:**
1. In Discord, type a message in the bridged channel
2. In Element, the message should appear in the room — posted by a ghost user with the Discord user's name

**Test 2 — Matrix → Discord (admin):**
1. In Element, type a message as the admin in the bridged room
2. In Discord, the message should appear with your Matrix bot display name (via the webhook)

**Test 3 — Matrix → Discord (non-admin):**
1. Register or log in as a different Element user
2. Join the bridged room
3. Send a message
4. In Discord, the message should appear with that user's display name

**If all three tests pass → move to step 5j or Part 6.**

---

### 5j: Bridge more channels

Repeat steps 5g and 5h for each Discord channel:

1. Create a **public** room in Element
2. Invite `@discordbot`
3. Copy the Discord channel ID
4. Type `!discord bridge <channelID>`
5. Type `!discord set-relay --create`
6. Test both directions

No redeployment, no config file editing. Just commands.

---

## Relay mode limitations

Relay mode uses Discord webhooks, which have limited capabilities. This affects what can cross from Element to Discord:

| Feature | Discord → Element | Element → Discord |
|---------|:-----------------:|:-----------------:|
| Messages | ✅ | ✅ |
| Files / images | ✅ | ✅ |
| Message edits | ✅ | ✅ |
| Reactions | ✅ | ❌ |
| Pins | ❌ | ❌ |
| Message deletes | ❌ | ✅ |
| Threads | ✅ | ✅ |
| Edit Messages | ✅ | ✅ |

The asymmetry exists because the bridge connects to Discord as a bot with full API access (can see all events), but sends to Discord via a webhook (which can only post and edit messages).

---

## ❌ Troubleshooting

- **Bridge logs: "cannot connect to homeserver"** → `SYNAPSE_DOMAIN` is wrong, or Synapse is not running. Verify with `curl`.
- **Bridge logs: "database connection failed"** → `discord_bridge` database doesn't exist (Part 5a), or PostgreSQL credentials are wrong.
- **Bridge logs: "authentication failed"** → `AS_TOKEN` or `HS_TOKEN` don't match between Synapse and mautrix-discord. They must be identical on both services.
- **`@discordbot` doesn't appear in Element** → Synapse hasn't loaded the registration. Check Synapse logs for appservice errors. Verify `BRIDGE_URL` is correct. Make sure to search the full `@discordbot:YOUR-SYNAPSE-DOMAIN` — partial search won't work.
- **`login-token` returns an error** → Double-check the bot token from Part 4a. Make sure there are no extra spaces or quotes. If the token was reset in the Discord Developer Portal, the old one is invalidated.
- **Bot stays offline in Discord after `login-token`** → Token might be wrong. Reset it in the Discord Developer Portal (Part 4a) and retry.
- **`!discord bridge` returns "not authorized"** → Your admin user isn't in the `bridge.permissions` with `admin` level. Check `BRIDGE_ADMIN` matches your Matrix username (without `@` and `:domain`).
- **`!discord bridge` returns "Channel not found"** → The bot is not connected to Discord. Complete step 5f first. Also verify the bot is in the Discord server (Part 4c).
- **`!discord bridge` returns no response** → The bot isn't in the room. Invite `@discordbot` first.
- **Admin's messages go to Discord but other users' messages don't** → Relay mode is not enabled. Run `!discord set-relay --create` in the bridged room (step 5h).
- **Messages appear as "Matrix Bridge" bot instead of the user's name** → Relay mode is not enabled. Run `!discord set-relay --create`.
- **Discord→Matrix works, Matrix→Discord doesn't** → Check that the bot has sufficient permissions in the Discord channel. Also ensure relay mode is enabled.
- **`M_LIMIT_EXCEEDED`** → Rate limiting from earlier crash loops. Wait 5-10 minutes.
- **Railway internal networking not working** → `BRIDGE_URL` might be wrong. Check that the mautrix-discord service name in Railway matches the hostname. You may need to verify Railway's internal DNS format.
- **Config file looks different from your template** → This is expected. v0.7.x rewrites the config file on startup, expanding your minimal template into its full default config. Your values are preserved.