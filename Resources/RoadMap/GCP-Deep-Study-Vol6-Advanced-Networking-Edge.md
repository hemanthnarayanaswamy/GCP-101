# GCP Deep Study Guide — Volume 6: Advanced Networking & Edge

**Service-by-service, maximum granularity.** Same study-card format as Volume 1.
**In this volume:** Cloud DNS, Cloud CDN, Cloud Load Balancing (advanced/global), Shared VPC + VPC Peering, Network Connectivity Center, Private Service Connect, Cloud Interconnect + Cloud VPN, Cloud NAT.

> Vol 1 covered VPC basics + LB basics. This is the scale-and-edge layer: connecting many networks/projects, going global, going hybrid, and private service connectivity. Networking depth is what makes Cloud Architect interviews hard — and GCP's **global VPC + Shared VPC + Private Service Connect** model differs meaningfully from AWS.

---

# 1. Cloud DNS

**🧠 Mental model:** Google's global, anycast, 100%-SLA DNS — turns names into addresses and routes with policies, public and private (inside your VPCs).

**🔬 Granular concepts**
- **Public** managed zones (internet) vs **private** zones (resolve inside specified VPCs)
- Record types: A/AAAA, CNAME, MX, TXT, NS, SOA, SRV, CAA; **alias-like** behavior via routing policies
- **Routing policies:** **weighted round robin** (canary/split), **geolocation** (route by client region), **failover** (primary/backup with health checks) — GCP's answer to Route 53 policies
- **DNS peering** + **forwarding** (hybrid: forward queries to on-prem, or on-prem → GCP) via inbound/outbound **server policies**
- **DNSSEC**; **Cloud Domains** (registration); split-horizon (same name, public vs private zone)
- Response policies (override/blackhole records)
- TTL + propagation

**🚧 Gotchas:** high TTLs slow failover (clients cache old answers) — lower TTL on records you fail over. Split-horizon surprises (public vs private zone for same name). Hybrid resolution needs inbound/outbound server policies + forwarding rules. Health checks must be wired for failover routing.

**💸 Cost:** per managed zone/month + per query (cheap).

**🚀 Projects**
- **A — Failover DR:** Two regional endpoints; **failover routing policy** with health checks; kill primary, watch DNS shift; tune TTL for RTO.
- **B (different approach — geo + weighted):** Geolocation routing to region-nearest endpoints; a weighted policy sending 10% to a new version.
- **C — hybrid DNS:** Inbound + outbound server policies + forwarding so a VPC resolves a simulated on-prem zone and vice versa.

**💬 Interview:** Walk through Cloud DNS routing policies + a use case each. Public vs private zones + split-horizon. Why does TTL affect RTO? How do you resolve on-prem names from a VPC (forwarding/peering)?

**✅ Mastery bar:** You pick the right routing policy, design DNS-level failover, and set up hybrid resolution.

---

# 2. Cloud CDN

**🧠 Mental model:** Caches your content at Google's global edge (the same edge as Search/YouTube), fronting the **global external Application LB** — low latency, origin offload, edge security.

**🔬 Granular concepts**
- Enabled **on a backend service/bucket** of the global external Application LB (not a standalone product)
- Origins: backend buckets (GCS), backend services (MIGs/NEGs/Cloud Run)
- **Cache modes:** use origin headers / cache-all-static / force-cache-all; TTL overrides
- **Cache keys** (include/exclude host, path, query string, headers, cookies) — mis-tuning tanks hit ratio or leaks personalized content
- **Cache invalidation** (purge by path); prefer **versioned filenames** (cache-busting) over frequent invalidations
- Signed URLs / signed cookies (private content); negative caching
- **Media CDN** (a separate, higher-scale product for large-scale video/media streaming)
- Pairs with Cloud Armor (edge security) + Google-managed certs

**🚧 Gotchas:** caching everything blindly (or including cookies/auth headers in the cache key) → poor hit ratio or serving one user's personalized content to another. Invalidations take time + cost → version filenames. It's tied to the global LB (architecturally). Media CDN vs Cloud CDN for large video.

**💸 Cost:** cache egress (often cheaper than origin egress) + cache-fill + invalidations. Net savings via origin offload.

**🚀 Projects**
- **A — Private static site via CDN:** GCS backend bucket (uniform access, public-access-prevention) behind the global LB with **Cloud CDN** + Google-managed cert; signed URLs for authed content.
- **B (different approach — dynamic + cache tuning):** Cache a dynamic backend selectively; measure hit ratio; optimize the **cache key** (strip unneeded query strings) and adopt versioned filenames.

**💬 Interview:** How does Cloud CDN relate to the global LB? What's a cache key and why does it matter? Invalidation vs versioned filenames. Cloud CDN vs Media CDN.

**✅ Mastery bar:** You design an edge layer that maximizes cache hits, protects personalized content, and locks the origin.

---

# 3. Cloud Load Balancing (advanced)

**🧠 Mental model:** GCP's LB is **global and anycast** at L7 — one IP, users enter Google's network at the nearest edge, and traffic is routed to the best healthy backend worldwide. This is a genuine architectural advantage; understand it deeply.

**🔬 Granular concepts**
- The chain: **global anycast IP → forwarding rule → target HTTP(S) proxy → URL map (host/path routing) → backend service → backends (MIG/NEG) + health checks**
- Global external Application LB (cross-region backends, one IP) vs regional; internal Application/Network LBs
- **Cross-region load balancing + failover** (backends in multiple regions; automatic routing to healthy region)
- **NEG types:** zonal (VMs), **serverless** (Cloud Run/Functions), internet, hybrid (on-prem/other-cloud backends), Private Service Connect NEGs
- **Traffic management:** URL rewrites, header-based routing, traffic splitting (weighted, for canary), outlier detection, request mirroring
- **Balancing modes** (RATE / UTILIZATION / CONNECTION) — determines when a backend is "full"
- Cloud CDN + Cloud Armor + IAP all attach here; SSL policies; HTTP/2/3

**🚧 Gotchas:** wrong **balancing mode** → uneven load / overload. Health-check **source ranges must be allowed** in the firewall or all backends show unhealthy. Serverless backends need serverless NEGs. Global anycast means one IP worldwide — design mental model accordingly.

**💸 Cost:** forwarding-rule hours + data processed + inter-region + CDN/Armor.

**🚀 Projects**
- **A — Global multi-region:** Backends (MIGs) in two regions behind one global LB; simulate a regional failure and watch automatic cross-region failover.
- **B (different approach — traffic mgmt):** URL-map routing + weighted **traffic splitting** for a canary; add outlier detection + a header-based route.

**💬 Interview:** Why is GCP's LB global/anycast and what advantage does that give? NEG types? Balancing modes and how a wrong one hurts. Why must you allow health-check ranges?

**✅ Mastery bar:** You design global, multi-region, canary-capable load balancing and reason about NEGs + balancing modes + health checks.

---

# 4. Shared VPC + VPC Network Peering

**🧠 Mental model:** Two ways to connect networks. **Shared VPC** = one central "host" project owns the network and *shares its subnets* with many "service" projects (centralized network, decentralized workloads — a GCP-signature pattern). **VPC Peering** = connect two separate VPCs so they route privately.

**🔬 Granular concepts**

*Shared VPC*
- **Host project** (owns the VPC + subnets + firewall) + **service projects** (run resources *in* the shared subnets)
- Centralized network/security admin; workload teams stay in their own projects
- IAM: network-user role on subnets to service projects; separation of duties
- The recommended enterprise pattern for centralized networking

*VPC Network Peering*
- Connects two VPCs (same or different orgs) for private RFC-1918 routing
- **Non-transitive** (A↔B and B↔C ≠ A↔C); no overlapping CIDRs
- Each side controls its own firewall; route exchange

*Compare*
- Shared VPC = one network many projects use (centralized); Peering = distinct networks talk (federated). Different tools for different org shapes.

**🚧 Gotchas:** peering is **non-transitive** (a hub-VPC peering mesh doesn't chain — use Network Connectivity Center for transitivity). Overlapping CIDRs break peering. Shared VPC needs careful IAM (network-user) + org structure. Global VPC means plan CIDRs org-wide.

**💸 Cost:** peering + shared VPC have no direct charge; you pay for egress (cross-region) + NAT + Interconnect.

**🚀 Projects**
- **A — Shared VPC:** A host project sharing subnets to two service projects; deploy a VM in each service project into the shared network; centralize firewall in the host.
- **B (different approach — peering):** Peer two standalone VPCs; prove private connectivity and demonstrate that a third peered VPC is **not** transitively reachable.

**💬 Interview:** Shared VPC vs VPC Peering — when each? Why is peering non-transitive and what replaces it at scale? How does Shared VPC separate network admin from workload teams?

**✅ Mastery bar:** You choose Shared VPC vs peering by org shape and understand non-transitivity + CIDR planning.

---

# 5. Network Connectivity Center (NCC)

**🧠 Mental model:** A **hub-and-spoke** router for connecting many VPCs + hybrid sites through one logical hub — GCP's answer to the "peering doesn't scale / isn't transitive" problem (the Transit-Gateway-shaped capability).

**🔬 Granular concepts**
- A **hub** with **spokes** (VPCs, VPN tunnels, Interconnect attachments, router appliances)
- **VPC spokes** give transitive connectivity between attached VPCs (transitivity that peering lacks)
- Centralized hybrid connectivity (on-prem via VPN/Interconnect spokes reaches all VPC spokes)
- Route exchange across spokes; site-to-site data transfer
- Works with Cloud Router (BGP) for dynamic routing

**🚧 Gotchas:** overlapping CIDRs across spokes break routing (plan centrally). It's the tool when you've outgrown a peering mesh — know *why* you'd migrate. Understand which spoke types give transitivity.

**💸 Cost:** per spoke/hour + data transfer.

**🚀 Project:** Build an NCC **hub** with 3 **VPC spokes**; demonstrate **transitive** connectivity between spoke VPCs (which plain peering wouldn't allow); add a VPN spoke to a simulated on-prem.

**💬 Interview:** When do you outgrow VPC peering and reach for NCC? What transitivity does NCC provide that peering doesn't? How does hybrid fit as a spoke?

**✅ Mastery bar:** You can design transitive hub-and-spoke connectivity across many VPCs + hybrid, and explain why peering fails at scale.

---

# 6. Private Service Connect (PSC)

**🧠 Mental model:** Private, one-way doorways to consume services over Google's backbone using **internal IPs** — reach Google APIs, managed services (Cloud SQL), or a producer's published service, with **no external IPs, no internet, no peering**. (The PrivateLink-shaped capability.)

**🔬 Granular concepts**
- **PSC endpoints** (a private IP in your VPC that forwards to a target): to **Google APIs** (private access to all/bundle of Google APIs via an internal IP), to **published services** (a producer exposes a service behind an internal LB; consumers connect privately across VPCs/orgs), or to managed services (Cloud SQL, etc.)
- **Service attachments** (producer side) + endpoints (consumer side)
- **PSC vs Private Google Access:** PGA lets a private VM reach Google APIs via the default paths; **PSC** gives a *dedicated internal IP* endpoint (finer control, custom DNS, works across peering/orgs)
- Avoids exposing services publicly + avoids VPC peering's CIDR/transitivity issues (PSC is not affected by overlapping CIDRs the way peering is)
- Backends can be PSC NEGs on the LB

**🚧 Gotchas:** PSC vs Private Google Access vs PSC-for-Google-APIs — know which solves which. DNS setup so the service name resolves to the endpoint. Producer/consumer role separation for published services.

**💸 Cost:** per endpoint/hour + data processed.

**🚀 Projects**
- **A:** Create a **PSC endpoint for Google APIs** so private VMs reach Cloud Storage/other APIs via an internal IP (with custom DNS); no external IP, no NAT for that path.
- **B (different approach — published service):** Publish an internal service (behind an internal LB) via a **service attachment**; consume it from a second VPC/project through a PSC endpoint — no peering.

**💬 Interview:** PSC vs Private Google Access vs VPC peering — what each solves. How does a SaaS/producer expose a service privately to consumers (service attachments)? Why is PSC unaffected by overlapping CIDRs?

**✅ Mastery bar:** You use PSC to consume/publish services privately across VPCs/orgs and can distinguish it from PGA + peering.

---

# 7. Cloud Interconnect + Cloud VPN (hybrid)

**🧠 Mental model:** Two ways to connect on-prem to GCP — **Cloud VPN** (encrypted tunnels over the internet; quick, cheap) and **Cloud Interconnect** (dedicated private physical connection; consistent, fast, pricier). Combine for resilience.

**🔬 Granular concepts**
- **Cloud VPN:** **HA VPN** (two interfaces, 99.99% SLA, BGP) vs Classic VPN (legacy); IPsec tunnels; **Cloud Router** for dynamic BGP routing
- **Cloud Interconnect:** **Dedicated** (your own physical connection at a colo, 10/100 Gbps) vs **Partner** (via a service provider, more flexible bandwidth); **VLAN attachments**; lower egress pricing + consistent latency
- **Cloud Router** (BGP; advertises VPC routes to on-prem + learns on-prem routes); regional vs global dynamic routing mode
- Resilience: HA VPN as backup to Interconnect; dual Interconnect across locations; the documented topologies for 99.9% vs 99.99%
- **Cross-Cloud Interconnect** (private link to another cloud)

**🚧 Gotchas:** **Interconnect is private but not encrypted** (add HA VPN over it or MACsec for encryption) — same trap as AWS Direct Connect. Interconnect takes weeks to provision (not on-demand). A single connection = SPOF → pair with VPN/second Interconnect. BGP misconfig breaks dynamic routing. Dynamic routing mode (regional vs global) affects which routes propagate.

**💸 Cost:** VPN = per tunnel-hour + egress. Interconnect = port/attachment + (much lower) egress + provider fees. Interconnect pays off at sustained high volume.

**🚀 Projects**
- **A:** Stand up **HA VPN** (with **Cloud Router**/BGP) to a simulated on-prem (another VPC as on-prem); verify dynamic route exchange both ways.
- **B (different approach — design):** Document a **Dedicated Interconnect + HA VPN backup** topology feeding an NCC hub; explain the failure modes + why you'd run VPN over Interconnect for encryption.

**💬 Interview:** VPN vs Interconnect — cost, latency, setup time, resilience. Is Interconnect encrypted? What does Cloud Router/BGP do? Regional vs global dynamic routing mode. How do you make hybrid both fast and resilient?

**✅ Mastery bar:** You choose + combine VPN/Interconnect for the right cost/resilience, know Interconnect isn't encrypted, and can wire BGP.

---

# 8. Cloud NAT (recap + placement)

**🧠 Mental model:** Managed, scalable outbound NAT so instances with **no external IP** can reach the internet — no per-instance NAT gateways to run.

**🔬 Granular concepts**
- Regional, software-defined, auto-scaling (no NAT instances/gateways to manage)
- Per-VPC/per-subnet configuration; static vs auto-allocated NAT IPs; **port allocation** (min ports per VM — exhaustion causes connection failures under high concurrency)
- Logging; works with private GKE, private VMs
- Complements Private Google Access (Google APIs) + PSC (services) — NAT is for *general internet* egress only

**🚧 Gotchas:** **port exhaustion** under many concurrent connections → tune min-ports-per-VM / dynamic port allocation. Don't NAT traffic that should use Private Google Access/PSC (cheaper, private). Regional (one config per region).

**💸 Cost:** per NAT gateway-hour + per-GB processed.

**🚀 Project:** Give a subnet of no-external-IP VMs internet egress via **Cloud NAT**; generate high concurrency and observe/tune port allocation; confirm Google-API traffic uses Private Google Access instead of NAT.

**💬 Interview:** How does Cloud NAT differ from AWS NAT gateways (regional, managed, auto-scaling)? What causes NAT port exhaustion and how do you fix it? When should traffic use PGA/PSC instead of NAT?

**✅ Mastery bar:** You provide private-VM egress via Cloud NAT, tune port allocation, and route Google-API/service traffic off NAT.

---

# 🏔️ Volume 6 Capstone — "Global, segmented, hybrid GCP network"
Design (and build what's feasible) an enterprise network:
1. **Shared VPC** host project sharing subnets to service projects (centralized network admin).
2. **Network Connectivity Center** hub giving **transitive** connectivity across VPC spokes + a hybrid (**HA VPN**/Interconnect) spoke.
3. **Private Service Connect** so workloads consume Google APIs + a published internal service privately (no external IPs); **Private Google Access** + **Cloud NAT** (tuned) for the rest.
4. **Hybrid:** HA VPN with Cloud Router/BGP (and a documented Dedicated Interconnect + VPN-backup design); **Cloud DNS** hybrid resolution (forwarding/peering).
5. **Edge:** global external Application LB (multi-region backends, canary traffic splitting) + **Cloud CDN** (tuned cache key, OAC-equivalent private origin) + **Cloud Armor** (OWASP + rate limiting) + Google-managed certs; **Cloud DNS** failover/geo routing across regions.

Deliver a network architecture diagram + doc: segmentation choices, where every dollar of NAT/NCC/PSC/Interconnect cost goes, failure domains, and the hybrid resilience design.

## Decision cheat-sheet (memorize)
- **Shared VPC:** centralize the network; many projects use one host network.
- **VPC Peering:** connect a few VPCs privately; non-transitive; no overlapping CIDRs.
- **Network Connectivity Center:** transitive hub-and-spoke across many VPCs + hybrid (peering-at-scale replacement).
- **Private Service Connect:** privately consume/publish Google APIs + managed/producer services (internal IP, cross-org, CIDR-independent).
- **Private Google Access:** private VMs reach Google APIs via default paths (no external IP).
- **Cloud NAT:** general internet egress for no-external-IP instances.
- **Cloud DNS:** DNS + routing policies + hybrid resolution.
- **Cloud CDN / Media CDN:** cacheable content at the edge (fronting the global LB) / large-scale media.
- **Global external Application LB:** global anycast L7 with cross-region failover, NEGs, traffic management.
- **Cloud VPN vs Interconnect:** encrypted-over-internet vs dedicated-private (not encrypted; combine for resilience).

## AWS→GCP map
Route 53→**Cloud DNS**; CloudFront→**Cloud CDN/Media CDN**; ELB→**Cloud Load Balancing (global)**; Transit Gateway→**Network Connectivity Center**; PrivateLink→**Private Service Connect**; VPC peering→**VPC Peering / Shared VPC**; Direct Connect→**Cloud Interconnect**; Site-to-Site VPN→**Cloud VPN (HA VPN)**; NAT Gateway→**Cloud NAT**; Global Accelerator→**(largely subsumed by the global anycast LB)**.

## Cadence
~1–2 weeks per service (NCC, PSC, Interconnect deserve extra); capstone is a 2–3 week push. ~8–10 weeks at 5–10 hrs/week.
