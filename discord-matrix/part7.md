# Part 7: Go Live

## Goal

All learning track channels bridged with relay mode enabled, all partners with accounts, token-based registration active.

## Steps

### 7a: Bridge all learning track channels

For each Discord channel you want to bridge:

1. In Element (as admin), create a **public** room
2. Invite `@discordbot:YOUR-SYNAPSE-DOMAIN`
3. In Discord, right-click the channel → **"Copy Channel ID"**
4. In the Element room, type: `!discord bridge <channelID>`
5. Enable relay mode: `!discord set-relay --create`
6. Test both directions (send a message from Discord, verify it appears in Element; send from Element, verify it appears in Discord)

No config files. No redeployments.

> **Don't skip step 5.** Without `!discord set-relay --create`, only the admin account can send messages to Discord. Other Element users' messages won't go through.

### 7b: Onboard partners

1. In Synapse Admin (Part 6), create a registration token with the number of uses matching the number of partners you're onboarding
2. Send each partner:

> 1. Open: `https://YOUR-ELEMENT-DOMAIN.up.railway.app`
> 2. Click "Create Account"
> 3. Enter the registration token: `YOUR_TOKEN`
> 4. Complete registration
> 5. If you see a "Verify your device" prompt, click "Can't confirm?" or "Skip" to dismiss it
> 6. Open the room directory (compass icon) to find your learning track rooms
>
> That's it — messages you send here will appear in the students' Discord, and vice versa.

### 7c: Verify token-based registration is active

Make sure these variables are set on the Synapse service (on Railway):

| Variable | Value |
|----------|-------|
| `ENABLE_REGISTRATION` | `true` |
| `REGISTRATION_REQUIRES_TOKEN` | `true` |

With both set, the registration page is accessible but requires a valid token. No need to toggle anything on and off.

### 7d: Adding a new partner later

1. In Synapse Admin, create a new registration token (set uses to `1`)
2. Send the partner the token and the Element URL
3. They register using the token
4. The token is spent after use — no cleanup needed

---

## Relay mode limitations

See the limitations table in Part 5. In short: messages, files, and edits work both ways. Reactions, pins, deletes, and threads do not fully bridge in both directions.

## ✅ Final Checkpoint

- [ ] Every learning track has a bridged Discord channel ↔ Matrix room pair
- [ ] Relay mode is enabled in every bridged room (`!discord set-relay --create`)
- [ ] Messages relay in both directions for all pairs — including from non-admin Element users
- [ ] All partners can authenticate and send/receive in Element
- [ ] `REGISTRATION_REQUIRES_TOKEN` is `true`
- [ ] Synapse Admin dashboard is accessible for user/token management