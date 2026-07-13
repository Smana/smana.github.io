+++
author = "Smaine Kahlouch"
title = "`RunLore`: the SRE buddy that investigates incidents — and learns from every fix"
date = "2026-07-12"
summary = "What you learn during an incident usually disappears the moment it's closed. RunLore is an open source SRE agent that investigates, points you at the likely root cause fast, and turns every resolution into knowledge you can reuse."
featured = false
codeMaxLines = 30
usePageBundles = true
toc = true
series = [
  "observability"
]
tags = [
    "observability",
    "ai"
]
thumbnail = "thumbnail.png"
+++

{{% notice info "The missing link in this series 🔗" %}}
Throughout this [series](/series/observability/), we've explored how to **detect** problems: collecting [metrics](/post/series/observability/metrics/), centralizing [logs](/post/series/observability/logs/), then configuring [meaningful alerts](/post/series/observability/alerts/).

But once the alert lands, the most time-consuming step is still ahead: **understanding what's happening, and why**. That's exactly what we tackle here.
{{% /notice %}}

An alert fires. And the **diagnosis** begins: digging through recent changes in Git, Grafana dashboards, logs, maybe a runbook, cross-checking clues until a first hypothesis takes shape.

This **investigation** phase sits at the heart of **SRE** (_Site Reliability Engineering_) work. It runs on deep familiarity with the platform: knowing *where* to look for the right signals. Which is why the **time** it takes to trace a cause varies so much with the seniority of whoever picks it up. And the knowledge built up along the way? It's usually gone the moment the incident is closed.

I decided to take this on a few weeks ago by starting an open source project: [`RunLore`](https://github.com/Smana/runlore). The initial idea was simple: a **buddy** that takes the data-gathering off your hands and **points the investigation at the most likely root cause, fast**. And one that, over time, **learns the context we work in together**: our platform, our constraints, our rules and standards.

To hold on to that knowledge, I picked **OKF** (_Open Knowledge Format_), a format designed precisely so an AI agent can read and write its knowledge efficiently. I'll come back to it [in detail further down](#okf-from-a-karpathy-idea-to-a-standard).

## 🎯 Goals

* 🤖 Get to know [**RunLore**](https://github.com/Smana/runlore) and how it works
* 🧠 Its **three design choices**: a single binary, the **learning loop**, and the **human who makes the call**
* 🛠️ **Deploy it** with Helm and watch it investigate a **real incident** on [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)

## 🔥 Why this project: the hidden cost of investigation

The previous post on alerts ended on an obvious point: technical configuration alone isn't enough, you also need **runbooks and incident response procedures**. In real life, though, those runbooks are incomplete, and much of that knowledge only ever lives in a few people's heads.

When an alert fires, you replay the same manual sequence every single time:

* **What changed?** A deployment, a version bump, an infrastructure change, an expired certificate?
* **What's wrong?** Saturation, network, nodes, failing dependencies?
* **Have we seen this before?** The answer is usually "yes", but nobody can find where it was written down.

Correlating all these sources (Git, metrics, logs, network flows) to reach a **diagnosis** is exactly the kind of task an agent can take on. **Several already do.** But the part I care about is different: **keeping a human in the loop on the decisions**, on what the agent _does_ as much as on what it _learns_.

## 🤖 RunLore in a nutshell

{{< img src="runlore-logo.png" alt="RunLore logo" width="120" >}}

**RunLore** is an open source **SRE agent** (Apache-2.0 licensed) that runs inside your Kubernetes cluster as a single Go binary, deployed with Helm. </br>
The idea is deliberately simple. An event comes in: an alert, a **GitOps** failure (continuous delivery driven from Git), and so on. RunLore investigates, answers two questions — _what changed?_ and _what's wrong?_ — then posts a **root cause** to your chat, with a **confidence score**, the evidence backing it, and the open questions it leaves to a human.

{{< img src="runlore-investigation-flow.png" alt="RunLore investigation flow diagram" width="1080" >}}
</br>

The diagram breaks down into three stages:

* **Triggers** — an alert, a GitOps failure or a webhook (e.g. PagerDuty) kicks off the investigation.
* **Data sources** — the **GitOps history** (the backbone of _what-changed_), metrics, logs, network flows, the cloud… _the more you plug in, the stronger the answer_.
* **Notification channels** — the verdict goes out to Slack, Matrix…

And all of it sits on top of a **knowledge base** that grows with every incident.

### Three design choices

Agents that investigate incidents already exist, and we'll get back to them. What sets RunLore apart comes down to three choices, and it's their **combination** that matters:

1. 📦 **A single binary, on your infra, running your models.** One Go binary in your cluster. You keep control of your data and your models, and with a self-hosted LLM nothing leaves your perimeter. No SaaS, no _lock-in_.
2. 🧠 **A memory that compounds.** Every investigation feeds a **knowledge catalog you own**, in the **OKF** standard. The same incident, the next time around, gets an instant answer instead of a fresh investigation.
3. ✋ **The human stays in charge.** The agent reads and recommends, it never acts on its own (_read-only → suggest → approve_). And above all, **nothing enters its memory without a PR reviewed by a human**. When it doesn't know something, it hands the question back instead of guessing.

The next two sections dig into the two least obvious ones: the memory, then where the human fits in.

## 🧠 A memory that compounds

The autonomous _alert → root cause → chat_ loop has become **commonplace**: plenty of tools do it. What's far less common is a knowledge base that **consolidates into a catalog you control**.

### The learning loop

RunLore implements a four-stage cycle:

1. **Retrieve** — faced with an incident, it first looks for a _trustworthy_ past answer in the knowledge base.
2. **Capture** — if nothing matches, it investigates live, records what it found, then **watches what follows**: did the incident actually resolve?
3. **Curate** — a finding that is both reliable and new enough becomes a **Pull Request** opened against your knowledge repo. This is the **quality gate**: an entry is reviewed the way code is reviewed.
4. **Compound** — once the PR is merged, the entry is re-indexed and immediately available. The same incident, next time, is recognized and served from the catalog, with no second investigation.

So it all hinges on **Retrieve**, which decides between two outcomes: serve a known answer **instantly**, or, with no trustworthy match, run the full loop — the one that feeds the catalog.

{{< img src="runlore-learning-loop.png" alt="RunLore learning loop" width="1080" >}}
</br>

What makes this cycle genuine **learning**, rather than a notepad, is that **outcomes loop back**. RunLore keeps an _outcome ledger_: every time an entry is recalled, it records whether the incident **actually resolved** afterwards. Confidence is therefore **derived from the observed resolution rate**, not asserted by the model:

* an entry that **resolves consistently** → it **gains confidence**;
* an entry **recalled without the incident ever resolving** → it **decays** and stops being offered, until a new investigation corrects it.

The memory isn't frozen: it stays **anchored to operational reality**.

That same score has a **human half**: an optional pair of **👍 / 👎** buttons under every notification on Slack and Matrix. The click isn't cosmetic. One tap moves **two levers**:

* **The quality of the diagnosis** — a 👍 confirms it, a 👎 declares it **wrong**. The vote feeds the entry's **confidence**, not its content. And it is sometimes the **only** judge available: some triggers, a GitOps failure for instance, never emit a resolution signal.
* **When the agent is allowed to repeat itself** — a 👎 that sticks **triggers a fresh investigation immediately**, instead of re-serving a contested answer.

### OKF: from a Karpathy idea to a standard

**Google Cloud** recently published a **portable standard** for what's known as the _LLM-wiki_: [**OKF**](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing) (_Open Knowledge Format_, June 2026). The concept itself, though, is anything but new.

The idea comes from **Andrej Karpathy**: rather than re-deriving knowledge from raw documents on every query, you hand the model a tree of **markdown** files that it reads, extends and maintains **like code**. A format designed to be **read efficiently by AI**, while staying plain text, versioned and reviewable by a human.

I was already using this pattern **locally**: first in **Obsidian**, then through **Tolaria**, the tooling I built on top of it so an agent could read and write in my knowledge base. Making it RunLore's memory felt like the natural move.

So the memory lives in a Git repo **you own**: BM25-indexed, reviewed through PRs, fully traceable. And you don't have to start from a blank page. The catalog can be **seeded** on day one (constraints, architecture, team conventions), and incidents build on it from there.

## ✋ The human stays in charge

RunLore takes a **deliberate stance**: keep a human at the center of the decisions. LLM answers are still debatable often enough that chasing full automation is a trap. Here, you stay **firmly in control**:

* **of how the knowledge base evolves** — nothing enters it without your approval. Every reliable finding becomes a **Pull Request** that you review and merge. The agent proposes; the team decides what joins its memory.
* **of judging diagnosis quality** — you're the one who confirms or overrules it (👍 / 👎), and the agent is calibrated never to oversell: an honest `unresolved` when it doesn't know, and a confidence that can only go **down** at verification.
* **of the domain knowledge that goes in** — you seed and enrich the catalog with _your_ context: company rules, team conventions, habits, platform specifics.

And the agent never acts alone. The posture is _read-only → suggest → approve_: it reads, correlates, recommends; a human validates.

{{% notice note "What about \"PR fatigue\"? 🤔" %}}
The question comes up fast: if nobody had time to document incidents yesterday, who reviews these PRs tomorrow? That's RunLore's **bet**, and it's a deliberate one: the review isn't a chore you put up with, it's what separates a memory you own from a dump of LLM output.

The bet holds because **the volume is bounded by design**: an incident that's already known produces **no PR at all** (it is served from the catalog), a duplicate is dropped, and an incident that already has an open PR gets a comment on it. Only a **new, verified, reliable enough** finding opens one.

And nothing says you have to review by hand: the efficient move is to keep an agent in the loop **during** the diagnosis, cross-checking RunLore's draft against what you just worked out and enriching it with your context. You keep the **decision**, not the line-by-line reading.
{{% /notice %}}

Keeping a human in the loop, on what gets done as much as on what gets learned, isn't a self-imposed limit. It's what makes the agent usable at all.

## 👀 Here's what it looks like

Enough theory, here is RunLore **at work**. The incident below is a real one, investigated on [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), my reference repo: a complete platform on **EKS** combining Cilium, VictoriaMetrics, Crossplane and Flux.

{{% notice info "The model behind the demo 🧠" %}}
The whole demo runs on **GLM 5.2** (Zhipu AI, through Z.ai's OpenAI-compatible API): quality **very close to frontier models** (Claude, GPT) at a **markedly lower cost per token**, which matters a lot when an agent wired to Alertmanager can kick off a lot of investigations. Since RunLore accepts any OpenAI-compatible endpoint, **switching takes a few lines**, all the way to a [self-hosted LLM stack](/post/series/agentic_ai/llm-self-hosted-stack/) (vLLM, Ollama) if you want to keep everything within your perimeter.
{{% /notice %}}

### 🔍 The incident

A **`HarborRegistryDown`** alert fires. RunLore investigates and correlates several signals:

* the **pod status**: `harbor-registry` is failing with `CreateContainerConfigError` — `couldn't find key username in Secret tooling/xplane-harbor-access-key`;
* the **Kubernetes events** in the `tooling` namespace: a persistent Warning on the Crossplane resource `AccessKey/xplane-harbor` — `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2`;
* the **causal link**: that failing Crossplane resource is the one meant to create the Secret the Harbor pod consumes.

The agent then identifies the root cause with **high confidence**: the Crossplane resource has hit the AWS IAM `AccessKeysPerUser: 2` quota, which prevents it from creating the registry's credentials. It posts all of this to Slack, suggests a remediation (delete an unused access key, flagged `reversible=false`), and, importantly, **lists what it doesn't know** (the _data gaps_).

The notification is _verdict-first_: it leads with a clear **actionability verdict** — _no action_, _action suggested_, _action required_ or _inconclusive_ — before any of the details. You know at a glance whether you need to step in.

{{< img src="runlore-investigation.png" alt="Slack notification of a RunLore investigation" width="760" >}}

Once the incident is reviewed and the PR merged, here is the **OKF** entry that joins the catalog:

```markdown
# file: harbor-registry-down-due-to-iam-access-key-quota-limit.md
---
type: Incident
title: Harbor Registry Down due to IAM Access Key Quota Limit
resource: tooling/harbor-registry
tags: [runlore, incident]
fingerprint: 2d6bd8279304b3e17a5d5e35a55fb0c115ffbeabde820af8cdd2494a4141a60b
---

## Decision
- **why keep:** The Crossplane resource `AccessKey/xplane-harbor` has hit an AWS
  IAM quota limit (`AccessKeysPerUser: 2`), preventing it from creating the
  credentials required by the Harbor registry.
- **confidence:** 95%

## Investigate
- pod_status shows harbor-registry failing with 'CreateContainerConfigError:
  couldn't find key username in Secret tooling/xplane-harbor-access-key'
- kube_events shows a persistent Warning on 'AccessKey/xplane-harbor':
  'LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2'
- The knowledge base article 'HarborRegistryDown' describes this exact scenario.

## Resolution
- An administrator should delete an old or unused access key, then reconcile
  the `AccessKey/xplane-harbor` resource. (reversible=false)

## Unresolved
- The name of the specific IAM user that has reached its quota.
- Which of the two existing access keys is safe to delete.
```

The `Unresolved` section shows that principle in action: the agent hands back anything that needs access or judgment it doesn't have. Here, *which* of the two access keys to delete.

### ⚡ Recurrence: an instant answer

Some time later, the same incident **happens again**. This time RunLore runs **no investigation at all**: it **recognizes** the incident, pulls the **already validated** answer out of the catalog (cause and resolution) and posts it to Slack **instantly**.

{{< img src="runlore-recall.png" alt="Slack notification of an instant RunLore recall" width="760" >}}

And it's **radically cheaper**: **~58,000 tokens** for the first investigation, **~3,700** for this recall. Same answer, in a matter of seconds.

{{% notice info "How does recall find the right entry? 🎯" %}}
Recall doesn't rely on keywords alone. Even when the alert shares **almost no wording** with the runbook that covers it, a **structural pre-filter** (the affected resource, here `tooling/harbor-registry`) narrows the candidates down first, then an **LLM reranker** judges whether the retrieved entry truly covers *this* resource and *this* symptom. And the cause is never taken for granted: it is **re-verified against the cluster's actual state** before publishing.
{{% /notice %}}

## 🔬 The alternatives

RunLore isn't landing in an empty field: **RCA** (_Root Cause Analysis_, automatically finding the **root cause**) is already crowded. Here's how I position it against what exists, starting with the closest:

| Tool | What it is | What RunLore adds |
|---|---|---|
| [**OpenSRE**](https://github.com/swapnildahiphale/OpenSRE) | The closest rival, still very young: a multi-agent system that genuinely **learns** (episodic memory + knowledge graph of past incidents) | OpenSRE stores every investigation **automatically, with no review**; RunLore builds a catalog **you own, reviewed through PRs, with confidence that decays when real outcomes don't back it up** |
| [**HolmesGPT**](https://github.com/HolmesGPT/holmesgpt) | The best open source investigation agent | With HolmesGPT, team knowledge comes from hand-written runbooks and it doesn't learn from past incidents; RunLore is _what-changed-first_ and improves on its own |
| [**k8sgpt**](https://github.com/k8sgpt-ai/k8sgpt) | A _detector_ (analyzers + LLM explanation) | An investigation loop, multi-signal correlation, real Git diffs and learning |
| [**kagent**](https://github.com/kagent-dev/kagent) | A generic in-cluster agent _framework_ | A focused, opinionated SRE agent (RunLore could actually run _on_ kagent) |

The table is far from exhaustive: [**Aurora**](https://github.com/Arvo-AI/aurora) (open source, multi-cloud) plays in the same space, and a whole wave of SaaS agents (Datadog Bits AI SRE, Cleric, Resolve.ai…) goes after the same need, but as closed products. That puts them outside the ground I care about here: open and self-hostable.

Change-oriented RCA is clearly **nothing new**: commercial tools have been computing change diffs for a long time. The space RunLore actually occupies is the **combination** that open tools don't offer: that same change signal feeding an **open, portable, reviewed catalog**, operated by an agent that **leaves the call to a human**.

## 🛠️ Try it yourself

Want to try it? Installation is a **Helm chart** and a `values.yaml`. You'll need at least one **data source**, an **LLM**, a **private GitHub repo** for the knowledge base (with a dedicated GitHub App) and a **notification target**. Credentials go into a Kubernetes `Secret`; the `values.yaml` does the wiring.

```yaml
# values.yaml (excerpt) — a standard example: Argo CD + Prometheus + VictoriaLogs + Slack
config:
  gitops:
    engine: argocd                   # or "flux"
  sources:
    alertmanager: {}                 # Alertmanager webhook
    gitops:
      enabled: true                  # also reacts to Argo CD failures
  model:                             # any OpenAI-compatible endpoint
    provider: openai
    model: glm-5.2
    base_url: https://api.z.ai/api/paas/v4/
    api_key_env: GLM_API_KEY
  metrics:
    url: http://kube-prometheus-stack-prometheus.monitoring.svc:9090   # Prometheus
  logs:
    url: http://victoria-logs-single-server.observability.svc:9428     # VictoriaLogs
  cloud:
    provider: aws
    region: eu-west-3
  catalog:
    instant_recall: {enabled: true}  # instant recall from the KB — off by default
  notify:
    slack:                           # bot token: threaded detail + 👍/👎 buttons
      bot_token_env: SLACK_BOT_TOKEN
      signing_secret_env: SLACK_SIGNING_SECRET
      channel: "#alerts"
      feedback_buttons: true
  forge:
    kb_repo: your-org/runlore-kb     # the repo that receives the PRs that grow the knowledge base
```

```bash
helm install runlore deploy/helm/runlore -n runlore --create-namespace -f values.yaml
```

All that's left is to route Alertmanager to `http://runlore.runlore.svc:8080/webhook/alertmanager`, and investigations start. I'm deliberately sticking to the essentials here: the [complete getting started guide](https://github.com/Smana/runlore/blob/main/docs/getting-started.md) covers creating the GitHub App, the full `values.yaml` reference and the verification steps.

The example above is a standard stack, but you can plug in whatever you want: **GitOps** (Flux/Argo CD), **metrics**, **logs**, **network flows**, **cloud**, several **LLMs** and **notifiers**, each one _pluggable_. The **full matrix**, which evolves with each release, lives in the [repo's README](https://github.com/Smana/runlore#-supported-integrations).

### 🔒 Security wasn't an afterthought

Wiring an LLM agent to your alerts raises two fair questions: **what can it break?** and **what gets sent to a third party?** So security is where I've put the most care. Here's what matters:

* **It can't modify anything.** The chart's `ClusterRole` grants **no write permission**, and remediation actions are **off by default**: the model *proposes*, it doesn't *execute*. RunLore's only writes are **PRs** to your knowledge-base repo.
* **Secrets are masked before they leave.** Keys, tokens, `user:pass@host` URLs, the `data:` block of a `kind: Secret` — even in the middle of a Git diff — are masked **before every call to the model**, then again before the PR and Slack.
* **The model is treated as untrusted — because what it reads is untrusted too.** Logs, events, alert annotations: its context is full of text an attacker can influence. So its output is never a decision, only a proposal the server validates: a successful injection **degrades an answer, not your cluster**.
* **And the rest of the groundwork**: least-privilege RBAC (raw logs are never readable cluster-wide), a non-root _Restricted_ pod, a NetworkPolicy, an authenticated webhook, a signed image with an SBOM.

That leaves the number-one fear: *"my data goes to a third party"*. Here, **the choice is yours**. You're most likely already using an LLM day to day, and how sensitive your data really is depends on your context and on the trade-offs your company has already made.

`base_url` takes any OpenAI-compatible endpoint: if you can afford it, point it at [a self-hosted stack](/post/series/agentic_ai/llm-self-hosted-stack/) and **nothing leaves your platform**. Not everyone can. If that's out of reach, you're left with the provider-side guardrails — **zero retention**, **no training on your data** — for whatever they're worth.

{{% notice tip "A few tips to get started 💸" %}}
An agent wired to Alertmanager can rack up LLM calls fast. You stay in control: start **deliberately low**, say **2 investigations/hour**, watch what happens, then raise the ceiling. The guardrails:

* **`investigation.rate_limit`** — the cap on investigations per window (start at 2/hour);
* **`triggers.incidents.debounce`** + **`triggers.incidents.dedup.window`** — ignore self-resolving noise and alerts already in flight;
* **`investigation.coalesce`** — folds a _storm_ of correlated alerts into a single investigation;
* **`investigation.max_tokens_per_investigation`** — a strict token budget;
* **`model.pricing`** — displays the **estimated cost** on every notification (`investigation_cost_usd` metric).
{{% /notice %}}

## 📊 Analyzing the agent's behavior

An agent that investigates your incidents has to be **observable** itself. RunLore exposes **Prometheus metrics** and provides a **Grafana dashboard** (in the repo, to import into your instance), organized **around the learning loop**, not just around technical health. Its first row answers a single question, "_is the loop working?_":

* **Recall fire-rate & precision** — how often it fires, and whether it picks the right entry.
* **Resolution rate** — the key signal: do the recalled answers really resolve the incident?
* **Tokens saved by recall** — the concrete gain against a full investigation.
* **Invalid KB entries** — knowledge that is degrading, worth keeping an eye on.

Many other indicators follow: **cost** (tokens), **performance**, **health** and **errors**.

{{< img src="runlore-dashboard.png" alt="RunLore Grafana dashboard" width="1080" >}}

It's also what will eventually settle the question this post closes on: does this memory actually pay off?

## 🧑‍💻 Built with agents, in full transparency

**RunLore itself was built with my agentic coding tooling**: Claude Code, _skills_, the _superpowers_ framework and multi-agent reviews. It's a direct extension of what I explore in the [Agentic AI](/series/agentic-ai/) series, in particular [the post on coding agents](/post/series/agentic_ai/ai-coding-agent/) and the one on [tips and workflows](/post/series/agentic_ai/ai-coding-tips/). On the model side, **Fable 5** did a lot of the specs/plans work, and **Opus 4.8** most of the code.

And it was nothing like autopilot: even with those models, **inconsistencies** surfaced across iterations, especially around the *learning loop*, the subtlest part of the whole thing. I do write **Go**, but mostly **DevOps tooling** on relatively simple projects. RunLore's internal logic (recall, scoring, decay…) went well beyond what I would have written on my own, and agentic coding was **essential** here.

But I don't want to settle into the role of **conductor**, the one who writes the specs and drives the agents. I want to **understand the internals**, so I had a proper **learning plan generated from the code** (the concepts, the key functions, the decision points) to be able to talk about the project **with confidence**. That's a work in progress.

## 💭 Final thoughts

I'm well aware the project is **very young**, and since I'm its **only contributor** for now, that alone can be off-putting. But this isn't some "vibe-coded" throwaway: I've put real care into **quality**, **performance** and **security**, and I have **actually tested it**, several times, in two environments. I'd really encourage you to take a look. Here's what I've taken away so far.

* **In practice, the diagnoses hold up well.** RunLore doesn't hand back a vague answer: it surfaces the **logs** and **events** that matter and proposes a **likely cause**, backed by evidence.

* **Their quality comes down to two things.** The **data sources** you plug in (without metrics, logs or GitOps history, the agent reasons blind) and the **model**. A solid reasoning model (Opus, GLM, GPT…) produces far more reliable diagnoses than a small, cost-optimized model tuned for latency. On **ambiguous, multi-signal** incidents, where you really have to correlate, the gap becomes obvious.

* **The upfront effort is real.** Without a minimum of **discipline**, the knowledge base won't fill itself. In an already **noisy** production environment, count on **a few hours** of analysis and **PR reviews** before it starts paying off. It's an investment, not an install-and-forget.

* **The fuller the knowledge base, the more useful the agent.** Every resolved investigation you validate (your analysis, your context, a precise RCA) becomes **reusable knowledge**: an incident you've already seen comes back out of the KB **in seconds**, without re-running a full analysis, and the entries that **actually resolve gain confidence** while those recalled without ever resolving are **dropped**. The real unknown is still **maintaining the base over time**: how investigations **map** onto the right entries, what happens when the **same symptom hides a different cause** (RunLore groups them and asks for a human check), and whether **duplicates and conflicts** stay manageable. Guardrails are in place, but only **weeks of real traffic** will tell whether they hold.

So I'll report back after a longer run. Until then, the project is **open** (Apache-2.0), and right now is when your feedback matters most: try it on your platform and tell me how it holds up, [open an issue](https://github.com/Smana/runlore/issues) for a bug, an idea or a use case, or send a PR. Everything that comes back from the field shapes what comes next.

## 🔖 References

* [RunLore — GitHub repo](https://github.com/Smana/runlore) · [getting started guide](https://github.com/Smana/runlore/blob/main/docs/getting-started.md)
* [Open Knowledge Format (OKF) — Knowledge Catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog)
* [Open Knowledge Format — Google Cloud announcement](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing)
* [Andrej Karpathy's _LLM-wiki_ pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
* [cloud-native-ref — reference platform](https://github.com/Smana/cloud-native-ref)
* [k8sgpt](https://github.com/k8sgpt-ai/k8sgpt) · [HolmesGPT](https://github.com/HolmesGPT/holmesgpt) · [kagent](https://github.com/kagent-dev/kagent)
