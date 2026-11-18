---
name: add-infra-resource
description: Provision a new infrastructure resource for StackShop both locally and in Kubernetes. Covers a Kafka topic (+ retry/DLQ topics), RabbitMQ exchange/queue/DLQ, SQS queue + DLQ, S3/MinIO bucket, MySQL schema + least-privilege user, MongoDB database/user, Redis config, or Elasticsearch index template. Use when a backend handoff or spec needs a topic, queue, bucket or database, or when adding a whole datastore/broker in its roadmap phase.
---

# Add an infrastructure resource

Every resource is defined **twice, kept identical**: locally in `stackshop-infra/local/init/` (idempotent scripts run by compose) and in K8s in `stackshop-infra/k8s/data/` (declarative CRs/values). Names and settings must match `docs/04-messaging-and-events.md` / `docs/03-data-architecture.md` exactly. If they don't, fix the docs first (`sync-docs-adr`).

## Kafka topic

- Local, `local/init/kafka/create-topics.sh`:
  ```bash
  kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic ordering.order-events \
    --partitions 6 --replication-factor 1 --config retention.ms=604800000
  ```
  Also create `<topic>.retry.1m` (if the consumer uses retry topics) and `<topic>.dlq` (1 partition). Compacted topics: `--config cleanup.policy=compact`.
- K8s, Strimzi `KafkaTopic` CR in `k8s/data/kafka/topics/<topic>.yaml` (replicas 3, `min.insync.replicas: 2` in staging/prod).

## RabbitMQ

- Local: add the exchange, queue (durable, `x-queue-type: quorum` in K8s / classic locally is fine), binding, the retry/delayed setup and the `.dlq` queue + DLX to `local/init/rabbitmq/definitions.json` (loaded via `load_definitions`). The delayed-message plugin is enabled in the image.
- K8s: messaging-topology-operator CRs (`Exchange`, `Queue`, `Binding`, `Policy` for the DLX) in `k8s/data/rabbitmq/`.

## SQS (LocalStack locally, AWS in prod)

- Local, `local/init/localstack/init-aws.sh`: create the DLQ first, then the main queue with `RedrivePolicy` (maxReceiveCount per docs) and `VisibilityTimeout`. FIFO queues get the `.fifo` suffix + `FifoQueue=true` (+ `ContentBasedDeduplication=false`).
- Cloud: Terraform (`terraform/sqs.tf`, stretch goal) or LocalStack in the dev cluster. Services get access through IRSA/workload identity, not static keys, in prod.

## S3 / MinIO bucket

Local: `mc mb --ignore-existing local/<bucket>` (+ `mc anonymous set download` for the public bucket only) in the init script. Add a CORS rule for pre-signed browser PUTs from the frontend origins. K8s/cloud: a Terraform/MinIO tenant config.

## MySQL schema + user

`local/init/mysql/01-create-schemas.sql` (append, idempotent):
```sql
CREATE DATABASE IF NOT EXISTS ordering_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE USER IF NOT EXISTS 'ordering'@'%' IDENTIFIED BY '${ORDERING_DB_PASSWORD}';
GRANT ALL PRIVILEGES ON ordering_db.* TO 'ordering'@'%';   -- only its own schema
```
One schema **and one user per service**. Never share users. K8s: an init Job or operator user CR. The password goes into a SealedSecret/ExternalSecret.

## MongoDB

Database + user with the `readWrite` role on its own DB only. The local container runs as a **single-node replica set** (`init-replica.js`), which transactions require.

## Elasticsearch

The index template + initial alias (`products` → `products-v1`) is created by the search service at startup. Infra only provides the cluster (ECK in K8s), and the ILM/snapshot repository if needed.

## Verify

- Local: `docker compose -f local/compose.yml up -d` (re-run the init container), then list the resource (`kafka-topics --describe`, the RabbitMQ UI/`rabbitmqctl list_queues`, `aws --endpoint-url http://localhost:4566 sqs get-queue-attributes`, `SHOW GRANTS FOR 'ordering'`). Run the init twice to prove it's idempotent.
- K8s: `kustomize build k8s/data | kubeconform -strict -ignore-missing-schemas` (CRDs → `-schema-location` with the CRD catalog).

## Handoff

**backend-engineer:** the exact names/URLs/env var keys for the resource and the secret name holding the credentials.
