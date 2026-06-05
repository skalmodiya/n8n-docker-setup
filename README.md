# n8n Docker Stack

A self-hosted [n8n](https://n8n.io) automation stack with persistent Postgres storage, a Qdrant vector database, and an ngrok tunnel for public webhook access.

## Stack

| Service | Image | Purpose | Default Port |
|---------|-------|---------|-------------|
| **n8n** | `docker.n8n.io/n8nio/n8n:latest` | Workflow automation UI & engine | `5678` |
| **postgres** | `postgres:16-alpine` | Persistent workflow/credential storage | `5432` |
| **qdrant** | `qdrant/qdrant:latest` | Vector database for AI workflows | `6333` (REST), `6334` (gRPC) |
| **ngrok** | `ngrok/ngrok:latest` | Reverse tunnel — exposes n8n to the internet | `4040` (inspector) |

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine + Compose v2)
- An [ngrok account](https://dashboard.ngrok.com) with an auth token
- A reserved ngrok domain (free tier includes one static domain)

## Setup

### 1. Clone and configure

```bash
git clone https://github.com/skalmodiya/n8n-docker-setup.git
cd n8n
cp .env.example .env
```

Open `.env` and fill in every value (see [Environment variables](#environment-variables) below).

### 2. Start the stack

```bash
docker compose up -d
```

Services start in dependency order: Postgres → Qdrant → n8n → ngrok.

### 3. Open the UI

| Interface | URL |
|-----------|-----|
| n8n (local) | http://localhost:5678 |
| n8n (public via ngrok) | https://\<your-domain\>.ngrok-free.app |
| Qdrant dashboard | http://localhost:6333/dashboard |
| ngrok inspector | http://localhost:4040 |

## Environment variables

Copy `.env.example` to `.env` and set each value before first run.

| Variable | Required | Description |
|----------|----------|-------------|
| `NGROK_AUTHTOKEN` | Yes | Auth token from https://dashboard.ngrok.com/get-started/your-authtoken |
| `NGROK_DOMAIN` | Yes | Your reserved ngrok domain (e.g. `foo.ngrok-free.app`) |
| `POSTGRES_USER` | Yes | Database username (default: `n8n`) |
| `POSTGRES_PASSWORD` | Yes | Database password — choose something strong |
| `POSTGRES_DB` | Yes | Database name (default: `n8n`) |
| `N8N_ENCRYPTION_KEY` | Yes | 32+ char random string — **set once, never change** |
| `QDRANT_API_KEY` | Recommended | API key for Qdrant REST/gRPC access |
| `QDRANT_READ_ONLY_API_KEY` | No | Optional read-only key for the Qdrant dashboard |
| `GENERIC_TIMEZONE` / `TZ` | No | Timezone for n8n scheduler (default: `UTC`) |

Generate secure random values:

```bash
openssl rand -hex 32
```

> **Warning:** `N8N_ENCRYPTION_KEY` encrypts stored credentials. If you lose it or change it, all saved credentials become unreadable and must be re-entered.

## Useful commands

```bash
# Start in the background
docker compose up -d

# Stop all services
docker compose down

# View logs for a specific service
docker compose logs -f n8n
docker compose logs -f ngrok

# Restart a single service
docker compose restart n8n

# Pull latest images and recreate containers
docker compose pull && docker compose up -d

# Remove all containers AND volumes (destroys all data)
docker compose down -v
```

## Data persistence

All data is stored in named Docker volumes:

| Volume | Contents |
|--------|----------|
| `postgres_data` | n8n workflows, credentials, execution history |
| `n8n_data` | n8n internal files (e.g. encryption metadata) |
| `qdrant_data` | Vector collections |
| `qdrant_snapshots` | Qdrant snapshot backups |

The `./files/` directory is bind-mounted into n8n at `/files` for sharing binary data between workflows and the host.

## Connecting n8n to Qdrant

In n8n, add a **Qdrant** credential:

- **URL:** `http://qdrant:6333`
- **API Key:** the value of `QDRANT_API_KEY` from your `.env`

Use the internal service name `qdrant` (not `localhost`) because n8n runs inside the Docker network.

## Troubleshooting

**n8n won't start / "waiting for postgres"**
Postgres health-checks must pass before n8n starts. Check logs: `docker compose logs postgres`

**Webhooks not receiving traffic**
Confirm ngrok is connected: `docker compose logs ngrok` and check http://localhost:4040. Make sure `NGROK_DOMAIN` in `.env` matches your actual reserved ngrok domain.

**"Invalid encryption key" after restart**
`N8N_ENCRYPTION_KEY` changed or was lost. Restore the original value — it must remain the same for the lifetime of the instance.

**Qdrant returns 401 Unauthorized**
The `QDRANT_API_KEY` in your n8n credential doesn't match the value in `.env`. Update the credential in n8n.
