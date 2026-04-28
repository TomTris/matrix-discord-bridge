# Reference

## Service inventory

| Service | Image / Source | Public endpoint | Persistent state |
|---------|---------------|----------------|-----------------|
| PostgreSQL | Railway managed | None (internal) | Railway-managed disk |
| Synapse | `synapse-config` repo | `synapse-xxxx.up.railway.app` | `/data` volume (signing keys, media) |
| Element Web | `element-web-config` repo | `element-web-xxxx.up.railway.app` | None (stateless SPA) |
| mautrix-discord | `mautrix-discord-config` repo | None (internal only) | Bridge state in `discord_bridge` database |

## Operational tasks

| Task | Action |
|------|--------|
| Bridge a new channel | In Element: create public room → invite `@discordbot` → `!discord bridge <channelID>` |
| Unbridge a channel | In the bridged Element room: `!discord unbridge` |
| Add a partner | Set `ENABLE_REGISTRATION=true` on Synapse → redeploy → partner registers → set back to `false` → redeploy |
| Rotate Discord token | Reset in Developer Portal → update `DISCORD_BOT_TOKEN` on mautrix-discord service → redeploy |
| Rotate bridge tokens | Generate new `AS_TOKEN` and `HS_TOKEN` → update on BOTH Synapse and mautrix-discord → redeploy both |
| SSH into Synapse | `railway link` (select Synapse) → `railway ssh` |
| View bridge logs | Railway → mautrix-discord service → Logs |
| View Synapse logs | Railway → synapse service → Logs |
| View bridge help | In Element, DM `@discordbot` → type `!discord help` |

## Bridge commands reference

All commands are typed in Element, inside a room where `@discordbot` is present:

| Command | What it does |
|---------|-------------|
| `!discord help` | Show all available commands |
| `!discord bridge <channelID>` | Bridge this room to a Discord channel |
| `!discord unbridge` | Unbridge this room |
| `!discord ping` | Check if the bridge is connected to Discord |
| `!discord guilds status` | Show which Discord servers the bot is in |

## Migrate to a real domain later

Since `server_name` is immutable in Matrix, migrating means rebuilding:

1. Add CNAME DNS record pointing your subdomain to Railway
2. Assign custom domain to Synapse and Element services in Railway
3. Update `SYNAPSE_SERVER_NAME` and `SYNAPSE_DOMAIN` variables on all services
4. Recreate Synapse (new signing key, new database)
5. Recreate user accounts (3 partners)
6. Re-bridge all channels via `!discord bridge` commands

If self-hosting PostgreSQL at that point, create the database with locale `C` (`initdb --locale=C`) so you can remove `allow_unsafe_locale: true`.

## How to open a shell

```bash
brew install railway       # first time only
railway login
railway link               # select project → environment → service
railway ssh
```

`railway ssh` connects to the running container. `railway shell` only opens a local shell with env vars — it does NOT connect to the remote container.