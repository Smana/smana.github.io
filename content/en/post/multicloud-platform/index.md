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

4. **Resilience**: The us-east-1 incident that opened this article is the recurring reminder: a single provider's control plane is a single blast radius, however good that provider is. If it ranks fourth here despite being the hook, that's because cross-cloud failover is the most expensive driver to actually build — few outages clear that bar. A dedicated section later assesses what it really requires.

5. **The unglamorous majority**: Most enterprises never decided to be multicloud — [they discovered they already were](https://gartsolutions.com/multi-cloud-kubernetes-the-power-and-peril/). **M&A** (mergers and acquisitions) brings a GCP estate into an AWS shop; a data team quietly signs up for BigQuery; a subsidiary keeps its existing contracts. The real question is rarely "should we go multicloud?" but "we already are — do we manage it deliberately or let it sprawl?"

### The counter-case

None of this is free, and pretending otherwise is how multicloud projects die. The complexity tax is structural: two IAM models with different semantics, two networking stacks that agree on almost nothing, double the quotas, double the per-service quirks, and an on-call rotation that now has to be fluent in both. Every engineer you hire either knows two clouds or gets trained on the second one, on your time.

Then there's the lowest-common-denominator trap. Abstract everything so it runs identically everywhere and you forfeit precisely the managed services that made each cloud attractive in the first place — you end up paying premium prices for a fleet of generic VMs and rebuilding the differentiated services yourself.

And egress still costs money. The Data Act kills *switching* fees, not day-to-day traffic: a chatty architecture spanning two clouds bleeds money on every cross-cloud call, in both directions, indefinitely.

My position, which the rest of this article defends: multicloud is a **capability you build selectively**, not a place you move to. Most teams need exit-*capability* and portability at the API layer — not symmetric active-active everything. The interesting work is deciding exactly how much of that capability to build, and where.

## 🧭 The doctrine: abstract the developer, not the platform

## 🌳 One Flux tree, two clusters

## 🔗 The seams: identity, DNS, secrets

## 🎮 GPUs: a consumption strategy for cost and availability

## 📈 When two clusters become a fleet

## 🛡️ The resilience posture

## 💭 Final thoughts: what it really cost

## 🔖 References
