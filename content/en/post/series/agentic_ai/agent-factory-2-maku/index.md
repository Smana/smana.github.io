+++
author = "Smaine Kahlouch"
title = "`Agent Factory`: Maku, the conductor"
date = "2026-11-21"
summary = "One prompt, and the factory handles everything up to the PR. We follow a real demo to discover `Maku`: its components, the room, workflows and merge policies."
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

Running an agent in a pod has become fairly common: just look at the talks at recent conferences (KubeCon NA in particular). Running a **software factory for a team** is another story: you have to coordinate several agents, keep budgets in check, isolate every run, decide what ships to production… and above all make sure the whole thing stays **pleasant to use**. If handing off a task means leaving your usual tools, nobody will bother.

That's the job of **[Maku](https://github.com/Smana/maku)**, the heart of the Agent Factory. The [first post](/post/series/agentic_ai/agent-factory-1-overview/) only touched on it; time to open the box.

For once, I won't start with the architecture. Let's take a dead-simple case instead: **I have a change to make, and I hand it to the factory from my coding agent**. We'll watch what happens, then look at the components involved. 🚀

## 🎯 Goals

* 🎬 Follow a **real task**, from prompt to merge
* 🏗️ Understand what happens under the hood: **Maku's components** and the **room**
* 🚪 Discover the other doors in: **Slack** and **GitHub**
* 🔄 Get a first look at **workflows** and **merge policies** (the next post is dedicated to them)

## 🎬 A task handed off without leaving the terminal

<center>
<video width="1200" controls autoplay loop muted playsinline>
  <source src="maku-demo.mp4" type="video/mp4">
  Your browser does not support video playback.
</video>
</center>
<center><i>The long version (2 minutes) is on the <a href="https://blog.ogenki.io/maku/">demo page</a>.</i></center>
<br>

Some context: I'm working on [airef.ogenki.io](https://airef.ogenki.io/) and, in passing, I spot a mistake in the configuration reference: the Slack app's budget shows the wrong values.

The fix is tiny, well-scoped and easy to verify: exactly the kind of task I hand to Maku so I can get on with something else. Without leaving my OpenCode session, I delegate:

```text
/maku handoff the config reference has the wrong Slack app budget; fix it to dailyUSD 5, maxCostUSD 2
```

Here's what happens next:

1. 📝 **Maku's planner drafts the issue.** From my one sentence, it builds a first proposal (what's wrong, what's expected, the acceptance criterion) that I can tweak before approving it.

2. ✅ **I confirm.** A form, which my coding agent can't answer on my behalf, shows me the issue, the workflow (`standard`) and the task's budget ($5.10 at most). I confirm: [issue #479](https://github.com/Smana/ai-native-ref/issues/479) is created under my GitHub account, the task starts, and I go back to my work.

3. 🤖 **An agent gets to work.** Somewhere on the cluster, it fixes the page, runs the repository's checks (6 out of 6), then opens [PR #480](https://github.com/Smana/ai-native-ref/pull/480): one line changed in one file.

4. 👀 **Another agent reviews.** It starts from scratch, knowing nothing of the first one's reasoning, reruns the checks and gives its verdict: `approve`. Had it asked for changes (`changes`), the implementer would have picked the PR back up to address its comments: the reviewer itself isn't allowed to push code.

5. 🔔 **I get pinged.** I didn't watch any of it: my coding agent was tracking the task in the background, and the room tells me what's expected of me:

   ```text
   Needs you · 1
   Approve the pull request
   a maintainer approves PR #480: approve the head you read on the PR page, […]; the merger merges that head
   Why: it would auto-merge, and the merge gate is in shadow
   ```

6. 🎉 **I approve, it's merged.** I read the diff, CI is green, I click **Approve**. Thirty seconds later, the PR is merged and the issue closed.

My side of it: one sentence in the terminal, one form, one line to review and one click. Twelve minutes end to end, for $0.14.

Getting there takes a single command:

```bash
# On my machine, once per coding agent
maku setup opencode      # or claude, codex, gemini, cursor, copilot
```

It opens the SSO login, registers the `maku` MCP server and, for Claude Code and OpenCode, also adds the `/maku` command:

* `/maku handoff <description>`: the one we just saw
* `/maku task [#issue]`: where the task stands, what it needs from me, how much it has cost
* `/maku inbox`: all my tasks waiting on a human
* `/maku cancel <task>`: stop a task, or drop it along with its PR and branch

{{% notice info "Prerequisites" %}}
`maku setup` connects to a factory that's **already deployed**: it asks for the URL of its rooms. No cluster yet? `task demo` spins everything up on a laptop (see [below](#-running-it-yourself)).
{{% /notice %}}

## 🔍 Behind the scenes

One sentence, one issue, two agents, a verdict, a merge: who did what? Time to look under the hood.

Maku follows the **Kubernetes operator** pattern: everything is declared in CRDs (`Task`, `Room`, `AgentRun`, `MergePolicy`…) that a handful of controllers written in Go reconcile in a loop. So a task is a Kubernetes resource like any other: you can run `kubectl get tasks`, and I love that! 😄

{{< img src="maku-components.png" alt="Maku's components: four doors in (coding agent, Web UI, Slack, GitHub), the control plane with maku-broker, maku-slack, maku-factory and maku-run-controller, the gVisor sandbox of each run with maku-harness and maku-bridge, and maku-gateway as the only way out to the models and MCP tools" width="1300" link="true" caption="Four doors in, one control plane, one sandbox per run." >}}
<br>

Let's replay the demo, one component at a time:

* 📓 **`maku-broker`**, the memory. Its planner drafted the issue from my prompt, through the MCP server my coding agent is connected to. It also keeps each **room**'s log in PostgreSQL and serves the Web UI.

* 🏭 **`maku-factory`**, the dispatcher. It picked up my issue, created the `Task`, chose the workflow and the budget, started the agents, then applied the **merge gate** at the end.

* 🚀 **`maku-run-controller`**, the launcher. For each agent, it creates an isolated pod in a **gVisor** sandbox, with its own `ServiceAccount` and a network that's closed by default. On my cluster, a Crossplane composition from [cloud-native-ref](https://github.com/Smana/cloud-native-ref) plays that part.

* 🤖 **`maku-harness`**, the agent. It's the image that runs in the sandbox: [OpenHands](https://github.com/OpenHands/OpenHands), `gh`, and what it takes to open the PR.

* 🌉 **`maku-bridge`**, the messenger. This sidecar relays the agent's events up to the room and brings my instructions back down (the *steers*, more on those below). The agent itself never holds the room's token.

* 🚪 **`maku-gateway`**, the way out. It's really just [agentgateway](https://agentgateway.dev/), which Maku merely configures. It's the only path to the models and the MCP tools. It verifies each run's identity and enforces the spending caps: the budget and cost shown in the room come from there.

On the GitHub side, in my setup, three GitHub Apps split the permissions: the factory's (issues, labels, comments), the agents' (they push a branch and open the PR), and the *merger*'s, the only one allowed to merge.

## 🎛️ The room: see everything, stay in control

Every task has its **room**: the place where you follow everything that happens in it. In the demo, its link showed up in my terminal.

{{< img src="room.gif" width="480" >}}

In *One Piece*, Trafalgar Law's *Room* is a sphere outside of which his power doesn't work. Inside it, though, he can examine anything, cut it apart and put it back together, and nothing dies. A Maku room does the same for a task: everything in it can be seen, everything can be steered, and nothing gets lost.

Why not settle for the pod's logs? Because they vanish with it, and they blur what the agent *says* it's doing with what it *actually* does. The room stays. It is:

* 🔒 **Append-only**: the broker can add and read events, never edit or delete them. When the room closes, the log is sealed.

* 🕵️ **Attributed and scrubbed**: every event carries the verified identity of its author (human, agent or factory), and secrets are masked by [gitleaks](https://github.com/gitleaks/gitleaks) before anything is written.

* 🎮 **Steerable live**: you can leave an instruction for the next run, redirect the agent mid-flight (that's a *steer*), or approve a request.

* 👥 **Shared**: a room hosts several humans and agents. Everyone can follow along and chat, but only one *driver* at a time can redirect the agent, and whoever guided a run can't approve its requests.

In the Web UI, the **NEXT** box tells you in one sentence what's expected of you:

{{< img src="maku-room.png" alt="A Maku room waiting for a maintainer: the status “Waiting — review and approve PR #42”, the token budget, the reviewer's latest verdict, the NEXT box saying that approving the current commit merges it, the exact action under “Needs you”, then the implementer's event feed" width="1100" caption="A room waiting for my approval: the next step, the action to take, and the run's event feed (try it in the [interactive demo](https://blog.ogenki.io/maku/))." >}}
<br>

And when several tasks run in parallel, the ones that need you rise to the top of the list:

{{< img src="maku-rooms.png" alt="The list of Maku rooms: tasks waiting on a human carry a “Needs you” badge and come first, each with its repository, status, PR and latest activity" width="1100" caption="The room list: “Needs you” first." >}}
<br>

## 🚪 The other doors in

The coding agent is my favorite door, but it's not the only one. Behind each of them, the path is exactly the same.

### 💬 From Slack

A passing remark in a channel can become a task. Mention `@factory`, in your own words:

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

The classic route: write an issue, add the `factory/ready` label, and Maku takes it from there. A few labels let you steer what follows: `workflow:careful` for a delicate task, `factory/hold` to freeze the merge, `factory/stop` to halt everything.

On the issue, Maku reports each step: the current run, the branch, the budget spent, and the link to the room.

## 🔄 Workflows and merge policies, in a nutshell

In the demo, two agents took turns, and the PR waited for me instead of merging on its own. Two mechanisms decide that; **the next post is entirely dedicated to them**.

### 👥 Workflows: who works on the task?

An agent that writes the code and then grades its own work quickly talks itself into believing it's done. So a **workflow** defines which roles take turns on a task:

* **`standard`**, the default workflow and the one from the demo: an **implementer** writes the change and opens the PR, then a **reviewer** goes over it with fresh eyes.
* **`quick`**: the implementer alone, for trivial changes.
* **`careful`**: the `standard` workflow, bracketed by a **scoper** that scopes the task first (and asks a question if the issue is vague) and a **judge** that checks the acceptance criteria before the review.

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

By default, a maintainer merges. On my side, I've turned on the `approveToMerge` option: **my approval of the current commit is enough**, and the *merger* GitHub App does the rest. In the demo, a policy covered this kind of docs fix, but it's still in `shadow`: in `live`, it would have merged on its own; here, the call was mine.

Automatic merging has to be earned. A `MergePolicy` describes a family of low-risk changes (fixing docs, for instance): allowed files, diff size, green CI, the reviewer's verdict. And it moves through stages:

1. **`shadow`**: the policy watches. It records what it *would have* decided, but a human still merges.
2. **`live`**: once it has enough good decisions under its belt, you promote it by changing one line. From then on it merges on its own, with a daily cap.
3. **Back to square one**: at the first incident, a circuit breaker shuts the lane down and hands control back to humans.

## 🧭 Running it yourself

**Maku stands on its own.** A Kubernetes cluster, a Helm chart, and you plug in what you already have: your OIDC provider, your PostgreSQL, your CNI. Nothing forces you to adopt my infrastructure. And for a first try, `task demo` brings the whole thing up on a laptop in fifteen minutes (no gVisor sandbox, on a public repository, with your own model key).

In my case, I've split things across three repositories (covered in the [previous post](/post/series/agentic_ai/agent-factory-1-overview/)):

* **[`maku`](https://github.com/Smana/maku)**: the controller and its charts, the part you install.
* **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)**: my factory configuration (budgets, allowed models, policies). It lives in its own repository, **out of the agents' reach**.
* **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)**: the cluster that hosts it all.

That's one way to do it, not a requirement.

{{% notice tip "Looking for a partner 🤝" %}}
I've put a lot of time into this project (and quite a few tokens 😅). Next, I'd like to put it to the test in the real world: I'm looking for a **partner company** willing to try it with its teams, on real use cases. If that sounds like you, get in touch!
{{% /notice %}}

## 💭 Final thoughts

At first, all I wanted was a small web interface to launch agents in sandboxes and keep a record of what they did. Then I got hooked and, one thing leading to another, it became the Agent Factory, with Maku at its heart.

The compass hasn't moved, though: it has to feel **natural for the developer**. Whether you start from Claude Code, OpenCode, a Slack thread or a GitHub issue, all the security machinery stays backstage, and the human keeps the final say where their judgment matters.

A few limitations worth knowing:

* the planner only reads public repositories;
* approve-to-merge is an option, off by default;
* for now, I'm the only maintainer using it day to day;
* the cost of a task depends heavily on the model you pick.

In the next post, we'll take a close look at **workflows** and **merge policies**: how the roles hand off to each other, and how a policy earns (or loses) the right to merge on its own.

## 🔖 References

* [`maku`](https://github.com/Smana/maku): the repository · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installation](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart on a laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Demo: the long version of the screencast and a room to play with](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref): the AI platform · [documentation](https://airef.ogenki.io/) · [user guide](https://airef.ogenki.io/docs/use/guide/) · [status and roadmap](https://airef.ogenki.io/docs/status/)
* [ADR-0057: why `ai-native-ref` has its own repository](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [agentgateway](https://agentgateway.dev/) · [OpenHands](https://github.com/OpenHands/OpenHands) · [gVisor](https://gvisor.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
