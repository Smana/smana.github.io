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

On October 20, 2025, a **latent race condition** in DynamoDB's automated DNS management [wiped the service's DNS records](https://www.thousandeyes.com/blog/aws-outage-analysis-october-20-2025) in us-east-1. DNS came back in about three hours, but EC2's internal **DropletWorkflow Manager** entered **congestive collapse** — recovery work piling up faster than it could be processed — and full recovery stretched to roughly fourteen hours. And because us-east-1 hosts global control-plane services such as **IAM** (Identity and Access Management) and **STS** (Security Token Service), the blast radius extended well beyond the region.

Every outage of this magnitude restarts the same conversation. ❓ **Should we be running on more than one cloud?**

**Multicloud** is usually a slide. Rarely a repo. What follows is neither a vendor comparison nor a thought experiment: it's a strategy derived from a platform that actually runs on both **AWS EKS** and **GCP GKE**, with its trade-offs measured rather than assumed.

## 🎯 Goals of this article

* Identify the **real drivers for multicloud in 2026** — and the case for staying on one cloud
* Decide **what to abstract and what to leave cloud-shaped**
* See how **one Git repository** drives **two production clusters**
* Define a **GPU consumption strategy**, with measured **cost per million tokens**
* Understand **when a fleet needs more than Flux**
* Assess **what surviving the loss of a cloud actually requires**
* Get the honest bill: **what it really cost**

{{% notice tip "The reference repo" %}}
Everything in this article runs today in the <strong><a href="https://cnref.ogenki.io">Cloud Native Ref</a></strong> project (<a href="https://github.com/Smana/cloud-native-ref">source on GitHub</a>): the same platform on AWS EKS and GCP GKE, reconciled from one repository and one Flux tree — not a fork each. Compositions live in a dedicated repo, <a href="https://github.com/Smana/crossplane-configuration">crossplane-configuration</a>.
{{% /notice %}}

## 🌍 Why multicloud — the honest version

Every decision in the rest of this article depends on *which* of the drivers below you actually have — so let's rank them, starting with the ones that carry legal weight.

1. **Regulatory exit-capability**: For European financial entities, being able to leave a cloud provider is no longer an aspiration but an obligation. **DORA** (Digital Operational Resilience Act), in force since January 17, 2025, requires firms to assess **ICT** (Information and Communication Technology) concentration risk and to be able to exit a critical provider ["without undue disruption"](https://schneiderdowns.com/our-thoughts-on/doras-approach-to-exit-strategy-and-termination/) (Article 28), with exit and transition provisions written into the contracts themselves (Article 30). The nuance that matters in practice: regulators expect **tested** transition plans, not perfect ones — an exit document nobody has ever rehearsed doesn't pass. And the pressure now extends beyond finance: the **EU Data Act**, fully applicable since September 12, 2025, [abolishes switching charges — including switch-related egress — from January 12, 2027](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/), removing the main financial penalty for actually leaving.

2. **Sovereignty**: Placing workloads by jurisdiction — and being able to move them — is becoming a design input rather than a talking point. AWS itself acknowledged as much by launching its **European Sovereign Cloud** on January 15, 2026: a Brandenburg-based cloud operated by a standalone German legal entity (GmbH), with €7.8B invested. And the EU's June 2026 [Technological Sovereignty Package](https://www.digital-chiefs.de/en/digital-sovereignty-2026-gaia-x-delos-cloud-and-europes-response-to-the-cloud-ac/) keeps the topic on European roadmaps.

3. **GPU capacity**: The quiet new driver. Accelerator supply remains tight — NVIDIA's **B200** is expected to stay constrained through mid-2026 — and when the GPU you need isn't available from your provider, in your region, the ability to consume capacity wherever it exists stops being a philosophical debate. It also doubles as pricing leverage: a credible option to run inference elsewhere changes the tone of GPU negotiations. We'll dedicate a full section to this later.

4. **Resilience**: The us-east-1 incident that opened this article is the recurring reminder: a single provider's control plane is a single blast radius, however good that provider is. If it ranks fourth here despite being the hook, that's because cross-cloud failover is the most expensive capability on this list to actually build — few outages clear that bar. A dedicated section later assesses what it really requires.

5. **The unglamorous majority**: Most enterprises never decided to be multicloud — [they discovered they already were](https://gartsolutions.com/multi-cloud-kubernetes-the-power-and-peril/). **M&A** (mergers and acquisitions) brings a GCP estate into an AWS shop; a data team quietly signs up for BigQuery; a subsidiary keeps its existing contracts. The real question is rarely "should we go multicloud?" but "we already are — do we manage it deliberately or let it sprawl?"

### The counter-case

None of this is free, and pretending otherwise is how multicloud projects die. The complexity tax is structural: two IAM models with different semantics, two networking stacks that agree on almost nothing, double the quotas, double the per-service quirks, and an on-call rotation that now has to be fluent in both. Every engineer you hire either knows two clouds or gets trained on the second one, on your time.

Then there's the lowest-common-denominator trap. Abstract everything so it runs identically everywhere and you forfeit precisely the managed services that made each cloud attractive in the first place — you end up paying premium prices for a fleet of generic VMs and rebuilding the differentiated services yourself.

And egress still costs money. The Data Act kills *switching* fees, not day-to-day traffic: a chatty architecture spanning two clouds bleeds money on every cross-cloud call, in both directions, indefinitely.

My position, which the rest of this article defends: multicloud is a **capability you build selectively**, not a place you move to. Most teams need exit-*capability* and portability at the API layer — not symmetric active-active everything. The interesting work is deciding exactly how much of that capability to build, and where.

## 🧭 The doctrine: abstract the developer, not the platform

The lowest-common-denominator trap has a structural fix: stop asking *how much* to abstract and ask *for whom*. The rule this platform runs on — split the API surface by audience — decides nearly every design question that follows, so it's worth stating precisely.

Some vocabulary first. The platform's APIs are built with **Crossplane**, which extends Kubernetes with custom infrastructure APIs: an **XRD** (Composite Resource Definition) declares the schema, and a **Composition** renders a **claim** — the manifest a user actually writes — into real cloud resources.

**Developer-facing claims are cloud-neutral.** `App`, `SQLInstance`, `InferenceService`: the kinds name intent, and so do the fields — `spec.objectStore`, never `spec.s3Bucket`. The same manifest applies unchanged on the AWS cluster and the GCP cluster; which cloud renders it is decided by the Composition installed on the target cluster — the claim itself carries no cloud signal. This is the layer where portability actually pays: application manifests are the platform's most numerous artifact, and the most expensive one to migrate by hand.

**Platform-facing APIs stay deliberately cloud-shaped**, as sibling XRDs per cloud. The clearest example: `EPI` (**EKS Pod Identity**, AWS's mechanism for granting IAM permissions to pods). Its central field is `spec.policyDocument` — inline AWS IAM policy JSON. There is no neutral form of that field. Its GCP sibling, `GCPWorkloadIdentity`, binds Google IAM roles to a ServiceAccount instead. A merged "WorkloadIdentity" API would have to carry both shapes and pick one at render time: two APIs wearing one name, with the pretense of portability as the only addition. [ADR-0007](https://cnref.ogenki.io/docs/decisions/0007-cloud-abstraction-boundaries/) states the principle we keep coming back to:

> An API that looks neutral but is not causes worse errors than one that is visibly cloud-shaped.

Platform engineers already know which cloud they are configuring. Hiding it from them buys nothing, and costs error messages that point at the wrong abstraction.

**An escape hatch keeps the neutral surface honest.** Neutral claims carry optional `aws {}` / `gcp {}` blocks for the provider-specific knobs that inevitably exist. Portability stays the default; reaching a cloud-specific feature costs a clearly-marked block, not a fork of the API — and a reviewer can measure a claim's cloud coupling by grepping for those two keys. And when a knob can stay neutral, the composition absorbs the difference instead: a `SQLInstance` backup asks for a plain `bucketName`, and the render turns it into an `s3://` or `gs://` destination depending on the cluster's cloud.

Does the neutral surface hold in practice? Here is the diff between the AWS and the GCP example of the same `InferenceService` claim, trimmed of repo-internal comments — in the full diff, every changed line above `apiVersion:` is a YAML comment:

```diff
--- inferenceservice-basic.yaml	2026-08-28 09:21:24
+++ inferenceservice-gcp.yaml	2026-08-28 09:21:24
@@ -1,23 +1,26 @@
 ---
-# Basic InferenceService — smallest viable model.
+# GCP InferenceService example — exercises the composition-gcp.yaml branch.
 [… remaining changed lines elided — all of them YAML comments …]
 apiVersion: cloud.ogenki.io/v1alpha1
 kind: InferenceService
 metadata:
-  name: xplane-qwen3-8b-basic
+  name: xplane-qwen3-8b-gcp
   namespace: llm
 spec:
   model:
```

The rest of both files — the entire `spec:` — is byte-identical. The only difference in the actual claim surface is `metadata.name`.

## 🌳 One Flux tree, two clusters

The doctrine decides API shapes; the repository decides who applies them. We reconcile both clusters with **Flux** — the GitOps engine that watches a Git repository and continuously applies what it finds — from one shared tree. `clusters/aws-0/` and `clusters/gcp-0/` are the two entrypoints, and they are thin by design: they declare what each cluster runs, pointing at shared definitions.

```text
clusters/
├── aws-0/           # entrypoint: what aws-0 runs, at which versions
└── gcp-0/           # same file layout, different paths and pins
infrastructure/
├── base/            # component definitions, written once
├── aws-0/           # AWS-only overlay
└── gcp-0/           # GCP-only overlay (ComputeClass, public DNS…)
observability/       # same base / aws-0 / gcp-0 split
security/            # same split
```

That base/overlay division is what makes a cross-cloud pull request legible. Touch `infrastructure/base/` or `observability/base/` and **both** clusters reconcile the change on their next sync; touch `infrastructure/gcp-0/` and only one does. The diff's paths announce the blast radius before the reviewer reads a single line of YAML.

Shared does not mean symmetric, though. The base holds *definitions*, and each cluster's entrypoint decides which ones it consumes: Karpenter lives in `infrastructure/base/`, yet only `aws-0` includes it — GKE node autoscaling goes through a `ComputeClass` in the `gcp-0` overlay instead. What a cluster runs is decided at its entrypoint.

The Compositions from the previous section are *not* in this tree. They are consumed as versioned **Configuration packages**: **OCI** (Open Container Initiative) artifacts — the same registry format as container images, which Crossplane reuses for packages — published from the dedicated [crossplane-configuration](https://github.com/Smana/crossplane-configuration) repo. Three packages exist: `crossplane-configuration-core` carries the cloud-neutral contracts, `-aws` and `-gcp` the per-cloud Compositions. The OCI tag *is* the git tag, so the version running in a cluster is always a commit you can check out.

Each cluster's Flux tree then pins what it installs: `clusters/aws-0/infrastructure/crossplane-configuration.yaml` reconciles the AWS pin, its `gcp-0` twin the GCP one. The pin itself is a single `Configuration` object:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: crossplane-configuration-aws
spec:
  package: ghcr.io/smana/crossplane-configuration-aws:v0.4.1
```

A platform API upgrade is therefore two small pull requests — bump the tag on one cluster, let it soak, then bump the other — never a synchronized big bang across clouds.

The mechanics of standing up a new cluster in this tree — accounts, OpenTofu stacks, Flux bootstrap, ordering — are a how-to rather than a strategy topic; the [Add a cloud provider](https://cnref.ogenki.io/docs/guides/add-a-cloud-provider/) guide walks through them end to end.

{{< img src="multicloud-overview.png" alt="One Git repository reconciled by Flux into EKS and GKE, with Crossplane configuration packages per cloud" width="1000" >}}

## 🔗 The seams: identity, DNS, secrets

However clean the abstraction, there are places where the two clouds must actually touch each other — not coexist side by side, but trust, resolve, or read one another. Keeping that list short is a strategy outcome in itself. Ours has three entries: workload identity, DNS, and secret stores plus sign-on.

**Workload identity.** Some workloads on GKE need to call AWS APIs (the DNS seam below explains why). They do it by assuming an AWS IAM role via `AssumeRoleWithWebIdentity`: an AWS **OIDC** (OpenID Connect) identity provider is declared to trust GKE's token issuer, the workload presents its **bound ServiceAccount token** — short-lived, audience-restricted, projected into the pod by Kubernetes — and STS exchanges it for temporary AWS credentials. Zero static keys anywhere: no access key sitting in a GCP secret, nothing to rotate, nothing to leak. And the trust is narrow by construction — the role grants only Route53 record changes on a single hosted zone (plus the read-only lookups to find it). The entire seam is one small OpenTofu stack, `opentofu/shared/aws-gcp-federation`, filed under `shared/` because these are resources whose only job is to couple the two clouds.

**DNS.** There is one authoritative public zone, `cloud.ogenki.io`, hosted in Route53. GCP services live under the child domain `gcp.cloud.ogenki.io`, so they never share a level with the AWS side's wildcard records — the one place where two clouds publishing into a single namespace collide. On the GKE side, each certificate covers a single hostname, so a stolen key buys one service, not the whole subdomain. The federated role from the identity seam is what makes all of this work from GCP: cert-manager on GKE proves domain control by solving **DNS-01** challenges — writing a TXT record the certificate authority then verifies — directly into Route53, and external-dns publishes each `HTTPRoute` hostname into the same zone the same way. The full reasoning, including the collision that forced the child-domain split, is recorded in [ADR-0019](https://cnref.ogenki.io/docs/decisions/0019-cross-cloud-dns-federation/).

**Secret stores and sign-on.** Manifests reference secrets through one portable `ClusterSecretStore` name, with a key grammar both clouds accept ([ADR-0023](https://cnref.ogenki.io/docs/decisions/0023-portable-secret-store-names/)); each cluster backs that name with its own cloud-managed store — AWS Secrets Manager on EKS, GCP Secret Manager on GKE ([ADR-0024](https://cnref.ogenki.io/docs/decisions/0024-cloud-managed-secret-stores/)). The manifests never encode which store answers. Above the stores sits a single identity provider for the whole platform: one ZITADEL instance, hosted on aws-0 and serving both clusters ([ADR-0022](https://cnref.ogenki.io/docs/decisions/0022-single-identity-provider-across-clouds/)) — because two instances would mean two unfederated user directories, and "log in twice, once per cloud" is how a platform convinces its users that multicloud was a mistake.

Three seams, each kept small, each with an ADR recording why it exists. Everything else stays cloud-local — which is exactly what makes the GPU question tractable.

## 🎮 GPUs: a consumption strategy for cost and availability

The GPU question is tractable here because of the layering above: the `InferenceService` claim is cloud-neutral, and everything GPU-shaped underneath it — Karpenter's node pools on AWS, the ComputeClass on GKE — stays cloud-local. When the same manifest runs on either cluster, GPU capacity stops being a constraint you inherit from your provider and becomes a market you shop: across pricing models first, and across clouds when a region runs dry. The byte-identical `spec` from the doctrine section is what makes that shopping credible — the option to move is proven rather than promised, and that proof is what a GPU renewal negotiation responds to.

### Match the pricing model to the workload

GPU pricing comes in three shapes, and the strategy is a portfolio, not a pick. **Committed-use / reserved** capacity covers the 24/7 baseline you know you will consume. **On-demand** absorbs burst above it. **Spot/preemptible** — steeply discounted capacity the provider can [reclaim with as little as 30 seconds' to two minutes' notice](https://www.thundercompute.com/blog/cloud-gpu-spot-instance-availability) — is for interruptible batch only: fine-tuning, evaluation runs, embedding backfills. It has no business under a real-time inference endpoint, where a reclaimed node is a user-facing outage. That segmentation — [classifying workloads by interruption tolerance before chasing discounts](https://www.cloudmagazin.com/en/2026/04/03/ai-inference-costs-cloud-finops-gpu-workloads-2026/) — is most of GPU FinOps; the discount itself is the easy part.

At the reserved end, the two clouds sell certainty differently. AWS **EC2 Capacity Blocks for ML** are defined-duration reservations: a block of accelerators booked for a set window at published per-accelerator-hour rates. GCP's **Dynamic Workload Scheduler** queues the request instead — flex-start mode provisions whenever capacity appears, calendar mode reserves a future window. With B200-class supply constrained through mid-2026 (the third driver on the opening list), these mechanisms are not about paying less; they are about getting scarce accelerators at all.

### Right-size the accelerator

The most expensive mistake precedes any pricing model: defaulting to "the AI GPU". An L4-class card (24 GB) comfortably serves the 8B-class models this platform runs; A100/H100-class parts exist for models an order of magnitude larger, and picking one by default costs 5–10x for headroom that idles. Our choice on both clouds is the smallest single-L4 instance — g6 on AWS, g2-standard on GCP: the same silicon on purpose, so the comparison below measures clouds, not GPUs.

### Make scale-to-zero your biggest lever

Cast AI's 2026 State of Kubernetes Optimization Report puts average GPU utilization in production clusters at [about 5%](https://cast.ai/blog/gpu-cloud-pricing/). That is the industry paying for roughly twenty times the GPU it uses. No discount negotiates that away — the fix is architectural. On this platform, every `InferenceService` scales to zero through **KEDA** (Kubernetes Event-driven Autoscaling), which scales on request activity rather than CPU — how that machinery works end to end (vLLM serving, the Envoy AI Gateway in front) is the subject of the [self-hosted LLM stack post](/post/series/agentic_ai/llm-self-hosted-stack/). The strategic consequence is what matters here: a model nobody is querying costs the object storage its weights sit in, not GPU-hours.

### Compare on cost per million tokens

Instance prices don't answer the question that matters — what does inference actually cost? The comparable unit is dollars per million output tokens, and the method fits in one line:

```text
cost_per_Mtok = (hourly_instance_cost / (throughput_tok_s * 3600)) * 1_000_000
```

Throughput comes from serving metrics you already collect (vLLM exports generation-token counters; sustained tokens/s over a busy window is the denominator that reflects real load). The method is the durable part of this section; the prices below are its perishable inputs.

<!-- TODO(author): fill tok/s and $/Mtok using the PromQL in docs/superpowers/plans/2026-08-28-multicloud-strategy-post-inputs.md and re-verify $/h before publishing. The AWS row MUST be measured; the GKE row may ship as pending (the note below explains why) if quota still hasn't landed (if the GKE quota lands before publication, also update the Final thoughts sentence that references the pending row). Also add the missing thumbnail.png to this bundle (front matter references it; homepage card renders broken without it). Then delete this comment. -->

| Cloud | GPU | Instance | $/h (on-demand) | tok/s | $/Mtok |
|---|---|---|---|---|---|
| AWS EKS | 1x NVIDIA L4 (24 GB) | g6.2xlarge (8 vCPU / 32 GB), eu-west-3 | $1.2410 | _pending_ | _pending_ |
| GCP GKE | 1x NVIDIA L4 (24 GB) | g2-standard-8 (8 vCPU / 32 GB), europe-west4 | $0.8972 | _pending_ | _pending_ |

Four caveats before anyone quotes this table. The prices are on-demand list rates as read on 2026-08-28 — they will drift, which is exactly why the method matters more than the cells. The rows are not like-for-like: the regions differ because that is where each cluster actually runs, and the GCP figure bundles the L4 into the machine price where the AWS figure is the instance's own rate. Although both rows use on-demand for comparability, the platform's actual buying policy is asymmetric — the AWS node pool mixes spot and on-demand, while the GKE ComputeClass is deliberately spot-only. And the GKE row may stay pending longer than the AWS one: that cluster has yet to be granted the quota to provision its first GPU node — a live illustration of the availability half of this section's title.

That is the strategy for one workload class on two clusters; the next question is what happens when two clusters stop being the whole story.

## 📈 When two clusters become a fleet

The answer starts with what we did *not* deploy. At two static clusters, per-cluster Flux entrypoints are the right model, for the properties the Flux-tree section demonstrated — a legible tree, path-readable blast radius, two-PR upgrades. A fleet manager layered on top of `aws-0` and `gcp-0` would add a control plane, a failure mode, and a stack of new concepts — and save nothing. If two clusters are your steady state, treat this section as a threshold you may never need to cross.

The threshold is clusters ceasing to be a pair of named, long-lived environments and becoming a *fleet*: a cluster per team, ephemeral clusters for previews or training runs, clusters created on demand by a workflow. The per-cluster overlay pattern survives that shift mechanically but fails economically — every new cluster is a copy-and-adjust of an entrypoint, and reviewing the fortieth copy teaches nothing the first thirty-nine didn't.

What replaces copying is label-matched distribution. **Sveltos** is a controller on a management cluster; its **ClusterProfiles** deploy add-ons and configuration to every registered cluster matching a label selector: declare once that clusters labeled `tier=inference` get the GPU stack, and carrying the label decides the rest. Its event framework turns that convenience into a capability — **cluster vending**: a Crossplane claim creates a cluster on either cloud, Sveltos detects and registers it, and profiles hydrate the full platform onto it with no human in the loop. The end-to-end flow is documented as the [GitOps bridge pattern](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/).

The ownership line matters more than the tool: Flux syncs Git; Sveltos distributes across the fleet. The two compose — Sveltos has integrated with Flux directly since v0.23, so manifests stay in Git and Sveltos consumes what Flux has already synced. Adoption would build on the Flux investment this platform has already made.

It is not the only candidate. **Karmada** (CNCF incubating) is the most mature project in the category, doing full multi-cluster *workload scheduling* — placing and rescheduling applications across clusters — considerably more machinery than a platform that schedules per cluster needs. **Open Cluster Management** (**OCM**, CNCF sandbox) comes at the problem from the governance side and is the foundation Red Hat ACM builds on. Both are credible; Sveltos is the one this platform would reach for, because the actual need is add-on distribution plus eventing that composes with a Flux tree already in place.

Whatever the cluster count, though, one question stays the same: what actually survives when a whole cloud goes away?

## 🛡️ The resilience posture

Driver #4 promised an assessment of what surviving the loss of a cloud actually requires, and it starts with a correction: **GitOps is not a backup**. Flux rebuilds anything declarative — Deployments, XRDs, HTTPRoutes — because Git holds their source of truth. It cannot rebuild what Git never contained: the bytes in a PostgreSQL volume, the contents of a cache, anything a workload wrote at runtime. A resilience posture therefore begins by drawing the **state boundary**: deciding, for every stateful component, which path brings it back.

On this platform the boundary has three zones, one per recovery path: stateful components with a path of their own, workloads rebuilt by GitOps, and accepted loss. The first zone has two members, and PostgreSQL is the one that matters most: **CloudNativePG** (**CNPG**), the Postgres operator, continuously archives its **WAL** (Write-Ahead Log — the append-only record of every database change, the primitive Postgres recovery is built on) to object storage. That archive does more than restore in place: CNPG [replica clusters](https://cloudnative-pg.io/docs/1.27/replica_cluster/) can bootstrap a standby in the *other* cloud from that same archive (S3 → GCS), with declarative switchover when the standby must take over. **RPO** (Recovery Point Objective — the data loss you accept, measured in time) is bounded by the WAL-archiving cadence.

The object stores are the first zone's second member: cross-cloud-accessible by nature, they are where anything that must survive gets written. One consequence is worth stating explicitly: there is no **Velero** (the de-facto Kubernetes backup tool) here, by design — the PersistentVolume snapshots it would otherwise take would capture nothing that isn't already in Git or in an object store.

The second zone takes the easy path: everything stateless is rebuilt by Flux on the surviving cluster, from the same tree, in minutes. That is the doctrine section paying out — the manifests never carried a cloud signal, so redeploying on the other cloud is ordinary reconciliation. The third zone is accepted loss: Valkey caches rebuild from their sources, so losing them costs a warm-up.

{{< img src="state-boundary.png" alt="What survives the loss of a cloud: PostgreSQL via WAL shipping to object storage, object stores as the cross-cloud state layer, stateless workloads rebuilt by GitOps, caches accepted as lost" width="800" >}}

State is only half of failover; traffic is the other half: repointing hostnames through the shared Route53 zone from the seams section. This is also where the residual risk lives. That single zone is a concentration point, and Route53's control plane runs in us-east-1 — the region from the opening incident — where it was those global control-plane dependencies that carried the impact beyond the region. In such an event, existing records keep resolving, but *changing* them can become impossible — uncomfortable when changing records is your failover mechanism. ADR-0019 records the trade-off: one zone buys the simplicity that keeps the seams small, and this is the risk accepted in exchange.

Which leaves the part tooling cannot supply: failover here is a **tested runbook**, not a button. Promote the CNPG standby, repoint DNS, scale up the survivor — each step is declarative, but the sequence is operational, and the bar is periodic drills: run it on a schedule, measure the recovery time, fix what surprised you. This is what regulators reading driver #1 mean by *tested* transition plans — an exit document nobody has rehearsed doesn't pass, and neither does a failover runbook. Until it has been drilled, a runbook is a hypothesis.

What the posture deliberately omits is live cross-cluster interconnection — Cilium Cluster Mesh-style networking where services span clouds at runtime — a standing cost most platforms don't need and this one hasn't paid: it sits on the backlog, and its absence is part of the bill the next section tallies.

## 💭 Final thoughts: what it really cost

The bill, then.

The second cloud was a second foundation layer: another OpenTofu stack — network, GKE, secrets, identity — a second Crossplane provider family with its own per-cloud Compositions, and a decision record of its own just for running Cilium self-managed on GKE. None of it was hard the way a research problem is hard; it was volume — weeks of evenings, not a weekend. And the stream of small asymmetries wore me down more than any of the big pieces did: quota requests that have no equivalent on the other side, ComputeClass semantics that only mostly map onto Karpenter's, a GPU pool that is spot-only on one cloud and mixed on the other — even the deliberate ones. The federation seams — DNS and identity — were the inverse: small in line count, large in thinking time.

The abstractions held better than I expected: the `SQLInstance` claim never learned which cloud it was on — the difference never climbed above the composition's backup render. The leak that stayed open is operational: the GKE row of the GPU table is still _pending_ because the quota request is.

Part of the bill is what I chose to leave unbuilt. There is no Cilium Cluster Mesh, so nothing spans the two clouds at runtime — the resilience section flagged that omission, and it stays on the books as a carried-forward obligation. There is no symmetric active-active, and no fleet manager over two named clusters. These omissions *are* the strategy — capability built selectively — and each of them can still be added later, without paying its standing cost in the meantime.

Several of those open threads deserve a measured article of their own, and the backlog currently looks like this:

* **GPU benchmarks** — spot, Dynamic Workload Scheduler and Capacity Blocks measured, with the $/Mtok method applied in full
* **Cluster vending with Crossplane + Sveltos** — the GitOps bridge built end to end
* **Losing a cloud on purpose** — a CNPG replica cluster, a controlled AWS kill, measured RTO/RPO
* **Cilium Cluster Mesh across EKS and GKE** — worth it? With the latency numbers and the egress bills

If one of these would be useful to you, say so in the comments or in the repo's GitHub issues — the votes decide the order.

The next us-east-1 morning will restart the conversation that opened this article; this time, the answer runs in a repo.

## 🔖 References

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
