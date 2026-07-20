# GCP Deep Study Guide — Volume 3: Infrastructure as Code & CI/CD

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** Terraform on GCP, Infrastructure Manager, Config Connector, Deployment Manager (legacy), Cloud Build, Cloud Deploy, GitHub Actions + Workload Identity Federation.

> On GCP, **Terraform is the de-facto IaC standard** (Google publishes the provider + modules and recommends it). This volume is where "I clicked it in the console" becomes "I rebuild the whole system from a git repo" — the biggest DevOps credibility multiplier.

---

# 1. Terraform on GCP (the primary IaC)

**🧠 Mental model:** Declarative, cloud-agnostic IaC that keeps a **state file** mapping config → real resources and computes an exact change **plan** before applying. The GCP standard.

**❓ Why it exists:** One IaC language/workflow across GCP + other providers, explicit plan/apply, and a huge module ecosystem (Google maintains the `google`/`google-beta` providers + the Cloud Foundation Toolkit modules).

**🔬 Granular concepts**
- HCL: `resource`, `data`, `variable`, `output`, `locals`, `module`, `provider`
- The **`google`** + **`google-beta`** providers; provider version pinning
- Lifecycle: `init` → `plan` → `apply` → `destroy`
- **State** (`terraform.tfstate`) = source of truth; plan symbols `+ - ~ -/+`
- **Remote state + locking** via a **GCS backend** (GCS supports object-level locking; historically DynamoDB-style lock tables aren't needed — GCS backend handles locking) — *essential for teams*
- **Modules** (reusable packages; the **Cloud Foundation Toolkit** provides Google-blessed modules for VPC, projects, GKE, etc.)
- `count` vs `for_each`; dynamic blocks; `depends_on`; lifecycle (`prevent_destroy`, `create_before_destroy`, `ignore_changes`)
- `terraform import`; drift detection via `plan`
- Auth: run Terraform as a **service account** via **impersonation** (no key file) — the secure pattern
- Project-factory pattern (Terraform vends new projects with baseline config)

**🎛️ Key knobs:** GCS backend config, variables (+ tfvars per env), `for_each`/`count`, lifecycle rules, provider versions, impersonation target.

**🚧 Limits & gotchas**
- **State is sacred:** losing it makes Terraform forget what it owns; committing it to git leaks secrets + causes conflicts. Use a **GCS backend** (with versioning on the bucket).
- Authenticating Terraform with a **downloaded SA key** is the anti-pattern — use **impersonation** / Workload Identity Federation.
- `google` vs `google-beta` provider: beta features need the beta provider (confusing "unknown field" errors otherwise).
- `terraform destroy` is brutally literal — wrong workspace/tfvars can wipe prod.
- Provider version bumps introduce breaking plan diffs.
- No native rollback — a failed apply can leave half-built infra (your responsibility).

**💸 Cost model:** Terraform CLI is free; you pay for resources + the GCS state bucket (trivial).

**💥 Failure modes:** state lock stuck → `force-unlock` carefully. Surprise replacement → a force-new attribute changed. "Already exists" → drift / created outside TF (use `import`). "Unknown field" → wrong provider (need beta).

**🚀 Projects**
- **A — VPC as code, keyless:** Build the Vol 1 VPC in Terraform with **GCS remote state (bucket versioning on)** and **service-account impersonation** (no key). Use `for_each` for subnets and a **CFT module** for the network.
- **B (different approach — modules + multi-env):** Refactor into a reusable `network` module; consume it for `dev` and `prod` with different tfvars.
- **C — recovery drills:** Cause + resolve a state lock; `import` a manually-created resource; add `prevent_destroy` to a database and prove `destroy` is blocked.

**💬 Interview questions**
- *Junior:* `plan` vs `apply`?
- *Mid:* What is Terraform state, why is it dangerous, and how do you run it safely on GCP (GCS backend)? How do you authenticate Terraform without a key?
- *Senior:* `google` vs `google-beta`? How do modules + `for_each` promote reuse? How do you handle drift/import?
- *Staff:* Design a safe multi-team, multi-project Terraform workflow: state isolation, PR-time plans, approvals, project factory, and drift detection across dozens of projects.

**✅ Mastery bar:** You treat state as sacred (GCS backend, keyless auth), structure with modules, and recover from lock/drift/import calmly.

---

# 2. Infrastructure Manager + Config Connector (GCP-native IaC)

**🧠 Mental model:** Two Google-native ways to do IaC without running your own Terraform pipeline: **Infrastructure Manager** (managed Terraform-as-a-service) and **Config Connector** (manage GCP resources *as Kubernetes objects*).

**🔬 Granular concepts**

*Infrastructure Manager (Infra Manager)*
- Google-hosted Terraform: you point it at Terraform config (in a repo/GCS); it runs plan/apply, stores state, and manages runs — no self-hosted CI runner for TF
- Runs as a service account; integrates with Cloud Build; deployment previews

*Config Connector*
- A GKE add-on (also underlies **Config Controller**) that lets you declare GCP resources as **Kubernetes CRDs** (e.g., a `StorageBucket` YAML) and reconciles them continuously
- GitOps for infrastructure (drift auto-corrected by the K8s reconcile loop)
- Part of the Anthos/Config Management story

*Deployment Manager (legacy)*
- GCP's original native IaC (YAML/Jinja/Python templates) — **largely superseded by Terraform + Infra Manager**; know it exists, don't start new projects on it.

**🚧 Gotchas:** Config Connector adds a running GKE dependency + a new mental model (infra as K8s objects). Infra Manager is convenient but still Terraform underneath (all TF caveats apply). Deployment Manager is legacy — don't invest.

**💸 Cost:** Infra Manager pricing is modest; Config Connector = the GKE cluster it runs on; Deployment Manager free.

**🚀 Project:** Deploy a bucket + service account two ways — once via **Infrastructure Manager** (managed Terraform) and once via **Config Connector** (a `StorageBucket` CRD) — and compare the reconcile/GitOps experience.

**💬 Interview:** Terraform vs Infra Manager vs Config Connector — when each? What does "infra as Kubernetes objects" buy you (continuous reconciliation)?

**✅ Mastery bar:** You can choose self-run Terraform vs managed (Infra Manager) vs GitOps-reconciled (Config Connector) deliberately.

---

# 3. Cloud Build (CI/CD)

**🧠 Mental model:** The orchestrator + build engine — runs your pipeline steps (each a container), builds/tests/deploys, and triggers on repo events. GCP's core CI service.

**🔬 Granular concepts**
- `cloudbuild.yaml`: **steps** (containerized commands, run in order; `waitFor` for parallel/DAG), `images`, `artifacts`, `substitutions`, `options`, `timeout`
- Triggers (GitHub/GitLab/Cloud Source Repos; on push/PR/tag) with included/ignored file filters
- Build **service account** (least privilege; it can deploy, so scope it)
- Secret Manager integration; **private pools** (build inside your VPC)
- Caching (Kaniko/layer); artifact + test result storage
- Deploy actions: to Cloud Run, GKE, App Engine; or hand off to **Cloud Deploy**
- Approvals (manual approval step)

**🚧 Gotchas:** over-privileged build SA (can deploy to prod!); no caching → slow/costly; secrets as plaintext substitutions; missing test step = no safety.

**💸 Cost:** per build-minute by machine size (free-tier minutes historically — verify) + private pool cost.

**🚀 Project:** Pipeline: source trigger → build+test → push to Artifact Registry (SHA tag) → deploy to a **staging** Cloud Run → **manual approval** → hand off to Cloud Deploy for prod. Include caching + Secret Manager.

**💬 Interview:** How are Cloud Build steps structured? What makes a pipeline safe to auto-deploy? Why scope the build SA tightly? Where do approvals belong?

---

# 4. Cloud Deploy

**🧠 Mental model:** Managed **continuous delivery** — define a delivery pipeline with ordered targets (dev → staging → prod) and progressive rollout strategies, with promotion + rollback.

**🔬 Granular concepts**
- **Delivery pipeline** + **targets** (each an environment: Cloud Run / GKE / GKE Enterprise)
- **Releases** (an immutable artifact promoted through targets) + **rollouts**
- Progressive deployment: **canary** (percentage-based, verify, then full) + phased rollouts
- **Approvals** between targets; automated **verification** (run tests as a rollout phase)
- **Rollback** to a previous release; deploy hooks (pre/post)
- Integrates with Skaffold (rendering manifests) for GKE

**🚧 Gotchas:** canary needs traffic-splitting support in the target (Cloud Run revisions / GKE); verification steps must actually gate; understand releases are immutable (redeploy = new release).

**💸 Cost:** priced per target/rollout (modest) + the underlying deploy compute.

**🚀 Project:** A Cloud Deploy pipeline: dev → staging → prod with a **canary** (25%→100%) to Cloud Run (or GKE), a **verification** phase, a **manual approval** before prod, and a demonstrated **rollback** from a bad release.

**💬 Interview:** Canary vs phased vs all-at-once — tradeoffs. How does Cloud Deploy separate build (Cloud Build) from delivery? How does rollback work with immutable releases?

**✅ Mastery bar:** You can build a promotion pipeline with canary + verification + approvals + rollback, cleanly separated from the build stage.

---

# 5. GitHub Actions + Workload Identity Federation (the keyless real-world path)

**🧠 Mental model:** Many teams run CI in GitHub Actions/GitLab, not Cloud Build. The key GCP skill is letting Actions deploy to GCP **without a service-account key** — using **Workload Identity Federation** so a short-lived GitHub OIDC token impersonates a GCP service account.

**🔬 Granular concepts**
- The problem: storing a **service-account JSON key** as a repo secret = long-lived, leakable credential (Google's most-warned-against pattern)
- The fix: **Workload Identity Federation** — a **workload identity pool** + an **OIDC provider** (GitHub) trust GitHub's tokens; a mapping lets those tokens **impersonate** a deploy SA
- **Attribute conditions/mappings** scope trust to your repo/branch/environment (e.g., `assertion.repository == 'org/repo'`) — critical to avoid *any* repo impersonating your SA
- `google-github-actions/auth` action exchanges the OIDC token for short-lived GCP credentials (no key)
- Grant the impersonated SA least-privilege roles
- GitHub Environments + required reviewers (approval gate on the GitHub side)

**🚧 Gotchas:** attribute condition too loose → any GitHub repo can assume your SA (severe). Forgetting to grant `roles/iam.workloadIdentityUser` on the SA to the federated principal. Over-broad deploy SA.

**🚀 Project:** A GitHub Actions workflow that, on merge to `main`, uses **Workload Identity Federation** (no key) to impersonate a deploy SA scoped **only** to your repo+branch, builds + pushes to Artifact Registry, and deploys to Cloud Run.

**💬 Interview:** Why is Workload Identity Federation better than a downloaded SA key? How do you scope the trust so only your repo/branch can impersonate the SA? Cloud Build vs GitHub Actions — when each?

**✅ Mastery bar:** You can build a fully keyless external CI→GCP pipeline and scope federation tightly.

---

## Terraform vs Infra Manager vs Config Connector vs Deployment Manager (interview gold)
- **Terraform (self-run):** the standard; full control; you run plan/apply in CI; manage GCS state.
- **Infrastructure Manager:** managed Terraform (Google runs it + stores state) — less pipeline to build.
- **Config Connector:** infra-as-Kubernetes-objects with continuous reconciliation (GitOps, drift auto-heal); needs GKE.
- **Deployment Manager:** legacy native IaC — don't start new work on it.
- Honest take: standardize on **Terraform** (or Infra Manager if you want it managed); pick one and go deep.

---

# 🏔️ Volume 3 Capstone — "One commit to production, the right way"
Put your Vol 2 containerized service under full GitOps on GCP:
1. **All infra in Terraform** (GCS backend + versioning, **keyless** via impersonation, modules incl. CFT).
2. **CI** (Cloud Build *or* GitHub Actions) builds, tests, scans, and pushes a SHA-tagged image to **Artifact Registry** — with **no SA keys** (Workload Identity Federation if GitHub; scoped build SA if Cloud Build), and **Binary Authorization** gating.
3. **CD** via **Cloud Deploy**: dev → staging (auto) → **manual approval** → prod with a **canary** rollout + **verification** + demonstrated **rollback**.
4. A deliberately broken release demonstrates the rollback live.
5. README lets a stranger clone and deploy the whole system from scratch.

This single repo is the most persuasive artifact you can show a GCP DevOps interviewer.

## Cadence
~1–2 weeks per tool (Terraform gets extra); capstone is a 2–3 week push. ~7–9 weeks at 5–10 hrs/week.
