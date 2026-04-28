# Part 4: Discord Bot

## Goal

A Discord bot application with a valid token, with the required Gateway Intents enabled, added to your Discord server.

## What this involves

Discord's API requires authentication via a bot token. The bot token is created in Discord's Developer Portal and scoped to a specific application. Gateway Intents are permission flags that control which events Discord sends to the bot over the WebSocket connection — specifically, `MESSAGE_CONTENT` is required for the bridge to read message text (Discord stopped sending message content to bots by default in 2022 as a privacy change).

## Steps

### 4a: Create the application

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. **"New Application"** → name: `Matrix Bridge` → Create
3. **"Bot"** tab (sidebar) → **"Reset Token"** → copy and store the token securely
4. This token is shown once. If lost, you must reset it (which invalidates the old one).

### 4b: Enable Privileged Gateway Intents

On the **"Bot"** tab → **"Privileged Gateway Intents"** section, enable:

- ✅ **Presence Intent** — receives user online/offline status events
- ✅ **Server Members Intent** — receives guild member join/leave events
- ✅ **Message Content Intent** — receives `content` field in MESSAGE_CREATE events. **Without this, the bridge receives empty messages.**

Save.

### 4c: Generate OAuth2 invite URL and add bot to server

1. **"OAuth2"** tab → **"URL Generator"**
2. Scopes: `bot`
3. Bot Permissions: `Administrator`

   Using `Administrator` simplifies permissions for the bridge. mautrix-discord needs to read messages, send messages, manage webhooks, manage channels, and more. If you prefer granular permissions, at minimum enable: `View Channels`, `Send Messages`, `Read Message History`, `Manage Webhooks`, `Manage Channels`, `Embed Links`, `Attach Files`, `Add Reactions`.

4. Copy the generated URL → open in browser → select your Discord server → Authorize

### 4d: Enable Developer Mode in Discord

This lets you copy channel IDs, which you'll need when bridging.

1. In Discord → click the gear icon (User Settings)
2. Go to **Advanced**
3. Enable **Developer Mode**
4. Now you can right-click any channel → **"Copy Channel ID"**

## ✅ Checkpoint

- The bot (`Matrix Bridge`) appears in your Discord server's member list with **offline** status
- Developer Mode is enabled (you can right-click a channel and see "Copy Channel ID")

Offline is correct — the bot exists and is in your server, but no process is using the token yet.

**If the bot is visible in the server → move to Part 5.**

## ❌ Troubleshooting

- **Bot not visible** → The OAuth2 URL was wrong. Regenerate with `bot` checked under Scopes.
- **"Bot requires code grant"** → In OAuth2 → General settings, disable "Require OAuth2 Code Grant".
- **"Private applications cannot have a default authorization link"** → Go to the **"Installation"** tab, find **"Default Install Link"** and set it to **"None"**. Then retry.
- **Lost the token** → Bot tab → "Reset Token" to generate a new one.