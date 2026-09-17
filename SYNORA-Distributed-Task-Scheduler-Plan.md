# Distributed Task Scheduler & Worker System — Project Plan

## Overview

This document describes an end-to-end plan to build the Distributed Task Scheduler & Worker System (Node.js + TypeScript, Redis + BullMQ, PostgreSQL, Next.js UI). The system provides at-least-once execution, delayed/cron jobs, worker heartbeats and reassignment, DLQ, retries, prioritization, and an observability dashboard.

## Acceptance Criteria (MVP)
- Enqueue jobs (immediate, delayed, cron) via REST API.
- Workers fetch/execute jobs, report success/failure.
- At-least-once semantics with configurable retries and DLQ for failed jobs.
- Worker heartbeat + lease mechanism: detect dead workers and reassign running jobs.
- Idempotency support for job handlers (idempotency key pattern).
- Observability UI: queue depth, worker list with heartbeat, job history, DLQ actions (requeue/delete).
- Automated tests: unit, integration (kill worker mid-job, confirm reassignment), basic load test.

## High-level Architecture

Devices/Clients → REST API (Scheduler) → Redis (BullMQ queues) & Postgres (metadata) ←→ Worker(s)

- Scheduler API: enqueues and manages jobs, stores metadata in Postgres.
- Redis: primary broker and queue store (BullMQ).
- Workers: subscribe to queues, execute jobs, publish status. Send heartbeats to scheduler via Redis or HTTP.
- Postgres: job metadata, audit logs, user/rule storage.
- UI (Next.js): dashboards, job CRUD, manual requeue/DLQ management, worker control.

## Key Components

- `scheduler-service` (Node.js + TypeScript): REST API, job validation, enqueueing, job metadata persistence.
- `worker-service` (Node.js + TypeScript): generic worker process with pluggable handlers, heartbeat, idempotency hooks.
- `db` (PostgreSQL): tables for jobs, job_attempts, workers, dlq, users.
- `redis` (Redis): BullMQ backend.
- `ui` (Next.js): dashboard and control plane.
- `infra` (docker-compose / k8s manifests)

## Data Model (relational)

- `jobs` (id UUID, name, queue, payload JSONB, status enum, priority int, scheduled_at timestamptz, created_at, updated_at)
- `job_attempts` (id, job_id, worker_id, attempt_number, status, error TEXT, started_at, finished_at)
- `workers` (id, hostname, pid, last_heartbeat timestamptz, status enum, metadata JSONB)
- `dlq` (id, job_id, reason, payload, failed_at)

Note: Redis holds queue state; Postgres is the authoritative metadata/audit store.

## API Surface (examples)

- `POST /api/v1/jobs` — Enqueue job.
  - body: `{ "name": "send-email", "queue": "default", "payload": {...}, "delay_ms": 0, "cron": null, "priority": 10, "idempotency_key": "abc" }`
- `GET /api/v1/jobs/:id` — Job details & attempts.
- `GET /api/v1/queues` — List queues and depths.
- `GET /api/v1/workers` — Worker list & heartbeat times.
- `POST /api/v1/dlq/:id/retry` — Requeue DLQ job.
- `POST /api/v1/workers/:id/kill` — (dev only) send shutdown signal.

Webhooks / callbacks: optional `on_complete` and `on_failure` URLs stored in job payload.

## Job Lifecycle & Guarantees

1. Scheduler validates request, persists `jobs` record in Postgres with status `queued`.
2. Scheduler enqueues job to BullMQ with job id and options (delay, attempts, backoff).
3. Worker reserves job (BullMQ lock), updates `job_attempts` (started), and sends periodic heartbeats/lease refresh.
4. Worker executes handler; on success: mark attempt succeeded, job status -> `completed` and call on_complete.
5. On failure: mark attempt failed; if retries remain, BullMQ requeues with backoff; otherwise move job to DLQ and record reason.
6. If worker heartbeat expires while executing, scheduler/worker supervisor reclaims job (BullMQ lock expiry or explicit reassign), increments attempt, and requeues.

Idempotency: job handlers should honor `idempotency_key` (store handled keys in Postgres or Redis with TTL) to avoid duplicate side-effects.

## Worker Design

- Generic worker process supporting pluggable handlers (JS modules). Handler API: `execute(payload, context) => { success:boolean, result }`.
- Heartbeat: worker writes TTL key `worker:{id}:hb` in Redis every `n` seconds and updates `workers.last_heartbeat` in Postgres periodically.
- Lease: BullMQ lock TTL should be shorter than heartbeat interval so worker refreshes lock; on lock loss, worker should abort handler if possible.
- Graceful shutdown: worker stops accepting new jobs, waits configurable grace window for running tasks, then either exit or release.

## Retry & DLQ Strategy

- Configure attempts and backoff per job. Use exponential backoff with jitter.
- When attempts exhausted, move to `dlq` table and emit event to UI/ops.
- Provide API to requeue DLQ item (optionally with payload edits) or delete.

## Scheduling & Cron

- Support immediate, delayed (timestamp or ms delay), and cron schedules via `bullmq` repeatable jobs.
- Scheduler creates a Postgres record for repeatable jobs and reconciles with Redis on startup.

## Observability & Metrics

- Expose Prometheus metrics: queue size, job success/failure rates, attempt durations, worker heartbeats.
- UI: realtime queue depth, worker list & last heartbeat, job timeline with attempts, DLQ table with actions.
- Logs: structured JSON logs; correlate traces via `job_id` and `attempt_id`.

## Testing Plan

- Unit tests: sched validation, DB mappers, worker handlers.
- Integration: compose stack (Redis, Postgres, scheduler, worker). Tests include enqueue → execute flow, delayed jobs, cron.
- Failure injection: kill worker mid-execution and verify job reassignment and at-least-once semantics. Use deterministic handlers to assert side-effect counts with idempotency.
- Load tests: simple throughput tests using k6 or Artillery to ensure queueing under load.

## Security & Ops

- Auth: API keys / JWT for scheduler API.
- Secure Redis/Postgres in production (TLS, auth), restrict network access.
- Secrets in environment variables or secret manager.

## Milestones & Timeline (suggested)

Sprint 1 (2 days): Repo + Docker Compose (redis, postgres, scheduler skeleton, worker skeleton, UI skeleton). Basic enqueue API and simple worker that logs payload.

Sprint 2 (3 days): Persist job metadata in Postgres, implement job attempts tracking, implement retries and DLQ handling, simple UI showing queue depth and job list.

Sprint 3 (4 days): Worker heartbeat + lease refresh, implement reassignment on worker failure, idempotency mechanism, and integration tests (kill worker mid-job).

Sprint 4 (3 days): Add delayed & cron jobs, observability metrics (Prometheus) + UI improvements, DLQ requeue UI.

Sprint 5 (2 days): Load testing, performance tuning, CI, docs, and runbook; prepare k8s manifests for production.

Total: ~2–3 weeks focused work (single developer), adjustable.

## Repo Layout (recommended)

```
repo/
├─ docker-compose.yml
├─ services/
│  ├─ scheduler/ (Node TS) 
│  ├─ worker/ (Node TS)
│  └─ ui/ (Next.js)
├─ infra/ (k8s manifests)
└─ tests/ (integration + failure-injection scripts)
```

## Docker Compose (minimal snippet)

```yaml
version: '3.8'
services:
  redis:
    image: redis:7
    ports: ['6379:6379']
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ['5432:5432']
  scheduler:
    build: ./services/scheduler
    depends_on: [redis, postgres]
  worker:
    build: ./services/worker
    depends_on: [redis, postgres]
  ui:
    build: ./services/ui
    ports: ['3000:3000']
```

## Local dev & quickstart

1. Start stack:

```bash
docker compose up --build
```

2. Create a job (example):

```bash
curl -X POST http://localhost:4000/api/v1/jobs -H 'Content-Type: application/json' -d '{"name":"hello","queue":"default","payload":{"msg":"hi"}}'
```

3. Watch worker logs and UI at `http://localhost:3000`.

## CI / CD

- Run lint, tests, build images in CI (GitHub Actions). Deploy to k8s using manifests; run migrations for Postgres; run Prometheus/Grafana in infra for monitoring.

## Deliverables

- Working Docker Compose dev environment.
- Scheduler & worker services with full job lifecycle.
- Next.js UI for observability and DLQ operations.
- Tests covering failure scenarios (worker kill, DLQ, retries).
- Documentation & runbook for deploying and operating the system.

---

If you approve this plan I will scaffold the repository and create the initial Docker Compose and service skeletons (Sprint 1). Reply `scaffold` to begin and I will create files and run the first smoke tests.
