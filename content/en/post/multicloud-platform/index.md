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

## 🧭 The doctrine: abstract the developer, not the platform

## 🌳 One Flux tree, two clusters

## 🔗 The seams: identity, DNS, secrets

## 🎮 GPUs: a consumption strategy for cost and availability

## 📈 When two clusters become a fleet

## 🛡️ The resilience posture

## 💭 Final thoughts: what it really cost

## 🔖 References
