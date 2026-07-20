# GCP Deep Study Guide — Volume 7: Data & Analytics

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** the lakehouse model, BigQuery, Dataproc, Dataflow, Dataplex, Pub/Sub ingestion, Looker / Looker Studio, Cloud Composer, BigQuery ML.

> This is GCP's strongest area, and **BigQuery is the crown jewel** — arguably the single most differentiating service on the platform. You don't need to be a data engineer for Architect/DevOps roles, but knowing where each piece fits (and BigQuery deeply) separates strong architects from name-droppers. Builds on Cloud Storage (Vol 1) + Pub/Sub (Vol 4).

---

# 0. The mental model: lake, warehouse, lakehouse

**🧠 Anchor this first.**
- **Data lake** = raw + processed data of any shape, cheap, in **Cloud Storage**, queried in place (schema-on-read).
- **Data warehouse** = **BigQuery** — serverless, columnar, massively parallel SQL over huge structured datasets (schema-on-write-ish, but very flexible).
- **Lakehouse** = GCS as storage + query with many engines (BigQuery, Dataproc/Spark) + a central catalog/governance layer (**Dataplex/Data Catalog**).
- **ETL vs ELT:** transform-then-load vs load-raw-then-transform (BigQuery's power makes ELT very common — load raw, transform in SQL).

**Cross-cutting ideas that drive cost + performance everywhere:**
- **Columnar formats** (Parquet/ORC/Avro) and BigQuery's native columnar storage >> row formats for analytics.
- **Partitioning + clustering** in BigQuery (and partitioned files in GCS) let engines scan far less data → cheaper + faster.
- **Compression** + right file sizes (avoid the "millions of tiny files" problem).
- **BigQuery bills primarily on bytes scanned** (on-demand) — so *how you query and lay out data* is your bill.

---

# 1. BigQuery (the crown jewel)

**🧠 Mental model:** A serverless, planet-scale data warehouse where storage and compute are separate — you just run SQL and Google throws a massive parallel engine (Dremel) at it. No clusters to size. You mostly pay for **bytes scanned** (or reserved slots).

**❓ Why it exists:** Fast, serverless analytics over terabytes–petabytes of structured data with plain SQL, plus built-in ML, streaming, geospatial, and multi-cloud querying.

**🔬 Granular concepts**

*Architecture*
- **Separation of storage + compute** (columnar storage; **Dremel** query engine; **slots** = units of compute; the Jupiter network + Colossus storage make this work)
- Datasets → tables/views; regions/multi-region location
- Storage: native columnar tables vs **external/BigLake tables** (query GCS/Parquet/other in place)

*Performance + cost levers (the whole game)*
- **Partitioning** (by ingestion time / date/timestamp column / integer range) → prune partitions → scan less
- **Clustering** (sort within partitions by columns) → skip blocks → scan less
- **Bytes scanned** determines on-demand cost → `SELECT` only needed columns (never `SELECT *` on wide tables), filter on partition/cluster columns
- **Materialized views** (precomputed, auto-maintained); **BI Engine** (in-memory acceleration for dashboards)
- Query plan + slot analysis; caching (free re-runs of identical queries)

*Pricing models*
- **On-demand** (per TB scanned) vs **capacity/Editions** (Standard/Enterprise/Enterprise Plus — buy slots, autoscaling reservations) — pick based on workload predictability + volume
- Storage pricing: active vs **long-term** (auto-discount for untouched data); physical vs logical storage billing

*Beyond basic SQL*
- **Streaming inserts** / the Storage Write API (real-time ingestion) + Pub/Sub → BigQuery direct subscriptions
- **BigQuery ML** (train/predict with SQL — regression, classification, forecasting, even LLM calls)
- **BigQuery Omni** (query data in AWS S3 / Azure from BigQuery — multi-cloud)
- **BigLake** (unified access + fine-grained governance over lake data)
- Federated queries (Cloud SQL/Spanner); **Data Transfer Service** (SaaS/warehouse ingestion)
- Authorized views / column-level + row-level security; scheduled queries

**🎛️ Key knobs:** partition column, clustering columns, on-demand vs slot reservations + autoscaling, materialized views, BI Engine, table expiration, query cost controls (max bytes billed), location.

**🚧 Limits & gotchas**
- **`SELECT *` on a big table = full column scan = big bill.** Select needed columns; filter on the partition column.
- **Not partitioning/clustering** = every query scans everything.
- BigQuery is **analytical (OLAP), not OLTP** — no high-frequency small row updates/point lookups (use Cloud SQL/Spanner/Bigtable for that).
- Streaming inserts cost more + have their own quotas (Storage Write API is the modern path).
- On-demand vs slots: heavy predictable workloads waste money on-demand; spiky small ones waste money on idle slots. Match the model.
- Multi-region vs region for residency + cost.

**💸 Cost model:** on-demand = per TB scanned (+ storage active/long-term); capacity = slot reservations (per slot-hour, autoscaling). Storage is cheap; **scanning is the cost** on-demand → partitioning/clustering/column-pruning are your bill control. Set **max-bytes-billed** guardrails.

**💥 Failure modes:** shocking query bill → `SELECT *` / no partition filter / no clustering. Slow dashboards → no BI Engine/materialized views. "Quota exceeded" → streaming/slot limits. Wrong tool → using BigQuery for point lookups.

**🚀 Projects**
- **A — Layout drives cost:** Load a large dataset; create a **partitioned + clustered** table; run the same query against a naive table vs the optimized one and **measure bytes scanned + cost difference** (often 10–100x).
- **B (different approach — lakehouse/BigLake):** Query Parquet in GCS via an **external/BigLake table** (no load); join it with a native table; apply **column-level security** to mask a PII column.
- **C — SQL ML + streaming:** Stream events (Pub/Sub → BigQuery) and train a **BigQuery ML** model (e.g., forecasting) with a few SQL statements; predict on new data.

**💬 Interview questions**
- *Junior:* Why is BigQuery serverless, and what do you mostly pay for?
- *Mid:* How do partitioning + clustering reduce cost? Why avoid `SELECT *`? On-demand vs slot pricing — when each?
- *Senior:* A query bill exploded — diagnose + fix. Why isn't BigQuery an OLTP database? What are materialized views/BI Engine for?
- *Staff:* Design a petabyte analytics platform: partitioning/clustering strategy, pricing model choice, multi-region/residency, governance, and cost guardrails across many teams.

**✅ Mastery bar:** You design partitioned/clustered tables, control cost via column-pruning + guardrails, pick on-demand vs slots correctly, and know BigQuery's OLAP niche.

---

# 2. Dataproc

**🧠 Mental model:** Managed **Spark / Hadoop** — spin up open-source big-data clusters on GCP for heavy or custom processing (the EMR equivalent).

**🔬 Granular concepts**
- Managed Spark/Hadoop/Hive/Presto/Flink; fast cluster provisioning
- **Ephemeral (job-scoped) clusters** (create → run → delete — cost-efficient) vs long-running
- **Dataproc Serverless** (submit Spark jobs with no cluster to manage — the modern default for Spark)
- Reads/writes **GCS** (via the connector) rather than relying on HDFS for durability
- **Spot/preemptible** secondary workers for big savings on tolerant jobs
- Autoscaling policies; initialization actions; workflow templates
- **Dataproc on GKE**; when to use Dataproc vs Dataflow vs BigQuery

**🚧 Gotchas:** leaving long-running clusters idle = burning money (prefer ephemeral / Serverless). Over-relying on HDFS instead of GCS loses durability. Cluster sizing skill. For most new pipelines, Dataflow (Beam) or BigQuery SQL is less ops than managing Spark.

**💸 Cost:** VM compute (heavy Spot savings on secondary workers) + a small Dataproc premium; Serverless bills per resource consumed.

**🚀 Project:** Run a **Spark job on Dataproc Serverless** (or an ephemeral cluster) reading from GCS, transforming, writing **partitioned Parquet** back; use Spot secondary workers + auto-delete.

**💬 Interview:** Dataproc vs Dataflow vs BigQuery — when do you actually need Spark? Ephemeral vs long-running clusters + cost. Why write to GCS not HDFS? Why Serverless?

**✅ Mastery bar:** You know when a workload needs Spark/Dataproc vs BigQuery/Dataflow, and run it cheaply (Serverless/ephemeral + Spot + GCS).

---

# 3. Dataflow

**🧠 Mental model:** Serverless **Apache Beam** for unified batch **and** streaming — the transform engine that autoscales and handles event-time correctness (the Kinesis-Data-Analytics/Flink + Glue-ETL overlap).

**🔬 Granular concepts**
- Apache Beam unified model (same code for batch + stream); PCollections, ParDo, GroupByKey, windowing
- **Windowing** (fixed/sliding/session) + **watermarks** + triggers → **event time vs processing time**, late/out-of-order data
- Sources/sinks: **Pub/Sub → Dataflow → BigQuery/GCS/Bigtable**
- **Autoscaling + dynamic work rebalancing** (Dataflow's differentiators — no manual shard/worker tuning); Streaming Engine
- Exactly-once processing within the pipeline
- **Templates** (Google-provided + custom **Flex** templates) — run common pipelines with no code
- Dataflow SQL; Dataflow Prime

**🚧 Gotchas:** stream processing is conceptually hard (state, event-time, late data). Always-on streaming cost vs batching when real-time isn't truly needed. Understanding when to use Dataflow (complex transforms/streaming) vs just loading into BigQuery and transforming in SQL (ELT).

**💸 Cost:** per vCPU/memory/Streaming-Engine usage while running (autoscaled). Batch is cheaper than always-on streaming.

**🚀 Projects**
- **A — Streaming aggregation:** Pub/Sub → Dataflow → **1-minute tumbling-window** counts → BigQuery, handling late data with watermarks (start from a Google-provided template, then a custom pipeline).
- **B (different approach — batch ETL):** A batch Dataflow job transforming GCS data → partitioned Parquet / BigQuery; compare to doing the same as BigQuery SQL (ELT) and discuss the tradeoff.

**💬 Interview:** Batch vs stream — when do you truly need real-time? Tumbling vs sliding windows; event vs processing time; why late data is hard. Dataflow (transform) vs BigQuery (load + SQL transform) — when each? What makes Dataflow autoscaling special?

**✅ Mastery bar:** You decide when streaming/Beam is justified vs BigQuery ELT, and reason about windowing + event-time correctness.

---

# 4. Dataplex (+ Data Catalog)

**🧠 Mental model:** The **governance + organization layer** over your lake — logically group + manage data across GCS + BigQuery, with a central catalog, discovery, quality, and fine-grained access (the Lake Formation analog).

**🔬 Granular concepts**
- **Lakes → zones → assets** (logically organize data across GCS buckets + BigQuery datasets regardless of physical location)
- **Data Catalog** (now part of Dataplex): searchable metadata, tags, tag templates, auto-discovery of schemas
- **Data quality** + **data profiling** scans; data lineage
- **Attribute-based access control** / fine-grained governance across the lake
- Auto-discovery (crawl GCS → register tables queryable by BigQuery/Dataproc)
- Integrates with BigLake for unified fine-grained access on lake data

**🚧 Gotchas:** governance value only shows once you actually organize + tag + apply policies (setup effort). Overlap/interaction between BigLake, BigQuery, and Dataplex permissions can confuse access debugging. It governs cataloged/lake access, not arbitrary direct GCS reads unless enforced.

**💸 Cost:** discovery/quality tasks + catalog usage (modest); underlying storage/query billed separately.

**🚀 Project:** Organize a lake in **Dataplex** (a lake with raw + curated zones over GCS + BigQuery); auto-discover schemas into the catalog; run a **data quality** scan; apply a tag template + fine-grained access so an analyst sees a table but a **PII column is masked**.

**💬 Interview:** What problem does Dataplex/Data Catalog solve over raw bucket/dataset IAM? How do you give analysts column-level least privilege across a lake? What is data lineage and why does it matter?

**✅ Mastery bar:** You can catalog + govern a multi-store lake with fine-grained access + quality, and reason about the layered permission model.

---

# 5. Pub/Sub for ingestion (recap) + Datastream

**🧠 Mental model:** **Pub/Sub** (Vol 4) is the real-time ingestion front door into analytics; **Datastream** is managed **change data capture** (CDC) that streams database changes into the lake/warehouse.

**🔬 Granular concepts**
- Pub/Sub → **BigQuery direct subscription** (no code) or → Dataflow → BigQuery; → GCS for raw landing
- **Datastream:** serverless CDC from Cloud SQL/MySQL/Postgres/Oracle → BigQuery/GCS (near-real-time replication of operational data into analytics without heavy ETL)
- Schema handling + backfill + ongoing change streams

**🚧 Gotchas:** direct Pub/Sub→BigQuery is great but less flexible than routing through Dataflow (no complex transforms). CDC schema drift handling. Cost of continuous replication.

**🚀 Project:** Use **Datastream** to replicate a Cloud SQL table into BigQuery (CDC), then query the near-real-time copy for analytics — no custom ETL.

**💬 Interview:** How do you get operational DB data into BigQuery for analytics with minimal ETL (CDC/Datastream)? Pub/Sub→BigQuery direct vs via Dataflow — the tradeoff.

**✅ Mastery bar:** You can land streaming + operational data into the warehouse via the right path (direct, Dataflow, or CDC).

---

# 6. Looker + Looker Studio (BI)

**🧠 Mental model:** The visualization/BI layer. **Looker** = governed, modeled enterprise BI (a semantic layer via LookML). **Looker Studio** = free, self-serve dashboards (the QuickSight-lite analog).

**🔬 Granular concepts**
- **Looker:** **LookML** semantic modeling layer (define metrics/dimensions once → consistent governed definitions), Explores, embedded analytics, row-level security, git-based model versioning, API; queries the warehouse live (in-database)
- **Looker Studio:** free, drag-and-drop dashboards on BigQuery/Sheets/many connectors; extract vs live; good for self-serve/ad-hoc
- **BI Engine** accelerates BigQuery-backed dashboards (in-memory)
- Row-level security + data governance in the BI layer

**🚧 Gotchas:** Looker (modeled, governed, licensed, enterprise) vs Looker Studio (free, ungoverned, self-serve) are different tools — don't conflate. Live queries can hammer BigQuery (cost) → BI Engine / materialized views / extracts. LookML modeling is a real skill/investment.

**💸 Cost:** Looker = platform/user licensing (enterprise). Looker Studio = free (Pro tier exists). Underlying BigQuery queries billed.

**🚀 Project:** Build a **Looker Studio** dashboard on the BigQuery dataset from earlier projects with a date filter + drill-down; enable **BI Engine** and compare responsiveness/cost; (conceptually) sketch the LookML model you'd build for governed metrics in Looker.

**💬 Interview:** Looker vs Looker Studio — when each? What does a semantic layer (LookML) give you? How do you keep BI dashboards from blowing up your BigQuery bill (BI Engine/materialized views)?

**✅ Mastery bar:** You can close the loop raw → BigQuery → dashboard, choose Looker vs Looker Studio, and control BI query cost.

---

# 7. Cloud Composer (orchestration)

**🧠 Mental model:** Managed **Apache Airflow** — author data pipelines as DAGs in Python, scheduled + monitored, orchestrating across BigQuery/Dataflow/Dataproc/GCS/external systems.

**🔬 Granular concepts**
- DAGs (Python), operators (BigQuery/Dataflow/Dataproc/GCS/HTTP/etc.), sensors, scheduling, backfills, retries
- Managed Airflow environment (GKE-backed); environment sizing
- When Composer (complex, multi-system, dependency-heavy pipelines) vs **Workflows** (Vol 4, lighter serverless orchestration) vs BigQuery **scheduled queries** (simple)
- Data lineage + monitoring integration

**🚧 Gotchas:** Composer runs a standing environment (not serverless → real cost even when idle) — for light needs, Workflows/scheduled queries are cheaper. Airflow has a learning curve + operational surface. Right-size the environment.

**💸 Cost:** the standing Composer environment (GKE + Airflow) — meaningful; only worth it for real pipeline complexity.

**🚀 Project:** A Composer DAG orchestrating: Dataflow/Dataproc transform → load to BigQuery → run a validation query → refresh a materialized view, with retries + a failure alert.

**💬 Interview:** Cloud Composer vs Workflows vs scheduled queries — deciding factors (complexity, cost, standing vs serverless). Why does Composer cost even when idle?

**✅ Mastery bar:** You choose Composer only when pipeline complexity justifies a standing Airflow, and orchestrate multi-service data flows.

---

# 🏔️ Volume 7 Capstone — "End-to-end GCP lakehouse"
Build the full pipeline and whiteboard the flow + cost drivers:
1. **Ingest:** stream events via **Pub/Sub**; land raw in **GCS** (partitioned) and near-real-time into BigQuery (direct subscription); replicate an operational DB via **Datastream** (CDC).
2. **Transform:** **Dataflow** for streaming aggregates + **Dataproc Serverless** for a heavy Spark batch → partitioned Parquet; and/or **BigQuery SQL (ELT)** for in-warehouse transforms.
3. **Warehouse:** **BigQuery** with **partitioned + clustered** tables; demonstrate a **measured bytes-scanned/cost comparison** (naive vs optimized); a **BigQuery ML** forecast; a **BigLake/external** table over GCS.
4. **Govern:** **Dataplex** lake with zones + catalog + a **data quality** scan + **column-level masking**.
5. **Visualize:** **Looker Studio** dashboard (+ **BI Engine**) with a date drill-down.
6. **Orchestrate:** **Cloud Composer** (or Workflows) DAG tying transform → load → validate → refresh with retries + alerting.

Deliver a data-architecture doc: lake→warehouse boundaries, why partitioning/clustering + column-pruning matter (with real numbers), where each engine fits, the governance model, on-demand-vs-slots pricing decision, and the top cost drivers + controls.

## Decision cheat-sheet (memorize)
- **GCS + BigQuery:** the default lakehouse — cheap lake storage + serverless SQL warehouse. Start here.
- **Dataproc:** heavy/custom Spark/Hadoop; ephemeral/Serverless + Spot for cost.
- **Dataflow:** unified batch + **streaming** with event-time correctness + autoscaling.
- **BigQuery SQL (ELT):** transform in-warehouse when you don't need Beam/Spark.
- **Dataplex/Data Catalog:** catalog + govern + quality across the lake.
- **Pub/Sub / Datastream:** streaming ingestion / operational-DB CDC into analytics.
- **Looker vs Looker Studio:** governed modeled enterprise BI vs free self-serve dashboards.
- **Cloud Composer vs Workflows vs scheduled queries:** complex standing Airflow vs light serverless orchestration vs simple.
- **Cross-cutting:** partitioning + clustering + column-pruning + format decide performance + your BigQuery bill.

## AWS→GCP map
S3-as-lake→**GCS**; Athena→**BigQuery (on-demand) / BigLake external tables**; Redshift→**BigQuery**; EMR→**Dataproc**; Glue ETL / Kinesis Analytics→**Dataflow**; Glue Catalog / Lake Formation→**Dataplex + Data Catalog**; Kinesis→**Pub/Sub (+ Dataflow)**; DMS/CDC→**Datastream**; QuickSight→**Looker Studio / Looker**; MWAA (Airflow)→**Cloud Composer**; Redshift ML/SageMaker-lite→**BigQuery ML**.

## Cadence
~1–2 weeks per service (BigQuery deserves the most time — it's the crown jewel); capstone is a 3-week push (touches the most services). ~8–10 weeks at 5–10 hrs/week.

---

# 🎓 GCP Volumes 1–7 complete
The seven volumes together span the core of Google Cloud for Cloud Architect + DevOps mastery:
1. **Foundational Core** — IAM & Hierarchy, VPC, Compute Engine, Cloud LB + MIGs, Cloud Storage, PD/Filestore, Cloud SQL/AlloyDB/Spanner, Firestore/Bigtable, Cloud Run/Functions, Cloud Operations
2. **Containers** — Docker, Artifact Registry, GKE (Autopilot/Standard), Cloud Run (deep), Cloud Build, Cloud Run Jobs/Batch
3. **IaC & CI/CD** — Terraform, Infra Manager, Config Connector, Cloud Build, Cloud Deploy, GitHub Actions + Workload Identity Federation
4. **Integration & Event-Driven** — Pub/Sub, Eventarc, Cloud Tasks, Workflows, Cloud Scheduler, API Gateway/Apigee, Dataflow (intro)
5. **Security & Governance** — Cloud KMS, Secret Manager, Certificate Manager, Security Command Center, Cloud Armor, Identity Platform, Resource Manager, Org Policy, **VPC Service Controls**, Cloud Identity
6. **Advanced Networking & Edge** — Cloud DNS, Cloud CDN, Cloud LB (advanced), Shared VPC/Peering, Network Connectivity Center, Private Service Connect, Interconnect/VPN, Cloud NAT
7. **Data & Analytics** — BigQuery, Dataproc, Dataflow, Dataplex, Pub/Sub/Datastream, Looker/Looker Studio, Cloud Composer, BigQuery ML

**The meta-capstone (staff-level):** fuse the volume capstones into one reference architecture — a governed multi-project org (Vol 5) on a global, Shared-VPC + NCC + PSC network (Vol 6), running containerized + serverless workloads (Vol 2/1), event-driven (Vol 4), delivered via keyless GitOps (Vol 3), feeding a governed BigQuery lakehouse (Vol 7), fully observable + secured with a VPC-SC perimeter. Write the design doc, model the cost, name the failure domains, and defend every decision. That document *is* your senior/staff GCP interview.

> ⚠️ Reminder: GCP renames + reprices services frequently. Verify current names, limits, tiers, and pricing against the official docs as you build — the architecture + concepts here are stable, the labels sometimes aren't.
