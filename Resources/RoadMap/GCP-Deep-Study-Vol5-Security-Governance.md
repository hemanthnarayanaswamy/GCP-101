# GCP Deep Study Guide — Volume 5: Security & Governance

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** Cloud KMS, Secret Manager, Certificate Manager, Security Command Center, Cloud Armor, Identity Platform, Resource Manager, Organization Policy Service, **VPC Service Controls**, Cloud Identity / IAM Identity.

> Security threads through every volume; this one goes deep on the dedicated services. Governance (org hierarchy + policy) is where senior crosses into staff. Pairs tightly with Vol 1 IAM. **VPC Service Controls is a GCP-signature capability** with no clean AWS analog — pay attention.

---

# 1. Cloud KMS (Key Management Service)

**🧠 Mental model:** A hardened vault holding your master keys that never leave it — you send small things to encrypt/decrypt, or ask it for data keys to encrypt big things yourself (envelope encryption).

**🔬 Granular concepts**
- **Key rings** (regional/multi-region containers) → **keys** → **key versions** (rotation creates new versions)
- Purposes: symmetric encrypt/decrypt, asymmetric sign/verify, asymmetric encrypt, MAC
- **Envelope encryption** (core concept): KMS gives a **data encryption key (DEK)**; you encrypt data locally with it; KMS encrypts the DEK with the **key encryption key (KEK)**; store ciphertext + wrapped DEK. KMS never sees bulk data.
- **CMEK** (customer-managed keys) — encrypt GCS/BigQuery/PD/etc. with *your* KMS key; **CSEK** (customer-supplied)
- Automatic **rotation** (set a period; new version becomes primary); manual rotation
- IAM on keys (`roles/cloudkms.cryptoKeyEncrypterDecrypter`) — least privilege per key
- **Cloud HSM** (FIPS 140-2 L3 hardware-backed keys) and **Cloud External Key Manager (EKM)** (keys stay in an external/on-prem HSM — you hold the key off-Google)
- Destroy = scheduled (24h+) destruction of a key version (data becomes unrecoverable)

**🚧 Limits & gotchas**
- Over-tight or wrong **key IAM** locks services out of CMEK-encrypted data (e.g., the service agent needs encrypter/decrypter on the key).
- High-volume per-object KMS calls cost + can throttle → services use envelope encryption internally, but be aware.
- Destroying a key version makes CMEK-encrypted data unrecoverable (scheduled, cancelable window).
- Multi-region vs regional key location must match data residency needs.

**💸 Cost model:** per active key version/month + per crypto operation. HSM/EKM cost more.

**💥 Failure modes:** service can't read CMEK data → the **service agent** lacks encrypter/decrypter on the key. Throttling → too many direct ops. Data lost → key version destroyed.

**🚀 Projects**
- **A:** Create a key ring + key with **auto-rotation**; enable **CMEK** on a GCS bucket + a Cloud SQL instance; grant decrypt to exactly one SA; prove another SA can't read.
- **B (different approach — envelope in code):** Use KMS to wrap/unwrap a DEK, encrypt a large file locally, store ciphertext + wrapped DEK, then decrypt — envelope encryption end to end.
- **C — external control:** Explore **EKM/HSM** conceptually and document when a regulated workload needs keys off-Google.

**💬 Interview:** Explain envelope encryption to a new hire. CMEK vs CSEK vs Google-managed vs EKM — when each? Why might a service get "permission denied" reading CMEK data (service agent on the key)? How do you avoid destroying data via key deletion?

**✅ Mastery bar:** You explain envelope encryption cold, wire CMEK with correct service-agent IAM, and know when EKM/HSM is required.

---

# 2. Secret Manager

**🧠 Mental model:** A managed safe for secrets (API keys, DB creds, tokens) with versioning, IAM-gated access, and rotation — no secrets in code or config.

**🔬 Granular concepts**
- **Secrets** → **versions** (immutable; enable/disable/destroy versions; `latest` alias)
- IAM per secret (`roles/secretmanager.secretAccessor`) — least privilege
- Encryption at rest (Google-managed or **CMEK**)
- **Rotation** (rotation schedule + a Pub/Sub notification → your rotation function; not fully automatic like some managed DB rotations — you implement the rotation logic)
- Access from Cloud Run/Functions/GKE (mount as env/volume) / Compute; regional + automatic replication policies
- Audit logging of access

**🚧 Gotchas:** fetching the secret on every request from cold → cache it, refresh on new version. Granting `secretAccessor` too broadly. Rotation requires you to wire the rotation logic (schedule + notification, not magic). Choosing Secret Manager for plain non-secret config (fine, but it's paid per access).

**💸 Cost:** per active secret version/month + per access operation.

**🚀 Projects**
- **A:** Store a DB credential; grant `secretAccessor` to one SA; consume it from Cloud Run (mounted as env); rotate to a new version and confirm the app picks it up.
- **B (different approach — rotation):** Set a rotation schedule + Pub/Sub notification → a function that generates + stores a new version; verify the flow.

**💬 Interview:** How does Secret Manager versioning work? How do you stop fetching a secret on every call? How is GCP rotation different from a fully-managed DB rotation (you wire the logic)?

**✅ Mastery bar:** You never hardcode a secret, scope access per-secret, and can implement rotation.

---

# 3. Certificate Manager (+ managed SSL)

**🧠 Mental model:** Managed TLS certificates that provision + auto-renew and attach to load balancers — no manual cert wrangling.

**🔬 Granular concepts**
- **Google-managed certificates** (free, auto-renew) vs self-managed (upload your own)
- **Certificate Manager** (scales to many domains/wildcards, DNS authorization) vs the older per-LB managed cert
- DNS authorization (prove domain control via a DNS record; enables wildcards + pre-provisioning)
- Certificate maps (attach many certs to a load balancer via SNI)
- Attaches to the **global external Application LB** / other LBs
- mTLS (Trust Config) for client-cert auth

**🚧 Gotchas:** Google-managed cert provisioning needs DNS pointing correctly first (or DNS authorization) or it stays stuck "provisioning." Wildcards need Certificate Manager + DNS auth. Certs are LB-attached (not exportable for arbitrary servers).

**💸 Cost:** Google-managed certs are free; Certificate Manager has modest per-cert/entry pricing at scale.

**🚀 Project:** Issue a **Google-managed cert via DNS authorization** (including a wildcard); attach via a certificate map to a global external Application LB; confirm auto-renewal.

**💬 Interview:** Google-managed vs self-managed certs? Why might a managed cert stay stuck provisioning (DNS)? How do you serve many domains on one LB (certificate maps/SNI)?

**✅ Mastery bar:** You can wire auto-renewing TLS (incl. wildcards) to load balancers and debug provisioning.

---

# 4. Security Command Center (SCC)

**🧠 Mental model:** The central security + risk dashboard for your whole org — finds misconfigurations, vulnerabilities, and active threats across all projects, in one place. (Rolls up the roles AWS splits across GuardDuty + Security Hub + Inspector + Macie.)

**🔬 Granular concepts**
- Tiers: **Standard** (free-ish: Security Health Analytics misconfig scanning, asset inventory) vs **Premium/Enterprise** (adds **Event Threat Detection**, **Container Threat Detection**, **Virtual Machine Threat Detection**, **Web Security Scanner**, attack path analysis, compliance reports, and **SDP — Sensitive Data Protection/DLP** discovery)
- **Findings** (misconfig, vulnerability, threat) + severity; **Security Health Analytics** (CIS/PCI/etc. benchmarks)
- **Event Threat Detection** (analyzes logs for threats — crypto-mining, exfil, brute force)
- **Sensitive Data Protection (Cloud DLP)** — discover/classify/redact PII (the Macie analog)
- **Web Security Scanner** (app vuln scanning)
- Asset inventory + attack path simulation (Premium)
- Findings → **Pub/Sub/Eventarc** → automated remediation or SIEM/ticketing
- Org-level enablement; export to BigQuery/Chronicle

**🚧 Gotchas:** these **detect**, they don't **fix** — wire remediation. Premium/Enterprise is a real cost — scope. Some detections need specific logs enabled. Findings without an action plan = noise.

**💸 Cost:** Standard largely free; Premium/Enterprise are significant (org-wide, usage-based). DLP priced by data inspected.

**🚀 Projects**
- **A:** Enable SCC; review **Security Health Analytics** misconfig findings; fix the top 3 (e.g., a public bucket, an over-broad firewall).
- **B (different approach — auto-remediation):** Route a finding (e.g., public bucket) via **Pub/Sub → Cloud Function** that auto-remediates.
- **C:** Run **Sensitive Data Protection** over a bucket to discover PII; document what it found.

**💬 Interview:** What does SCC Standard vs Premium give you? Detection vs remediation. How do you auto-fix a public bucket end to end? How do you control SCC/DLP cost at scale?

**✅ Mastery bar:** You enable SCC org-wide, know the tier boundaries, and wire detection→remediation rather than just collecting findings.

---

# 5. Cloud Armor

**🧠 Mental model:** Edge WAF + DDoS protection attached to your global load balancer — filter malicious web traffic and absorb volumetric attacks at Google's edge.

**🔬 Granular concepts**
- **Security policies** attached to backend services on the global external Application LB
- Rules: IP allow/deny, geo, **preconfigured WAF rules** (OWASP Top 10 — SQLi/XSS/etc. via ModSecurity CRS), rate-based (throttle/ban), custom rules (CEL expressions on headers/paths/etc.)
- **DDoS protection**: always-on L3/L4 via Google's edge; **Cloud Armor Managed Protection Plus** (advanced L7 DDoS + Adaptive Protection ML)
- **Adaptive Protection** (ML detects + suggests rules for anomalous attacks)
- Preview mode (evaluate rules without enforcing); edge security policies; bot management (reCAPTCHA integration)

**🚧 Gotchas:** deploying rules straight to enforce can block legit traffic → use **preview mode** first, then enforce. Rate limits too low = false positives. Cloud Armor protects the LB/edge — lock the backend so it's only reachable via the LB.

**💸 Cost:** per policy + per rule + per request evaluated; Managed Protection Plus is a subscription.

**🚀 Project:** Attach a Cloud Armor policy to the global LB with **preconfigured OWASP rules** + a **rate-based** rule; run in **preview**, analyze logs, then enforce; enable **Adaptive Protection**.

**💬 Interview:** How does Cloud Armor relate to the global LB? Why deploy rules in preview first? What does Adaptive Protection add? How do you layer edge defenses?

**✅ Mastery bar:** You build layered L7 + DDoS protection and tune rules (preview → enforce) without breaking users.

---

# 6. Identity Platform (+ Firebase Auth)

**🧠 Mental model:** Managed end-user identity for your apps — sign-up/sign-in, MFA, social/SAML/OIDC federation, issuing tokens. (The Cognito analog; Firebase Auth is the same engine with a mobile/web SDK focus.)

**🔬 Granular concepts**
- User authentication: email/password, phone, social (Google/Apple/etc.), SAML/OIDC federation, anonymous
- Issues **Firebase/Google ID tokens (JWTs)**; MFA; blocking functions (auth hooks)
- Multi-tenancy (isolated user pools per tenant)
- Integrates with **API Gateway** / your backend for authZ (verify the JWT); with Firebase security rules for direct client access to Firestore/Storage
- Distinct from **Cloud Identity** (workforce/employee identity + SSO — see next card): Identity Platform = *your app's end users*; Cloud Identity = *your employees*

**🚧 Gotchas:** confusing Identity Platform (customer/end-user identity) with Cloud Identity/IAM (workforce). Token verification/refresh handling. Multi-tenant isolation setup.

**💸 Cost:** per monthly active user (tiered; free tier).

**🚀 Project:** Identity Platform with email + Google sign-in + MFA; protect an **API Gateway** route by verifying the ID token; a signed-in user calls the API with their JWT.

**💬 Interview:** Identity Platform vs Cloud Identity — who is each for? How do you protect an API with end-user JWTs? What is multi-tenancy for?

**✅ Mastery bar:** You can stand up end-user auth and gate a backend with verified JWTs, and you don't confuse it with workforce identity.

---

# 7. Resource Manager + Organization Policy Service (governance)

**🧠 Mental model:** Resource Manager is the **hierarchy** (Org → Folders → Projects); Organization Policy Service applies **guardrail constraints** across that hierarchy (the closest GCP analog to AWS SCPs — but constraint-based, not IAM-action-based).

**🔬 Granular concepts**

*Resource Manager*
- Organization (tied to Cloud Identity/Workspace) → **Folders** → **Projects**
- Projects hold resources, billing links, enabled APIs, quotas
- Labels + tags (tags can gate IAM conditions + org policies); liens (prevent deletion)

*Organization Policy Service*
- **Constraints** (predefined + custom) applied at org/folder/project → **inherited downward**
- Common guardrails: restrict resource **locations**, **disable SA key creation**, **enforce uniform bucket-level access**, **restrict external IPs on VMs**, restrict allowed VPC/shared-VPC, require OS Login, restrict allowed images
- **Boolean** vs **list** constraints; enforced/merged/inheritance rules; **custom org policies** (CEL)
- Difference from IAM: **org policies constrain *what configurations are allowed*** (e.g., "no external IPs," "only in EU"); **IAM controls *who can do what*.** They're complementary.

**🚧 Limits & gotchas**
- Org policies **constrain configuration**, they don't grant/deny IAM actions — people expect SCP-like action-blocking; some overlap but the model differs.
- A constraint at the org root affects **every** current + future project — a too-broad restriction breaks workloads.
- The **"disable service account key creation"** + **"restrict external IPs"** policies are among the highest-value guardrails — know them.
- Custom constraints (CEL) are powerful but need testing.

**💸 Cost:** free.

**🚀 Projects**
- **A:** Build Org → Folders (prod/dev) → Projects; apply org policies: **restrict locations to your region(s)**, **disable SA key creation**, **enforce uniform bucket-level access**; prove a violating action is blocked.
- **B (different approach — custom constraint):** Write a custom org policy (CEL) enforcing a naming/label standard and test it.

**💬 Interview:** How is the resource hierarchy structured and how do org policies inherit? Org Policy constraints vs IAM vs AWS SCPs — contrast. Which guardrails give the most security value? Why keep the org root's constraints minimal?

**✅ Mastery bar:** You design a folder/project hierarchy with high-value guardrails and can articulate org-policy-vs-IAM.

---

# 8. VPC Service Controls (a GCP-signature capability)

**🧠 Mental model:** A **data-exfiltration firewall around your GCP services** — draw a **service perimeter** so that even a principal with valid IAM permissions **cannot move data out** of the perimeter (e.g., copy from an in-perimeter BigQuery/GCS to an outside project). It defends against stolen-credential + insider exfiltration in a way IAM alone can't.

**🔬 Granular concepts**
- **Service perimeters** enclose projects + specified GCP APIs (GCS, BigQuery, etc.)
- Blocks data movement **across** the perimeter boundary even with valid IAM (IAM says "can read"; perimeter says "but not to outside")
- **Ingress/egress rules** (allow specific controlled flows across the boundary)
- **Access levels** (via Access Context Manager: require corporate IP / device / identity to reach in-perimeter resources)
- Bridges (share between perimeters); dry-run mode (observe violations without enforcing)
- Complements Private Google Access + private connectivity

**🚧 Gotchas:** notoriously easy to **break legitimate workflows** — always start in **dry-run** and analyze violations before enforcing. Cross-perimeter and CI/CD/analytics flows need explicit ingress/egress rules. Steep learning curve. It protects **GCP-API data paths**, not everything.

**💸 Cost:** no direct charge (part of the platform).

**🚀 Project:** Put a project's **BigQuery + GCS** in a **service perimeter** in **dry-run**; attempt to export data to an outside project and observe the violation logged; add a narrow egress rule for one legitimate flow; then enforce.

**💬 Interview:** What threat does VPC Service Controls address that IAM can't (exfiltration with valid creds)? Why always start in dry-run? How do ingress/egress rules + access levels work? (This question separates people who've run regulated GCP environments from those who haven't.)

**✅ Mastery bar:** You can design an exfiltration perimeter, roll it out safely via dry-run, and reason about why it's distinct from IAM.

---

# 9. Cloud Identity + IAM (workforce SSO)

**🧠 Mental model:** Cloud Identity is the **workforce directory** (employees, groups) that your Org is built on; combined with IAM it gives centralized SSO + group-based access across all projects — no per-project human accounts.

**🔬 Granular concepts**
- Cloud Identity / Google Workspace as the identity source for the Organization
- **Groups** (grant IAM roles to groups, not individuals — the scalable pattern)
- **Federation** with external IdPs (Okta/Entra/AD) via SAML/OIDC + directory sync
- **Workforce Identity Federation** (let external-IdP users access GCP without provisioning Google accounts)
- Context-Aware Access (device/IP conditions via Access Context Manager); IAP for app-level access
- Admin roles, 2SV/MFA enforcement, session controls

**🚧 Gotchas:** granting roles to individuals instead of groups → unmanageable at scale. Over-broad groups re-introduce least-privilege problems. Offboarding must remove group membership everywhere.

**💸 Cost:** Cloud Identity Free tier + premium; federation is included.

**🚀 Project:** Model workforce access with **groups** (e.g., `developers@`, `sre@`) granted least-privilege roles at folder level; (conceptually) federate an external IdP; add a Context-Aware Access level requiring corp network for a sensitive resource.

**💬 Interview:** Why grant IAM to groups not individuals? Cloud Identity vs Identity Platform. How do you onboard/offboard 500 engineers cleanly? What is Workforce Identity Federation?

**✅ Mastery bar:** You design group-based, federated, least-privilege workforce access and can explain context-aware controls.

---

# 🏔️ Volume 5 Capstone — "Governed, self-defending GCP organization"
Design (and build what's feasible) a secure org:
1. **Resource Manager** hierarchy (Org → prod/dev folders → projects) with high-value **Organization Policies** (restrict locations, **disable SA key creation**, enforce uniform bucket access, restrict external IPs).
2. **Cloud Identity + IAM** group-based least-privilege access (no individual grants, no human owner roles); federated IdP concept.
3. **Security Command Center** enabled org-wide; findings centralized; **Sensitive Data Protection** scanning key buckets.
4. At least one **automated remediation** (public-bucket auto-fix via finding → Pub/Sub → function).
5. **Cloud KMS** CMEK for sensitive data, **Secret Manager** for credentials, **Certificate Manager** TLS everywhere, **Cloud Armor** on the edge.
6. A **VPC Service Controls** perimeter (dry-run → enforced) around BigQuery + GCS to prevent exfiltration.

Deliver a one-page security architecture doc: the guardrails, detection→response flow, encryption strategy, the exfiltration perimeter design, and blast-radius reasoning behind the project boundaries.

## AWS→GCP map (memorize)
KMS→**Cloud KMS**; Secrets Manager→**Secret Manager**; ACM→**Certificate Manager**; GuardDuty+SecurityHub+Inspector+Macie→**Security Command Center (+ Sensitive Data Protection)**; WAF/Shield→**Cloud Armor**; Cognito→**Identity Platform**; Organizations→**Resource Manager**; SCPs→**Organization Policy Service** (constraint-based); IAM Identity Center→**Cloud Identity + IAM groups / Workforce Identity Federation**; (no clean AWS analog)→**VPC Service Controls**.

## Cadence
~1–2 weeks per service (KMS, Org Policy, and VPC Service Controls deserve extra); capstone is a 2–3 week push. ~8–10 weeks at 5–10 hrs/week.
