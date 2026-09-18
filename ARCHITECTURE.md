# CDPForge architecture

Understanding this platform used to mean reading 26 READMEs. This is the map:
what each repository is, how an event travels through the system, which
contracts hold it together, and where to change what.

It describes the platform as a whole. Each repository's own README stays the
place for its details.

---

## The one-paragraph version

A visitor's browser sends an event to **core-input**, which publishes it onto
an Apache Pulsar topic. A chain of **pipeline stages** consumes it, each one
enriching it — geography, identity, behavioural scores, catalogue, triggers —
and passing it on. The last stage indexes it into OpenSearch. **core-api** is
the product backend that reads all of that and serves the two admin panels,
the e-commerce integrations and the MCP server. Everything is deployed by one
Helm chart.

```
 browser / shop
   │  plugin-tracking-web, integration-plugin-*
   ▼
 core-input ─────────────────────────────► Pulsar topic `logs`
   POST /events                                  │
                                                 ▼
                             core-pipeline-stage        (priority 0, blocking)
                             plugin-pipeline-geo        (1, blocking)
                             plugin-pipeline-identity   (2, blocking)
                             plugin-pipeline-trigger    (5, blocking)
                             plugin-pipeline-scoring    (6, parallel)
                             plugin-pipeline-catalog    (7, parallel)
                             plugin-pipeline-gorse-…    (100, parallel)
                                                 │
                                                 ▼
                             core-pipeline-output ──► OpenSearch
                                                      users-logs-<client>

 core-api ◄── reads OpenSearch + MySQL ──► core-web-panel, core-manager-panel
```

---

## The repositories

### The event path

| Repository | What it is |
|---|---|
| `plugin-tracking-web` | The JavaScript tracker that runs in the visitor's browser. Queues events and retries, so a slow network does not lose data. |
| `integration-plugin-wordpress` `-woocommerce` `-prestashop` `-magento` `-shopify` | Platform plugins that install the tracker on a shop and emit its commerce events (product views, cart, purchase) and catalogue exports. |
| `core-input` | The ingestion API. `POST /events` is the only producer on the Pulsar `logs` topic. Applies per-instance security rules and the consent gate before anything is published. |
| `core-pipeline-manager` | The pipeline registry. Stages register themselves; it assigns their Pulsar topics, persists the topology to MySQL and broadcasts changes. |
| `core-pipeline-stage` | The priority-0 stage. Every event on the platform passes through it: PV quota, instance active, per-instance ingestion gates. |
| `plugin-pipeline-geo` | Resolves the visitor's IP to a location, from a local IP2Location database. |
| `plugin-pipeline-identity` | The identity graph. Merges anonymous and identified sessions into one Person via device ids and external identifiers. Union-find over MySQL. |
| `plugin-pipeline-trigger` | Evaluates the client's active trigger rules against the event and dispatches webhooks and communications. |
| `plugin-pipeline-scoring` | Person scores: RFM features plus churn, propensity and lifetime value. Incremental in real time, from scratch in batch. |
| `plugin-pipeline-catalog` | Builds the product and article catalogues from live traffic, so the platform is useful without a batch import. |
| `plugin-pipeline-gorse-output` | Forwards user/item interactions to the Gorse recommender. |
| `core-pipeline-output` | The terminal stage. Bulk-indexes events into `users-logs-<client>` in OpenSearch. |

### The product

| Repository | What it is |
|---|---|
| `core-api` | The backend. NestJS, ~43k lines, 38 controllers, 242 routes. Tenants, auth, segments, personalizations, triggers, communications, billing, GDPR, recommendations, the AI assistant. |
| `core-web-panel` | The customer-facing panel. Vue 3, Vuetify, Pinia. |
| `core-manager-panel` | The operator panel: provisioning tenants and running migrations. |
| `web-site` | The public marketing site. Nuxt 3, fully static, on Cloudflare Pages. |
| `core-os-mcp` | An MCP server over OpenSearch, so an assistant can query the data. |

### The shared pieces

| Repository | What it is |
|---|---|
| `core-types` | The shared contract: `Log`, `Event`, `Config`, queue names. Published as `@cdp-forge/types`. Everything on the event path depends on it. |
| `plugin-pipeline-sdk` | The base every stage is built on: Pulsar consumer and producer, retry and dead-letter policy, tenant connection pooling, config listener, health endpoint. Published as `@cdp-forge/plugin-pipeline-sdk`. |
| `helm` | The single chart that deploys all of the above, with Pulsar, OpenSearch, MySQL and Redis as subchart dependencies. |
| `.github` | Organisation profile, licence, contribution guide — and this document. |

---

## How an event travels

**1. Collection.** The tracker (or an e-commerce plugin) posts a batch of
events to `core-input`. Nothing is written to a database here: `core-input`
validates, applies the per-instance security rules and the consent gate, and
publishes onto Pulsar. Both checks are served from in-memory caches that
refresh in the background, because a database round trip per event does not
survive the ingestion rate.

**2. The stage chain.** Each stage subscribes to its own input topic and
publishes to its output topic. Which topic that is comes from
`core-pipeline-manager`, not from configuration: a stage starts, calls
`POST /register` with its name, type and priority, and is told where to read
and write. When the topology changes, the manager broadcasts it on the
`config` topic and every stage re-subscribes.

Priority is the order. `blocking` stages run in sequence and the next one sees
what the previous one added; `parallel` stages run concurrently off the same
point in the chain, because nothing downstream depends on them. Priority 0 is
reserved for `core-pipeline-stage`.

**3. Enrichment.** Each stage adds its own field to the event — `log.geo`,
`log.identity`, `log.scores` — or performs a side effect, like writing a
catalogue document or firing a webhook. The identity stage is the load-bearing
one: scoring, triggers and catalogue all skip an event that carries no
`log.identity`.

**4. Indexing.** `core-pipeline-output` buffers and bulk-indexes into
OpenSearch, one index per client. That index is what the panels' dashboards,
the segment engine and the AI assistant read.

**5. Reading.** `core-api` queries OpenSearch for behavioural data and MySQL
for configuration and tenant state.

---

## The contracts

Three things hold the platform together, and breaking any of them breaks
several repositories at once.

**`@cdp-forge/types`** is the event shape. A field added to `Log` is a field
every stage can read, and a field renamed is a field several stages silently
stop finding. Bump it deliberately; Dependabot propagates it.

**`@cdp-forge/plugin-pipeline-sdk`** is the stage runtime. A stage implements
`elaborate(log)` and the SDK does everything else: subscribing, acking,
nacking, dead-lettering, shutting down cleanly, serving health.

**The registration protocol** is how a stage joins the chain. It is guarded by
a shared internal secret (`INTERNAL_SERVICE_SECRET`), because those routes
rewrite the topology for every tenant — an unauthenticated caller could
register itself and be handed a copy of the whole event stream.

---

## Multi-tenancy

One MySQL schema per client. The tenant a request acts on comes from the
authenticated token, never from a parameter, and inside the pipeline it comes
from the event's `client` field. `TenantConnectionResolver` in the SDK turns a
client id into a connection to that client's schema, pooled and closed when
idle.

OpenSearch is separated by index name rather than by schema:
`users-logs-<client>`, `persons-<client>`, `products-<client>-<instance>-<lang>`.

The practical consequence: there is no global query. Anything that spans
tenants is a loop over tenants, and anything that looks like a cross-tenant
join is a bug.

---

## Where state lives

| Store | What it holds |
|---|---|
| **MySQL** | Configuration and tenant state: clients, instances, users, segments, triggers, subscriptions, the identity graph, the pipeline topology. |
| **OpenSearch** | Everything behavioural: the event log, person profiles, catalogues, materialised segment membership. |
| **Redis** | Hot state and coordination: PV quotas, scoring accumulators, dedup markers, trigger cooldowns, BullMQ queues. Everything here is reconstructible. |
| **Pulsar** | Events in flight, plus the dead-letter queues holding what could not be processed. |

---

## Adding a pipeline stage

1. Start from an existing stage — `plugin-pipeline-geo` is the smallest.
2. Depend on `@cdp-forge/plugin-pipeline-sdk` and `@cdp-forge/types`.
3. Implement `PipelinePluginI`: `init()`, `elaborate(log)`, optionally
   `close()` for a flush on shutdown.
4. Declare a name, a priority and a type (`blocking` or `parallel`) in
   `src/config/plugin.ts`.
5. Call `start(plugin, pluginConfig)` from `src/index.ts`. The SDK registers
   the stage, subscribes it, serves its health endpoints and handles shutdown.
6. Add a deployment to the Helm chart.

Two rules the SDK cannot enforce for you:

- **Never log the event body.** It carries personal data — IP, user agent,
  traits, form values. Log identifiers and counts.
- **Let a dependency being down throw.** The SDK nacks, redelivers and
  dead-letters, and that only works if your `elaborate` does not swallow the
  error. Swallow what is wrong with *this event*; rethrow what is wrong with
  the *dependency*. `rethrowIfInfrastructure(err)` draws that line. The
  exception is a stage that has already performed an irreversible side effect
  for this event, where a redelivery would repeat it.

---

## Deployment

One Helm chart in the `helm` repository deploys every first-party service,
with Pulsar, OpenSearch, MySQL and Redis as subchart dependencies. Each service
is a Deployment with its replica count in values; scaling one past a single
replica also gives it a PodDisruptionBudget.

Images are published to `ghcr.io/cdpforge/<repository>` and pinned by tag in
`values.yaml`.

Liveness and readiness probes exist for every first-party service, on `/health`
and `/ready`. They are off by default (`probes.enabled`) because the pinned
image tags may predate the endpoints — bump the tags, then turn them on.
