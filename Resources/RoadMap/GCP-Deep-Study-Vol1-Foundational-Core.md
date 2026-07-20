# GCP Deep Study Guide — Volume 1: The Foundational Core

**Service-by-service, maximum granularity.** Google Cloud counterpart to the AWS series, same study-card format. Built for ~5–10 hrs/week, junior → staff depth.

## What's in this volume

The foundational core: **IAM & Resource Hierarchy, VPC, Compute Engine, Cloud Load Balancing + Managed Instance Groups, Cloud Storage, Persistent Disk + Filestore, Cloud SQL / AlloyDB / Spanner, Firestore / Bigtable, Cloud Run / Cloud Functions, Cloud Operations (Monitoring + Logging).**

## How each service card works

- **🧠 Mental model** · **❓ Why it exists** · **🔬 Granular concepts** · **🎛️ Key knobs** · **🚧 Limits & gotchas** · **💸 Cost model** · **💥 Failure modes** · **🚀 Projects** (2–3, different approaches) · **💬 Interview questions** (junior→staff) · **✅ Mastery bar**

**Study loop:** read the card → do Project A in the console/`gcloud` → redo as IaC (Terraform) → break it → answer interview questions out loud → log it.

> ⚠️ **GCP renames things often** (Stackdriver → Cloud Operations, Container Registry → Artifact Registry, load balancer naming). Verify current names/limits/pricing in the official docs as you go.

> 🔑 **The single biggest AWS→GCP mental shift:** in AWS the account is the boundary and VPCs are regional. In GCP, the **project** is the fundamental unit, sits inside an **Org → Folder → Project** hierarchy, and a **VPC is global** (subnets are regional). Internalize this before anything else.

---

# 1. IAM & Resource Hierarchy

_Learn this first and deepest. GCP's model is genuinely different from AWS's._

**🧠 Mental model:** A tree of resources (Org → Folders → Projects → resources) where permissions granted higher up **flow downward**, and every grant is a simple sentence: "_this principal_ has _this role_ on _this resource_."

**❓ Why it exists:** Centralized, inheritable access control across many projects, plus a clean way for workloads (not just humans) to authenticate.

**🔬 Granular concepts**

_Resource hierarchy_

- **Organization** (root, tied to your Cloud Identity/Workspace domain) → **Folders** (departments/teams/environments) → **Projects** (the unit that holds resources + billing + APIs) → **resources**
- Policies **inherit downward** and are **additive** (a grant at the org applies to everything beneath; you can't take away an inherited allow with a lower-level policy — different from AWS SCP deny logic)
- Projects: project ID (globally unique, immutable) vs project number vs display name

_IAM model_

- **Principals (members):** Google account, Google group, service account, Cloud Identity domain, `allUsers`/`allAuthenticatedUsers`, and workload-identity federated principals
- **Roles** (a role = a bundle of permissions):
  - **Basic** roles (Owner/Editor/Viewer — legacy, too broad, avoid in prod)
  - **Predefined** roles (service-specific, least-privilege-ish, e.g., `roles/storage.objectViewer`)
  - **Custom** roles (you pick exact permissions)
- **Allow policy** = binding of principals ↔ role ↔ resource
- **IAM Conditions** (attribute-based: time, resource name, request attributes)
- **Deny policies** (explicit deny, evaluated before allow — newer)

_Service accounts (huge topic)_

- A **service account** is an identity for a workload (not a human); has its own email
- **Service account keys** (JSON key files) = long-lived credentials → **avoid** (they leak); Google actively discourages them
- **Attached service accounts** (a VM/GKE pod/Cloud Run service runs _as_ a service account — the good pattern; no keys)
- **Workload Identity Federation** (let external identities — GitHub Actions, AWS, OIDC — impersonate a service account with _no_ key)
- **Service account impersonation** (`--impersonate-service-account`; short-lived tokens) — the modern way to "act as" without keys
- The `iam.serviceAccounts.actAs` permission (the GCP analog of AWS `PassRole` — guard it; it's a privilege-escalation vector)

_Tooling_

- Policy Analyzer / Policy Troubleshooter (why does/doesn't X have access?)
- IAM Recommender (suggests least-privilege based on usage)
- `gcloud`, Cloud SDK, the metadata server (how attached SAs get tokens)

**🎛️ Key knobs:** which level you grant at (org/folder/project/resource), predefined vs custom role, IAM conditions, deny policies, service-account attachment, key vs keyless.

**🚧 Limits & gotchas**

- **Inheritance is additive and downward** — a broad grant at the org/folder level leaks to every project below. People over-grant at high levels.
- **Basic roles (Editor/Owner) are dangerously broad** — Editor can modify almost everything; use predefined/custom.
- **Service account keys are the #1 credential-leak source** — prefer attachment / impersonation / Workload Identity Federation.
- `actAs`/`serviceAccountUser` lets a principal run jobs as a more-privileged SA → escalation. Guard tightly.
- Deleting a service account still-referenced by a resource breaks that resource silently.
- IAM changes are eventually consistent (seconds).

**💸 Cost model:** IAM is free. (You pay for what the permissions enable.)

**💥 Failure modes**

- "Permission denied" → missing role binding, wrong resource level, condition not met, or a deny policy. Use Policy Troubleshooter.
- Workload can't authenticate → SA not attached, or missing scope/role.
- Escalation incident → over-broad `actAs` or leaked SA key.

**🚀 Projects**

- **A — Least-privilege, keyless (console + gcloud):** Create a service account with only `roles/storage.objectViewer` on **one** bucket; attach it to a Compute Engine VM; from the VM, prove it can read that bucket and nothing else — with **no key file**.
- **B — Cross-project access (different approach: hierarchy):** Two projects under one folder; grant a service account in project A a role on a resource in project B; observe how you did it _without_ keys via impersonation.
- **C — Keyless external CI:** Set up **Workload Identity Federation** so a GitHub Actions run can impersonate a deploy service account with no stored key; scope the federation to your repo.

**💬 Interview questions**

- _Junior:_ What are the levels of the resource hierarchy, and how does permission inheritance work?
- _Mid:_ Basic vs predefined vs custom roles? What's a service account and why avoid JSON keys?
- _Senior:_ How does a VM/pod get GCP permissions without keys? What is `actAs` and why is it dangerous? GCP inheritance (additive) vs AWS SCP (deny caps) — contrast.
- _Staff:_ Design org-wide access for 500 engineers across many projects: hierarchy design, least-privilege strategy, keyless workloads, and blast-radius containment.

**✅ Mastery bar:** You can design a hierarchy, grant least privilege at the right level, and run every workload keyless (attachment/impersonation/federation).

---

# 2. VPC (Virtual Private Cloud)

_The concept that most surprises AWS people: VPC is **global**._

**🧠 Mental model:** One global private network per project; you carve **regional subnets** out of it, and traffic between regions stays on Google's backbone without extra plumbing.

**❓ Why it exists:** A software-defined, global private network with fine-grained firewalling and hybrid connectivity.

**🔬 Granular concepts**

_Structure_

- **VPC network** = global; **subnets** = regional (a subnet spans all zones in its region)
- Auto-mode (a subnet auto-created per region — convenient, not for prod) vs **custom-mode** (you define subnets — production default)
- IP ranges: primary + **secondary ranges** (secondary ranges power GKE pods/services — "alias IPs")
- Global routing vs regional routing (dynamic routing mode)

_Firewalling & routes_

- **VPC firewall rules** are **stateful**, defined at the network level, with priorities, direction (ingress/egress), targets (by tag/SA), and sources (CIDR/tag/SA) — note: they attach by **network tags** or **service accounts**, not per-instance SGs like AWS
- Implied rules (default allow egress, deny ingress)
- Hierarchical firewall policies (org/folder level)
- **Routes** (system-generated + custom); next hops
- **Cloud NAT** (managed egress for instances with no external IP — regional, no per-instance NAT gateways)
- **Private Google Access** (reach Google APIs/services from internal IPs without external IPs)

_Connectivity (preview; deep in Vol 6)_

- **Shared VPC** (host project shares subnets with service projects — a GCP-signature pattern for centralized networking)
- VPC Network Peering; Cloud VPN / Interconnect; Private Service Connect

_DNS & visibility_

- Internal DNS; Cloud DNS (Vol 6)
- **VPC Flow Logs**; Firewall Rules Logging

**🎛️ Key knobs:** custom vs auto mode, subnet CIDRs + secondary ranges, firewall priorities/targets/tags, dynamic routing mode (regional vs global), Cloud NAT, Private Google Access.

**🚧 Limits & gotchas**

- **Firewall rules attach by network tag or service account, not to the instance directly** — and tags are unauthenticated (anyone who can set a tag gets the rule); prefer **service-account-based** rules for security.
- A "public" VM needs an external IP _and_ a firewall allow — but best practice is **no external IP + Cloud NAT + IAP** for SSH.
- Global VPC means overlapping CIDRs across peered/shared networks bite you — plan ranges centrally.
- GKE needs secondary ranges sized correctly or you exhaust pod IPs.
- Auto-mode VPCs use predictable ranges that collide when peering — use custom mode.

**💸 Cost model:** the network/subnets/firewall are free. You pay for: **Cloud NAT** (hourly + per-GB), external IPs, **egress** (especially cross-region + internet; egress pricing tiers), and Interconnect/VPN.

**💥 Failure modes**

- VM can't reach internet → no external IP _and_ no Cloud NAT, or egress firewall/route issue.
- Can't SSH → firewall rule missing, or (better path) IAP not configured.
- Can't reach Google APIs from a private VM → Private Google Access off.
- GKE pods failing to get IPs → secondary range too small.

**🚀 Projects**

- **A — Secure custom VPC (no public IPs):** Custom-mode VPC, regional subnets, firewall rules scoped by **service account** (not tags), a private VM reachable via **IAP SSH**, and **Cloud NAT** for egress. Prove no instance has an external IP.
- **B — Same VPC as Terraform (different approach: IaC):** Rebuild it in Terraform; add **Private Google Access** and confirm the private VM reaches Cloud Storage APIs with no NAT/internet.
- **C — Debug lab:** Turn on Flow Logs + firewall logging; break connectivity with a bad rule priority; find and fix it from the logs.

**💬 Interview questions**

- _Junior:_ Why is a GCP VPC global but subnets regional? What makes a VM reachable from the internet?
- _Mid:_ Firewall rules by tag vs by service account — why is SA-based safer? What does Cloud NAT solve?
- _Senior:_ What is Private Google Access and when do you need it? Design a no-public-IP network with IAP + NAT.
- _Staff:_ Design networking for a 40-project org: Shared VPC vs peering, centralized egress, IP planning, and firewall governance via hierarchical policies.

**✅ Mastery bar:** You can build a no-public-IP network with SA-based firewalls, IAP, NAT, and Private Google Access, and explain global-VPC implications.

---

# 3. Compute Engine (GCE)

**🧠 Mental model:** Google's VMs — rent a virtual machine by the second, pick the machine type and image, and drop it into a subnet.

**🔬 Granular concepts**

_Machine types_

- Families: **E2** (cost-optimized), **N2/N2D/N4** (balanced), **C3/C3D/H3** (compute), **M-series** (memory), **A/G** (GPU/accelerator)
- **Custom machine types** (pick exact vCPU + memory — a GCP advantage)
- Shared-core (e2-micro/small) with bursting

_Provisioning & bootstrap_

- Images (public, custom, from a family); **machine images** (full VM snapshots)
- **Startup scripts** + custom **metadata** (key/value; the **metadata server** at `169.254.169.254` gives SA tokens)
- **OS Login** (IAM-managed SSH — the recommended access model) vs metadata SSH keys
- Instance templates (blueprint for MIGs)

_Pricing (interview-relevant)_

- On-demand + **sustained-use discounts** (automatic for steady usage — GCP-unique)
- **Committed-use discounts** (1/3-yr commit)
- **Spot VMs** (formerly preemptible; deep discount, can be reclaimed; no >24h guarantee)
- Sole-tenant nodes (compliance/licensing)

_Reliability & placement_

- Zones vs regions; live migration (Google moves running VMs during host maintenance — a differentiator)
- Placement policies (spread/compact)
- Local SSD (ephemeral, fast) vs Persistent Disk (durable)

**🎛️ Key knobs:** machine type (or custom), image, startup script, attached service account + scopes, network tags, Spot vs standard, disk type/size, OS Login, deletion protection.

**🚧 Limits & gotchas**

- **OS Login vs metadata SSH keys** — mixing them causes access confusion; standardize on OS Login (IAM-driven).
- Attached SA + **access scopes** both gate API access on older VMs (scopes are a legacy extra filter on top of IAM) — set scopes to `cloud-platform` and control via IAM, or you'll get confusing denials.
- Spot VMs vanish with ~30s notice → design for interruption.
- vCPU quotas per region (raisable).
- External IP by default on some paths → prefer none + NAT/IAP.

**💸 Cost model:** per-second compute (1-min min) + sustained-use auto-discount + disks + egress + external IPs. Spot ~60–91% cheaper. Custom types avoid paying for unused vCPU/RAM.

**💥 Failure modes:** SSH fails → OS Login/IAM role, firewall, or IAP config. VM can't call APIs → SA not attached or access scope too narrow. Vanished VM → Spot reclaim or MIG autohealing.

**🚀 Projects**

- **A — Bootstrapped, keyless, keyless-SSH VM:** Startup-script VM running a web app; access via **IAP + OS Login** (no external IP, no SSH keys in metadata); attached SA reads one bucket.
- **B — Custom machine type + Spot (different approach: cost):** Run a fault-tolerant batch job on a **custom-sized Spot VM**; handle the preemption notice; compare cost vs on-demand + sustained-use.
- **C — Golden image:** Bake a custom image with the app pre-installed; launch identical VMs from the image family; compare to startup-script boot time.

**💬 Interview questions**

- _Junior:_ How does a VM get permission to call Cloud Storage the right way?
- _Mid:_ OS Login vs metadata SSH keys? What are access scopes and how do they interact with IAM? Custom machine types — why useful?
- _Senior:_ Design a workload on Spot VMs safely. What is live migration and why does it matter? Sustained vs committed-use discounts.
- _Staff:_ Optimize compute spend across hundreds of VMs: right-sizing, CUDs, Spot, and custom types — without hurting reliability.

**✅ Mastery bar:** You launch keyless, no-external-IP VMs accessed via IAP, pick the right machine/pricing, and never store SA keys on a box.

---

# 4. Cloud Load Balancing + Managed Instance Groups (MIGs)

**🧠 Mental model:** Cloud Load Balancing is a **global, anycast** front door (one IP, users hit the nearest edge); MIGs keep the right number of healthy VMs behind it and heal/scale them.

**🔬 Granular concepts**

_Load balancing_

- **Global external Application LB** (L7 HTTP(S), single anycast IP, edge-terminated, integrates Cloud CDN + Cloud Armor) — the flagship
- **Regional external Application LB**; **Network LB** (L4 TCP/UDP: passthrough or proxy)
- **Internal** Application/Network LBs (private, for internal services)
- Components: forwarding rule → target proxy → **URL map** (path/host routing) → **backend service** → **backends** (MIGs/NEGs) with **health checks**
- **NEGs** (Network Endpoint Groups): zonal (VMs), serverless (Cloud Run/Functions/App Engine), internet, hybrid
- SSL certs (Google-managed or self); SSL policies; global anycast IP

_Managed Instance Groups_

- Instance template–based; **autohealing** (recreate on failed health check), **autoscaling** (CPU, LB utilization, custom/Cloud Monitoring metrics, schedules)
- **Rolling updates** + **canary** (versioned templates, surge/unavailable settings)
- Regional (multi-zone, resilient) vs zonal MIGs
- Stateful MIGs (preserve disks/IPs where needed)

**🎛️ Key knobs:** LB type (global vs regional, external vs internal, L4 vs L7), URL-map routing, health-check config, backend balancing mode (rate/utilization), autoscaler signal + min/max, rolling-update surge/canary %, regional vs zonal MIG.

**🚧 Limits & gotchas**

- Wrong/failing **health check** → autohealing kills healthy VMs in a loop, or LB sends no traffic.
- Global LB needs backends configured with correct balancing mode or it under/over-loads.
- Use **regional MIGs** for HA (a zonal MIG dies with its zone).
- Serverless backends attach via **serverless NEGs** (not the same as VM backends).
- Global anycast means you get one IP worldwide — great, but understand it terminates at Google's edge.

**💸 Cost model:** LB = forwarding-rule hours + data processed (+ Cloud CDN/Armor if used). MIGs are free; you pay for the VMs. Global LB egress + edge.

**💥 Failure modes:** all backends unhealthy → health-check path/firewall (health-check source ranges must be allowed). No scaling → wrong autoscaler metric or max hit. Canary broke prod → surge/rollback misconfigured.

**🚀 Projects**

- **A — Self-healing, global web tier:** Regional MIG + autoscaling behind a **global external Application LB** with a Google-managed cert; delete a VM (watch autohealing) and load-test (watch autoscaling).
- **B — Path routing + serverless mix (different approach):** One URL map routing `/api` → a **serverless NEG** (Cloud Run) and `/` → a MIG backend, behind one anycast IP.
- **C — Zero-downtime rollout:** Push a new instance-template version; do a **canary** rolling update while traffic flows; verify no dropped requests, then promote.

**💬 Interview questions**

- _Junior:_ What does a load balancer + MIG do together?
- _Mid:_ Global vs regional LB; L4 vs L7; what's a NEG? Autohealing vs autoscaling.
- _Senior:_ Your MIG isn't scaling — diagnose. Why must you allow health-check source ranges in the firewall? Regional vs zonal MIG for HA.
- _Staff:_ Design global, multi-region traffic distribution with canary deploys and a strict latency SLO under spiky load.

**✅ Mastery bar:** You can build a globally-load-balanced, self-healing, canary-deployable tier and reason about health checks + NEGs.

---

# 5. Cloud Storage (GCS)

**🧠 Mental model:** Infinite, durable, HTTPS-accessible object storage — buckets of objects with fine-grained IAM and lifecycle automation.

**🔬 Granular concepts**

- Buckets (globally-unique names), objects; **location types**: regional, dual-region, multi-region (availability vs cost vs latency)
- **Storage classes**: Standard, **Nearline** (~30-day), **Coldline** (~90-day), **Archive** (~365-day) — cheaper storage, higher retrieval + minimum-storage-duration fees
- **Autoclass** (auto-moves objects between classes)
- Strong global consistency (read-after-write)
- **Object lifecycle management** (transition class / delete by age/version)
- **Versioning**; retention policies + **bucket lock** (WORM/compliance); Object Holds
- Access control: **Uniform bucket-level access** (IAM-only — recommended) vs fine-grained (IAM + legacy ACLs)
- **Signed URLs** (time-limited access without credentials); signed policy documents
- Encryption: Google-managed (default) vs **CMEK** (Cloud KMS) vs CSEK (customer-supplied)
- Integration: **Pub/Sub notifications** on object changes; static website hosting; behind Cloud CDN
- Transfer: `gsutil`/`gcloud storage`, Storage Transfer Service, composite uploads, requester-pays

**🎛️ Key knobs:** location type, storage class, lifecycle rules, uniform vs fine-grained access, versioning, retention/bucket-lock, CMEK, public access prevention, Pub/Sub notifications.

**🚧 Limits & gotchas**

- **Public access prevention** should be on org-wide — public buckets are a classic breach.
- Nearline/Coldline/Archive have **minimum storage durations + retrieval fees** — churny data in Archive costs _more_, not less.
- Multi-region ≠ multi-zone-within-region; pick location for your access + compliance needs.
- Uniform bucket-level access is recommended; mixing ACLs + IAM causes confusing access outcomes.
- Object versioning without lifecycle cleanup silently grows cost.

**💸 Cost model:** storage per-GB (by class + location) + operations (Class A/B ops priced differently) + egress + retrieval fees (cold classes) + early-deletion fees. Multi-region storage costs more than regional.

**💥 Failure modes:** 403 → public access prevention / IAM / CMEK key perms. Surprise bill → cold-class retrieval/early-delete, versioning growth, or cross-region egress. Object "deleted" but present → versioning.

**🚀 Projects**

- **A — Private static site via CDN:** Private bucket + uniform access → **Cloud CDN** in front of a global LB with a signed-URL path for authed users; lifecycle-tier old objects to Coldline.
- **B — Event-driven processing (different approach: serverless):** Object upload → **Pub/Sub notification** → **Cloud Run/Function** generates a thumbnail / writes metadata to Firestore; add a dead-letter path.
- **C — Compliance bucket:** Enforce **CMEK + retention policy + bucket lock + versioning**, public access prevention on, and dual-region for resilience.

**💬 Interview questions**

- _Junior:_ Storage classes — pick one for data read twice a year; how do you keep a bucket private but served publicly?
- _Mid:_ Uniform vs fine-grained access; why prefer uniform? Google-managed vs CMEK; signed URLs.
- _Senior:_ Why can cold storage cost _more_ for churny data? Design encryption + access for sensitive data with compliance retention.
- _Staff:_ Design GCS for a petabyte platform: location/class strategy, cross-project access governance, DR, and cost control.

**✅ Mastery bar:** You pick the right class/location for any pattern, lock a bucket down (uniform + public-access-prevention + CMEK), and explain every cost driver.

---

# 6. Persistent Disk + Filestore

**🧠 Mental model:** Persistent Disk = a network block disk for one VM (or read-only many). Filestore = managed NFS many VMs mount at once.

**🔬 Granular concepts**

_Persistent Disk (PD)_

- Types: **pd-standard** (HDD), **pd-balanced**, **pd-ssd**, **pd-extreme** (provisioned IOPS); Hyperdisk (newer, decouple IOPS/throughput/capacity)
- Zonal vs **regional PD** (synchronously replicated across two zones — HA)
- **Snapshots** (incremental, global, can restore cross-zone/region); snapshot schedules
- Resize online; CMEK encryption; read-only multi-attach
- **Local SSD** (ephemeral, physically attached, fastest, lost on stop)

_Filestore_

- Managed NFS; tiers (Basic, Zonal, Enterprise/regional); multi-VM shared access; backups; snapshots

**🚧 Limits & gotchas**

- Zonal PD dies with its zone → use **regional PD** or snapshots for DR.
- Local SSD data is lost on stop/terminate — never durable state.
- PD performance scales with size + type — a tiny pd-standard is slow.
- Filestore is pricier per-GB than PD — only for genuine shared-file needs.

**💸 Cost model:** PD = per-GB provisioned (+ provisioned IOPS/throughput for extreme/Hyperdisk) + snapshot storage. Filestore = per-GB by tier. Local SSD = per-GB, cheap but ephemeral.

**🚀 Projects**

- **A:** Provision pd-balanced, resize it online, snapshot it, restore into another zone, attach to a new VM.
- **B (different approach: shared):** Mount Filestore on three VMs behind an LB so they share uploads.
- **C:** Automate scheduled snapshots with retention; test a restore.

**💬 Interview:** PD vs Local SSD vs Filestore — one differentiator each. Zonal vs regional PD for HA. How do you move a disk between zones (snapshot/restore)?

**✅ Mastery bar:** You never lose data to the wrong storage choice and can design snapshot-based DR with real restores.

---

# 7. Cloud SQL + AlloyDB + Spanner (relational)

**🧠 Mental model:** Managed relational databases at three tiers — Cloud SQL (standard managed MySQL/Postgres/SQL Server), AlloyDB (high-performance Postgres), Spanner (globally-distributed, horizontally-scalable, strongly-consistent SQL).

**🔬 Granular concepts**

_Cloud SQL_

- Engines: MySQL, PostgreSQL, SQL Server
- **HA (regional)**: synchronous standby in another zone → automatic failover (HA, not read scaling)
- **Read replicas** (async; scale reads; cross-region possible; can be promoted)
- Automated backups + **point-in-time recovery**; maintenance windows
- Connectivity: **Private IP (recommended)**, public IP + authorized networks, or the **Cloud SQL Auth Proxy** / connectors (IAM-based, encrypted)
- IAM database authentication; CMEK

_AlloyDB_ — PostgreSQL-compatible, much faster for transactional + analytical (columnar engine) workloads; separates compute/storage; read pools; for demanding Postgres needs.

_Spanner_ — globally-distributed relational DB with **horizontal scaling + strong consistency** via **TrueTime**; 99.999% SLA (multi-region); no failover step (it's inherently distributed); scales by nodes/processing units; use for global, huge, strongly-consistent workloads (fintech, global inventory).

**🚧 Limits & gotchas**

- **HA vs read replica confusion** (same as AWS): HA = failover; read replica = read scaling (async, can lag).
- Connecting via **public IP + authorized networks** is the insecure old way → use **Private IP + Auth Proxy/connector + IAM auth**.
- Connection storms (esp. from serverless) exhaust connections → use the connector/proxy + pooling (PgBouncer/AlloyDB).
- Spanner is powerful but pricey + needs schema/design discipline (interleaving, avoiding hotspots on monotonically-increasing keys).
- Choosing Spanner when Cloud SQL suffices = massive overspend.

**💸 Cost model:** Cloud SQL = instance + storage + HA doubles compute + backups + egress. AlloyDB = compute + storage + more for read pools. Spanner = per node/processing-unit-hour + storage (expensive; multi-region more).

**💥 Failure modes:** can't connect → private IP/proxy/firewall or authorized-network miss. Stale reads → lagging replica. "Too many connections" → no pooling. Spanner hotspot → monotonic key.

**🚀 Projects**

- **A — HA + private + keyless:** Cloud SQL (Postgres) with **HA + Private IP**; connect from a VM/Cloud Run via the **connector with IAM auth** (no password in code); force a failover and confirm reconnect.
- **B — Read scaling (different approach):** Add read replicas; route reads to a replica, writes to primary; measure throughput + lag.
- **C — Spanner taste:** Model a globally-consistent counter/inventory in Spanner; deliberately create then fix a hotspot key; contrast with Cloud SQL.

**💬 Interview questions**

- _Junior:_ What does Cloud SQL manage for you?
- _Mid:_ HA vs read replica — what each solves. Public IP + authorized networks vs private IP + proxy — why the latter?
- _Senior:_ Read-heavy app hammering the DB — options in order. When does Spanner beat Cloud SQL, and what's the cost/design tradeoff?
- _Staff:_ Design a global, low-RTO relational data tier. Where does Spanner earn its price vs a multi-region Cloud SQL/AlloyDB design?

**✅ Mastery bar:** You never confuse HA with read-scaling, connect privately + keyless, and know when Spanner is (and isn't) justified.

---

# 8. Firestore + Bigtable (NoSQL)

**🧠 Mental model:** Firestore = a serverless document DB for app data with real-time sync. Bigtable = a wide-column store for massive, low-latency, high-throughput workloads (time-series, IoT, analytics).

**🔬 Granular concepts**

_Firestore_

- Document/collection model; serverless, auto-scaling; strong consistency
- Real-time listeners; offline sync (mobile/web SDKs)
- Composite indexes (queries need indexes; auto + manual); query limitations (no arbitrary joins)
- Native mode vs Datastore mode
- Security rules (for direct client access) vs server access via IAM
- Transactions; TTL policies

_Bigtable_

- Wide-column, HBase-compatible API; single **row key** design is everything
- Massive scale (petabytes), low-latency, high-throughput; **linear scaling by nodes**
- **Hotspotting** on sequential row keys (design keys to distribute — reverse timestamps, salting, field promotion)
- No secondary indexes (design row keys for access patterns); no multi-row transactions across the board
- SSD vs HDD clusters; replication (multi-cluster) for HA + geo

**🚧 Limits & gotchas**

- **Bigtable hotspotting**: sequential/timestamp-prefixed row keys funnel writes to one node → throttling. Row-key design is the whole game.
- Firestore requires composite indexes for many queries — missing index = query error.
- Firestore has per-document/second write limits (avoid single hot documents/counters → sharded counters).
- Choosing Bigtable for small data = overkill; choosing Firestore for high-throughput time-series = wrong tool.

**💸 Cost model:** Firestore = per document read/write/delete + storage + network (serverless, scales to zero-ish). Bigtable = per node-hour (always-on) + storage + network — Bigtable has a real standing cost, so it's for sustained high scale.

**🚀 Projects**

- **A — Firestore app data:** Model users + orders; add the composite indexes queries need; implement a **sharded counter** to beat the single-document write limit.
- **B — Bigtable at scale (different approach):** Store time-series/IoT-style data; design a **non-hotspotting row key** (reverse-timestamp/salted); load-test and observe node scaling.

**💬 Interview questions**

- _Junior:_ Firestore vs Bigtable — what workload is each for?
- _Mid:_ Explain Bigtable hotspotting and how row-key design fixes it. Why does Firestore need indexes?
- _Senior:_ When Firestore vs Cloud SQL vs Bigtable? How do you avoid a hot document/row?
- _Staff:_ Design the data layer for a global, high-throughput event platform — choose stores per access pattern and defend it.

**✅ Mastery bar:** You choose NoSQL vs relational deliberately, design non-hotspotting Bigtable keys, and handle Firestore index/write-limit constraints.

---

# 9. Cloud Run + Cloud Functions (serverless)

**🧠 Mental model:** Run code without managing servers — **Cloud Run** runs _containers_ that scale to zero; **Cloud Functions** runs _event-driven functions_ (gen 2 is built on Cloud Run).

**🔬 Granular concepts**

_Cloud Run_

- Deploy any container; **scales to zero**; request-driven autoscaling
- **Concurrency** (requests per instance — a key Cloud Run lever; unlike Lambda's 1-per-invocation) → big cost/perf impact
- Services (HTTP) vs **Jobs** (run-to-completion batch)
- Min instances (kill cold starts) / max instances (cap); CPU allocation (always-on vs during-request); CPU boost
- Revisions + traffic splitting (**built-in canary/blue-green** by percentage)
- Runs as a **service account** (keyless); ingress controls; VPC connector/Direct VPC egress (reach private resources)
- Triggers: HTTPS, **Eventarc** (events), Pub/Sub push, Cloud Scheduler

_Cloud Functions_

- **Gen 1** vs **Gen 2** (Gen 2 = Cloud Run + Eventarc underneath: more concurrency, bigger limits, longer timeouts)
- Triggers: HTTP, Pub/Sub, Cloud Storage, Firestore, Eventarc
- Per-function runtime (Node/Python/Go/Java/etc.)

**🚧 Limits & gotchas**

- **Concurrency tuning** is the Cloud Run superpower and footgun: concurrency=1 wastes money; too high overloads the instance/DB. Tune it.
- Scale-to-zero → cold starts on the first request (min instances or CPU boost to mitigate).
- Connection-per-instance to Cloud SQL → use the connector + pooling.
- Cloud Functions **Gen 1 vs Gen 2** differ a lot (limits, concurrency) — know which you're on.
- Request timeout limits (long jobs → Cloud Run Jobs / Workflows / Batch).

**💸 Cost model:** Cloud Run = CPU + memory × request-duration + requests; **higher concurrency = fewer instances = lower cost**. Min instances add standing cost. Very cheap for spiky/low traffic; can lose to GKE at sustained high load.

**💥 Failure modes:** slow p99 → cold starts / concurrency too low. Overloaded downstream → concurrency too high. Can't reach private DB → no VPC connector/egress.

**🚀 Projects**

- **A — Serverless API:** Container on **Cloud Run** → Firestore/Cloud SQL; tune concurrency + min instances; do a **10%→100% traffic split** canary across revisions.
- **B (different approach: event function):** A **Cloud Functions (Gen 2)** triggered by a GCS upload that processes the file; contrast Gen 1 vs Gen 2 limits.
- **C — batch job:** A **Cloud Run Job** processing a queue/dataset to completion, triggered by Cloud Scheduler.

**💬 Interview questions**

- _Junior:_ What does "scale to zero" mean and how are you billed?
- _Mid:_ Cloud Run concurrency — how does it change cost/behavior vs Lambda's model? Cloud Functions Gen 1 vs Gen 2?
- _Senior:_ Two scenarios where Cloud Run is the wrong tool. How do you kill cold starts, and what's the tradeoff? How does traffic splitting enable safe deploys?
- _Staff:_ Design a serverless platform for spiky 50k-rps traffic with cost + latency targets; where does Cloud Run stop making sense vs GKE?

**✅ Mastery bar:** You can tune Cloud Run concurrency/min-instances, deploy safely with traffic splitting, and know when serverless is wrong.

---

# 10. Cloud Operations (Monitoring + Logging + Trace)

_Formerly Stackdriver. Your eyes and ears — and most "app is slow" interview answers._

**🧠 Mental model:** The observability suite — every metric, log, trace, and alert for your projects in one place.

**🔬 Granular concepts**

_Cloud Monitoring_

- Metrics (built-in + custom), labels, MQL/PromQL, **Managed Service for Prometheus**
- Dashboards; **alerting policies** (conditions → notification channels); SLOs + error budgets (native SLO monitoring)
- Uptime checks (synthetic monitoring); metrics scopes (multi-project monitoring)

_Cloud Logging_

- **Log buckets**, **log sinks** (route logs to GCS/BigQuery/Pub/Sub for analytics/retention), **log-based metrics**
- **Logs Explorer** + query language; **Log Analytics** (SQL over logs via BigQuery)
- Audit logs: **Admin Activity** (always on) vs **Data Access** (opt-in) vs System Event — the "who did what" trail
- Retention + exclusion filters (cost control)

_Others in the suite_

- **Cloud Trace** (distributed tracing), **Cloud Profiler** (continuous profiling), **Error Reporting** (aggregates exceptions)

**🚧 Limits & gotchas**

- **Data Access audit logs are off by default** — enable them for a real audit trail (they cost/volume, so scope).
- Default log retention + ingesting everything → surprise cost; use exclusion filters + sinks + retention.
- VM guest metrics (memory/disk) need the **Ops Agent** installed (like the CloudWatch agent).
- Noisy, non-actionable alerts → alert fatigue.

**💸 Cost model:** logging per-GB ingested (+ retention beyond free), monitoring for custom/Prometheus metrics + API calls, trace per span. Verbose logging at scale is a real cost driver.

**💥 Failure modes:** no memory metric → Ops Agent not installed. Alert never fires → wrong metric/condition. Log bill spike → no exclusion filters / unbounded ingestion. No audit trail → Data Access logs disabled.

**🚀 Projects**

- **A — Full observability:** Ship structured logs; dashboard for p99 latency + error rate + throughput; alerting policy → notification channel; define a native **SLO + error budget**.
- **B — Log-based metric (different approach):** Create a log-based metric from an error pattern and alert on it; route logs to **BigQuery via a sink** for SQL analysis.
- **C — Incident drill:** Inject a fault; diagnose using Logs Explorer + Trace + dashboards only; write a mini post-mortem.

**💬 Interview questions**

- _Junior:_ Why doesn't a VM show memory by default?
- _Mid:_ Admin Activity vs Data Access audit logs — which is on by default and why does it matter? What's a log sink for?
- _Senior:_ "App is slow" — diagnose in order with the Ops suite. What makes an alert good vs noise?
- _Staff:_ Design observability for 50 services across many projects: metrics scope, log cost control, tracing, SLOs, and avoiding alert fatigue.

**✅ Mastery bar:** You can stand up meaningful SLO-based monitoring, control log cost, ensure an audit trail, and diagnose a live incident from telemetry.

---

# 🏔️ Volume 1 Capstone — "Production-shaped GCP application"

A full app you can demo and defend: custom-mode **VPC** with SA-based firewalls + **Cloud NAT** + **Private Google Access** (no external IPs), a **global external Application LB** in front of an autoscaling **regional MIG** (and/or a **Cloud Run** service via serverless NEG), a **Cloud SQL HA (Private IP + IAM auth)** backend, private **Cloud Storage + Cloud CDN** for assets, **IAP** for access, least-privilege **keyless service accounts** throughout, and full **Cloud Operations** dashboards/alerts/SLOs. Write a one-page design doc justifying each choice and naming the remaining single points of failure.

## AWS→GCP quick map (memorize)

IAM users→**principals + service accounts**; account→**project**; regional VPC→**global VPC (regional subnets)**; Security Groups→**firewall rules (by tag/SA)**; EC2→**Compute Engine**; ASG→**MIG**; ELB→**Cloud Load Balancing (global anycast)**; S3→**Cloud Storage**; EBS→**Persistent Disk**; EFS→**Filestore**; RDS→**Cloud SQL/AlloyDB**; DynamoDB→**Firestore/Bigtable**; (no direct AWS analog)→**Spanner**; Lambda→**Cloud Functions/Cloud Run**; CloudWatch→**Cloud Operations**; PassRole→**actAs**; NAT Gateway→**Cloud NAT**.

## Cadence

~1–2 weeks per service; do Project A by hand, redo as Terraform, break it. Order: IAM → VPC → Compute Engine → LB/MIG → Cloud Storage → PD/Filestore → Cloud SQL → Firestore/Bigtable → Cloud Run → Cloud Operations. ~4–5 months at 5–10 hrs/week.
