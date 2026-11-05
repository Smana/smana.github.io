+++
author = "Smaine Kahlouch"
title = "`Agent Factory`: agents and humans, one crew"
date = "2026-11-05"
slug = "agent-factory-1-overview"
summary = "More autonomy and better collaboration, starting from a plain GitHub issue: one label opens an isolated workspace where agents and humans take turns until a pull request comes out reviewed, secured and traceable. Episode one: the problem, and the solution from above."
featured = true
draft = false
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.jpg"
+++

{{% notice info "Agentic AI series: enter the Agent Factory 🏭" %}}
Previously in this series: [the fundamentals of coding agents](/post/series/agentic_ai/ai-coding-agent/), [my day-to-day Claude Code workflows](/post/series/agentic_ai/ai-coding-tips/) and [a self-hosted LLM stack](/post/series/agentic_ai/llm-self-hosted-stack/).

Time to scale up: a **shared** environment where the team works alongside agents and can trust what they deliver. This post opens a series about that **Agent Factory**, one building block per article.
{{% /notice %}}

For a few months now, I have hardly written any code without an agent, and I see the same shift around me: we write less code and **steer** more agents.

The catch is that what works on my laptop doesn't carry over to a team as is. Everyone shows up with their own instructions and tools, and practices drift from one machine to the next. In an organization, it's not enough for a coding agent to be effective: you need to guarantee the quality and security of what **actually** ships to production, whoever started the agent.

Meanwhile, agents are getting reliable enough that we can consider handing them some changes end to end, from the issue all the way to the merge. Provided we know which criteria earn that trust, and can measure them. That's the whole point of **software factories**.

This series describes the **Agent Factory**, my attempt at an answer: a pipeline from issue to merged pull request, built from cloud native open source tools. It sits on [`cloud-native-ref`](https://cnref.ogenki.io/), my reference platform, which already provided most of the foundation (Crossplane, Cilium, ZITADEL, OpenBao, observability). I added a layer dedicated to agents, plus the glue that keeps the machinery turning 😉.

This first post lays out the problem and sketches the solution. The next ones will go into each building block, starting with Maku. Everything is open source, and nothing forces you to adopt it all: pick the ideas that speak to you and adapt them to your own context. 🚀

## 🎯 Goals

* 🏭 Understand what a **software factory** is, and where this one sits on the autonomy scale
* 🧩 See what the factory **actually brings you**, as a developer and as an organization
* 🗺️ Discover the overall architecture: **three repositories, three layers**, and the open source building blocks behind them
* 🔄 Follow one task **end to end**, from issue to merged pull request

## 🏭 Software factory: what are we talking about?

A **software factory** is a pipeline that turns a queue of work (issues, a backlog) into shipped changes, with agents at every step. The most quoted examples come from large engineering teams:

* [Stripe's Minions](https://www.infoq.com/news/2026/03/stripe-autonomous-coding-agents/), which produce more than **1,300 PRs a week**;
* [Honk, at Spotify](https://engineering.atspotify.com/2025/11/spotifys-background-coding-agent-part-1), behind more than **1,500 merged PRs**.

Behind those numbers, three notions are enough to describe any factory:

* 🔁 **The loop**: each agent observes, acts and checks, until the task is done.
* 🧰 **The harness**: the frame around the loop. Where the agent runs, with which tools and permissions, and what "done" means.
* 🚦 **The gate**: one last validation step that decides what reaches production.

### 🪜 Where do humans fit in?

What sets factories apart most is how much room they leave for humans. Dan Shapiro proposed [a scale](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/) modelled on the levels of driving automation. The higher you go, the further the human steps back:

| Level | Your role | Example |
|---|---|---|
| **0 · Spicy autocomplete** | You write the code, the AI suggests what comes next | The editor completes the loop you just started |
| **1 · One-off tasks** | You hand over small, well-bounded tasks, the rest of your work doesn't change | "Write a unit test for this function" |
| **2 · Pair programming** | You work with the AI like with a junior colleague, and leave it the repetitive parts | A Claude Code session where you approve every change |
| **3 · The manager** | The agent writes most of the code, you **review every diff** | Several agents in parallel, and you hop from one to the next to review |
| **4 · The spec writer** | You write the spec, review the plan, then come back later to check that the tests pass | An issue written in the evening, a PR to review the next morning |
| **5 · The *dark factory*** | **Nobody reads the code anymore**: specs go in, software comes out | At [StrongDM](https://simonwillison.net/2026/Feb/7/software-factory/), "Code must not be reviewed by humans", and the test scenarios are kept out of the repository so agents can't see them |

{{< img src="autonomy-levels.png" width="900" >}}
</br>

The dark factory is not where I'm heading. When nobody reads the code anymore, one question becomes unavoidable: **who still understands the system?** Knowledge thins out, and the day something breaks, nobody knows where to look. As with [RunLore](/post/series/observability/runlore/), I'd rather have agents do the work while **humans keep their hands on the decision that matters**: what ships to production. Steering the boat rather than watching it drift. ⛵

{{% notice note "Where does this factory sit? 🧭" %}}
**Level 4 for almost everything, level 5 for a single lane.** I write the issue, the agents scope, implement, test and review each other's work, and **my approval of the commit is the merge**. Only one family of changes merges without me: broken-link fixes in the documentation, at most one a day, with a circuit breaker that sends the lane back to human review at the first incident. And that autonomy comes with one condition: knowing **exactly** what must be true before a machine merges anything.
{{% /notice %}}

## 🧩 From laptop to team: what's in it for me?

On my laptop, trust is hardly a question: I start the agent, watch it work, review, push. **I am the guardrail.** The moment the agent works for a team, on a shared cluster, with nobody in front of the screen, that guardrail has to move into the architecture. An autonomous agent is **unreviewed code running with privileges**.

What the factory gives back in return depends on whether you're the one at the keyboard, or the one accountable for what ships.

### 👩‍💻 For the developer

* ⏱️ **Delegate and move on.** A well-written issue is enough, nobody stays in front of the screen, and the PR that comes back has already been reviewed by an agent. On October 29th, a RunLore alert fired at 6:28 am; the PR fixing the rule was open at 6:33 and merged, with my approval, at 6:47.

* 🚪 **Start a task from wherever you are.** A label on the issue, a Slack message, the CLI, or straight from your own coding agent. No context switch.

* 🧰 **Nothing to install, nothing to maintain.** The harness, the tools, the access to models and to GitHub all come from the platform. No keys on your laptop, no configuration to keep current, and the same tooling as your colleagues.

* 🎛️ **Stay in control without reading everything.** The room, the shared log of the task, shows what the agent is doing live; you can steer it along the way or stop it with a label. At the end, a single action is yours: approve the commit, and it's merged.

* 📝 **A report that writes itself.** Budget, steps, link to the room, who merged: everything is posted on the issue as it happens. No more "what had I done here again?" when you come back to it.

### 🏢 For the organization

New questions arise the moment code leaves the developer's laptop, and the factory has to answer them with **guarantees**:

| Question | On my laptop | In the factory |
|---|---|---|
| 🪪 **Who is acting?** | Me, with my own tokens | Every agent run has **its own short-lived identity**, and its spend is charged to the person who started it |
| 🌐 **What can it reach?** | The whole Internet | Only **named** destinations; everything else doesn't even resolve |
| 🚦 **Who decides the merge?** | Me | **Rules versioned in the repository**, and a human as soon as they're not enough |
| 📓 **Where is the evidence?** | My terminal, then nothing | A **shared, tamper-proof log** per task, where every action is attributed. An audit, a post-mortem or a new colleague's onboarding all start from there, even six months later |

On top of those guarantees comes what you'd expect from any shared platform:

* 🧱 **Quality and security that no longer depend on the machine.** Junior or senior, well or badly configured laptop: everyone goes through the same agent roles, the same reviewer and the same gate. The rules improve in one place, for everyone.

* 💸 **A cost you know before you start, and that stays bounded.** The budget (model, tokens, duration) is displayed at startup. A queue caps concurrent runs, and a circuit breaker cuts a lane the moment it goes off the rails. The October 29th alert fix cost 490k tokens, and I knew it before merging.

* 📈 **Autonomy that grows at your own pace, through written rules.** You start at level 4, where a human approves everything. Once a family of changes has proven itself, a versioned rule grants it auto-merge, and takes it away at the first incident. The policy is reviewed and debated like code.

* 🤝 **A single front door for everything that wants to change the repository.** Humans, but also other agents: an alert handled by RunLore can become an issue, hence a task, hence a PR, without a human typing a line, and under the same controls.

* 🔓 **No vendor lock-in.** Every building block is open source and replaceable, the model is one line of configuration, and humans keep their understanding of the system because they review what ships.

In all honesty, I've tested a lot (37 agent PRs merged since late September), but mostly on fairly simple changes, a handful of files each time. That's where a factory earns its keep: recurring, well-bounded changes. For a big feature that chains many tasks, the laptop is probably still faster. I'm curious to see how far this goes.

## 🗺️ The solution from above

The Agent Factory rests on three repositories ([`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), [`ai-native-ref`](https://github.com/Smana/ai-native-ref) and [`maku`](https://github.com/Smana/maku)), each with a clearly defined scope.

{{< img src="solution-layers.png" width="800" >}}
</br>

The AI platform is shipped to the Kubernetes cluster as a **signed OCI artifact**, disabled by default: nothing turns on without an explicit decision. The cluster commits, in a written *contract*, to what it provides (namespaces, identity, secrets, gVisor nodes, observability).

{{% notice tip "Agent Factory and Maku: who's who? 🎭" %}}
The **Agent Factory** is a set of open source components that, assembled together, turn an issue into a reviewed pull request.

**[Maku](https://github.com/Smana/maku)** (幕, "the curtain" in Japanese) is one of them: the controller, the one that orchestrates the others, with a promise: *"Run coding agents on your Kubernetes cluster without trusting them."*

{{% /notice %}}

Apart from Maku, every building block is an existing open source project, used for what it does best:

| Building block | Its role in the factory |
|---|---|
| [OpenHands](https://github.com/OpenHands/OpenHands) | The agent itself: the loop and its developer tools |
| [gVisor](https://gvisor.dev/) and [`agent-sandbox`](https://agent-sandbox.sigs.k8s.io/) | The sandbox for each run, with a user-space kernel |
| [Crossplane](https://www.crossplane.io/) | Composes, from a single resource, everything a run needs |
| [Cilium](https://cilium.io/) | The network, closed by default and opened one name at a time |
| [agentgateway](https://agentgateway.dev/) | The single exit door: identity, access to models and tools, budgets |
| [octo-sts](https://github.com/octo-sts/app) | One-hour GitHub tokens, scoped to one repository and one role |
| [policy-bot](https://github.com/palantir/policy-bot) | The merge policy, versioned in the repository |
| [Kueue](https://kueue.sigs.k8s.io/) | A capped admission queue, so a runaway factory can't fill the cluster |
| [ZITADEL](https://zitadel.com/), [OpenBao](https://openbao.org/), [Flux](https://fluxcd.io/), [VictoriaMetrics](https://victoriametrics.com/) | Human identity, secrets, GitOps and observability, provided by the host cluster |

## 🔄 One task, end to end

Here is the journey of a task, in broad strokes:

{{< img src="agent-factory-overview.png" link="true" alt="Overview: a maintainer labels an issue, Maku opens a room and starts a run in a gVisor sandbox; every call goes through the agent gateway to the models, the MCP tools and octo-sts; the run pushes a branch and opens a PR that the maintainer reviews" width="1200" >}}
</br>

1. **A maintainer** puts the `factory/ready` label on an issue. The same label from anyone else triggers nothing. A task can also start from Slack, from the `maku` CLI, or straight from your coding agent.

2. **Maku** creates a task and a **room**, the shared log of that task. It picks a budget (model, tokens, maximum duration), then starts the agents.

3. **Each agent** runs in its own sandbox, under its own identity. Everything it calls (model, tool, GitHub) goes through the **agent gateway**.

4. **The agents take turns**: one scopes the task, another writes the change and opens the PR, a judge checks that it's done, a reviewer reads it with fresh eyes.

5. **The merge gate decides**: no human needed for the rare changes that have earned that right; otherwise a single action is yours, and **your approval of the commit is the merge**.

## 💭 Final thoughts

For a project that is only a few weeks old, the first results are stunning! It helps that I started from a cloud native foundation that was already battle-tested 😉. Until now, I had mostly been busy getting an efficient local setup with Claude Code or OpenCode, and keeping up with the breakneck pace of new models. Company needs pushed me to look more closely at the organizational side: **giving autonomy and power while keeping control**.

This software factory started as a way to explore a lot of topics, but I got hooked, and the result is getting more serious by the week. The next step, already on the [roadmap](https://airef.ogenki.io/docs/status/), proves it: it's called **"anyone can run it"**, and it's quite a challenge, one I'll tell you about in the adventures to come.

The next post will be about **Maku**, the layer that orchestrates this whole crowd. Here's a sneak peek:

<figure style="text-align:center;">
  <a href="https://blog.ogenki.io/maku/"><img src="maku-room.webp" width="800" loading="lazy" style="border-radius:6px;"></a>
  <figcaption><small><a href="https://blog.ogenki.io/maku/">Open the interactive demo ↗</a></small></figcaption>
</figure>

## 🔖 References

### Repositories, docs and demo

* [`ai-native-ref`](https://github.com/Smana/ai-native-ref): the AI platform · [documentation](https://airef.ogenki.io) · [status and roadmap](https://airef.ogenki.io/docs/status/)
* [`maku`](https://github.com/Smana/maku): the controller · [public demo of a room](https://blog.ogenki.io/maku/)
* [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref): the host cluster · [documentation](https://cnref.ogenki.io)

### Software factories

* [Addy Osmani: Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/)
* [InfoQ: Stripe's Minions, autonomous coding agents](https://www.infoq.com/news/2026/03/stripe-autonomous-coding-agents/)
* [Spotify Engineering: 1,500+ PRs Later, Honk Part 1](https://engineering.atspotify.com/2025/11/spotifys-background-coding-agent-part-1)
* [Dan Shapiro: The five levels, from spicy autocomplete to the software factory](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/)
* [Simon Willison: StrongDM's software factory](https://simonwillison.net/2026/Feb/7/software-factory/)
* [Agent Baseline: The Secure Software Factory, reference architecture](https://agentbaseline.org/software-factory.pdf)
* [awesome-software-factories](https://github.com/varun1505/awesome-software-factories): the landscape, regularly updated
