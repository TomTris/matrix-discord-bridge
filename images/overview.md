# Discord ↔ Matrix Bridge: Setup Guide

## Architecture Overview

```
Partner (browser)                    Student (Discord client)
       │                                      │
       │ HTTPS (Matrix Client-Server API)     │ WSS (Discord Gateway API)
       ▼                                      ▼
┌─────────────┐                       ┌──────────────┐
│ Element Web  │                       │   Discord    │
│ (static SPA) │                       │   (hosted)   │
└──────┬──────┘                       └──────┬───────┘
       │                                      │
       │ REST API calls                       │
       ▼                                      │
┌─────────────┐    Application Service    ┌───┴──────────┐
│   Synapse    │◄────── HTTP API ────────►│ mautrix-     │
│ (homeserver) │    (bidirectional)        │ discord      │
└──────┬──────┘                           └──────────────┘
       │              Synapse pushes events TO the bridge.
       │              Bridge pushes events TO Synapse.
       │              Bridge connects to Discord via Gateway API.
       │
       │◄──── Admin API ────┐
       │                     │
┌──────┴──────┐       ┌─────┴────────┐
│ PostgreSQL   │       │ Synapse Admin │
│ (persistence)│       │ (admin SPA)   │
└─────────────┘       └──────────────┘
```

**Components:**

- **PostgreSQL** — Relational database. Synapse and mautrix-discord each use their own database on the same PostgreSQL instance. Synapse stores user accounts, room membership, event DAGs, device keys, and media metadata. mautrix-discord stores bridge state: which Matrix rooms are linked to which Discord channels, puppet user mappings, and message ID correlations.
- **Synapse** — Reference implementation of a Matrix homeserver. Implements the Matrix Client-Server API (used by Element) and the Application Service API (used by mautrix-discord). Manages authentication, room state, event ordering, and push notifications. Runs as a Python process (Twisted-based).
- **Element Web** — Single-page application (React) that implements a Matrix client. Served as static files (HTML/JS/CSS) by nginx inside the Docker container. All logic runs client-side; it communicates with Synapse exclusively via the Matrix Client-Server REST API.
- **mautrix-discord** — A Matrix Application Service bridge written in Go. Unlike a relay bot (e.g., Matterbridge), it registers directly into Synapse as an appservice. Synapse pushes events to it over HTTP, and it pushes events back via the Client-Server API. It connects to Discord via the Gateway API. Bridging is managed via chat commands inside Element (`!discord bridge <channelID>`) — no config file editing or redeployment needed to add channels. **Note:** In v0.7.x, the Discord bot token is no longer set via config file — it's provided through a chat command (`login-token bot <token>`) in Element after deployment.
- **Synapse Admin** — A React SPA that connects to Synapse's Admin API. Provides a web UI for managing users (create, deactivate, reset passwords), rooms, and registration tokens. Deployed as a standalone Docker image — no GitHub repo needed.

**Key difference from Matterbridge:** Matterbridge is a standalone relay that logs into both platforms as a regular user. mautrix-discord is an Application Service — it's a registered extension of Synapse itself. This means:
- Synapse and mautrix-discord share authentication tokens (`as_token`, `hs_token`)
- Synapse needs to reach mautrix-discord over HTTP (Railway internal networking)
- Channel bridging is managed via commands in Element, not config files

**Dependency chain:** PostgreSQL → Synapse → Element (for verification) → mautrix-discord → Synapse Admin. Each service requires the previous one to be operational.

**Relay mode:** By default, only the admin user (who authenticated the bot) can send messages to Discord. To allow all Element users to send messages, each bridged room needs relay mode enabled via `!discord set-relay --create`. This creates a Discord webhook per channel. Messages from Element users are relayed with their display names. Relay mode has limitations — reactions, pins, and threads only flow from Discord to Element, not the other direction (see Part 5 for the full table).

**Registration:** Partner access is managed via registration tokens. Registration stays open (`ENABLE_REGISTRATION=true`) but requires a valid token (`REGISTRATION_REQUIRES_TOKEN=true`). Tokens are created and managed through Synapse Admin's web UI — no CLI or redeployment needed.

**Hosting:** All services run as Docker containers on Railway. Railway provides container orchestration, persistent volumes, managed PostgreSQL, automatic HTTPS via generated subdomains, GitHub-based CI/CD (push to repo → auto-redeploy), and internal networking between services.