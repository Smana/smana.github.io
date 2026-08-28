# Multicloud Strategy Blog Post — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish one consolidated multicloud-strategy post (EN + FR) — "One repo, two clouds: a pragmatic multicloud strategy" — grounded in the running cloud-native-ref implementation, covering drivers, abstraction doctrine, GPU consumption strategy, fleet-scaling stance, and resilience posture.

**Architecture:** A single standalone post (~4,500–5,500 words) where each strategy pillar is treated at the same altitude: position → reasoning → one running proof → pointer to docs. Heavy engineering (DR chaos test, Cluster Mesh, Sveltos vending build, GPU benchmarks) is explicitly deferred to a deep-dive backlog announced in the post's conclusion. English is written first, French is a full translation task.

**Tech Stack:** Hugo 0.156.0 extended (hugo-clarity theme, TOML front matter, page bundles), cloud-native-ref (EKS + GKE, Flux, Crossplane, Cilium, VictoriaMetrics), crossplane-configuration (KCL compositions, OCI packages), vLLM `InferenceService`.

---

## Status

- **Date:** 2026-08-28
- **Author:** Smaine Kahlouch (with Claude)
- **State:** Plan reviewed and researched; execution not started. Publication PR to be opened by the author when ready.

## Research annex — verified claims and sources (as of 2026-08-28)

Every factual claim below was verified against a primary or recent secondary source. **Re-verify anything time-sensitive (pricing, versions) at drafting time** — this annex records what was true when the plan was written.

### Regulatory / sovereignty drivers

| Claim | Detail | Source |
|---|---|---|
| EU Data Act applicability | In force 2024-01-11, fully applicable since **2025-09-12** | [cloudmagazin.com](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/) |
| Egress/switching fee ban | From **2027-01-12** EU cloud providers may no longer charge switching or egress fees | same |
| DORA | In force since **2025-01-17**. Art. 28: concentration-risk assessment + exit "without undue disruption". Art. 30: contractual exit/transition provisions. Regulators expect **tested** transition plans, not perfect ones. Hyperscalers designated Critical ICT Third-Party Providers under direct EU oversight | [schneiderdowns.com](https://schneiderdowns.com/our-thoughts-on/doras-approach-to-exit-strategy-and-termination/), [openmetal.io](https://openmetal.io/resources/blog/what-doras-ict-concentration-risk-requirements-mean-for-eu-financial-infrastructure/) |
| AWS European Sovereign Cloud | Launched **2026-01-15** (Brandenburg, standalone German GmbH, €7.8B) | [digital-chiefs.de](https://www.digital-chiefs.de/en/digital-sovereignty-2026-gaia-x-delos-cloud-and-europes-response-to-the-cloud-ac/) |

### The hook incident (verified)

**2025-10-20 AWS us-east-1 outage**: latent race condition in DynamoDB's automated DNS management wiped DynamoDB DNS records; ~14h total (DNS restored in ~3h, EC2 DropletWorkflow Manager congestive collapse added 5+h). Because us-east-1 hosts global control-plane services (IAM, STS), impact exceeded the region. Sources: [ThousandEyes analysis](https://www.thousandeyes.com/blog/aws-outage-analysis-october-20-2025), [techupkeep.dev timeline](https://www.techupkeep.dev/blog/aws-outage-october-2025-analysis).

### GPU cost & availability (verify pricing again at drafting time)

- Spot/preemptible: up to ~90% off on-demand, but **wrong for real-time inference endpoints**; fits interruptible batch, fine-tuning, checkpointed training. [thundercompute.com](https://www.thundercompute.com/blog/cloud-gpu-spot-instance-availability)
- Best practice is a **portfolio**: committed/reserved for 24/7 baseline, on-demand for burst, spot for interruptible batch. [cloudmagazin.com FinOps GPU 2026](https://www.cloudmagazin.com/en/2026/04/03/ai-inference-costs-cloud-finops-gpu-workloads-2026/)
- Reservation mechanisms: AWS **EC2 Capacity Blocks for ML** (published per-accelerator-hour rates, e.g. P5 $5.191/acc/h as of 2026-07-01) vs GCP **Dynamic Workload Scheduler** (flex-start / calendar). [cast.ai GPU pricing 2026](https://cast.ai/blog/gpu-cloud-pricing/)
- B200 supply constrained through mid-2026; average production GPU utilization ~5% (Cast AI 2026 report) — scale-to-zero is the biggest single lever.

### Fleet management (alternatives considered)

- **Sveltos**: management-cluster controller; label-matched `ClusterProfiles` distribute add-ons/config; event framework; Flux integration since v0.23 (Flux syncs Git → Sveltos distributes). Crossplane + Sveltos "GitOps bridge" pattern is documented upstream. [projectsveltos.io](https://projectsveltos.io/main/), [GitOps bridge example](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/)
- **Karmada**: CNCF incubating, most mature in category, **workload scheduling** across clusters — more than this platform needs. **OCM**: CNCF sandbox, governance-oriented, basis of RH ACM. [CNCF comparison](https://www.cncf.io/blog/2022/09/26/karmada-and-open-cluster-management-two-new-approaches-to-the-multicluster-fleet-management-challenge/), [Brian Grant's fleet survey](https://itnext.io/managing-applications-across-fleets-of-kubernetes-clusters-b71b96764e41)
- Position for the post: at N=2 static clusters, per-cluster Flux trees win; Sveltos becomes relevant at fleet scale for **add-on distribution + cluster vending**, complementing (not replacing) Flux.

### Resilience best practices (review corrections — MUST appear in the post)

1. **GitOps is not a backup.** Flux rebuilds declarative resources only — not volume data or runtime state. The post must state the platform's explicit state boundary: PostgreSQL protected via CNPG **WAL shipping to object storage** (replica clusters possible S3→GCS), object stores are the cross-cloud-accessible state layer, Valkey caches are accepted-loss. Therefore no Velero **by design** — say it explicitly, reviewers will ask. [plural.sh DR guide](https://www.plural.sh/blog/kubernetes-disaster-recovery-guide/)
2. **DNS is the residual concentration risk.** ADR-0019 deliberately keeps a single Route53 public zone — in a "lose AWS" scenario, DNS repointing depends on the failed provider (and Route53's control plane lives in us-east-1). Name this honestly as a known trade-off.
3. **Tested runbooks + periodic drills** are the DORA-era bar (quarterly drills are common guidance). "Failover is a runbook, not a button" — and an untested runbook doesn't count.

### Cilium Cluster Mesh (for the backlog teaser only)

Requirements: non-overlapping pod CIDRs across clusters, node-level IP connectivity (VPN/peering), WireGuard on **all** meshed clusters (no mixed mode, UDP 51871), KVStoreMesh default in current releases. [cilium docs](https://docs.cilium.io/en/stable/network/clustermesh/)

### CloudNativePG replica clusters (for resilience section + backlog)

Replica cluster in a second Kubernetes cluster bootstraps from the barman object store and keeps replaying WAL from it; declarative controlled switchover; RPO bounded by WAL-archiving cadence. [CNPG replica clusters](https://cloudnative-pg.io/docs/1.27/replica_cluster/)

## Writing conventions (inlined — the `ogenki-blog-style` / `hugo-new-post` skills referenced by older specs no longer exist)

- **Canonical structure:** Hook/Lead (concrete problem, never an abstract definition) → 🎯 Objectives → reference-repo notice → concept sections (lean) → hands-on proofs → 💭 Final thoughts (reflection, not summary) → 🔖 References.
- **Voice:** "we" for exploration, "you" for instructions, "I" for personal experience/caveats; concise, theory-first, precise terminology; no weak analogies; honesty over polish; occasional self-deprecating aside.
- **Terms:** every technical term **bold** on first use with a brief definition; acronyms spelled out on first mention (DORA, RTO, RPO, WAL…). French post keeps English technical terms untranslated.
- **Formatting:** TOML front matter (`+++`), `usePageBundles = true`, `toc = true`, emoji-decorated H2s, `{{% notice %}}` boxes, `{{< img src="..." >}}` for images (src relative to bundle).
- **Link policy:** main project link is **https://cnref.ogenki.io**; GitHub only for source deep-links. EN internal links are root-relative `/post/...` (never `/en/post/...`); FR uses `/fr/post/...`.
- **Scope guard:** the in-progress FR draft `llm-platform-abstraction` owns the InferenceService abstraction-design story — this post cites the InferenceService only as a *portability and cost* case study.

## File structure

- Create: `content/en/post/multicloud-platform/index.md` — the EN post (page bundle)
- Create: `content/en/post/multicloud-platform/thumbnail.png` — thumbnail (exported by author)
- Create: `content/en/post/multicloud-platform/multicloud-overview.png` — diagram 1 (exported from mermaid source in Task 7)
- Create: `content/en/post/multicloud-platform/state-boundary.png` — diagram 2 (exported from mermaid source in Task 7)
- Create: `content/fr/post/multicloud-platform/index.md` — the FR translation (bundle reuses images via copy)
- Create: `docs/superpowers/plans/2026-08-28-multicloud-strategy-post-inputs.md` — measured inputs produced by Task 1 (external)

---

### Task 1: Collect measured inputs from cloud-native-ref (EXTERNAL PREREQUISITE)

This task runs against the **cloud-native-ref platform**, not this repo. It produces the only new data the post needs.

**Files:**
- Create: `docs/superpowers/plans/2026-08-28-multicloud-strategy-post-inputs.md`

- [ ] **Step 1: Capture the same-claim proof.** In the `crossplane-configuration` repo, diff the AWS and GCP example claims and save the output:

```bash
cd ~/Sources/crossplane-configuration
diff -u examples/inferenceservice-basic.yaml examples/inferenceservice-gcp.yaml
diff -u examples/sqlinstance-basic.yaml examples/sqlinstance-gcp.yaml
```

Expected: small diffs (a provider block / storage class), proving the claim surface is near-identical. Paste both diffs verbatim into the inputs file.

- [ ] **Step 2: Measure cost per million tokens on both clouds.** For one model (pick the primary coder model), record for each cloud: GPU type, effective hourly instance cost (on-demand price at measurement date), and sustained generation throughput (tokens/s) from vLLM metrics (`vllm:generation_tokens_total` rate over a 10-minute steady load window in VictoriaMetrics). Compute:

```
cost_per_Mtok = (hourly_cost_usd / (throughput_tok_s * 3600)) * 1_000_000
```

- [ ] **Step 3: Write the inputs file** with exactly this structure:

```markdown
# Inputs — multicloud strategy post (measured YYYY-MM-DD)

## Same-claim diffs
### inferenceservice (aws vs gcp)
<verbatim diff>
### sqlinstance (aws vs gcp)
<verbatim diff>

## Cost per million tokens
| Cloud | GPU | Instance | $/h (on-demand) | tok/s | $/Mtok |
|---|---|---|---|---|---|
| AWS EKS | ... | ... | ... | ... | ... |
| GCP GKE | ... | ... | ... | ... | ... |
Measurement window: <date, duration, model, quantization>
```

- [ ] **Step 4: Commit**

```bash
git add docs/superpowers/plans/2026-08-28-multicloud-strategy-post-inputs.md
git commit -m "docs(multicloud-post): add measured inputs (claim diffs, cost/Mtok)"
```

### Task 2: Scaffold the EN post

**Files:**
- Create: `content/en/post/multicloud-platform/index.md`

- [ ] **Step 1: Create the page bundle with this exact front matter and section skeleton:**

```markdown
+++
author = "Smaine Kahlouch"
title = "One repo, two clouds: a pragmatic `multicloud` strategy"
date = "2026-09-15"
draft = true
summary = "**Multicloud** is usually a slide, rarely a repo. A strategy walkthrough of a platform that actually runs on **AWS EKS and GCP GKE** from one Git repository: what to abstract, what to leave cloud-shaped, how to buy GPUs, and what resilience really requires."
featured = true
codeMaxLines = 25
usePageBundles = true
toc = true
tags = [
    "multicloud",
    "platform-engineering",
    "crossplane",
    "gitops",
    "gpu"
]
thumbnail = "thumbnail.png"
+++

<!-- Hook / lead -->

## 🎯 Goals of this article

## 🌍 Why multicloud — the honest version

## 🧭 The doctrine: abstract the developer, not the platform

## 🌳 One Flux tree, two clusters

## 🔗 The seams: identity, DNS, secrets

## 🎮 GPUs: a consumption strategy for cost and availability

## 📈 When two clusters become a fleet

## 🛡️ The resilience posture

## 💭 Final thoughts: what it really cost

## 🔖 References
```

- [ ] **Step 2: Verify the skeleton builds.**

Run: `hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK`
Expected: `OK` (draft pages don't need to render; the build must not error)

- [ ] **Step 3: Commit**

```bash
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): scaffold EN multicloud strategy post"
```

### Task 3: Hook, goals, reference-repo notice

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md` (lead + first two sections)

- [ ] **Step 1: Write the lead (~150 words).** Open on the **2025-10-20 us-east-1 outage** (facts from the research annex: DynamoDB DNS automation race, ~14h, IAM/STS global blast radius) — then the pivot sentence: "Multicloud is usually a slide. Rarely a repo." State what the reader gets: a strategy derived from a platform that actually runs on both clouds, with its trade-offs measured. No abstract multicloud definition.

- [ ] **Step 2: Write 🎯 Goals** as bullets: the real 2026 drivers (and the counter-case) · what to abstract vs leave cloud-shaped · one Flux tree, two clusters · a GPU consumption strategy with measured cost per million tokens · when a fleet needs more than Flux · what surviving a cloud actually requires · what it really cost.

- [ ] **Step 3: Add the reference-repo notice** right after goals (mirrors other posts):

```markdown
{{% notice tip "The reference repo" %}}
Everything in this article runs today in the <strong><a href="https://cnref.ogenki.io">Cloud Native Ref</a></strong> project (<a href="https://github.com/Smana/cloud-native-ref">source on GitHub</a>): the same platform on AWS EKS and GCP GKE, reconciled from one repository and one Flux tree — not a fork each. Compositions live in a dedicated repo, <a href="https://github.com/Smana/crossplane-configuration">crossplane-configuration</a>.
{{% /notice %}}
```

- [ ] **Step 4: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): write lead, goals and repo notice"
```

### Task 4: Section "Why multicloud — the honest version" (~700 words)

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md`

- [ ] **Step 1: Write the drivers, ranked, each with its annex fact + source link:** (1) regulatory exit capability — DORA Art. 28/30 "tested transition plans", EU Data Act fee ban 2027-01-12; (2) sovereignty — AWS European Sovereign Cloud, EU sovereignty package; (3) **GPU capacity** as the new driver — scarcity, capacity shopping, portability as pricing leverage; (4) resilience/blast-radius — the us-east-1 incident from the lead; (5) the unglamorous majority: M&A and organic sprawl.

- [ ] **Step 2: Write the counter-case (do not skip):** the complexity tax (two IAM models, two networking stacks, double the quotas and quirks), the lowest-common-denominator trap, egress. Position statement: multicloud is a **capability built selectively**, not a place you move to. Most teams need exit-*capability* and portability at the API layer, not symmetric active-active everything.

- [ ] **Step 3: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): write drivers and counter-case section"
```

### Task 5: Sections "Doctrine" + "One Flux tree" + "Seams" (~1,400 words total)

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md`

- [ ] **Step 1: Write 🧭 Doctrine (~500 words).** Core thesis from ADR-0007, quote it: *"An API that looks neutral but is not causes worse errors than one that is visibly cloud-shaped."* Developer-facing claims stay neutral (`App`, `SQLInstance`, `InferenceService`); platform-facing APIs stay cloud-shaped as sibling XRDs (`EPI` vs `GCPWorkloadIdentity` — there is no neutral form of an IAM policy document); escape-hatch `aws {}` / `gcp {}` blocks. Paste the **same-claim diff** from the Task 1 inputs file as the proof. Link [ADR-0007](https://cnref.ogenki.io/docs/decisions/0007-cloud-abstraction-boundaries/).

- [ ] **Step 2: Write 🌳 One Flux tree (~400 words).** `clusters/aws-0` / `clusters/gcp-0` share bases, diverge in overlays; the composition packages `crossplane-configuration-{core,aws,gcp}` (OCI tag = git tag) select per cloud; what a PR touching both clouds looks like. Insert diagram 1: `{{< img src="multicloud-overview.png" alt="One Git repository reconciled by Flux into EKS and GKE, with Crossplane configuration packages per cloud" width="1000" >}}`.

- [ ] **Step 3: Write 🔗 Seams (~500 words).** The three places clouds must touch, each one paragraph: **identity** — GKE workloads assume an AWS IAM role via `AssumeRoleWithWebIdentity`, zero static keys (`opentofu/shared/aws-gcp-federation`); **DNS** — one Route53 zone, `gcp.cloud.ogenki.io` scoping, per-hostname certs, cert-manager DNS-01 from GKE ([ADR-0019](https://cnref.ogenki.io/docs/decisions/0019-cross-cloud-dns-federation/)); **secrets** — portable store names ([ADR-0023](https://cnref.ogenki.io/docs/decisions/0023-portable-secret-store-names/)) on cloud-managed stores ([ADR-0024](https://cnref.ogenki.io/docs/decisions/0024-cloud-managed-secret-stores/)); one IdP ([ADR-0022](https://cnref.ogenki.io/docs/decisions/0022-single-identity-provider-across-clouds/)).

- [ ] **Step 4: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): write doctrine, flux-tree and seams sections"
```

### Task 6: Section "GPUs: a consumption strategy" (~800 words)

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md`

- [ ] **Step 1: Write the strategy content** using annex facts: pricing-model portfolio mapped to workload profiles (committed = 24/7 baseline; on-demand = burst; spot = interruptible batch **only**, never real-time endpoints); reservation mechanisms compared in 2–3 sentences (Capacity Blocks vs Dynamic Workload Scheduler); right-sizing (L4 class vs A100/H100 class per model size); **KEDA scale-to-zero as the biggest lever** (cite ~5% average GPU utilization industry figure); portability as capacity hedge and negotiating leverage.

- [ ] **Step 2: Paste the measured cost table** from the Task 1 inputs file, with the measurement-date caveat sentence: prices go stale — the method is the durable part. Add the formula in a short code block.

- [ ] **Step 3: Add the scope-guard link:** one sentence pointing readers to the [llm-self-hosted-stack post](/post/series/agentic_ai/llm-self-hosted-stack/) for how the inference layer itself works.

- [ ] **Step 4: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): write GPU consumption strategy section"
```

### Task 7: Sections "Fleet" + "Resilience" (~900 words total) and diagrams

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md`
- Create: `content/en/post/multicloud-platform/multicloud-overview.png`, `content/en/post/multicloud-platform/state-boundary.png`

- [ ] **Step 1: Write 📈 Fleet (~400 words).** Honest position first: at two static clusters, per-cluster Flux trees are correct and a fleet manager is overhead. Trigger for changing the model: per-team / ephemeral / dynamically-created clusters. The shape it takes: label-matched distribution (**Sveltos** ClusterProfiles) instead of copied overlays, and cluster vending via the [Crossplane + Sveltos GitOps bridge](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/); ownership line: Flux syncs Git, Sveltos distributes across the fleet. **Alternatives-considered paragraph** (from annex): Karmada = workload scheduling (more than needed), OCM = governance/RH ACM; Sveltos fits because the need is add-on distribution + eventing, composing with Flux.

- [ ] **Step 2: Write 🛡️ Resilience (~500 words).** Structure: (a) **GitOps is not a backup** — state the platform's state boundary explicitly: PostgreSQL via CNPG WAL shipping to object storage (replica clusters S3→GCS possible, declarative switchover, RPO = WAL cadence), object stores as the cross-cloud state layer, Valkey caches accepted-loss, hence no Velero *by design*; (b) stateless workloads rebuilt by Flux in minutes; (c) traffic repointing through the shared zone — and the honest residual risk: **the single Route53 zone is itself a concentration point** (ADR-0019 trade-off, control plane in us-east-1); (d) failover is a **tested runbook**, not a button — name periodic drills as the bar DORA-era reviewers expect. Close: live cross-cluster interconnection (Cluster Mesh) is a cost most platforms don't need — one sentence, backlog teaser.

- [ ] **Step 3: Author the two diagrams.** Export PNGs from these mermaid sources (author tool of choice — excalidraw/draw.io restyling welcome, content must match):

```
flowchart LR
  G["Git — cloud-native-ref (one Flux tree)"] --> A["EKS · aws-0"]
  G --> P["GKE · gcp-0"]
  X["crossplane-configuration OCI packages (core/aws/gcp)"] --> A
  X --> P
  A ---|"one Route53 zone + keyless OIDC federation"| P
```

```
flowchart TB
  subgraph state["Stateful — has its own path"]
    PG["PostgreSQL — CNPG WAL → object store (S3→GCS)"]
    OBJ["Object stores — cross-cloud accessible"]
  end
  subgraph rebuild["Rebuilt by GitOps"]
    APPS["Stateless workloads — Flux, minutes"]
  end
  subgraph lost["Accepted loss"]
    KV["Valkey caches"]
  end
```

- [ ] **Step 4: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/
git commit -m "docs(multicloud): write fleet and resilience sections, add diagrams"
```

### Task 8: Final thoughts + references (~400 words)

**Files:**
- Modify: `content/en/post/multicloud-platform/index.md`

- [ ] **Step 1: Write 💭 Final thoughts** — reflection, not summary: what the second cloud actually cost (time, quota fights, where abstractions leaked); what stays deliberately unsolved. End with the deep-dive backlog as bullets and an invitation to vote (comments/GitHub): GPU benchmarks (spot/DWS/Capacity Blocks measured) · cluster vending with Crossplane + Sveltos · losing a cloud on purpose (CNPG replica cluster, measured RTO/RPO) · Cilium Cluster Mesh EKS↔GKE, worth it?

- [ ] **Step 2: Write 🔖 References** exactly:

```markdown
### Platform
- [`cloud-native-ref`](https://cnref.ogenki.io) — The complete platform documentation · [source on GitHub](https://github.com/Smana/cloud-native-ref)
- [`crossplane-configuration`](https://github.com/Smana/crossplane-configuration) — The Crossplane compositions
- [Architecture decisions](https://cnref.ogenki.io/docs/decisions/) — ADR-0007, 0017–0024 underpin this article
- [Add a cloud provider](https://cnref.ogenki.io/docs/guides/add-a-cloud-provider/) — the how-to this article deliberately doesn't duplicate

### Context
- [ThousandEyes — AWS outage analysis, Oct 20 2025](https://www.thousandeyes.com/blog/aws-outage-analysis-october-20-2025)
- [EU Data Act: cloud switching and the 2027 fee ban](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/)
- [DORA: exit strategy and concentration risk](https://schneiderdowns.com/our-thoughts-on/doras-approach-to-exit-strategy-and-termination/)
- [FinOps for GPU inference (2026)](https://www.cloudmagazin.com/en/2026/04/03/ai-inference-costs-cloud-finops-gpu-workloads-2026/)
- [Sveltos](https://projectsveltos.io/main/) · [Crossplane + Sveltos GitOps bridge](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/)
- [CloudNativePG replica clusters](https://cloudnative-pg.io/docs/1.27/replica_cluster/)
- [Cilium Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/)
```

- [ ] **Step 3: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/en/post/multicloud-platform/index.md
git commit -m "docs(multicloud): write final thoughts and references"
```

### Task 9: French translation

**Files:**
- Create: `content/fr/post/multicloud-platform/index.md` (+ copy the three PNGs into the FR bundle)

- [ ] **Step 1: Translate the full post.** Title: « Un repo, deux clouds : une stratégie `multicloud` pragmatique ». Keep English technical terms untranslated (claim, workload, control plane, egress…). Internal links become `/fr/post/...` (e.g. `/fr/post/series/agentic_ai/llm-self-hosted-stack/`). Same front matter fields; translate `summary`.

- [ ] **Step 2: Copy images into the FR bundle:**

```bash
cp content/en/post/multicloud-platform/{thumbnail.png,multicloud-overview.png,state-boundary.png} content/fr/post/multicloud-platform/
```

- [ ] **Step 3: Build check + commit**

```bash
hugo --minify --quiet --destination /tmp/hugo-mc-check && echo OK
git add content/fr/post/multicloud-platform/
git commit -m "docs(multicloud): add French translation"
```

### Task 10: Verification pass before review

**Files:**
- Modify: none expected (fixes only)

- [ ] **Step 1: Link-policy checks — all must return no output:**

```bash
grep -n "](/en/post" content/{en,fr}/post/multicloud-platform/index.md
grep -n "https://github.com/Smana/cloud-native-ref\"" content/{en,fr}/post/multicloud-platform/index.md | grep -v "source"
grep -rn "infrastructure/base/crossplane" content/{en,fr}/post/multicloud-platform/
```

- [ ] **Step 2: External links resolve** — spot-check every URL in the References section (curl -sI, expect 200/301).

- [ ] **Step 3: Full build with drafts:**

Run: `hugo --minify --buildDrafts --quiet --destination /tmp/hugo-mc-check && echo OK`
Expected: `OK`; open `hugo server -D` and visually check both language versions (toc, images, notice boxes, code collapse at 25 lines).

- [ ] **Step 4: Editorial checklist** (from the conventions section): honest counter-case present · every bolded term defined on first use · no section-ranking hook sentences · measured numbers timestamped · state-boundary and DNS-residual-risk paragraphs present · scope guard respected (no InferenceService design internals).

- [ ] **Step 5: Flip `draft = true` → `false` only when the author decides to publish; final commit**

```bash
git add -A content/{en,fr}/post/multicloud-platform/
git commit -m "docs(multicloud): verification fixes"
```

---

## Self-review (done 2026-08-28)

- **Spec coverage:** all eight pillars of the consolidated strategy (drivers, doctrine, Flux tree, seams, GPU, fleet, resilience, honest close) map to Tasks 3–8; FR translation Task 9; measured proof Task 1. ✓
- **Review corrections incorporated:** GitOps≠backup state boundary (Task 7), DNS residual risk (Task 7), tested-runbook/drills bar (Task 7), fleet alternatives-considered (Task 7), verified Oct-2025 hook (Task 3). ✓
- **Placeholder scan:** measured values are explicit inputs produced by Task 1 with a defined schema — no TBDs elsewhere. Diagrams have complete mermaid sources. ✓
- **Consistency:** section names in Task 2 skeleton match Tasks 3–8; file paths consistent throughout. ✓

## Out of scope

- Building any of the deep-dive backlog items (GPU benchmarks, Sveltos vending, DR chaos test, Cluster Mesh).
- Changes to cloud-native-ref or crossplane-configuration beyond the read-only measurements in Task 1.
- Publishing: the author flips `draft`, opens the PR, and announces.
