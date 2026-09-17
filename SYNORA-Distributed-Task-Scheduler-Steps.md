# Distributed Task Scheduler — Step‑by‑Step Implementation Guide

This document lists concrete, ordered steps to build the Distributed Task Scheduler & Worker System from zero to a fully working, tested, deployable system (Node.js + TypeScript, Redis + BullMQ, PostgreSQL, Next.js UI). Follow each step and run the included commands. Do not skip steps.

Prerequisites
- Git, Docker (Engine + Compose v2), Node.js 18+ and npm/yarn, PostgreSQL client (psql), Redis CLI.
- Recommended: Linux/macOS or WSL2 on Windows for best Docker compatibility.

Step 0 — Create repository and branches
- Create a new repo folder and initialize git.

```bash
mkdir synora-task-scheduler && cd synora-task-scheduler
git init
```

- Create two main branches: `main` and `develop`.

```bash
git checkout -b develop
```

Step 1 — Add monorepo skeleton and Docker Compose
- Create directory layout and `docker-compose.yml` with `redis`, `postgres`, `scheduler`, `worker`, `ui` services.
- Add `.env.example` with database and redis connection strings.

Commands (create files):
1. `docker-compose.yml` (see project plan snippet)
2. `.env` (copy `.env.example` and set values)

Quick compose start (dev):
```bash
docker compose up -d --build
```

Step 2 — Scheduler service skeleton (Node + TypeScript)
- Create `services/scheduler` folder. Initialize Node + TypeScript project and add deps: `express`, `bullmq`, `ioredis`, `pg`, `knex` or `typeorm`, `dotenv`, `winston`, `ts-node-dev`, `jest`.

Example commands:
```bash
cd services
mkdir scheduler && cd scheduler
npm init -y
npm install express bullmq ioredis pg knex dotenv winston
npm install -D typescript ts-node-dev @types/node @types/express jest ts-jest @types/jest
npx tsc --init
```

- Implement minimal files:
  - `src/index.ts` — start Express server, health endpoint.
  - `src/queue.ts` — BullMQ connection factory.
  - `src/api/jobs.ts` — `POST /api/v1/jobs` endpoint to validate and enqueue into BullMQ and insert metadata into Postgres.
  - `Dockerfile` and `dev` npm script.

Step 3 — Worker service skeleton
- Create `services/worker` folder.
- Install same runtime deps plus any task‑specific packages.
- Implement `src/worker.ts` that connects to BullMQ and registers processors for `default` queue; log received payload and mark job complete.
- Add basic idempotency store integration with Redis (set idempotency key with TTL) and record attempts to Postgres.

Step 4 — Database schema & migrations
- Use `knex` or `typeorm` migrations to create tables: `jobs`, `job_attempts`, `workers`, `dlq`.
- Add a `migrations/` folder and SQL files. Run migrations in `scheduler` start script if DB not initialized.

Step 5 — Basic enqueue → execute E2E test
- Start compose stack.
- Start `scheduler` and `worker` services locally.
- Run curl to enqueue job.

```bash
curl -X POST http://localhost:4000/api/v1/jobs \
  -H 'Content-Type: application/json' \
  -d '{"name":"hello","queue":"default","payload":{"msg":"hello world"}}'
```

- Observe worker logs; check `jobs` and `job_attempts` tables in Postgres.

Step 6 — Retries, backoff, and DLQ
- Update enqueue API to set BullMQ job options: `attempts`, `backoff` (exponential + jitter).
- Implement BullMQ `failed` event listener in scheduler or worker that writes to `dlq` table when attempts exhausted.

Step 7 — Heartbeat & lease handling
- Workers periodically update `workers` table and set `worker:{id}:hb` key in Redis with TTL.
- Configure BullMQ lock TTL to be slightly less than heartbeat TTL; worker refreshes lock during long jobs.
- Implement detection script in scheduler: when worker heartbeat expired and worker has running jobs, reclaim via BullMQ APIs (move job back to waiting or failover queue).

Step 8 — Idempotency and side‑effects handling
- Implement idempotency key handling: when a job with `idempotency_key` is processed, worker checks Redis key `idem:{key}`; if present and marked success, skip side-effect and mark job success; otherwise proceed and set key on success.

Step 9 — Delayed & cron jobs
- Use BullMQ repeatable jobs for cron. For delayed one‑offs, use job options `delay`.
- Add scheduler endpoints to create repeatable jobs and list them.

Step 10 — Observability UI (Next.js)
- Create `services/ui` with Next.js, React, Tailwind.
- Implement pages: `/` (overview), `/queues`, `/workers`, `/jobs/:id`, `/dlq`.
- UI fetches REST API endpoints and subscribes to server‑sent events or WebSocket for live updates (queue depth, new jobs, worker heartbeats).

Step 11 — Testing & failure injection
- Integration test harness (`tests/integration`) that:
  - Starts Docker Compose stack programmatically (use `docker-compose` CLI in scripts).
  - Enqueues a deterministic job that sleeps for X seconds and increments a counter (e.g., a Redis counter keyed by `test:<job_id>`).
  - While job running, kill worker container (`docker compose kill worker`) to simulate failure.
  - Restart worker and assert counter shows exactly 1 successful side-effect when idempotency is used, assert job finished and audit logs recorded.

Step 12 — Load testing
- Write a simple k6 or artillery script to enqueue many jobs per second and measure latency to completion and queue depth.

Step 13 — CI, lints, and preflight
- GitHub Actions pipeline: `lint`, `unit test`, `build images`, `integration tests` (optional step that runs compose), `publish artifacts`.

Step 14 — Deployment
- Provide `k8s/` manifests or Helm charts: Deploy Redis (managed Redis recommended), Postgres (managed recommended), scheduler & worker as deployments with autoscaling, UI as deployment.
- Add readiness/liveness probes for scheduler/worker and Prometheus scraping annotations.

Step 15 — Acceptance checklist (what to verify)
- Enqueue immediate job → worker executes → Postgres attempt recorded → UI shows completed.
- Delayed job triggers at approximate scheduled time (± 1s for dev infra).
- Cron job repeats per schedule.
- Worker killed mid-job → job reassigned and completed after restart.
- Failed job after retries appears in DLQ; UI can requeue it.
- Idempotent job executed only once side‑effect observed.

Step 16 — Deliverables & docs
- Deliver code for `services/{scheduler,worker,ui}`, `docker-compose.yml`, `migrations/`, `tests/`, `README.md` with exact run commands.

Troubleshooting Tips
- If jobs aren't running: check Redis connectivity, BullMQ connection settings, and queue names. Inspect `bull-board` or BullMQ events.
- If jobs duplicate side-effects: implement idempotency key storage and verify worker updates the key on completion.

Next actions (if you want me to continue):
- Reply `scaffold` and I will create the monorepo skeleton, `docker-compose.yml`, initial scheduler and worker TypeScript skeletons, Postgres migrations, and a minimal Next.js UI skeleton, then run the smoke test locally.
