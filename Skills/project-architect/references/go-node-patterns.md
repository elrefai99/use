# Go↔Node distributed-systems playbook

Reusable reasoning for the recurring decisions in a polyglot Go + Node.js backend. Adapt to the specifics the user gives you — don't paste these verbatim, use them as the grounded starting point for the why/problem/when/tradeoffs/alternatives breakdown.

## 1. Communication protocol: REST vs gRPC vs async messaging

**REST (HTTP/JSON)**
- Why: ubiquitous tooling, human-readable, trivial to debug with curl/Postman, no shared codegen step.
- Problem it solves: fastest path to a working integration between two services owned by different teams/languages with no coordination overhead.
- When useful: low-to-medium call volume, external-facing or partner-facing APIs, endpoints that change shape often during early development.
- Drawbacks: JSON (de)serialization overhead at scale, no compile-time contract enforcement between Go and Node (schema drift only caught at runtime), chattier over the wire, weaker streaming support.
- Alternatives: gRPC when the contract is internal and stable; async messaging when the caller shouldn't block on the callee.

**gRPC (protobuf)**
- Why: compile-time-checked contracts shared across Go and Node via generated stubs, binary encoding, native streaming.
- Problem it solves: schema drift between services in different languages, and the latency/CPU cost of JSON at high call volume.
- When useful: internal service-to-service calls with high volume or low-latency requirements, especially once the interface has stabilized. Go's gRPC support is first-class; Node's is workable but has more friction (codegen tooling, less idiomatic streaming ergonomics) — worth naming as a real cost, not glossing over.
- Drawbacks: codegen step in the build pipeline for both languages, harder to debug ad hoc (no curl), a proto schema to version and coordinate, less natural fit if the endpoint is also consumed by a browser client.
- Alternatives: REST for anything externally consumed; REST also wins short-term if the team doesn't already have protobuf tooling and the call volume doesn't yet justify the investment.

**Async messaging (BullMQ/Redis, SQS, Kafka)**
- Why: decouples the caller from the callee's availability and latency; the producer doesn't block on the consumer's processing time.
- Problem it solves: synchronous coupling — if Go's fraud-scoring service is slow or down, a REST/gRPC call from Node blocks or fails the user-facing request. A queue absorbs that.
- When useful: fire-and-forget or eventually-consistent work (notifications, fraud scoring, image processing, webhooks fan-out), anything where retries/backoff matter, anything where the producer and consumer scale independently.
- Drawbacks: eventual rather than immediate consistency (the caller often needs a separate way to observe the result — polling, websocket push, or a callback), operational surface of another broker if not already using one, harder to reason about end-to-end request tracing without deliberate correlation IDs.
- Alternatives: sync REST/gRPC when the caller genuinely needs the result before responding to its own caller (e.g. an auth check).

## 2. Data ownership across services

**Database-per-service**
- Why: each service (Go or Node) owns its schema and evolves it without coordinating migrations with the other language's ORM/model layer.
- Problem it solves: the two-languages-one-schema trap — Prisma/Mongoose models on the Node side and a Go ORM/struct layer on the other drifting out of sync against a shared table.
- When useful: as soon as a Go service exists as a genuinely separate deployable with its own lifecycle, not just a library called in-process.
- Drawbacks: no cheap cross-service JOINs, need an explicit mechanism (API call, event, or read replica) to get data owned by the other service, eventual consistency between the two datasets.
- Alternatives: shared DB is sometimes pragmatic short-term for a small team, but name the coupling cost explicitly — it re-creates the schema-drift problem this pattern exists to avoid.

**Shared DB**
- Why: no cross-service data-fetching mechanism needed, one source of truth, simplest to query for reporting.
- Problem it solves: avoids building event/read-replica plumbing when the team is small and the services are tightly related.
- When useful: early-stage, same team owns both sides, no independent scaling/deploy cadence needed yet.
- Drawbacks: exactly the coupling database-per-service avoids — a Go struct and a Prisma/Mongoose model both reading the same table will drift, migrations need coordinating across both codebases, defeats independent deploys.
- Alternatives: database-per-service once either side needs to scale, deploy, or evolve its schema independently.

**CDC-based read replication (Debezium, or DB-native logical replication)**
- Why: lets one service keep a local, queryable copy of data it doesn't own, without synchronous calls to fetch it.
- Problem it solves: "Go needs to filter/join on data Node owns, but a network call per query is too slow and a shared DB isn't acceptable."
- When useful: read-heavy cross-service access patterns, reporting/analytics needs, when eventual consistency (seconds, not milliseconds) is acceptable.
- Drawbacks: real operational weight (CDC pipeline to run and monitor), replica lag as a source of bugs if the consumer assumes freshness, another moving part to explain to the team.
- Alternatives: a simple "get by ID" API call is often good enough if the access pattern is sparse; reach for CDC only when call volume or join complexity makes that impractical.

## 3. Consistency across services

**Outbox pattern**
- Why: guarantees a DB write and the event announcing it either both happen or neither does, without a distributed transaction.
- Problem it solves: the classic "wrote to Postgres, then crashed before publishing to the queue" — inconsistent state between the source of truth and everything downstream of it (Go service included).
- When useful: any time a service needs to reliably publish an event after a local write — order placed, payment captured, etc., especially when the consumer is a different language/service.
- Drawbacks: extra table and a relay process (polling or CDC) to actually publish from the outbox, added latency between write and publish.
- Alternatives: 2PC (avoid — poor fit across heterogeneous stores/languages, blocks on the slowest participant, most modern brokers don't support it well); "just publish after commit and hope" (avoid — the exact failure mode outbox exists to prevent).

**Sagas**
- Why: coordinates a multi-step transaction across services (some Go, some Node) via a sequence of local transactions plus compensating actions, instead of a distributed lock.
- Problem it solves: multi-service workflows (e.g. reserve inventory in one service, charge payment in another, notify a third) where a true ACID transaction across languages/datastores isn't available.
- When useful: any cross-service workflow with multiple steps that can fail partway through and needs defined rollback behavior.
- Drawbacks: compensating actions have to be designed for every step (not always possible — you can't "un-send" an email, only send a correction), orchestration vs choreography adds its own complexity choice, harder to reason about than a single transaction.
- Alternatives: keep the workflow inside one service/datastore if at all possible — sagas are a cost paid only because the workflow genuinely spans services.

**Idempotency keys**
- Why: makes retried requests/messages safe to process more than once — essential once you have async messaging or gRPC retries between Go and Node.
- Problem it solves: at-least-once delivery (the default for most queues and retry policies) turning into double-charges, duplicate records, or double-sends without it.
- When useful: anywhere a request or message might legitimately be retried — webhook receivers, queue consumers, payment calls.
- Drawbacks: requires a dedupe store (even a short-TTL Redis key) and discipline about what "the same request" means for a given endpoint.
- Alternatives: none, really — treat this as close to mandatory for async or retried paths rather than a tradeoff to weigh.

## 4. Cross-language contract stability

- **Schema sharing**: protobuf (if using gRPC) or a generated OpenAPI client (if REST) checked into a shared location both the Go and Node repos pull from, so a schema change surfaces as a build failure in both languages rather than a runtime surprise in one.
- **Auth token propagation**: the user issues PASETO v4.local tokens on the Node side; a Go consumer needs a PASETO v4.local library (exists, less mainstream than the JS ecosystem's) and the same shared symmetric key management — call out key rotation/distribution as the actual hard part, not the token format itself.
- **Distributed tracing**: OpenTelemetry has both Go and Node SDKs and is the natural choice for propagating trace context across a language boundary — without it, debugging a request that crosses Node → queue → Go means correlating logs by hand via a manually-threaded request/correlation ID at minimum.
