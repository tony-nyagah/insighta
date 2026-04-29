# Insighta

Deployment repo for the Insighta profile intelligence platform. Contains a single `docker-compose.yml` that pulls the three service images from ghcr.io and wires them together.

The services themselves live in their own repos:

| Service | Repo | Image |
|---|---|---|
| REST API | insighta-backend | `ghcr.io/<owner>/insighta-backend:latest` |
| Web portal | insighta-web | `ghcr.io/<owner>/insighta-web:latest` |
| CLI | insighta-cli | `ghcr.io/<owner>/insighta-cli:latest` |

Images are built and pushed to ghcr.io automatically on every merge to `main` in each service repo.

---

## Architecture

```
                        ┌─────────────────────────────┐
                        │         GitHub OAuth         │
                        └────────────┬────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                       │
              ▼                      ▼                       ▼
   ┌─────────────────┐   ┌─────────────────────┐  ┌──────────────────┐
   │  insighta-cli   │   │   insighta-web       │  │                  │
   │  (Cobra CLI)    │   │   (Go + HTMX)        │  │   Direct API     │
   │                 │   │   :8069              │  │   consumers      │
   └────────┬────────┘   └──────────┬──────────┘  └────────┬─────────┘
            │                       │                       │
            └───────────────────────┴───────────────────────┘
                                    │  REST + JWT
                                    ▼
                       ┌────────────────────────┐
                       │    insighta-backend     │
                       │    (Go + chi)  :8080    │
                       └────────────┬────────────┘
                                    │
                                    ▼
                              ┌──────────┐
                              │  SQLite  │
                              └──────────┘
```

---

## Running locally

### 1. Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with the Compose plugin (`docker compose version`)
- A GitHub OAuth App

### 2. Create a GitHub OAuth App

1. Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App**
2. Fill in:
   - **Homepage URL**: `http://localhost:8069`
   - **Authorization callback URL**: `http://localhost:8080/auth/github/callback`
3. Click **Register application**, then generate a client secret.

### 3. Configure environment

```bash
cp .env.local.example .env.local
```

Open `.env.local` and fill in every value. The required ones are:

| Variable | Description |
|---|---|
| `GHCR_OWNER` | Your GitHub username or org (used to pull the images) |
| `GITHUB_CLIENT_ID` | From your OAuth App |
| `GITHUB_CLIENT_SECRET` | From your OAuth App |
| `JWT_SECRET` | Any long random string — `openssl rand -hex 32` |
| `SESSION_HASH_KEY` | 32+ byte HMAC key — `openssl rand -hex 32` |
| `SESSION_BLOCK_KEY` | Exactly 32 bytes — `openssl rand -base64 24` |
| `CSRF_AUTH_KEY` | Exactly 32 bytes — `openssl rand -base64 24 \| head -c 32` |

### 4. Pull and start

```bash
docker compose --env-file .env.local pull
docker compose --env-file .env.local up -d
```

| URL | What you'll see |
|---|---|
| http://localhost:8069 | Web portal login page |
| http://localhost:8080/health | Backend health check |

### Stopping

```bash
docker compose down        # stop containers (data volume preserved)
docker compose down -v     # stop and wipe the SQLite volume
```

### Updating to the latest images

```bash
docker compose --env-file .env.local pull
docker compose --env-file .env.local up -d
```

---

## Using the CLI

The CLI is included as an on-demand service (under the `tools` profile) so it doesn't start with `docker compose up`. Run commands against the already-running backend:

```bash
# Optional alias to save typing
alias insighta='docker compose --env-file .env.local run --rm cli'
# (the CLI image is pulled with the rest of the stack — no extra steps needed)

insighta login                                            # GitHub OAuth (opens browser)
insighta whoami                                           # show current user
insighta profiles list                                    # list all profiles
insighta profiles list --country NG --min-age 25 --limit 20
insighta profiles get <id>
insighta profiles search "young males from Lagos"
insighta profiles export --format csv --country NG        # saves CSV to current dir
insighta logout
```

Login credentials are persisted to `./cli-data/` on your host so you only need to log in once per session.

---

## Promoting a user to admin

New users are assigned the `analyst` role (read-only) on first login. To grant admin access:

```bash
docker compose exec backend sh
sqlite3 /app/data/insighta.db \
  "UPDATE users SET role = 'admin' WHERE username = 'your-github-login';"
```

---

## Repo contents

```
insighta/
├── docker-compose.yml      # pulls images from ghcr and wires the stack together
├── .env.local.example      # copy to .env.local and fill in values
└── cli-data/               # created on first CLI run; stores login credentials
```
