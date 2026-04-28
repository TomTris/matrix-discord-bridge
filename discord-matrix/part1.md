# Part 1: PostgreSQL

## Goal

A running PostgreSQL instance with two databases: one for Synapse, one for the Discord bridge.

## Why first

Synapse requires a database connection at startup. Without it, the process exits immediately. PostgreSQL is used over SQLite because SQLite doesn't support concurrent writes, which causes lock contention under any real usage. mautrix-discord also needs its own database to store bridge state (room mappings, puppet users, message correlations).

## Steps

### 1a: Create the PostgreSQL service

1. Go to [railway.app](https://railway.app) → sign up or log in
2. Click **"New Project"** → **"Empty Project"** → name it `matrix-chat`
3. Inside the project: **"+ New"** → **"Database"** → **"PostgreSQL"**
4. Click on the PostgreSQL service → **"Variables"** tab
5. Note these values (you'll use them in Part 2 and Part 5):
   - `PGHOST` — hostname of the database server
   - `PGPORT` — port (typically 5432)
   - `PGUSER` — database user
   - `PGPASSWORD` — database password
   - `PGDATABASE` — database name (this is Synapse's database)

### 1b: Create the bridge database
(You can do this now or at step 5a)

The default Railway PostgreSQL comes with one database. mautrix-discord needs its own. Connect to PostgreSQL and create it:

1. In Railway, click on the PostgreSQL service tile
2. Go to the **"Data"** tab (or use the query console if available)
3. If no query console is available, you'll create the database later via Synapse's shell in Part 2. Skip to the checkpoint for now and come back to this step after Part 2 is running.

From a shell (Synapse container after Part 2, or any container with `psql`):

```bash
apt-get update && apt-get install -y postgresql-client
psql "postgres://PGUSER:PGPASSWORD@PGHOST:5432/PGDATABASE"
```

Replace with your actual PostgreSQL values. Then inside psql:

```sql
CREATE DATABASE discord_bridge;
\q
```

## ✅ Checkpoint

- The PostgreSQL service is in "Active" state
- All five variables (`PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`) have values
- The `discord_bridge` database exists (verify with `\l` in psql if you've already created it)

**If PostgreSQL is active → move to Part 2.**