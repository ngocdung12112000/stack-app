---
name: local-env
description: Start, check, stop or troubleshoot the StackShop local development environment across the three repos (stackshop-infra compose, stackshop-backend services, stackshop-frontend Next.js). Use when the user says "start the app", "run everything locally", "why is X not connecting", "reset local data", or before a feature demo.
---

# Local environment

The order matters: **infra → backend → frontend**. See `docs/07-devops-cicd.md` §1 for the ports table.

## Start

```bash
# 1. Third-party infra. Creates docker network "stackshop". Use profiles to save RAM.
cd stackshop-infra && docker compose -f local/compose.yml up -d
#    (only some profiles: docker compose -f local/compose.yml --profile messaging --profile search up -d)

# 2. Backend services (gateway :5000)
cd stackshop-backend && docker compose up -d --build
#    or a single service from source: cd services/<svc> && dotnet watch --project src/<Svc>.Api

# 3. Frontend (web :3000, admin :3001)
cd stackshop-frontend && pnpm install && pnpm dev
```

Seed data (once, or after a reset): run `tools/Seeder` from the backend repo (`dotnet run --project tools/Seeder -- --seed`).

## Health check (do this before saying "it's running")

1. `docker compose -f local/compose.yml ps` → every container is `healthy` / `running`.
2. Check each layer:
   - MySQL: `docker exec <mysql> mysql -uroot -p... -e "show databases"` → one schema per service exists
   - MongoDB: `rs.status().ok == 1` (the replica set must be initiated or transactions fail)
   - Kafka: the topic list matches `docs/04-messaging-and-events.md` §3.2 (Kafka UI :8085)
   - RabbitMQ :15672: exchanges/queues from the definitions exist
   - LocalStack: `aws --endpoint-url http://localhost:4566 sqs list-queues`
   - Elasticsearch: `curl localhost:9200/_cluster/health` → green/yellow
3. Backend: `curl localhost:5000/health/ready` and each service's `/health/ready` → Healthy. Prometheus :9090 targets are all UP.
4. Frontend: open `http://localhost:3000` and check it renders data from the gateway.

Report what's up, what's down, and the exact error for anything down.

## Common problems

| Symptom | Likely cause → fix |
|---|---|
| Backend can't resolve `mysql`/`kafka` hostnames | Backend compose isn't on the external `stackshop` network, or infra isn't started → start infra first, check `networks: stackshop: external: true` |
| Mongo "Transaction numbers are only allowed on a replica set" | Replica set not initiated → re-run `local/init/mongo/init-replica.js` |
| Kafka consumers get no messages | Wrong `KAFKA_ADVERTISED_LISTENERS` (host vs container) → containers use `kafka:9092`, the host uses `localhost:9092` |
| Elasticsearch exits with code 137 | Out of memory → set the ES heap to 1g, raise Docker Desktop memory (≥ 10–12 GB for the full stack) |
| Port already in use | Another local service → change the port in an override file, or stop the other process |
| `401` on everything from the frontend | Identity signing keys changed / cookies stale → clear cookies, check the JWKS URL config |
| Frontend type errors after pull | API client changed → use the `bump-api-client` skill |

## Stop and reset

```bash
cd stackshop-frontend   # Ctrl+C on pnpm dev
cd stackshop-backend && docker compose down
cd stackshop-infra && docker compose -f local/compose.yml down        # keep data
cd stackshop-infra && docker compose -f local/compose.yml down -v     # ⚠ deletes all local data volumes
```

Always confirm with the user before `down -v`, because it wipes every local database.
