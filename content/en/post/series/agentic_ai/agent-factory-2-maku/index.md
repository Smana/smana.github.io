+++
author = "Smaine Kahlouch"
title = "`Agent Factory`: Maku, the conductor"
date = "2026-11-21"
summary = "One prompt, and the factory handles everything up to the PR. We follow a real demo to explore Maku: its components, the room, workflows, and merge policies."
featured = false
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.jpg"
+++

{{% notice info "Agent Factory series 🏭" %}}
1. [Overview: agents and humans, one crew](/post/series/agentic_ai/agent-factory-1-overview/)
2. **Maku, the conductor** (you are here)
3. Workflows and merge policies (coming up)
4. The harness: a sandbox for every agent (coming up)
5. Observing and keeping a fleet of agents in check (coming up)
6. Agent to agent: RunLore hands its fixes to the factory (coming up)
{{% /notice %}}

Running an agent in a pod has become fairly common—just look at the talks at recent conferences (especially KubeCon NA). Running a **software factory for a team** is another story: you have to coordinate several agents, keep budgets in check, isolate every run, decide what ships to production… and above all make sure the whole thing stays **user-friendly**. If handing off a task means leaving your usual tools, nobody will bother.

That's the job of **[Maku](https://github.com/Smana/maku)**, the heart of the Agent Factory. The [first post](/post/series/agentic_ai/agent-factory-1-overview/) only touched on it—time to open it up.

This time, I'm skipping the architecture diagram up front. Let's start with a dead-simple example instead: **I have a change to make, and I hand it to the factory from my coding agent**. We'll watch what happens, then look at the components involved. 🚀

## 🎯 Goals

* 🎬 Follow a **real task**, from prompt to merge
* 🏗️ Understand what happens under the hood: **Maku's components** and the **room**
* 🚪 Explore the other ways in: **Slack** and **GitHub**
* 🔄 Get a first look at **workflows** and **merge policies** (the next post is dedicated to them)

## 🎬 Handing off a task without leaving the terminal

<center>
<video width="1200" controls autoplay loop muted playsinline>
  <source src="maku-demo.mp4" type="video/mp4">
  Your browser does not support video playback.
</video>
</center>
<center><i>The long version (2 minutes) is on the <a href="https://blog.ogenki.io/maku/">demo page</a>.</i></center>
<br>

Some context: I'm working on [airef.ogenki.io](https://airef.ogenki.io/) and along the way, I spot a mistake in the configuration reference—the Slack app's budget shows the wrong values.

The fix is tiny, well-scoped, and easy to verify: exactly the kind of task I hand to Maku so I can get on with something else. Without leaving my OpenCode session, I delegate:

```text
/maku handoff the config reference has the wrong Slack app budget; fix it to dailyUSD 5, maxCostUSD 2
```

Here's what happens next:

1. 📝 **Maku's planner drafts the issue.** From that single sentence, it puts together a draft (what's wrong, what's expected, and the acceptance criteria) that I can tweak before approving it.

2. ✅ **I confirm.** A confirmation form pops up—one my coding agent can't submit on my behalf—showing the issue, the workflow (`standard`), and the max budget ($5.10). I confirm: [issue #479](https://github.com/Smana/ai-native-ref/issues/479) is created under my GitHub account, the task starts, and I go back to my work.

3. 🤖 **An agent gets to work.** Somewhere on the cluster, it fixes the page, runs the repository's checks (6 out of 6), then opens [PR #480](https://github.com/Smana/ai-native-ref/pull/480): one line changed in one file.

4. 👀 **Another agent reviews.** It starts from scratch without seeing the first agent's reasoning, reruns the checks, and gives its verdict: `approve`. If it had requested changes (`changes`), the implementer would have picked the PR back up to address the feedback—the reviewer isn't allowed to push code.

5. 🔔 **I get pinged.** I didn't watch any of it—my coding agent was tracking the task in the background, and the room tells me what it needs from me:

   ```text
   Needs you · 1
   Approve the pull request
   a maintainer approves PR #480: approve the head you read on the PR page, […]; the merger merges that head
   Why: it would auto-merge, and the merge gate is in shadow
   ```

6. 🎉 **I approve, it's merged.** I read the diff, CI is green, I click **Approve**. Thirty seconds later, the PR is merged and the issue closed.

On my end: one sentence in the terminal, one form, one line to review, and one click. Twelve minutes end to end, for $0.14.

Setting this up takes a single command:

```bash
# On my machine, once per coding agent
maku setup opencode      # or claude, codex, gemini, cursor, copilot
```

It opens the SSO login, registers the `maku` MCP (Model Context Protocol) server, and—for Claude Code and OpenCode—also adds the `/maku` command:

* `/maku handoff <description>`: the one we just saw
* `/maku task [#issue]`: where the task stands, what it needs from me, how much it has cost
* `/maku inbox`: all my tasks waiting on a human
* `/maku cancel <task>`: stop a task, or drop it along with its PR and branch

{{% notice info "Prerequisites" %}}
`maku setup` connects to a factory that's **already deployed**: it asks for the URL of its rooms. No cluster yet? `task demo` spins everything up on a laptop (see [below](#-running-it-yourself)).
{{% /notice %}}

## 🔍 Behind the scenes

One sentence, one issue, two agents, a verdict, a merge: who did what? Time to look under the hood.

Maku follows the **Kubernetes operator** pattern: everything is declared in CRDs (Custom Resource Definitions—`Task`, `Room`, `AgentRun`, `MergePolicy`…) that a handful of Go controllers reconcile continuously. So a task is a Kubernetes resource like any other—you can run `kubectl get tasks`, and I love that! 😄

{{< img src="maku-components.png" alt="Maku's components: four ways in (coding agent, Web UI, Slack, GitHub), the control plane with maku-broker, maku-slack, maku-factory and maku-run-controller, the gVisor sandbox of each run with maku-harness and maku-bridge, and maku-gateway as the only way out to the models and MCP tools" width="1300" link="true" caption="Four ways in, one control plane, one sandbox per run." >}}
<br>

Let's replay the demo, one component at a time:

* 📓 **`maku-broker`**, the memory. Its planner drafted the issue from my prompt via the MCP server connected to my coding agent. It also keeps each **room**'s log in PostgreSQL and serves the Web UI.

* 💬 **`maku-slack`**, the Slack bridge. It listens for `@factory` mentions, runs the same planning flow in a thread, and posts concise status updates.

* 🏭 **`maku-factory`**, the dispatcher. It picked up my issue, created the `Task`, chose the workflow and the budget, started the agents, then applied the **merge gate** at the end.

* 🚀 **`maku-run-controller`**, the launcher. For each agent, it creates an isolated pod in a **gVisor** sandbox, with its own `ServiceAccount` and default-deny networking. On my cluster, a Crossplane composition from [cloud-native-ref](https://github.com/Smana/cloud-native-ref) handles that.

* 🤖 **`maku-harness`**, the agent. It's the image that runs in the sandbox: [OpenHands](https://github.com/OpenHands/OpenHands), `gh`, and everything needed to open a PR.

* 🌉 **`maku-bridge`**, the messenger. This sidecar streams the agent's events up to the room and sends my instructions back down (the *steers*—more on those below). The agent itself never holds the room's token.

* 🚪 **`maku-gateway`**, the way out. Under the hood, it's just [agentgateway](https://agentgateway.dev/), configured by Maku. It's the only path to the models and the MCP tools. It verifies each run's identity and enforces the spending caps—the budget and cost shown in the room come from there.

On the GitHub side, my setup splits permissions across three GitHub Apps:

* **Factory**: manages issues, labels, and comments.
* **Agents**: push branches and open PRs.
* **Merger**: the only one allowed to merge.

## 🎛️ The room: see everything, stay in control

Every task has its **room**—the space where you can follow everything happening on that task. In the demo, a link to it popped up right in my terminal.

{{< img src="room.gif" width="480" >}}

In *One Piece*, Trafalgar Law's *Room* is a sphere where his power applies—outside it, nothing; inside it, he can inspect anything, take it apart, and put it back together, and nothing dies. A Maku room does the same for a task: everything in it can be seen, everything can be steered, and nothing gets lost.

Why not settle for the pod's logs? Because they disappear as soon as the pod is gone, and they mix up what the agent *says* it's doing with what it *actually* does. The room stays. It is:

* 🔒 **Append-only**: the broker can add and read events, never edit or delete them. When the room closes, the log is sealed.

* 🕵️ **Attributed and scrubbed**: every event carries the verified identity of its author (human, agent, or factory), and secrets are masked by [gitleaks](https://github.com/gitleaks/gitleaks) before anything is written.

* 🎮 **Steerable live**: you can leave an instruction for the next run, redirect the agent mid-flight (that's a *steer*), or approve a request (like a PR approval or a budget bump).

* 👥 **Shared**: a room hosts several humans and agents. Everyone can follow along and chat, but only one *driver* at a time can redirect the agent, and whoever guided a run can't approve its requests.

In the Web UI, the **NEXT** box tells you in one sentence what you need to do next:

{{< img src="maku-room.png" alt="A Maku room waiting for a maintainer: the status “Waiting — review and approve PR #530”, the token and dollar budget, the reviewer's approve verdict, the NEXT box explaining that the merge gate is in shadow so a maintainer approves the current commit, the “Approve the pull request” card under “Needs you”, and the rail with the implementer and the reviewer, each with its time and cost" width="1100" caption="A room waiting for my approval: the next step, the action to take, and what each stage cost (try it in the [interactive demo](https://blog.ogenki.io/maku/))." >}}
<br>

And when several tasks run in parallel, the ones that need you rise to the top of the list:

{{< img src="maku-rooms.png" alt="The list of Maku rooms filtered to what needs me: each task waiting on a human carries a badge saying why (review, failed), with its repository, status, PR and latest activity; New task, top right, plans a request into an issue" width="1100" caption="The room list: “Needs you” first." >}}
<br>

## 🚪 Other ways in

Starting from my coding agent is my favorite way in, but it's not the only one. You can also kick off tasks from Slack or GitHub (the Web UI we just saw is mainly for following, steering, and approving rooms)—no matter how you start, the flow behind it is identical.

### 💬 From Slack

A quick message in a channel can become a task. Mention `@factory`, in your own words:

```text
@factory the install link in the README still points to the old page, it should target docs/install.md
```

Maku replies in the thread with a plan card:

```text
Plan for Smana/ai-native-ref (revision 1)
docs: the install link in the README is dead
Workflow  [ standard (implementer, reviewer) ▾ ]
Budget    up to $2.00
[ Confirm ]  [ Cancel ]
```

You adjust it by replying in the thread, then a maintainer clicks **Confirm** and off it goes. After that, Slack stays quiet: one message when the task starts, one if it needs you, one when the PR opens, and a one-line recap at the end (PR merged, cost, duration).

### 🐙 From GitHub

The classic route: write an issue, add the `factory/ready` label, and Maku takes it from there. A few labels let you control what happens next:

| Label | Effect |
| --- | --- |
| `factory/ready` | Hands the issue over to Maku |
| `workflow:careful` | Uses the `careful` workflow for tricky tasks |
| `factory/hold` | Freezes the merge |
| `factory/stop` | Halts the task immediately |

On the issue, Maku reports each step: the current run, the branch, the budget spent, and the link to the room.

## 🔄 Workflows and merge policies, in a nutshell

In the demo, two agents took turns, and the PR waited for me instead of merging on its own. Two mechanisms control that—and **the next post is all about them**.

### 👥 Workflows: who works on the task?

An agent that writes the code and then grades its own work quickly talks itself into believing it's done. So a **workflow** defines which roles take turns on a task:

* **`standard`**, the default workflow and the one from the demo: an **implementer** writes the change and opens the PR, then a **reviewer** goes over it with fresh eyes.
* **`quick`**: the implementer alone, for trivial changes.
* **`careful`**: the `standard` workflow, wrapped with a **scoper** that frames the task upfront (and asks a question if the issue is vague) and a **judge** that checks the acceptance criteria before the review.

<center>
<video width="540" autoplay loop muted playsinline>
  <source src="maku-workflows.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
</center>
<center><i>The workflow picks the roles; the merge gate judges the diff.</i></center>
<br>

{{% notice tip "What about a complex task?" %}}
The factory isn't limited to small fixes. For a new feature, everything hinges on a **well-written issue**: clear specs and verifiable acceptance criteria. With `careful`, the scoper uses them to scope the work, the judge to check the result, and the team follows it all in the same room.
{{% /notice %}}

### 🚦 Merge policies: who gets to merge?

By default, a maintainer merges. In my setup, I turned on the `approveToMerge` option: **my approval of the current commit is enough**, and the *merger* GitHub App does the rest. In the demo, a policy already covered this kind of docs fix, but it's still in `shadow` mode. In `live` mode, it would have auto-merged; here, I still made the final call.

Auto-merging has to be earned. A `MergePolicy` describes a class of low-risk changes (fixing docs, for instance): allowed files, diff size, green CI, and the reviewer's verdict. It moves through stages:

1. **`shadow`**: the policy watches. It records what it *would have* decided, but a human still merges.
2. **`live`**: once it has enough good decisions under its belt, you promote it by changing one line. From then on it merges on its own, with a daily cap.
3. **Back to square one**: at the first incident, a circuit breaker shuts the lane down and hands control back to humans.

## 🧭 Running it yourself

**Maku stands on its own.** Bring a Kubernetes cluster and a Helm chart, and plug in what you already have—your OIDC provider, PostgreSQL, and CNI. You don't have to adopt my stack at all. And for a first try, `task demo` brings the whole thing up on a laptop in about 15 minutes (without gVisor, against a public repo, using your own model API key).

In my case, I've split things across three repositories (covered in the [previous post](/post/series/agentic_ai/agent-factory-1-overview/)):

* **[`maku`](https://github.com/Smana/maku)**: the controller and its charts, the part you install.
* **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)**: my factory configuration (budgets, allowed models, policies). It lives in its own repository, **out of the agents' reach**.
* **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)**: the cluster that hosts it all.

That's one way to do it, not a requirement.

{{% notice tip "Looking for a partner 🤝" %}}
I've put a lot of time into this project (and quite a few tokens 😅). Next, I'd like to put it to the test in the real world: I'm looking for a **partner company** willing to try it with its teams, on real use cases. If that sounds like you, get in touch!
{{% /notice %}}

## 💭 Final thoughts

At first, all I wanted was a small web interface to launch agents in sandboxes and keep a record of what they did. Then I got hooked, one thing led to another, and it grew into the Agent Factory, with Maku at its heart.

My north star hasn't changed, though: it has to feel **natural for the developer**. Whether you start from Claude Code, OpenCode, a Slack thread, or a GitHub issue, all the security machinery stays backstage, and the human keeps the final say where their judgment matters.

A few things to keep in mind:

* The planner only reads public repositories.
* Approve-to-merge is opt-in (off by default).
* For now, I'm the only maintainer using it day to day.
* The cost of a task depends heavily on the model you pick.

In the next post, we'll take a close look at **workflows** and **merge policies**: how the roles hand off to each other, and how a policy earns (or loses) the right to merge on its own.

## 🔖 References

* [`maku`](https://github.com/Smana/maku): the repository · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installation](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart on a laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Demo: the long version of the screencast and a room to play with](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref): the AI platform · [documentation](https://airef.ogenki.io/) · [user guide](https://airef.ogenki.io/docs/use/guide/) · [status and roadmap](https://airef.ogenki.io/docs/status/)
* [ADR-0057: why `ai-native-ref` has its own repository](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [agentgateway](https://agentgateway.dev/) · [OpenHands](https://github.com/OpenHands/OpenHands) · [gVisor](https://gvisor.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
