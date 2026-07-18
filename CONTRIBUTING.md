# Contributing

## The 3-service split (read this first)

This repo has three services, each with a clear owner. Understanding the split will save you from stepping on someone else's code:

- **`apps/web`** (Frontend) — Next.js. Upload UI, paper viewer, topic dashboard. Talks to the Core API only, never directly to the ML Service or any external AI provider.
- **`apps/api`** (Backend) — FastAPI. Auth, uploads, job orchestration (Celery), CRUD, database writes. This service is deliberately kept lightweight — no ML libraries, no direct LLM/embedding calls. When it needs parsing, topic extraction, or article recommendations, it calls the ML Service over HTTP.
- **`apps/ml-service`** (ML) — FastAPI, internal-only (not publicly reachable). Owns PDF parsing (GROBID/PyMuPDF), chunking, embeddings, LLM prompting, and retrieval/reranking. Exposes a small internal contract: `POST /parse`, `POST /extract-topics`, `POST /recommend-articles`.

**Why split it this way:** the ML engineer can change prompts, swap models, add fine-tuning, or run experiments and deploy `ml-service` independently — no backend deploy required. The backend engineer never needs torch/transformers installed locally to work on auth or job orchestration. Neither blocks the other.

**The internal contract is the thing to protect.** If you're changing a request/response schema on `POST /parse`, `/extract-topics`, or `/recommend-articles`, that's a two-person conversation (backend + ML) before you touch code, not a solo change — the other service depends on that shape.

## Local setup

See the [README](./README.md#getting-started). tl;dr: `uv sync` at the repo root installs both Python services in one shot (it's a uv workspace); `pnpm install` in `apps/web` for the frontend.

## Working in the uv workspace

Both Python services share one root `pyproject.toml` (`[tool.uv.workspace]`) and one lockfile (`uv.lock`), but each service still has its own dependency list in its own `apps/*/pyproject.toml`.

- Adding a dependency to just one service: `uv add --package api <name>` or `uv add --package ml-service <name>`. This updates that service's `pyproject.toml` and the shared `uv.lock`.
- Running a command inside one service's environment: `uv run --package api <cmd>` / `uv run --package ml-service <cmd>`.
- **Do not** add ML dependencies (torch, transformers, sentence-transformers, etc.) to `apps/api`. If the Core API needs something from the ML Service, that's a sign it should be an HTTP call to `ml-service`, not a shared import.
- After pulling changes that touch `uv.lock`, just re-run `uv sync` at the root — it's fast, uv resolves incrementally.

## Branching & PRs

- Branch naming: `<track>/<short-description>` — e.g. `frontend/upload-ui`, `backend/job-orchestration`, `ml/topic-extraction-prompt`.
- Keep PRs scoped to one service where possible. A PR that touches `apps/api` and `apps/ml-service` together should generally mean you're changing the contract between them — call that out explicitly in the PR description and tag the other track's owner as a reviewer.
- PR description should state: what changed, why, and (if it touches the internal API contract) the exact request/response shape before and after.
- CI must pass before merge (each service has its own path-filtered GitHub Actions workflow — see `.github/workflows/`).

## Code style

| Service | Lint | Format | Test |
|---|---|---|---|
| `apps/web` | ESLint | Prettier (via ESLint) | Vitest/Jest — `pnpm test` |
| `apps/api` | `uv run --package api ruff check` | `uv run --package api ruff format` | `uv run --package api pytest` |
| `apps/ml-service` | `uv run --package ml-service ruff check` | `uv run --package ml-service ruff format` | `uv run --package ml-service pytest` |

Run the relevant lint/test commands before pushing. CI will catch it either way, but catching it locally is faster for everyone.

## Commit messages

Conventional commits preferred, not strictly enforced: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`. Scope with the service name when it's not obvious from context, e.g. `feat(ml-service): add reading-level reranking step`.

## Database changes

Only `apps/api` writes to the schema-defining tables (`users`, `papers`, `jobs`) — `apps/ml-service` writes to `sections`, `topics`, `articles`, and embedding columns, but the schema itself (migrations) lives in and is owned by `apps/api` via Alembic. If `ml-service` needs a new column or table, open a PR against `apps/api`'s migrations rather than writing raw SQL from the ML Service — this keeps one source of truth for the schema.

```bash
# Generate a new migration
uv run --package api alembic revision --autogenerate -m "add topic_articles table"

# Apply migrations
uv run --package api alembic upgrade head
```

## Issues & labels

- `enhancement` — new functionality (most Phase 0+ work falls here)
- `frontend` / `backend` / `ml` / `infra` — track ownership, stack alongside `enhancement`
- `bug` — something broken
- `epic` — tracking issue with sub-issue checklist, one per phase

## Environment variables & secrets

Never commit `.env` files. Each service has a `.env.example` documenting required variables — copy it to `.env` (or `.env.local` for `apps/web`) and fill in your own credentials for Neon, Upstash, R2, and OpenAI/Anthropic. See the README for which free-tier service each variable maps to.

## Questions

If it's about the internal API contract between `api` and `ml-service`, loop in both the backend and ML owner before making a decision — that boundary is the one place where an independent change in one service can silently break the other.
