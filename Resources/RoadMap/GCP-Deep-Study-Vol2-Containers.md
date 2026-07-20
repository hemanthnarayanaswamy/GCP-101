# GCP Deep Study Guide — Volume 2: Containers & Compute Orchestration

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** Docker fundamentals, Artifact Registry, GKE (Standard + Autopilot), Cloud Run (deep), Cloud Build (for containers), Cloud Run Jobs + Cloud Batch.

> Kubernetes was born at Google, and **GKE is the flagship** — this volume goes deepest there. Prereq from Vol 1: IAM/service accounts, VPC, Cloud Load Balancing, Cloud Operations.

---

# 0. Docker fundamentals (mandatory first)

**🧠 Mental model:** A container = a process + a frozen snapshot of everything it needs, so it runs identically anywhere. Image = template; container = running instance.

**🔬 Granular concepts**
- Image **layers** + layer caching; Dockerfile (`FROM`, `RUN`, `COPY`, `ENV`, `ENTRYPOINT` vs `CMD`, `WORKDIR`, `USER`)
- **Multi-stage builds** (build fat, ship tiny)
- Base image choice (distroless/alpine = small + fewer CVEs)
- Tags vs digests; `latest` is a trap (mutable)
- **Cloud-native build option:** GCP's **Buildpacks** / `pack` (build a container from source with *no* Dockerfile) — used by Cloud Run + Cloud Build
- 12-factor ideas (config via env, stateless, logs to stdout)

**🚧 Gotchas:** running as root; giant no-multi-stage images; secrets baked into layers; `latest` → non-reproducible deploys.

**🚀 Drill:** Write a multi-stage Dockerfile producing a <50 MB non-root image logging to stdout; then build the *same* app with Buildpacks (no Dockerfile) and compare.

**💬 Interview:** ENTRYPOINT vs CMD? Why multi-stage? Why is `latest` dangerous? What do Buildpacks give you?

---

# 1. Artifact Registry

**🧠 Mental model:** GCP's private, IAM-controlled registry for container images **and** language packages (npm/Maven/PyPI/etc.) — the successor to Container Registry (GCR).

**❓ Why it exists:** Store, version, scan, and secure build artifacts close to where they run, with GCP-native IAM and vulnerability scanning.

**🔬 Granular concepts**
- Repositories per format (Docker, Maven, npm, Python, apt/yum, generic); regional/multi-region location
- Auth via IAM (`gcloud auth configure-docker <region>-docker.pkg.dev`); no separate registry creds
- **Vulnerability scanning** (Container Analysis / Artifact Analysis — on-push + continuous)
- **Binary Authorization** integration (only allow signed/attested images to deploy — supply-chain security)
- Cleanup policies (auto-delete old/untagged images — cost control)
- Immutable tags; remote repositories (proxy/cache Docker Hub etc.); virtual repositories
- Replaces GCR (`gcr.io`) — migrate to `pkg.dev`

**🎛️ Key knobs:** repo format/location, immutable tags, cleanup policies, scanning on/off, Binary Authorization policy, IAM per-repo.

**🚧 Limits & gotchas**
- Migrating off legacy GCR (`gcr.io`) to Artifact Registry (`pkg.dev`) — know both exist.
- GKE/Cloud Run **service account** needs `roles/artifactregistry.reader` to pull — a common "image pull failed."
- Untagged images accumulate without cleanup policies → cost.
- Cross-project pulls need IAM on the repo.

**💸 Cost model:** storage per-GB + egress; scanning per image (Artifact Analysis). Cleanup policies keep it in check.

**💥 Failure modes:** pull fails → runtime SA lacks reader role, or wrong repo path/region. Registry bloat → no cleanup policy.

**🚀 Projects**
- **A:** Push a multi-stage image to Artifact Registry; enable vulnerability scanning; read findings; rebuild on a patched base to clear them.
- **B (different approach — supply chain):** Turn on **Binary Authorization** so only attested images deploy to GKE/Cloud Run; try to deploy an unsigned image and watch it blocked.
- **C:** Add a cleanup policy keeping the last 5 tags; script build+push tagged with the git SHA (kill `latest`).

**💬 Interview:** How does Artifact Registry auth work? What is Binary Authorization and what supply-chain risk does it address? How do you stop a registry growing forever?

**✅ Mastery bar:** Reproducible, scanned, attested, cleanup-managed artifacts that only the right SAs can pull.

---

# 2. GKE (Google Kubernetes Engine) — the flagship

**🧠 Mental model:** Managed Kubernetes where Google runs the control plane. **Autopilot** manages nodes for you too (you think in pods); **Standard** gives you node-level control.

**❓ Why it exists:** Production Kubernetes with Google's operational muscle, deep GCP integration, and a hands-off (Autopilot) option.

**🔬 Granular concepts**

*Kubernetes core (portable knowledge)*
- **Pod** (smallest unit), **Deployment** (stateless, ReplicaSets, rolling updates), **Service** (ClusterIP/NodePort/LoadBalancer), **Ingress/Gateway** (L7 routing in)
- **ConfigMap/Secret**, **Namespace**, **StatefulSet**, **DaemonSet**, **Job/CronJob**
- Labels/selectors; the reconciliation loop; **requests vs limits**; liveness/readiness probes; HPA/VPA

*GKE modes*
- **Autopilot** (Google manages nodes, scaling, security posture; you pay per pod resource request; less to break — the recommended default for most) vs **Standard** (you manage node pools, machine types, more control/cost-tuning)
- **Node pools** (groups of same-type nodes; Spot node pools; GPU pools) — Standard only
- Cluster types: zonal vs **regional** (control plane + nodes across zones = HA)

*GKE-specific integrations*
- **Workload Identity** (map a K8s service account → a GCP service account so pods get GCP permissions **keyless** — the essential pattern, GKE's IRSA equivalent)
- **VPC-native clusters** (pods get real VPC IPs via **alias IP** secondary ranges — plan ranges!)
- **GKE Ingress / Gateway API** → provisions Cloud Load Balancing; Container-native LB (NEGs → pods directly)
- **Cluster Autoscaler** + **Node Auto-Provisioning**; **Autopilot** auto-scales inherently
- Config Sync / Anthos Config Management (GitOps); Binary Authorization; Shielded/Confidential nodes
- Release channels (rapid/regular/stable) for auto-upgrades

**🎛️ Key knobs:** Autopilot vs Standard, regional vs zonal, node pool machine types + Spot, Workload Identity, requests/limits, HPA targets, release channel, network policy.

**🚧 Limits & gotchas**
- **VPC-native + alias IPs**: pods consume real subnet IPs → undersized secondary ranges = pod IP exhaustion at scale (plan CIDRs).
- **Workload Identity** is the correct way to give pods GCP access — using SA keys in a pod is the anti-pattern.
- No requests/limits → noisy neighbors + eviction chaos.
- Autopilot removes footguns but restricts some low-level tweaks (privileged pods, certain daemonsets) — know the constraints.
- Regional control plane has an hourly charge; Standard needs upgrade/ops discipline.
- Two permission systems (GCP IAM + K8s RBAC) must agree.

**💸 Cost model:** cluster management fee per cluster/hour (one zonal cluster/account can be in the free tier historically — verify) + node compute (Standard) or per-pod resource requests (Autopilot) + LBs + egress. The hidden cost is operational time (Standard) — Autopilot trades some cost for far less ops.

**💥 Failure modes:** pod `Pending` → no schedulable node / insufficient resources / no pod IP. `CrashLoopBackOff` → app/probe/config. Pod can't call GCP APIs → Workload Identity misconfigured. Locked out → RBAC/IAM mismatch.

**🚀 Projects**
- **A — Autopilot first:** Deploy a Deployment + Service + **Gateway/Ingress** (auto-provisions a global LB) on an **Autopilot** cluster; do a rolling update; scale with HPA.
- **B (different approach — Standard + Spot + Workload Identity):** Standard cluster with a **Spot node pool**; give a pod least-privilege GCP access via **Workload Identity** (read one bucket only); tolerate node preemption.
- **C — container-native LB + autoscaling:** Use NEG-based container-native load balancing; add HPA (pods) + Cluster Autoscaler (nodes); load-test to watch both scale.

**💬 Interview questions**
- *Junior:* Pod vs Deployment vs Service?
- *Mid:* Autopilot vs Standard — the tradeoff. How does a pod get GCP permissions (**Workload Identity**)? Requests vs limits?
- *Senior:* Why can VPC-native clusters exhaust IPs? GKE vs Cloud Run — honest decision. Regional vs zonal cluster for HA.
- *Staff:* Justify (or reject) GKE for a 200-engineer org: Autopilot vs Standard fleet strategy, multi-tenancy, upgrades, cost, and the ops burden.

**✅ Mastery bar:** You can deploy + scale on GKE (Autopilot and Standard), wire Workload Identity correctly, and give an honest GKE-vs-Cloud-Run recommendation.

---

# 3. Cloud Run (deep)

**🧠 Mental model:** Serverless containers — deploy an image, it scales (to zero) on requests, terminates HTTPS, and needs no cluster. Often the *right default* on GCP before reaching for GKE.

**🔬 Granular concepts**
- Services (HTTP, scale-to-zero) vs **Jobs** (run-to-completion)
- **Concurrency** (requests per instance) — the core cost/perf lever (higher = fewer instances = cheaper, until you overload)
- Min instances (kill cold starts) / max instances (cap); CPU always-on vs during-request; startup CPU boost
- **Revisions + traffic splitting** (built-in canary/blue-green by %)
- Runs as a **service account** (keyless); ingress settings (all / internal / internal+LB)
- **Direct VPC egress** or **Serverless VPC Access connector** (reach private resources: Cloud SQL, internal services)
- Triggers: HTTPS, **Eventarc**, Pub/Sub push, Cloud Scheduler; behind a global LB via serverless NEG
- Second-generation execution environment; gVisor sandbox
- Cloud Run **for Anthos**/GKE (run the Cloud Run API on your cluster)

**🚧 Limits & gotchas**
- Concurrency mis-tuning: =1 wastes money; too high overloads instance/DB.
- Cold starts on scale-to-zero → min instances / CPU boost (cost tradeoff).
- Reaching a private Cloud SQL needs the connector/Direct VPC egress + the Cloud SQL connector.
- Per-request timeout cap (long work → Cloud Run **Jobs** / Workflows / Batch).
- CPU-during-request only means background work after the response may be throttled (use CPU always-on or Jobs).

**💸 Cost model:** CPU + memory × duration + requests; **higher concurrency lowers cost**; min instances add standing cost; scale-to-zero = pay nothing idle. Cheap for spiky/low, can lose to GKE at sustained high load.

**🚀 Projects**
- **A — Service + canary:** Container → Cloud Run → Firestore; tune concurrency + min instances; deploy a new revision with **10%→100% traffic split**.
- **B (different approach — private backend):** Add a **VPC connector**/Direct VPC egress so Cloud Run reaches a **private Cloud SQL** with IAM auth.
- **C — Job:** A **Cloud Run Job** processing a dataset to completion, triggered by Cloud Scheduler; contrast with a service.

**💬 Interview:** Cloud Run concurrency vs Lambda's 1-per-invocation — cost implications. How do you kill cold starts and what's the tradeoff? How does traffic splitting enable safe deploys? Cloud Run vs GKE — when each?

**✅ Mastery bar:** You tune concurrency/min-instances, deploy safely with traffic splitting, reach private resources correctly, and know when to graduate to GKE.

---

# 4. Cloud Build (for containers)

**🧠 Mental model:** A managed, ephemeral build service — runs your build steps (each a container) in sequence, produces artifacts/images, and disappears. (Full CI/CD in Vol 3; here it's the container builder.)

**🔬 Granular concepts**
- `cloudbuild.yaml`: **steps** (each step is a container image running a command), `images`, `artifacts`, `substitutions`, `options`
- Builds from Dockerfile **or** Buildpacks; pushes to Artifact Registry
- Triggers (on push/PR/tag from GitHub/Cloud Source/GitLab)
- Runs as a **service account** (grant it least privilege); Secret Manager integration for build secrets
- Caching (Kaniko cache / layer caching) for faster image builds
- Private pools (build inside your VPC to reach private deps)

**🚧 Gotchas:** the Cloud Build SA can be over-privileged (it can deploy!) — scope it. No caching → slow, costly builds. Secrets as plaintext substitutions instead of Secret Manager. Default logs bucket permissions.

**💸 Cost:** per build-minute by machine size (a free tier of build-minutes historically — verify).

**🚀 Project:** A `cloudbuild.yaml` that builds a container (Dockerfile *or* Buildpacks), runs tests, pushes to Artifact Registry tagged with the commit SHA, with layer caching and secrets pulled from Secret Manager.

**💬 Interview:** How are Cloud Build steps structured (containers)? How do you keep secrets out of a build? Why scope the build service account tightly?

---

# 5. Cloud Run Jobs + Cloud Batch

**🧠 Mental model:** For work that runs to completion rather than serving requests — **Cloud Run Jobs** for containerized tasks, **Cloud Batch** for large-scale/HPC batch scheduling.

**🔬 Granular concepts**

*Cloud Run Jobs* — run a container to completion; task arrays (parallel indexed tasks); retries; triggered by Scheduler/Workflows/manually. Great for migrations, ETL steps, scheduled processing.

*Cloud Batch* — managed batch scheduler: submit jobs, it provisions compute (incl. **Spot**), runs, and tears down; task groups, dependencies, arrays; integrates with Compute Engine machine types + GPUs; for HPC/rendering/large parallel workloads.

**🚧 Gotchas:** these are throughput/completion tools, not request/response. Spot interruptions → make jobs idempotent/retryable. Right-sizing task resources.

**💸 Cost:** the underlying compute (heavy Spot use = cheap); no premium for Batch orchestration.

**🚀 Projects**
- **A:** Run a data-processing container as a **Cloud Run Job** task array over many inputs.
- **B (different approach):** Submit a large parallel job to **Cloud Batch** on Spot VMs; add a dependency between task groups.

**💬 Interview:** Cloud Run Jobs vs Cloud Batch vs GKE Jobs vs Dataflow — deciding factors (scale, duration, parallelism, framework). Why run batch on Spot and what does idempotency buy you?

**✅ Mastery bar:** You pick the right run-to-completion tool for a job shape and run it cheaply on Spot.

---

# 🏔️ Volume 2 Capstone — "Container platform, three runtimes"
Ship the *same* containerized service three ways + write a decision memo:
1. **Cloud Run** (fastest, serverless, scale-to-zero, traffic-split deploys)
2. **GKE Autopilot** (Kubernetes without node ops)
3. **GKE Standard** (+ Spot node pool, full control)

For each: reproducible **Artifact Registry** image (scanned, Binary-Authorization-gated, cleanup-managed), **keyless** identity (Cloud Run SA / GKE **Workload Identity**), global LB / Gateway, autoscaling, **Cloud Operations** logs+metrics, and a safe deploy (traffic split / rolling+canary). The memo recommends one for a 5-person startup and a different one for a 200-engineer enterprise — and defends both.

## Decision cheat-sheet (memorize)
- **Cloud Run:** HTTP/event services, scale-to-zero, minimal ops → the GCP default for most services.
- **Cloud Run Jobs / Cloud Batch:** run-to-completion + large parallel batch (Spot).
- **GKE Autopilot:** need Kubernetes (ecosystem/portability) but not node management.
- **GKE Standard:** need node-level control, Spot pools, GPUs, custom daemonsets, tight cost tuning.
- **Compute Engine MIG (Vol 1):** plain VMs / legacy / non-container workloads.

## Cadence
~1–2 weeks per service (GKE deserves extra); capstone is a 2–3 week push. ~6–8 weeks at 5–10 hrs/week.
