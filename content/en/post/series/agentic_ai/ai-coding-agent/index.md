+++
author = "Smaine Kahlouch"
title = "`Agentic Coding`: concepts and hands-on Platform Engineering use cases"
date = "2026-02-06"
summary = "Exploring **agentic coding** through `Claude Code`: from fundamentals (tokens, MCPs, skills) to real-world use cases, with an enthusiastic yet honest take on this new way of working."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = [
    "ai",
    "devxp",
    "tooling"
]
aliases = ["/en/post/ai-coding-agent/"]
thumbnail = "thumbnail.png"
+++

We can all see it — AI is shaking things up in a major way. The field is evolving so fast that keeping up with every new development is nearly impossible. As for measuring the impact on our daily lives and how we work, it's still too early to tell. One thing is certain though: in tech, it's a **revolution**!

In this post, I'll walk you through a practical application in **Platform Engineering**, exploring how a **coding agent** can help with common tasks in our field.

Most importantly, I'll try to demonstrate through concrete examples that this new way of working **truly** boosts our productivity. Really!

## :dart: Goals of this article

* Understand what a **coding agent** is
* Discover the key concepts: tokens, MCPs, skills, agents
* **Hands-on use cases** in Platform Engineering
* Thoughts on limitations, pitfalls to avoid, and alternatives
* For tips and workflows I've picked up along the way, check the [dedicated article](/post/series/agentic_ai/ai-coding-tips/)

{{% notice tip "The reference repo" %}}
<table>
  <tr>
    <td><img src="repo_gift.png" style="width:80%;"></td>
    <td style="vertical-align:middle; padding-left:10px;" width="70%">
The examples below come from my work on the <strong><a href="https://github.com/Smana/cloud-native-ref">Cloud Native Ref</a></strong> repository. It's a full-fledged platform combining EKS, Cilium, VictoriaMetrics, Crossplane, Flux and many other tools.
    </td>
  </tr>
</table>
{{% /notice %}}

{{% notice info "Already familiar with the concepts?" %}}
If you already know the basics of coding agents, tokens and MCPs, jump straight to the [hands-on Platform Engineering use cases](#-hands-on-platform-engineeringsre-use-cases).
{{% /notice %}}

---

## :brain: Why _Coding Agents_?

### How an agent works

You probably already use ChatGPT, LeChat or Gemini to ask questions. That's great, but it's essentially **one-shot**: you ask a question, and you get an answer whose relevance depends on the quality of your prompt.

A **coding agent** works differently. It runs tools in a loop to achieve a goal. This is called an [**agentic loop**](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/).

{{< img src="agentic-loop.png" width="580" >}}

The cycle is simple: **reason → act → observe → repeat**. The agent calls a tool, analyzes the result, then decides on the next action. That's why it needs access to the **output of each action** — a compilation error, a failing test, an unexpected result. This ability to react and **iterate autonomously** on our local environment is what sets it apart from a simple chatbot.

A coding agent combines several components:

* **LLM**: The "brain" that reasons (Claude Opus 4.6, Gemini 3 Pro, Devstral 2...)
* **Tools**: Available actions (read/write files, execute commands, search the web...)
* **Memory**: Preserved context (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`... depending on the tool, plus conversation history)
* **Planning**: The ability to break down a complex task into sub-steps

### Choosing the right model — hard to keep up 🤯

New models and versions appear at a breakneck pace. However, you need to be careful when choosing a model because effectiveness (code quality, hallucinations, up-to-date context) can **vary drastically**.

The [**SWE-bench Verified**](https://www.swebench.com/) benchmark has become the reference for evaluating model capabilities in software development. It measures the ability to solve real bugs from GitHub repositories and helps guide our choices.

{{< img src="swe-bench-leaderboard.png" width="750" >}}

{{% notice warning "These numbers change fast!" %}}
Check [vals.ai](https://www.vals.ai/benchmarks/swebench) for the latest independent results. At the time of writing, Claude Opus 4.6 leads with **79.2%**, closely followed by Gemini 3 Flash (**76.2%**) and GPT-5.2 (**75.4%**).
{{% /notice %}}

In practice, today's top models are all capable enough for most _Platform Engineering_ tasks.

{{% notice info "Why model choice matters" %}}
Boris Cherny, creator of Claude Code, shared his take on model selection (about Opus 4.5 — the reasoning still holds):

{{< img src="boris-opus4.5.png" width="600" >}}

My experience aligns: with a more capable model, you spend less time rephrasing and correcting, which more than compensates for the extra latency.

{{% /notice %}}

### Why Claude Code?

There are many coding agent options out there. Here are a few examples:

| Tool | Type | Strengths |
|------|------|-----------|
| [**Claude Code**](https://code.claude.com/docs/en/overview) | Terminal | 200K context (1M in beta), high SWE-bench score, hooks & MCP |
| [**opencode**](https://opencode.ai/) | Terminal | **Open source**, multi-provider, local models (Ollama) |
| [**Cursor**](https://cursor.sh/) | IDE | Visual workflow, Composer mode |
| [**Antigravity**](https://antigravity.google/) | IDE | Parallel agents, Manager view |

Other notable alternatives (non-exhaustive): [Gemini CLI](https://github.com/google-gemini/gemini-cli), [Mistral Vibe](https://mistral.ai/news/devstral-2-vibe-cli), [GitHub Copilot](https://github.com/features/copilot)...

I started with Cursor, then switched to Claude Code — probably because of my **sysadmin background** and natural affinity for the terminal. While others prefer working exclusively in their IDE, I feel more at home with a CLI.

---

## :books: Essential Claude Code concepts

This section cuts straight to the point: **tokens, MCPs, Skills, and Tasks**. I'll skip the initial setup (the [official docs](https://code.claude.com/docs/en/overview) cover that well) and subagents — that's internal plumbing; what matters is what you can *build* with them. Most of these concepts **also apply to other coding agents**.

### Tokens and context window

#### The essentials about tokens

A **token** is the basic unit the model processes — roughly 4 characters in English, 2-3 in French. Why does this matter? Because **everything costs tokens**: input, output, and context.

The **context window** (200K tokens for Claude, up to 1M in beta) represents the model's "working memory". The `/context` command lets you see how this space is used:

```console
/context
```

{{< img src="cmd_context.png" width="650" >}}

This view breaks down context usage across different components:

* **System prompt/tools**: Fixed cost of Claude Code (~10%)
* **MCP tools**: Definitions of enabled MCPs
* **Memory files**: `CLAUDE.md`, `AGENTS.md`...
* **Messages**: Conversation history
* **Autocompact buffer**: Reserved for automatic compression
* **Free space**: Available space to continue

<table>
  <tr>
    <td style="vertical-align:middle;" width="70%">
Once the limit is reached, the oldest information is simply <strong>forgotten</strong>. Fortunately, Claude Code has an <strong>auto-compaction</strong> mechanism: as the conversation approaches 200K tokens, it <strong>intelligently compresses</strong> the history while retaining important decisions and discarding verbose exchanges. This lets you work through long sessions without losing the thread — but frequent compaction degrades context quality. That's why it's worth using <code>/clear</code> between distinct tasks.
    </td>
    <td><img src="dori_forgot.png" style="width:100%;"></td>
  </tr>
</table>

### MCPs: a universal language

The **Model Context Protocol** (MCP) is an open standard created by Anthropic that allows AI agents to connect to external data sources and tools in a standardized way.

{{% notice info "Open governance" %}}
In December 2025, Anthropic [handed MCP over to the Linux Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) through the Agentic AI Foundation. OpenAI, Google, Microsoft and AWS are among the founding members.
{{% /notice %}}

There are many MCP servers available. Here are the ones I use regularly to interact with my platform — **configuration, troubleshooting, analysis**:

| MCP | What it does | Concrete example |
|-----|-------------|------------------|
| **[context7](https://github.com/upstash/context7)** | Up-to-date docs for libs/frameworks | "Use context7 for the Cilium 1.18 docs" → avoids hallucinations on changed APIs |
| **[flux](https://fluxcd.control-plane.io/mcp/)** | Debug GitOps, reconciliation state | "Why is my HelmRelease stuck?" → Claude inspects Flux state directly |
| **[victoriametrics](https://github.com/VictoriaMetrics-Community/mcp-victoriametrics)** | PromQL queries, metric exploration | "What Karpenter metrics are available?" → lists and queries in real time |
| **[victorialogs](https://github.com/VictoriaMetrics-Community/mcp-victorialogs)** | LogsQL queries, log analysis | "Find Crossplane errors from the last 2 hours" → root cause analysis |
| **[grafana](https://github.com/grafana/mcp-grafana)** | Dashboards, alerts, annotations | "Create a dashboard for these metrics" → generates and deploys the JSON |
| **[steampipe](https://github.com/turbot/steampipe-mcp)** | SQL queries on cloud infra | "List public S3 buckets" → multi-cloud audit in one question |

{{% notice tip "Global or local configuration?" %}}
MCPs can be configured globally (`~/.claude/mcp.json`) or per project (`.mcp.json`). I use `context7` globally since I rely on it almost all the time, and the others at the repo level.
{{% /notice %}}

### Skills: unlocking new powers

{{< img src="skill-acquired-notif.png" width="450" >}}

This is probably the feature that generates the most excitement in the community — and for a good reason, it really lets you extend the agent's capabilities! A **skill** is a Markdown file (`.claude/skills/*/SKILL.md`) that lets you inject project-specific **conventions**, **patterns**, and **procedures**.

In practice? You define once how to create a clean PR, how to validate a Crossplane composition, or how to debug a Cilium issue — and Claude applies those rules in every situation. It's **encapsulated know-how** that you can share with your team.

**Two loading modes:**

* **Automatic**: Claude analyzes the skill description and loads it when relevant
* **Explicit**: You invoke it directly via `/skill-name`

{{% notice info "A format that's catching on" %}}
The `SKILL.md` format introduced by Anthropic has become a **de facto convention**: [GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills), [Google Antigravity](https://codelabs.developers.google.com/getting-started-with-antigravity-skills), Cursor, OpenAI Codex and others adopt the same format (YAML frontmatter + Markdown). Only the directory changes (`.claude/skills/`, `.github/skills/`...). The skills you create are therefore **reusable across tools**.
{{% /notice %}}

#### Anatomy of a skill

A skill consists of a **YAML frontmatter** (metadata) and **Markdown content** (instructions). Here's the `/create-pr` skill from [cloud-native-ref](https://github.com/Smana/cloud-native-ref/tree/main/.claude/skills) — it generates PRs with a structured description and Mermaid diagram:

```markdown
<!-- .claude/skills/create-pr/SKILL.md -->
---
name: create-pr
description: Create Pull Requests with AI-generated descriptions and mermaid diagrams
allowed-tools: Bash(git:*), Bash(gh:*)
---

## Usage
/create-pr [base-branch]       # New PR (default: main)
/create-pr --update <number>   # Update an existing PR

## Workflow
1. Gather: git log, git diff --stat, git diff (in parallel)
2. Detect: Change type (composition, infrastructure, security...)
3. Generate: Summary, Mermaid diagram, file table
4. Create: git push + gh pr create
```

| Field | Role |
|-------|------|
| `name` | Skill name and `/create-pr` command |
| `description` | Helps Claude decide when to auto-load |
| `allowed-tools` | Tools authorized without confirmation (`git`, `gh`) |

This pull request example shows how you can frame the agent's behavior to achieve the result you want — here, a structured PR with a diagram. This avoids iterating on the agent's proposals and helps you be more efficient.

### Tasks: never losing track

**Tasks** (v2.1.16+) solve a real problem in autonomous workflows: how do you keep track of a complex task that spans over time?

Tasks replace the former "Todos" system and bring three key improvements: **persistence across sessions**, **shared visibility between agents**, and **dependency tracking**.

In practice, when Claude works on a long-running task, it can:
- Break down the work into Tasks with dependencies
- Delegate certain Tasks to the background
- Resume work after an interruption without losing context

{{% notice tip "/tasks command" %}}
Use `/tasks` to see the status of ongoing tasks. Handy for tracking where Claude is on a complex workflow.
{{% /notice %}}

---

## :rocket: Hands-on Platform Engineering/SRE use cases

Enough theory! Let's get to what really matters: how Claude Code can help us day to day. I'll share two detailed, concrete use cases that showcase the power of MCPs and the Claude workflow.

### :mag: Full Karpenter observability with MCPs

This case perfectly illustrates the power of the **agentic loop** introduced earlier. Thanks to MCPs, Claude has full context about my environment (metrics, logs, up-to-date documentation, cluster state) and can **iterate autonomously**: create resources, deploy them, visually validate the result, then correct if needed.

#### The prompt

Prompt structure is essential for guiding the agent effectively. A well-organized prompt — with context, goal, steps and constraints — helps Claude understand not only *what* to do, but also *how* to do it. The [Anthropic prompt engineering guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) details these best practices.

Here's the prompt used for this task:

```markdown
## Context
I manage a Kubernetes cluster with Karpenter for autoscaling.
Available MCPs: grafana, victoriametrics, victorialogs, context7, chrome.

## Goal
Create a complete observability system for Karpenter: alerts + unified dashboard.

## Steps
1. **Documentation**: Via context7, fetch the latest Grafana docs
   (alerting, dashboards) and Victoria datasources
2. **Alerts**: Create alerts for:
    - Node provisioning errors
    - AWS API call failures
    - Quota exceeded
3. **Dashboard**: Create a unified Grafana dashboard integrating:
    - Metrics (provisioning time, costs, capacity)
    - Karpenter error logs
    - Kubernetes events related to nodes
4. **Validation**: Deploy via kubectl, then visually validate with
   the grafana and chrome MCPs
5. **Finalization**: If the rendering looks good, apply via the
   Grafana operator, commit and create the PR

## Constraints
- Use recent Grafana features (v11+)
- Follow best practices: dashboard variables, annotations,
  progressive alert thresholds
```

#### Step 1: Planning and decomposition

Claude analyzes the prompt and automatically generates a **structured plan** broken into sub-tasks. This decomposition lets you track progress and ensures each step is completed before moving to the next.

{{< img src="karpenter_plan.png" width="600" >}}

Here you can see the 4 identified tasks: create VMRule alerts, build the unified dashboard, validate with kubectl and Chrome, then finalize with commit and PR.

#### Step 2: Leveraging MCPs for context

This is where the **power of MCPs** becomes apparent. Claude uses **several simultaneously** to gather full context:

{{< img src="karpenter_mcp.png" width="1200" >}}

- **context7**: Retrieves Grafana v11+ documentation for alerting rules and dashboard JSON format
- **victoriametrics**: Lists all `karpenter_*` metrics available in my cluster
- **victorialogs**: Analyzes Karpenter logs to identify scaling events, provisioning errors and behavioral patterns

This combination allows Claude to generate code **tailored to my actual environment** rather than generic, potentially outdated examples.

#### Step 3: Visual validation with Chrome MCP

Once the dashboard is deployed via `kubectl`, Claude uses the **Chrome MCP** to open Grafana and visually validate the rendering. It can verify that panels display correctly, that queries return data, and adjust if necessary.

<center>
<video width="1200" autoplay loop muted playsinline onloadedmetadata="this.playbackRate=1.5;">
  <source src="chrome_karpenter.mp4" type="video/mp4">
  Your browser does not support video playback.
</video>
</center>

This is a concrete example of a **feedback loop**: Claude **observes the results of its actions** and can iterate until the desired outcome is achieved.

#### Result: complete observability

At the end of this workflow, Claude created a **complete PR**: 12 VMRule alerts (provisioning, AWS API, quotas, Spot interruptions) and a unified Grafana dashboard combining metrics, logs and Kubernetes events.

{{< img src="karpenter_summary.png" width="750" >}}

The ability to interact with my platform, identify errors and inconsistencies, then make adjustments automatically really **blew me away** 🤩. Rather than parsing Grafana JSON or listing metrics and logs through the various VictoriaMetrics UIs, I define my goal and the agent takes care of reaching it while consulting up-to-date documentation. A significant **productivity boost**!

---

### :building_construction: The spec as source of truth — building a new self-service capability

I've discussed in several previous articles the value of Crossplane for providing the right level of abstraction to platform users. This second use case puts that approach into practice: creating a **Crossplane composition** with the agent's help. This is one of the key principles of **Platform Engineering** — offering self-service tailored to the context while maintaining control over the underlying infrastructure.

{{% notice info "What is Spec-Driven Development (SDD)?" %}}
**Spec-Driven Development** is a paradigm where specifications — not code — serve as the primary artifact. In the age of agentic AI, SDD provides the guardrails needed to prevent "Vibe Coding" (unstructured prompting) and ensure agents produce maintainable code.

For those steeped in Kubernetes, here's an analogy 😉: the spec defines the **desired state**, and once validated by a human, the AI agent behaves somewhat like a *controller* — iterating based on results (tests, validations) until that state is reached. The difference: the human stays in the loop (**HITL**) to validate the spec *before* the agent starts, and to review the final result.

**Major frameworks in 2026:**

| Framework | Key strength | Ideal use case |
|-----------|-------------|----------------|
| **[GitHub Spec Kit](https://github.com/github/spec-kit)** | Native GitHub/Copilot integration | Greenfield projects, structured workflow |
| **[BMAD](https://github.com/bmad-code-org/BMAD-METHOD)** | Multi-agent teams (PM, Architect, Dev) | Complex multi-repo systems |
| **[OpenSpec](https://github.com/Fission-AI/OpenSpec)** | Lightweight, change-focused | Brownfield projects, rapid iteration |
{{% /notice %}}

{{% notice tip "My SDD variant for Platform Engineering" %}}
For [cloud-native-ref](https://github.com/Smana/cloud-native-ref), I created a variant inspired by GitHub Spec Kit that I'm evolving over time. I'll admit it's still quite experimental, but the results are already impressive.

**🛡️ Platform Constitution** — Non-negotiable principles are codified in a [constitution](https://github.com/Smana/cloud-native-ref/blob/main/docs/specs/constitution.md): `xplane-*` prefix for IAM scoping, mandatory zero-trust networking, secrets via External Secrets only. Claude checks every spec and implementation against these rules.

**👥 4 review personas** — Each spec goes through a checklist that forces you to consider multiple angles:

| Persona | Focus |
|---------|-------|
| **PM** | Problem clarity, user stories aligned with real needs |
| **Platform Engineer** | API consistency, KCL patterns followed |
| **Security** | Zero-trust, least privilege, externalized secrets |
| **SRE** | Health probes, observability, failure modes |

**⚡ Claude Code Skills** — The workflow is orchestrated by [skills](/en/post/series/agentic_ai/ai-coding-agent/#skills-unlocking-new-powers) (see previous section) that automate each step:

| Skill | Action |
|-------|--------|
| `/spec` | Creates the GitHub issue + pre-filled spec file |
| `/clarify` | Resolves `[NEEDS CLARIFICATION]` items with structured options |
| `/validate` | Checks completeness before implementation |
| `/create-pr` | Creates the PR with automatic spec reference |

{{< img src="sdd_workflow.png" width="700" >}}
{{% /notice %}}

#### Why SDD for Platform Engineering?

Creating a Crossplane composition isn't just a script — it's designing an **API for your users**. Every decision has lasting implications:

| Decision | Impact |
|----------|--------|
| API structure (XRD) | Contract with product teams — hard to change after adoption |
| Resources created | Cloud costs, security surface, operational dependencies |
| Default values | What 80% of users will get without thinking about it |
| Integrations (IAM, Network, Secrets) | Compliance, isolation, auditability |

SDD forces you to **think before coding** and **document decisions** — exactly what you need for a platform API.

#### Our goal: building a Queue composition

The product team needs a queuing system for their applications. Depending on the context, they want to choose between:
- **Kafka (via [Strimzi](https://strimzi.io/))**: for cases requiring streaming, long retention, or replay
- **AWS SQS**: for simple, serverless cases with native AWS integration

Rather than asking them to configure Strimzi or SQS directly (dozens of parameters), we'll expose a **simple, unified API**.

#### Step 1: Create the spec with `/spec` 📝

The `/spec` skill is the workflow entry point. It automatically creates:
- A **GitHub Issue** with the `spec:draft` label for tracking and discussions
- A **spec file** in `docs/specs/` pre-filled with the project template

```
/spec composition "Add queuing composition supporting Strimzi (Kafka) or SQS"
```

{{< img src="sdd_spec.png" width="950" >}}

Claude analyzes the project context (existing compositions, constitution, ADRs) and pre-fills the spec with an **initial design**. It also identifies **clarification points** — here, 3 key questions about scope and authentication.

The GitHub issue serves as a **centralized reference point** — that's where discussions happen and decision history lives — while the spec file evolves with the detailed design.

#### Step 2: Clarify design choices with `/clarify` 🤔

The generated spec contains `[NEEDS CLARIFICATION]` markers for decisions Claude can't make on its own. The `/clarify` skill presents them as **structured questions with options**:

{{< img src="sdd_clarify_1.png" width="950" >}}

Each question proposes options analyzed from **4 perspectives** (PM, Platform Engineer, Security, SRE) with a recommendation. You simply pick by navigating the proposed options.

Once all clarifications are resolved, Claude updates the spec with a decision summary:

{{< img src="sdd_clarify_2.png" width="900" >}}

These decisions are **documented in the spec** — six months from now, when someone asks "why no mTLS?", the answer will be right there.

#### Step 3: Validate and implement ⚙️

Before starting implementation, the `/validate` skill checks the spec's completeness:
- All required sections are present
- All `[NEEDS CLARIFICATION]` markers are resolved
- The GitHub issue is linked
- The project constitution is referenced

Once validated, I can start the implementation. Claude enters **plan mode** and launches exploration agents **in parallel** to understand existing patterns:

{{< img src="sdd_implement_1.png" width="1100" >}}

Claude explores existing compositions (`SQLInstance`, `EKS Pod Identity`, the Strimzi configuration) to understand the project's conventions **before writing a single line of code**.

The implementation generates the appropriate resources based on the chosen backend:

{{< img src="sdd_implement_summary.png" width="450" >}}

For **each backend**, the composition creates the necessary resources while following the project's conventions:
- `xplane-*` prefix for all resources (IAM convention)
- `CiliumNetworkPolicy` for zero-trust networking
- `ExternalSecret` for credentials (no hardcoded secrets)
- `VMServiceScrape` for observability

#### Step 4: Final validation 🛂

The `/validate` skill checks not only the spec but also the **implementation**:

{{< img src="sdd_validation.png" width="800" >}}

The validation covers:
- **Spec**: Sections present, clarifications resolved, issue linked
- **Implementation**: Phases completed, examples created, CI passing
- **Review checklist**: The 4 personas (PM, Platform Engineer, Security, SRE)

Items marked "N/A" (E2E tests, documentation, failure modes) are clearly identified as optional for this type of composition.

#### Result: the final user API 🎉

Developers can now declare their needs in just a few lines:

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: Queue
metadata:
  name: orders-queue
  namespace: ecommerce
spec:
  # Kafka for streaming with retention
  type: kafka
  clusterRef:
    name: main-kafka
  config:
    partitions: 6
    retentionDays: 7
```

Or for SQS:

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: Queue
metadata:
  name: notifications-queue
  namespace: notifications
spec:
  # SQS for simple cases
  type: sqs
  config:
    visibilityTimeout: 30
    enableDLQ: true
```

In both cases, the platform automatically handles:
- Resource creation (Kafka topics or SQS queues)
- Authentication (SASL/SCRAM or IAM)
- Monitoring (metrics exported to VictoriaMetrics)
- Network security (CiliumNetworkPolicy)
- Credential injection into the application's namespace

Without SDD, I would have probably jumped straight into writing the Crossplane composition, without stepping back to take a **proper product approach** or **flesh out the specifications**. And even then, delivering this new service would have taken much longer.

By structuring the thinking upfront, **every decision is documented** and justified before the first line of code. The four perspectives (PM, Platform, Security, SRE) ensure no angle is missed, and the final PR references the spec — the reviewer has all the context they need.


## :thought_balloon: Final thoughts

Through this article, we've explored agentic AI and how its principles can be useful on a daily basis. An agent with access to rich context (`CLAUDE.md`, skills, MCPs...) can be **truly** effective: quality results and, above all, impressive speed! The SDD workflow also helps formalize your intent and better guide the agent for more complex projects.

### Things to watch out for

That said, as impressive as the results may be, it's important to stay **clear-eyed**. Here are some lessons I've learned after several months of use:

* **Avoid dependency and keep learning** — systematically review the specs and generated code, understand *why* that solution was chosen
* **Force yourself to work without AI** — I make a point of at least 2 "old school" sessions per week
* **Use AI as a teacher** — asking it to explain its reasoning and choices is an excellent way to learn

{{% notice warning "Confidentiality and proprietary code" %}}
If you work with sensitive or proprietary code:
- Use the **Team** or **Enterprise** plan — your data isn't used for training
- Request the **Zero-Data-Retention** (ZDR) option if needed
- **Never** use the Free/Pro plan for confidential code

See the [privacy documentation](https://www.anthropic.com/legal/privacy) for more details.
{{% /notice %}}

### :bulb: Getting the most out of it

{{% notice info "Dedicated article" %}}
Tips and workflows I've picked up along the way (CLAUDE.md, hooks, context management, worktrees, plugins...) have been compiled in a dedicated article: [A few months with Claude Code: tips and workflows that helped me](/post/series/agentic_ai/ai-coding-tips/).
{{% /notice %}}

### My next steps

This is a concern I share with many developers: **what happens if Anthropic changes the rules of the game?** This fear actually materialized in early January 2026, when Anthropic [blocked without warning](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses) access to Claude through third-party tools like [OpenCode](https://github.com/charmbracelet/crush).

Given my affinity for open source, I'm looking at exploring open alternatives: **[Mistral Vibe](https://mistral.ai/news/devstral-2-vibe-cli)** with Devstral 2 (72.2% SWE-bench) and **[Crush](https://github.com/charmbracelet/crush)** (formerly OpenCode) (multi-provider, local models via Ollama) for instance.

---

## :bookmark: References

### Guides and best practices
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) — Comprehensive guide by sshh

### Spec-Driven Development
- [GitHub Spec Kit](https://github.com/github/spec-kit) — GitHub's SDD toolkit
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) — Lightweight SDD for brownfield projects
- [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD) — Multi-agent SDD

### Plugins, Skills and MCPs
- [Code-Simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) — AI code cleanup
- [Claude-Mem](https://github.com/thedotmack/claude-mem) — Persistent memory across sessions
- [CC-DevOps-Skills](https://github.com/akin-ozer/cc-devops-skills) — 31 ready-to-use DevOps skills
- [Awesome Claude Code Plugins](https://github.com/ccplugins/awesome-claude-code-plugins) — Curated list

### Cited studies
- [METR Study on AI Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) — Productivity study
- [State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/) — Qodo
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Research

### Resources
- [Cloud Native Ref](https://github.com/Smana/cloud-native-ref) — My reference repo
- [SWE-bench Leaderboards](https://www.swebench.com/) — Reference benchmark
