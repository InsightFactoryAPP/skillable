# Scene schema

The renderer in [../assets/stack-map.html](../assets/stack-map.html) draws whatever scene JSON it is given. Write the scene. Never edit the renderer per project. `scripts/build_scene.py` validates the scene and injects it.

## Top level

```json
{
  "title": "acme-shop",
  "subtitle": "pnpm monorepo · Next.js · NestJS · Postgres · SQS",
  "repoUrl": "https://github.com/acme/shop",
  "commit": "3f9c2ab",
  "layers": [], "nodes": [], "links": [], "zones": [], "flows": []
}
```

`repoUrl` and every `url` must be `https://`. The renderer turns anything else into plain text.

## Layers: the tech stack, bottom first

Each layer is one floating platform. Order is **bottom → top**, so the data tier sits at the bottom and clients sit at the top:

```json
[{ "id": "data", "label": "Data" }, { "id": "services", "label": "Services" },
 { "id": "apps", "label": "Apps & edge" }, { "id": "clients", "label": "Clients" }]
```

Use 3–5 layers. Typical sets:

| Project shape | Layers (bottom → top) |
|---|---|
| Web app + API + DB | Data · Services · Apps & edge · Clients |
| Kubernetes platform | Infra (cluster, ingress, mesh) · Data · Services · Edge · Clients |
| Serverless | Data & storage · Functions · Gateway · Clients |
| Library / monorepo with no runtime | Foundation packages · Core · Adapters · Apps / CLIs |
| Data pipeline | Sources · Ingest · Processing · Storage · Consumers |

## Nodes: components

```json
{ "id": "api", "label": "api", "kind": "service", "layer": "services", "state": "on",
  "tech": ["NestJS 10", "Node 22"], "port": 8080,
  "desc": "REST API for catalog, cart and checkout.",
  "cite": [{ "path": "apps/api/src/main.ts", "line": 12, "url": "https://github.com/acme/shop/blob/3f9c2ab/apps/api/src/main.ts#L12" }] }
```

| Field | Notes |
|---|---|
| `id` | Unique, stable, kebab-case. |
| `label` | Short name as the team says it (≤ 16 chars reads best). |
| `kind` | Picks the glyph (see below). |
| `layer` | A layer id. |
| `state` | `on` glows (part of the default runtime). `idle` is dashed (defined but optional: compose profiles, dev-only, feature-flagged). `off` is grey (deprecated or disabled). `external` is dashed and has no fill (third-party or not in this repo). |
| `tech` | 1–3 chips, most defining first. The first one shows under the label. |
| `port` | Listening port when known. |
| `desc` | One sentence. |
| `cite` | Files that prove the component exists: `{path, line?, url?}` or a plain `"path:line"` string. |
| `x`, `y`, `size` | Optional overrides. Leave them out and the renderer lays out each layer by neighbour barycenter. `size` can override `w`, `d`, `h`, `r`. |

### Kinds → glyphs

| kind | glyph | use for |
|---|---|---|
| `service` | box | APIs, backends, monolith processes |
| `frontend` | slab + screen | SSR/SPA web apps, admin UIs |
| `client` | small slab + screen | browsers, mobile apps, CLIs used by people |
| `gateway` | truncated pyramid | load balancers, ingress, API gateways, CDNs, reverse proxies |
| `worker` | hexagonal prism | queue consumers, cron jobs, schedulers |
| `function` | small cube | lambdas, edge functions, cloud functions |
| `database` | tall cylinder with rings | SQL/NoSQL databases, search indexes |
| `cache` | short cylinder | Redis, Memcached, in-memory stores |
| `storage` | open-top cylinder | S3/GCS/blob storage, volumes |
| `queue` | long slotted bar | SQS, Kafka topics, RabbitMQ, BullMQ, Pub/Sub |
| `external` | box (use `state: "external"`) | Stripe, Auth0, OpenAI, Twilio, SaaS APIs |
| `library` | flat box | internal packages, when mapping a monorepo's package graph |
| `infra` | wide flat box | cluster, service mesh, observability stack |

## Links: the network

```json
{ "from": "api", "to": "postgres", "protocol": "SQL", "port": 5432,
  "async": false, "critical": true, "kind": "call", "rate": 1,
  "desc": "Prisma client", "cite": [{ "path": "apps/api/src/db.ts", "line": 4 }] }
```

- **Direction = packet direction = who initiates.** `from` calls `to`.
- `kind`: `call` (default) means `from` depends on `to`. `consume` or `subscribe` means the packets flow `from` producer/queue `to` consumer, but the **consumer** depends on the source. Use this for queue → worker and topic → subscriber.
- `async: true` draws a dashed cable with sparse packets (queues, events, webhooks).
- `critical: false` means the caller keeps working if the target is down (caches with fallbacks, analytics, optional SaaS). It is excluded from failure propagation.
- `rate` scales packet speed (0.5–3). Use it only when the code shows a clear hot path.
- One link per ordered pair. Merge several protocols into one label (`"HTTP + WS"`).

## Zones: network boundaries

Dashed wireframe volumes drawn around their member nodes, across layers:

```json
{ "id": "vpc", "label": "VPC 10.0.0.0/16", "nodes": ["api", "worker", "postgres"] }
```

Use them for VPCs, subnets, Kubernetes clusters and namespaces, Docker networks, and trust boundaries ("PCI scope"). Keep to 3 or fewer. Overlapping zones get hard to read.

## Flows: key request paths

```json
{ "id": "checkout", "label": "Checkout", "steps": [
  { "from": "browser", "to": "web", "note": "POST /checkout", "cite": { "path": "apps/web/app/checkout/actions.ts", "line": 18 } }
] }
```

Every step must match a link (either direction). Use 2–4 flows of 4–10 steps each, for the paths that matter most: a typical read, a write, a background job, auth.
