# Part 6: Synapse Admin

## Goal

A web-based admin dashboard for managing users, rooms, and registration tokens — no CLI or `curl` commands needed.

## What this involves

Synapse Admin is a React SPA that connects to Synapse's Admin API. It provides a UI for tasks like creating users, resetting passwords, deactivating accounts, managing rooms, and creating registration tokens. It runs as a separate service on Railway — no GitHub repo needed, just a Docker image.

## Steps

### 6a: Deploy on Railway

1. In your Railway project: **"+ New"** → **"Docker Image"** → enter `awesometechnologies/synapse-admin`
2. Rename the service to `synapse-admin`
3. Go to **"Settings"** → **"Networking"** → **"Generate Domain"** → port **80**
4. Wait for it to deploy — no environment variables needed

### 6b: Log in

1. Open the generated URL in your browser
2. On the login page, enter:
   - **Homeserver URL**: `https://YOUR-SYNAPSE-DOMAIN` (your Synapse Railway domain)
   - **Username**: your admin username (e.g., `admin`)
   - **Password**: the admin password from Part 3d
3. You should see the admin dashboard

Note: Save this admin account somewhere. You will use it again and again in the future.

### ✅ Checkpoint

The Synapse Admin dashboard loads and you can see the **Users** section with your admin account listed.

---

### 6c: Create a registration token

Registration tokens let you control who can sign up without toggling environment variables or redeploying.

1. In Synapse Admin, go to **"Registration Tokens"** (in the sidebar)
2. Click **"Create"**
3. Fill in:
   - **Token**: a value partners will enter during registration (use something hard to guess)
   - **Uses allowed**: how many people can use this token (e.g., `1` for a single partner, `3` for a batch)
   - **Expiry time**: optional — when the token should stop working
4. Click **"Save"**

You can create, list, and delete tokens here anytime. No redeployment needed.

### 6d: Enable token-required registration

Now lock down registration so a token is required:

1. In Railway, go to the **Synapse service** → **"Variables"** tab
2. Set `REGISTRATION_REQUIRES_TOKEN` to `true`
3. Synapse redeploys automatically

Registration is still open, but the Element registration page now shows a **"Registration Token"** field. Partners must enter a valid token to sign up.

---

## What you can do in Synapse Admin

| Task | Where |
|------|-------|
| List all users | Users section |
| Create a user | Users → Create |
| Reset a user's password | Users → select user → edit |
| Deactivate a user | Users → select user → deactivate |
| List all rooms | Rooms section |
| Delete a room | Rooms → select room → delete |
| Create registration tokens | Registration Tokens section |
| Delete/expire tokens | Registration Tokens → select token |

and more.

---

## Security note

The Synapse Admin dashboard gives full control over your server to anyone who can log in with admin credentials. The obscure Railway subdomain provides some security through obscurity, but if you want to be more careful, you can delete the public domain after use and regenerate it only when you need it.

**If the dashboard is working → move to Part 7.**