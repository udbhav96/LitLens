# PaperPrereq

Upload a research paper (PDF) and get back a ranked list of prerequisite topics you need to understand it — each with recommended articles to read first.

## How it's built

Three independently deployable services, one repo:

| Service | Path | Owner | Stack |
|---|---|---|---|
| **Web** | `apps/web` | Frontend | Next.js 15 (App Router), React 19, TypeScript, Tailwind + shadcn/ui |
| **Core API** | `apps/api` | Backend | FastAPI, PostgreSQL (Neon) + pgvector, Celery + Redis (Upstash) |
| **ML Service** | `apps/ml-service` | ML | FastAPI (internal-only), GROBID, PyMuPDF, OpenAI/Anthropic/HF |

The Core API owns auth, uploads, jobs, and CRUD. It never calls an LLM or a parsing library directly — it delegates that to the ML Service over a small internal REST contract (`/parse`, `/extract-topics`, `/recommend-articles`). This split means the backend and ML engineer can each iterate and deploy independently. See [`docs/architecture.md`](./docs/architecture.md) for the full design doc if present, or ask in the team channel for the latest architecture writeup.

## Prerequisites

- [uv](https://docs.astral.sh/uv/) (Python package manager — installs and manages Python itself too, no separate Python install needed)
- [pnpm](https://pnpm.io/) (Node package manager for the frontend)
- Docker + Docker Compose (for running GROBID locally)
- Accounts for the free-tier services below (all have a permanent free tier, no credit card required for the ones marked ✅):

| Service | Used for | Free tier |
|---|---|---|
| [Neon](https://neon.tech) | Postgres + pgvector | ✅ 0.5 GB storage, 100 compute-hrs/month |
| [Upstash](https://upstash.com) | Redis (Celery broker) | ✅ 256 MB, 500K commands/month |
| [Cloudflare R2](https://developers.cloudflare.com/r2/) | Object storage (PDFs) | ✅ 10 GB, 1M writes/10M reads per month, zero egress |
| [Render](https://render.com) | Hosting for `api` / `ml-service` | ✅ Free web services (no card) |
| OpenAI / Anthropic | LLM calls | Paid — get an API key, low usage in dev is cheap |

## Getting started

Clone the repo, then from the root:

```bash
# Python services (installs BOTH apps/api and apps/ml-service in one shot —
# this is a uv workspace, not two separate installs)
uv sync

# Frontend
cd apps/web && pnpm install && cd ../..

# GROBID (runs locally in dev — no hosting cost)
docker compose up -d grobid
```

Copy the example env files and fill in your own service credentials (Neon connection string, Upstash Redis URL, R2 keys, OpenAI/Anthropic keys):

```bash
cp apps/api/.env.example apps/api/.env
cp apps/ml-service/.env.example apps/ml-service/.env
cp apps/web/.env.example apps/web/.env.local
```

Run the database migrations:

```bash
uv run --package api alembic upgrade head
```

Start everything (three terminals, or use your process manager of choice):

```bash
# Terminal 1 — Core API
uv run --package api uvicorn app.main:app --reload --port 8000

# Terminal 2 — ML Service
uv run --package ml-service uvicorn app.main:app --reload --port 8001

# Terminal 3 — Frontend
cd apps/web && pnpm dev
```

Open http://localhost:3000.

## Running tests

```bash
uv run --package api pytest
uv run --package ml-service pytest
cd apps/web && pnpm test
```

## Adding a dependency

```bash
# Add to the Core API only
uv add --package api <package-name>

# Add to the ML Service only
uv add --package ml-service <package-name>

# Frontend
cd apps/web && pnpm add <package-name>
```

Never install a heavy ML dependency (torch, transformers, etc.) into `apps/api` — it's meant to stay lightweight. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the full reasoning behind the service split.

## Project structure

```
.
├── apps/
│   ├── web/           # Next.js frontend
│   ├── api/            # Core API (FastAPI) — auth, uploads, jobs, CRUD
│   └── ml-service/      # ML Service (FastAPI, internal-only) — parsing, topic extraction, retrieval
├── infra/               # Terraform (Neon, Upstash, Cloudflare, Render)
├── docker-compose.yml   # Local GROBID for dev
├── pyproject.toml       # uv workspace root
└── uv.lock              # Shared lockfile for apps/api + apps/ml-service
```

## Deployment

`apps/web` deploys to Vercel. `apps/api` and `apps/ml-service` deploy to Render as separate web services, each built via `uv sync --package <name> --frozen --no-dev` from the shared workspace lockfile. See `infra/` for the Terraform config.

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md).
