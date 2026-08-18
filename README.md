# UP IT Pulse

AI-assisted IT incident triage and early-warning system for Mahasarakham University's CITCOMS (มหาวิทยาลัยพะเยา). Reporters submit Thai/English tickets; AI classifies, masks PII, finds similar tickets, and flags suspected major incidents — humans review and decide on every action.

Full spec: [docs/SPEC.md](docs/SPEC.md)

## Status

Spec-only. No application code has been scaffolded yet. Setup instructions below reflect the planned stack from the spec (Section 14–16) and will apply once `apps/web` and `apps/api` exist.

## Planned Stack

- **Frontend**: Next.js + TypeScript, Tailwind CSS, shadcn/ui, TanStack Query
- **Backend**: FastAPI (Python), Pydantic, SQLAlchemy + Alembic
- **Data**: PostgreSQL with `pgvector`, optional Redis
- **AI**: Provider-agnostic LLM + embedding model via adapter (OpenAI-compatible endpoint)
- **Deployment**: Docker Compose (`web`, `api`, `postgres`, optional `redis`)

## Setup (once scaffolded)

1. Clone the repo:
   ```bash
   git clone https://github.com/Padungkiet/up-it-pulse.git
   cd up-it-pulse
   ```
2. Copy environment template and fill in secrets:
   ```bash
   cp .env.example .env
   ```
3. Start services:
   ```bash
   docker compose -f infra/docker-compose.yml up --build
   ```
4. Run database migrations and seed data:
   ```bash
   docker compose exec api alembic upgrade head
   docker compose exec api python -m scripts.seed
   ```
5. Open the app:
   - Web: http://localhost:3000
   - API: http://localhost:8000/api/v1

## Demo Logins

Three demo roles (`REPORTER`, `OFFICER`, `ADMIN`) — see `docs/SPEC.md` Section 5 for role permissions and Section 20 for demo scenarios.

## Tests

```bash
docker compose exec api pytest
docker compose exec web pnpm test
```

## Repository Structure

See `docs/SPEC.md` Section 15 for the full suggested layout (`apps/web`, `apps/api`, `packages/shared-types`, `infra/`, `data/`, `docs/`).
