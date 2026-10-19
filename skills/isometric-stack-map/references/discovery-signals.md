# Discovery signals

Where components, links, and network boundaries hide in a repo. Read in this order. The deployment and runtime descriptors are the strongest evidence. Imports confirm what code actually talks to.

## 1. Runtime topology (strongest)

| File | Gives you |
|---|---|
| `docker-compose*.yml`, `compose*.yaml` | services → nodes; `image:` → tech; `ports:` → port; `depends_on:` and env URLs → links; `networks:` → zones; `profiles:` → `state: "idle"` |
| `k8s/`, `manifests/`, `charts/*/templates`, `kustomization.yaml` | Deployments and StatefulSets → nodes; Services → ports; Ingress → gateway; namespaces → zones; NetworkPolicy → allowed links; CronJob → worker |
| Terraform / Pulumi / CDK / CloudFormation / SST | managed resources (RDS, ElastiCache, SQS, S3, Lambda, ALB, CloudFront) → nodes; VPC/subnets/security groups → zones; SG ingress rules → links and ports |
| `serverless.yml`, `sst.config.*`, `vercel.json`, `netlify.toml`, `wrangler.toml`, `fly.toml`, `render.yaml`, `Procfile`, `app.yaml` | functions, routes, workers, bindings (KV, D1, R2, queues) |
| `Dockerfile*` | which apps ship as containers; `EXPOSE` → port; base image → runtime chip |
| Reverse-proxy config (`nginx.conf`, `Caddyfile`, `traefik.*`, `envoy.yaml`) | gateway → upstream links |

## 2. Configuration names (medium)

`.env.example`, `.env.sample`, config schemas (`zod`/`envalid`/`pydantic-settings`), Helm `values.yaml`. Use the **names** only: `DATABASE_URL`, `REDIS_URL`, `KAFKA_BROKERS`, `STRIPE_SECRET_KEY`, `S3_BUCKET` each imply a node and a link. **Never read or copy values from real `.env` files or secrets stores.**

## 3. Code imports (confirms the links)

Search each app's source for client libraries and attribute the link to the app that imports them:

| Import / dependency | Node kind · protocol |
|---|---|
| `pg`, `postgres`, `prisma`, `drizzle-orm`, `typeorm`, `sequelize`, `knex`, `psycopg`, `sqlalchemy`, `gorm`, `diesel`, `sqlx` | database · SQL |
| `mongodb`, `mongoose`, `pymongo` | database · Mongo wire |
| `redis`, `ioredis`, `@upstash/redis`, `go-redis` | cache · RESP |
| `bullmq`, `bee-queue`, `celery`, `sidekiq`, `rq` | queue + worker · async |
| `kafkajs`, `@confluentinc/*`, `confluent-kafka`, `sarama` | queue (topic) · Kafka, async |
| `amqplib`, `pika` | queue · AMQP, async |
| `@aws-sdk/client-sqs`, `boto3` sqs | queue · SQS, async |
| `@aws-sdk/client-s3`, `@google-cloud/storage`, `@azure/storage-blob` | storage · HTTPS |
| `@elastic/elasticsearch`, `meilisearch`, `typesense`, `algoliasearch` | database (search) · HTTPS |
| `stripe`, `@sendgrid/*`, `resend`, `twilio`, `openai`, `@anthropic-ai/sdk`, `auth0`, `@clerk/*`, `posthog`, `@sentry/*` | external · HTTPS (Sentry and analytics: `critical: false`) |
| `fetch`/`axios`/`got`/`httpx`/`requests` to an env-configured base URL | link to the service that owns that URL |
| `@grpc/grpc-js`, `grpcio`, `.proto` files | link · gRPC |
| `socket.io`, `ws`, `graphql-ws` | link · WebSocket |
| `@trpc/*`, `graphql`, `apollo` | link · tRPC / GraphQL over HTTP |

Use the stack's tooling to list workspace packages and their internal imports: `pnpm -r ls --depth -1`, `npm query .workspace`, `go list ./...`, `cargo metadata`, Nx/Turbo graphs. A monorepo with no runtime topology is drawn as a **package stack** with `library` nodes and `imports` links.

## 4. Entry points (fills the top of the stack)

Next.js `app/`/`pages/`, Remix/SvelteKit/Nuxt routes → `frontend`. `bin` in `package.json`, `cmd/`, `cli.py` → `client` (CLI). Mobile roots (`ios/`, `android/`, Expo `app.json`) → `client`. Webhook route handlers → an `external` node calling in with `async: true`.

## State heuristics

- `on`: part of the default `docker compose up`, deployed by CI, or a managed resource in IaC.
- `idle`: behind a compose profile, a feature flag, a `dev`-only script, or commented out in the deploy config.
- `off`: marked deprecated, unused (no imports), or disabled in IaC.
- `external`: SaaS or anything owned by another repo or team.

When a component is inferred from config but not seen in code (or the reverse), keep it, and say so in `desc` ("inferred from REDIS_URL; no client import found").
