# GCP Deep Study Guide — Volume 4: Integration & Event-Driven Architecture

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** Pub/Sub, Eventarc, Cloud Tasks, Workflows, Cloud Scheduler, API Gateway / Apigee, and Dataflow (streaming intro).

> This is where "a bunch of services" becomes "a system." Decoupling + event-driven design is the heart of resilient, scalable architecture — and heavily probed in senior/staff interviews.

---

# 1. Pub/Sub

**🧠 Mental model:** A global, serverless messaging bus. Publishers send messages to a **topic**; each **subscription** delivers a copy to its consumers. It's both a work queue *and* a fan-out broadcaster, depending on how you wire subscriptions.

**❓ Why it exists:** Decouple producers from consumers at global scale — absorb spikes, let components fail independently, and fan events out to many consumers reliably.

**🔬 Granular concepts**

*Core model*
- **Topic** (where publishers send) → **Subscriptions** (each subscription = an independent stream of the topic's messages)
- **One subscription, many consumers** = a work queue (load-balanced across consumers)
- **Many subscriptions on one topic** = fan-out (each subscription gets every message) — this is how Pub/Sub does both patterns
- **Pull** (consumers ask for messages; StreamingPull) vs **Push** (Pub/Sub POSTs to an HTTPS endpoint / Cloud Run / Function)

*Delivery & reliability*
- **Ack / nack**; **ack deadline** (if not acked in time, redelivered — the duplicate cause; extend for long processing)
- **At-least-once** by default → consumers must be **idempotent**
- **Exactly-once delivery** (optional, per-subscription, pull only) — reduces duplicates within a window
- **Message ordering** (optional, via **ordering keys** — ordered per key within a region)
- **Dead-letter topics** + max delivery attempts; **retry policy** (exponential backoff)
- **Message retention** (default + configurable; can seek/replay to a timestamp or snapshot)
- **Schemas** (Avro/Protobuf) + schema enforcement on topics

*Scale & integration*
- Global topics; automatic scaling (no shards to manage — contrast with Kinesis)
- **Pub/Sub Lite** (cheaper, zonal, capacity-based — for very high-volume cost-sensitive cases)
- Native sinks: Cloud Storage / BigQuery **direct subscriptions** (no code), Dataflow
- Flow control (client-side) to avoid overwhelming consumers

**🎛️ Key knobs:** push vs pull, ack deadline, exactly-once toggle, ordering keys, dead-letter topic + max attempts, retry backoff, retention, schema, subscription filter.

**🚧 Limits & gotchas**
- **Ack deadline shorter than processing time** → redelivery → duplicate work (extend the deadline or use lease extension).
- At-least-once → **idempotency is mandatory** (even with exactly-once, design defensively).
- Ordering keys reduce throughput for that key + must be enabled at subscription create.
- Push subscriptions to an unauthenticated endpoint are a security hole — use OIDC-authenticated push to Cloud Run/Functions.
- Forgetting a dead-letter topic → poison messages retry forever.
- Pub/Sub vs Pub/Sub Lite are different products (Lite = capacity-provisioned, zonal).

**💸 Cost model:** per volume of message data (published + delivered) + retained storage + any Dataflow/BigQuery sink. No per-shard cost (unlike Kinesis). Lite is cheaper for sustained high volume but you manage capacity.

**💥 Failure modes:** duplicates → short ack deadline / no idempotency. Messages stuck → consumer down or piling into dead-letter. Out-of-order → ordering key not set. Push failing → endpoint auth/5xx.

**🚀 Projects**
- **A — Work queue + dead-letter:** Topic → one **pull** subscription → idempotent Cloud Run/worker; add a **dead-letter topic** + max attempts; force failures into it and replay.
- **B (different approach — fan-out + push):** One topic → two subscriptions → two independent consumers (one **push** to Cloud Run with OIDC auth); prove each gets every message and one consumer being down loses nothing.
- **C — ordering + exactly-once:** Enable **ordering keys** so per-customer events stay ordered; enable **exactly-once** on a pull subscription; measure the throughput tradeoff.

**💬 Interview questions**
- *Junior:* Topic vs subscription? How does Pub/Sub do *both* a work queue and fan-out?
- *Mid:* Explain ack deadline and how it causes duplicates. At-least-once vs exactly-once; why idempotency anyway? Push vs pull.
- *Senior:* "Process each message exactly once" — why hard, what do you do? How do ordering keys work and what do they cost? Pub/Sub vs Kinesis (no shards) — contrast.
- *Staff:* Design a durable ingestion layer for spiky 100k msg/sec feeding a fragile downstream — backpressure, dead-letters, poison handling, replay.

**✅ Mastery bar:** You design idempotent consumers, tune ack deadlines, use dead-letter topics by habit, and know when ordering/exactly-once/Lite apply.

---

# 2. Eventarc

**🧠 Mental model:** The event router that connects **GCP service events** (and custom events) to your handlers — the standardized "when X happens anywhere in GCP, run Y" layer.

**❓ Why it exists:** Uniform, CloudEvents-based routing from ~any GCP source (via Audit Logs or direct) to targets like Cloud Run/Functions/Workflows — without wiring each integration by hand.

**🔬 Granular concepts**
- Sources: direct events (Pub/Sub, Cloud Storage), and **Cloud Audit Logs** events (huge coverage — almost any GCP admin action becomes an event)
- Targets: Cloud Run, Cloud Functions (Gen 2), Workflows, GKE
- **Triggers** with event filters (event type, resource, attributes)
- **CloudEvents** format (standardized event schema); delivered via Pub/Sub under the hood
- Channels (for third-party/partner event sources)
- Runs as a service account; ordering + retry via the underlying Pub/Sub

**🚧 Gotchas:** Audit-Log-based triggers require the relevant **Data Access audit logs enabled** (off by default) → trigger silently never fires. Filters too broad/narrow. Latency includes the audit-log path.

**💸 Cost:** based on the underlying Pub/Sub delivery + event volume.

**🚀 Projects**
- **A:** A GCS finalize event → **Eventarc** trigger → Cloud Run processes the object (CloudEvents payload).
- **B (different approach — audit-log trigger):** Trigger a function whenever someone creates a resource (via a Cloud Audit Log event) — e.g., notify on new firewall rules; requires enabling the right audit logs.

**💬 Interview:** What's Eventarc for, and how is it different from wiring Pub/Sub directly? Why might an audit-log trigger not fire (Data Access logs off)? What is CloudEvents?

**✅ Mastery bar:** You can route any GCP event to a handler and know the audit-log-enablement dependency.

---

# 3. Cloud Tasks

**🧠 Mental model:** A managed **task queue for deferred/rate-limited HTTP work** — enqueue a task now, and Cloud Tasks delivers it to your handler later, with per-queue rate limits, retries, and scheduling. (Contrast Pub/Sub: Tasks gives you *explicit per-task control* + dispatch rate control.)

**🔬 Granular concepts**
- Queues with **rate limiting** (max dispatches/sec) + **concurrency** control — protect a fragile backend
- HTTP targets (any endpoint / Cloud Run / Functions) with OIDC/OAuth auth
- **Scheduled** tasks (dispatch at a future time); **task deduplication** (task name); retries with backoff
- Explicit task creation (you control each task) vs Pub/Sub's stream model

**🚧 Gotchas:** Tasks vs Pub/Sub confusion — Tasks = you want to *throttle* and control individual task delivery to an HTTP endpoint; Pub/Sub = event streaming/fan-out. Handler must be idempotent (retries). Auth on the target endpoint.

**💸 Cost:** per operation (task creation/dispatch). No standing cost.

**🚀 Project:** Rate-limit calls to a fragile downstream: enqueue work to a **Cloud Tasks** queue with a max-dispatches/sec cap; watch it smooth a burst into a steady trickle; add a scheduled task.

**💬 Interview:** Cloud Tasks vs Pub/Sub — the deciding factor (explicit per-task + rate control vs streaming/fan-out). How do you protect a rate-limited backend?

**✅ Mastery bar:** You reach for Cloud Tasks when you need dispatch-rate control to an HTTP endpoint, and Pub/Sub when you need streaming/fan-out.

---

# 4. Workflows

**🧠 Mental model:** A serverless **orchestrator** — define a sequence/branch/parallel flow in YAML/JSON that calls services (any HTTP/GCP API), handles retries + errors, and holds state across steps. (GCP's Step Functions equivalent.)

**🔬 Granular concepts**
- Steps: assignments, calls (HTTP/connectors), **switch** (branch), **parallel**, **for** loops, `try/retry/except` (error handling with backoff)
- **Connectors** (call GCP APIs without boilerplate); call Cloud Run/Functions/any HTTP
- **Callbacks** (pause until an external event/human approval resumes)
- Long-running (up to ~1 year); execution history for debugging
- Triggered by Scheduler, Eventarc, Pub/Sub, or directly

**🚧 Gotchas:** Workflows orchestrates but isn't a high-volume stream processor (that's Dataflow). Over-orchestrating simple flows adds complexity (sometimes Pub/Sub + a worker is enough). State/payload size limits (pass references).

**💸 Cost:** per step executed (internal steps cheap; external API calls billed separately) — very cheap for typical orchestration.

**🚀 Projects**
- **A — Orchestrated pipeline (Saga):** Order flow: validate → (parallel) charge + reserve inventory → fulfill → notify, with **retry/except** and a **compensating refund** step on failure.
- **B (different approach — human approval):** Add a **callback** step that pauses until a manual approval resumes it.

**💬 Interview:** Orchestration (Workflows) vs choreography (Pub/Sub/Eventarc chains) — tradeoffs. Explain the Saga pattern with compensation. Workflows vs Dataflow — when each?

**✅ Mastery bar:** You choose orchestration vs choreography deliberately and build a workflow with real error handling + compensation + human-in-the-loop.

---

# 5. Cloud Scheduler

**🧠 Mental model:** Managed cron — reliably trigger HTTP endpoints, Pub/Sub, or Workflows on a schedule.

**🔬 Granular concepts**
- Cron syntax + timezones; targets: HTTP (with OIDC/OAuth auth), Pub/Sub, App Engine, Workflows
- Retries + backoff; at-least-once trigger semantics (make targets idempotent)
- Pairs with Cloud Run Jobs / Workflows / Functions for scheduled batch

**🚧 Gotchas:** timezone/cron mistakes; unauthenticated HTTP targets; assuming exactly-once (it's at-least-once → idempotent handlers).

**💸 Cost:** per job/month (a few free jobs historically — verify).

**🚀 Project:** Schedule a nightly **Cloud Run Job** (or Workflow) via Cloud Scheduler with OIDC auth; verify retries on failure.

**💬 Interview:** How do you run scheduled work on GCP? Why must scheduled targets be idempotent?

---

# 6. API Gateway + Apigee

**🧠 Mental model:** The managed front door for your APIs. **API Gateway** = lightweight, serverless, cheap (auth/keys/routing to Cloud Run/Functions). **Apigee** = full enterprise API management (monetization, advanced policies, developer portal, analytics).

**🔬 Granular concepts**

*API Gateway*
- OpenAPI-defined config; routes to Cloud Run/Functions/App Engine/HTTP backends
- Auth: API keys, JWT (Firebase/Auth0/Google ID tokens), service-account auth to backends
- Managed, serverless, per-request billing; basic rate limiting/quotas

*Apigee*
- Full API management platform: **policies** (transformation, mediation, threat protection, spike arrest/quota), **developer portal**, **API products** + keys, **monetization**, deep analytics
- Proxies (fronting backends anywhere — GCP/other-cloud/on-prem); environments; deployment
- For orgs running APIs as a product / many external consumers

*General*
- The 30-ish-second-ish backend timeout considerations (long work → async: return a job ID, poll/notify)
- CORS; versioning via config revisions

**🚧 Gotchas:** API keys ≠ authentication (metering only) — use JWT/service-account auth for real authZ. Choosing Apigee (heavy/enterprise) when API Gateway suffices = big overspend; choosing API Gateway when you need Apigee's policy/monetization = a rebuild. Long backends need async patterns.

**💸 Cost:** API Gateway = per call (cheap). Apigee = significant platform pricing (enterprise).

**🚀 Projects**
- **A — Serverless API front:** OpenAPI-defined **API Gateway** → Cloud Run backend with JWT auth + a quota.
- **B (different approach — long job):** API returns a job ID immediately, kicks off **Workflows**, and notifies on completion (dodging the backend timeout).

**💬 Interview:** API Gateway vs Apigee — when each? Why aren't API keys authentication? How do you handle a backend slower than the gateway timeout?

**✅ Mastery bar:** You pick the right API front door, wire real auth + quotas, and design around timeouts for long operations.

---

# 7. Dataflow (streaming intro; deep in Vol 7)

**🧠 Mental model:** Serverless **Apache Beam** for unified batch + stream processing — the engine that transforms/aggregates Pub/Sub streams in real time (GCP's Kinesis-Data-Analytics/Flink equivalent).

**🔬 Granular concepts**
- Apache Beam model (unified batch + streaming); PCollections, transforms, pipelines
- **Windowing** (fixed/sliding/session) + **watermarks** + triggers (handle late/out-of-order data — **event time** vs processing time)
- Sources/sinks: Pub/Sub → Dataflow → BigQuery/GCS/Bigtable
- **Autoscaling** + dynamic work rebalancing (Dataflow's differentiator); streaming engine
- Templates (Google-provided + custom/Flex) for common pipelines with no code
- Exactly-once processing semantics within the pipeline

**🚧 Gotchas:** stream processing is conceptually hard (state, event-time, late data). Cost of always-on streaming vs batching when real-time isn't truly needed. Pipeline sizing/parallelism.

**💸 Cost:** per vCPU/memory/streaming-engine usage while running (autoscaled).

**🚀 Project:** A Dataflow streaming pipeline: Pub/Sub → 1-minute tumbling-window aggregation → BigQuery, handling late data with watermarks. (Use a Google-provided template first, then a custom pipeline.)

**💬 Interview:** Batch vs stream — when do you truly need real-time? Tumbling vs sliding windows. Event time vs processing time and why late data is hard. Pub/Sub + Dataflow vs Kinesis — contrast.

**✅ Mastery bar:** You can decide when streaming is justified and reason about windowing + event-time correctness.

---

# 🏔️ Volume 4 Capstone — "Event-driven order system"
Build a realistic decoupled system and whiteboard it:
- **API Gateway** (JWT auth + quota) accepts orders, returns a job ID immediately.
- Order event published to **Pub/Sub** → routed to multiple subscriptions (fan-out) + **Eventarc** for GCP-service reactions.
- **Cloud Tasks** rate-limits calls to a fragile downstream; workers (Cloud Run) are **idempotent** + dead-letter-protected.
- **Workflows** orchestrates fulfillment (validate → charge → reserve → ship) with retries + a **compensating refund** (Saga) + a human-approval **callback**.
- **Cloud Scheduler** drives nightly reconciliation jobs.
- **Dataflow** streams every event from Pub/Sub into **BigQuery** for analytics.
- All in Terraform (Vol 3), observable (Vol 1 Cloud Operations), keyless least-privilege SAs (Vol 1).

Then write up: where ordering matters (ordering keys), orchestration vs choreography choices, poison-message handling (dead-letter topics), and behavior under a 10x spike.

## Decision cheat-sheet (memorize)
- **Pub/Sub:** decouple + buffer + fan-out streaming; at-least-once → be idempotent; the messaging backbone.
- **Cloud Tasks:** explicit per-task deferral + **dispatch-rate control** to HTTP endpoints (protect a fragile backend).
- **Eventarc:** route GCP service/audit-log events to handlers (CloudEvents).
- **Workflows:** orchestrate multi-step processes with retries/compensation/human approval.
- **Cloud Scheduler:** cron triggers.
- **Dataflow:** high-volume stream/batch processing (windowing, event-time).
- **API Gateway vs Apigee:** lightweight serverless front door vs enterprise API management.

## AWS→GCP map
SQS→**Pub/Sub subscription (or Cloud Tasks)**; SNS→**Pub/Sub fan-out**; EventBridge→**Eventarc**; Step Functions→**Workflows**; Kinesis→**Pub/Sub + Dataflow**; API Gateway→**API Gateway/Apigee**; (rate-limited task queue)→**Cloud Tasks** (no direct AWS analog).

## Cadence
~1–2 weeks per service; capstone is a 2–3 week push. ~7–9 weeks at 5–10 hrs/week.
