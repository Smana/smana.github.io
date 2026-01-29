+++
author = "Smaine Kahlouch"
title = "A few months with `Claude Code`: tips and workflows that helped me"
date = "2026-01-29"
summary = "CLAUDE.md, hooks, context management, worktrees, plugins, anti-patterns: everything I wish I'd known from the start."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
aliases = ["/en/post/ai-coding-tips/"]
tags = ["ai", "devxp", "tooling"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Agentic AI Series — Part 2" %}}
This article follows [Agentic Coding: concepts and hands-on use cases](/en/post/series/agentic_ai/ai-coding-agent/), where we explored the fundamentals of agentic AI — tokens, MCPs, Skills, Tasks — and two detailed practical examples. **Here, we dive into advanced practices**: how to get the most out of Claude Code on a daily basis.
{{% /notice %}}

As with any tool you adopt, it takes time to refine how you use it. After iterating on my config and workflows, I've found an efficient rhythm with Claude Code. Here's what works for me.

{{% notice info "Living article" %}}
This article will be updated as I discover new things and as tools evolve. Feel free to check back from time to time for new tips.
{{% /notice %}}

---

## :scroll: CLAUDE.md: persistent memory

The `CLAUDE.md` file is the **first optimization lever**. These are instructions automatically injected into every conversation. If you don't have one yet, it's the first thing to set up.

### Loading hierarchy

Claude Code loads `CLAUDE.md` files according to a specific hierarchy:

| Location | Scope | Use case |
|----------|-------|----------|
| `~/.claude/CLAUDE.md` | All sessions, all projects | Global preferences (language, commit style) |
| `./CLAUDE.md` | Project (shared via git) | Team conventions, build/test commands |
| `./CLAUDE.local.md` | Project (not versioned) | Personal preferences on this project |
| `./subfolder/CLAUDE.md` | Subtree | Instructions specific to a module |

Files are **cumulative**: Claude loads all of them from most global to most local. In case of conflict, the most local one wins.

### What I put in mine

Here's what the `CLAUDE.md` for [cloud-native-ref](https://github.com/Smana/cloud-native-ref) actually contains:

- **Build/test/lint commands** — the first lines, so Claude knows how to validate its work
- **Project conventions** — `xplane-*` prefix for IAM, Crossplane composition structure, KCL patterns
- **Architecture summary** — key folder structure, not a novel
- **Common pitfalls** — traps Claude keeps falling into if you don't warn it (e.g., wrong default namespace, label format)

What I **don't** put in it: exhaustive documentation (that's what [Skills](/en/post/series/agentic_ai/ai-coding-agent/#skills-unlocking-new-powers) are for), long code examples (I reference existing files instead), and obvious instructions Claude already knows.

Here's a condensed excerpt from the `CLAUDE.md` of [cloud-native-ref](https://github.com/Smana/cloud-native-ref) to illustrate:

```markdown
## Common Commands

### Terramate Operations
# Deploy entire platform
terramate script run deploy
# Preview changes across all stacks
terramate script run preview
# Check for configuration drift
terramate script run drift detect

## Crossplane Resources
- **Resource naming**: All Crossplane-managed resources prefixed with `xplane-`

## KCL Formatting Rules
**CRITICAL**: Always run `kcl fmt` before committing KCL code. The CI enforces strict formatting.

### Avoid Mutation Pattern (Issue #285)
Mutating dictionaries after creation causes function-kcl to create duplicate resources.
   # ❌ WRONG - Mutation causes DUPLICATES
   _deployment = { ... }
   _deployment.metadata.annotations["key"] = "value"  # MUTATION!

   # ✅ CORRECT - Use inline conditionals
   _deployment = {
       metadata.annotations = {
           if _ready:
               "krm.kcl.dev/ready" = "True"
       }
   }
```

You'll find the three essential ingredients: **build/deploy commands** first (so Claude knows how to validate its work), **naming conventions**, and **specific pitfalls** Claude would keep reproducing if not warned.

{{% notice warning "The ~500-line rule" %}}
Every token in `CLAUDE.md` is loaded in **every conversation**. An overly long file wastes precious context. Aim for ~500 lines maximum and move specialized instructions to **Skills** that only load on demand.
{{% /notice %}}

### Iterating on CLAUDE.md

Treat `CLAUDE.md` as a **production prompt**: iterate on it regularly. The `#` shortcut lets you ask Claude itself to suggest improvements to your file. You can also use Anthropic's [prompt improver](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-improver) to refine the instructions.

---

## :hook: Hooks: my first automation

**Hooks** are shell commands that run automatically in response to Claude Code events. The key difference with `CLAUDE.md`: CLAUDE.md instructions are *advisory* (Claude can ignore them), hooks are **deterministic** — they always run.

### Getting notified when Claude is waiting

The first hook I set up — and the one I recommend to everyone — is the **desktop notification**. When Claude finishes a task or is waiting for your input, you get a system notification with a sound. No more checking the terminal every 30 seconds.

Configuration in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION\" --icon=dialog-information && paplay /usr/share/sounds/freedesktop/stereo/complete.oga"
          }
        ]
      }
    ]
  }
}
```

On Linux, `notify-send` is provided by the `libnotify` package and `paplay` by `pulseaudio-utils` (or `pipewire-pulse`). Other mechanisms exist for macOS (`osascript`) or other environments — see the [hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for alternatives.

### Other possibilities

Hooks cover several events (`PreToolUse`, `PostToolUse`, `Notification`, `Stop`, `SessionStart`) and enable things like:

- **Auto-format** after each edit (e.g., `gofmt` on Go files)
- **Sensitive file protection** — a `PreToolUse` hook that blocks writes to `.env`, `.pem`, `.key` files (exit code 2 = action blocked)
- **Audit** of executed commands in a log file

I won't detail every variant here — the [official hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) does a great job. The key takeaway is that hooks are your **deterministic** safety net where `CLAUDE.md` is merely advice.

---

## :brain: Mastering the context window

The context window (200K tokens) is **the most critical resource**. Once saturated, old information gets compressed and quality degrades. This is THE topic that makes the difference between an efficient user and someone who "loses" Claude after 20 minutes.

### `/compact` with custom instructions

The `/compact` command compresses conversation history while preserving key decisions. The trick: you can pass **focus instructions** to keep what matters:

```
/compact focus on the Crossplane composition decisions and ignore the debugging steps
```

This is especially useful after a long debugging session where 80% of the context is noise (failed attempts, stack traces).

### `/clear` strategy

The `/clear` command resets the context to zero. When to use it?

- **Always** between two distinct tasks — this is the most important rule
- When Claude starts **hallucinating** or repeating mistakes
- After a long debugging session (context is polluted with failed attempts)
- When `/context` shows < 20% free space

I've made a habit of starting every new task with a `/clear`. It seems counterintuitive (you lose context), but in practice it's much more effective than a polluted context.

### Tool Search: when MCPs eat your context

The problem: each enabled MCP injects its **tool definitions** into the context. With 6-7 configured MCPs (common in platform engineering), that can represent over **10% of your window** — consumed before you even start working.

The solution: enable **Tool Search**:

```bash
export ENABLE_TOOL_SEARCH=auto:10
```

With this option, Claude only loads tool definitions **when it needs them**, rather than keeping them all in memory. The `10` is the threshold (as a percentage of context) at which the mechanism kicks in.

### Delegating to subagents

For large exploration tasks (browsing a codebase, searching through logs), Claude can delegate to **subagents** that have their own isolated context. The condensed result is sent back to the main session — saving precious tokens.

In practice, Claude does this automatically when it deems it relevant (file exploration, broad searches). But you can also guide it explicitly: *"use a subagent to explore the networking module structure"*.

### A few simple rules

1. **`/clear` between tasks**: Each new task should start with a `/clear`
2. **Keep CLAUDE.md concise**: Every token is loaded in every conversation
3. **CLIs > MCPs**: For mature tools (`kubectl`, `git`, `gh`...), prefer the CLI directly — LLMs know them perfectly and it avoids loading an MCP
4. **`/context` to audit**: Identify what's consuming context and disable unused MCPs

---

## :arrows_counterclockwise: Multi-session workflows

### Git Worktrees: parallelizing sessions

Rather than juggling branches and `stash`, I use **git worktrees** to work on multiple features in parallel with independent Claude sessions.

```console
# Create two features in parallel
git worktree add ../worktrees/feature-a -b feat/feature-a
git worktree add ../worktrees/feature-b -b feat/feature-b

# Launch two separate Claude sessions
cd ../worktrees/feature-a && claude  # Terminal 1
cd ../worktrees/feature-b && claude  # Terminal 2
```

Each session has its **own context** and its own memory — no interference between tasks.

{{% notice tip "Worktree vs repo copy" %}}
Unlike a separate `git clone`:
- **Shared history**: A single `.git`, commits are immediately visible everywhere
- **Disk space**: ~90% savings (no git object duplication)
- **Synchronized branches**: `git fetch` in one worktree updates all others
{{% /notice %}}

When the change is done (PR merged), just go back to the main repo and clean up.

```console
cd <path_to_main_repo>
git worktree remove ../worktrees/feature-a
git branch -d feat/feature-a  # after PR merge
```

**Useful commands:**

```console
git worktree list              # See all active worktrees
git worktree prune             # Clean up orphaned references
```

### Writer/Reviewer pattern

A pattern I use more and more involves running **two sessions in parallel**:

| Session | Role | Prompt |
|---------|------|--------|
| **Writer** | Implements the code | "Implement feature X according to the spec" |
| **Reviewer** | Reviews the code | "Review the changes on the feat/X branch, check security and edge cases" |

The reviewer works on the same repo (via worktree or read-only) and provides independent feedback, without the author's bias. This is especially effective for infrastructure changes where a mistake can be costly.

### Fan-out with `-p`

The `-p` flag (non-interactive prompt) lets you run Claude in **headless** mode and parallelize tasks:

```bash
# Launch 3 tasks in parallel
claude -p "Add unit tests for the auth module" &
claude -p "Document the REST API for the orders service" &
claude -p "Refactor the billing module to use the new SDK" &
wait
```

Each instance has its own context. This is ideal for independent tasks that don't require interaction.

---

## :desktop_computer: Hybrid IDE + Claude Code workflow

In practice, I alternate between two modes: sometimes pure terminal — old habits — and sometimes hybrid with **Cursor** for editing and **Claude Code** in the terminal. The hybrid workflow is clearly more comfortable, and I'm moving towards it more and more.

{{< img src="cursor+claude.png" alt="Cursor + Claude Code" width="1200" >}}

| Need | Tool | Why |
|------|------|-----|
| Quick editing, autocomplete | Cursor | Minimal latency, you stay in the flow |
| Refactoring, multi-file debugging | Claude Code | Deep reasoning, autonomous loops |

**What I like about hybrid mode**: Claude makes changes via the terminal, and I review the diffs in the Cursor interface — much more readable than `git diff`. Changes appear in real time in the editor, which lets you follow what Claude is doing and step in quickly if needed.

---

## :electric_plug: Plugins

Claude Code has a **plugin** ecosystem that extends its capabilities. Here are the two I use daily:

### Code-Simplifier: cleaning up generated code

The **[code-simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)** plugin is developed by Anthropic and used internally by the Claude Code team. It automatically cleans up AI-generated code while preserving functionality.

I discovered this plugin recently and intend to make a habit of using it **before creating a PR** after an intensive session. It runs on Opus and should help reduce the technical debt introduced by AI code — duplicated code, unnecessarily complex structures, inconsistent style.

### Claude-Mem: persistent memory across sessions

The **[claude-mem](https://github.com/thedotmack/claude-mem)** plugin automatically captures your session context and reinjects it in future sessions. No more re-explaining your project at the start of every conversation.

**Its two main strengths:**
- **Semantic search**: easily find information from a past session via the `mem-search` skill
- **Token consumption optimization**: a 3-layer workflow that significantly reduces usage (~10x savings)

Usage examples:

  - "Search my sessions for when I debugged Karpenter"
  - "Find what I learned about OpenBao PKI last week"
  - "Look at my previous work on the App composition"

{{% notice warning "Privacy considerations" %}}
Claude-mem stores session data locally. For sensitive projects, use `<private>` tags to exclude information from capture.
{{% /notice %}}

---

## :warning: Anti-patterns

Here are the traps I fell into — and learned to avoid:

| Anti-pattern | Symptom | Solution |
|-------------|---------|----------|
| **Kitchen sink session** | Mixing debugging, features, refactoring in the same session | `/clear` between each distinct task |
| **Correction spiral** | Claude fixes a bug, creates another, loops endlessly | Stop, `/clear`, rephrase with more context |
| **Bloated CLAUDE.md** | Context consumed from the start, degraded responses | Target ~500 lines, move the rest to Skills |
| **Trust-then-verify gap** | Accepting code without review, finding bugs in prod | Always read the diff before committing |
| **Infinite exploration** | Claude browses the entire codebase instead of acting | Give specific files/paths in the prompt |

{{% notice tip "The most insidious trap" %}}
The **correction spiral** is by far the most dangerous. A real example: Claude was supposed to add a `CiliumNetworkPolicy` to a Crossplane composition. First attempt, wrong endpoint selector. It fixes it, but breaks the KCL format. It fixes the format, but reverts to the original bad selector. After 5 iterations and ~40K tokens consumed, I hit `/clear` and rephrased in 3 lines, specifying the target namespace and an example of an existing policy. Correct result on the first try. The lesson: when Claude loops after 2-3 attempts, it's a sign it's missing **context**, not persistence. Better to cut and rephrase than to let it spin.
{{% /notice %}}

---

## :checkered_flag: Conclusion

Over time, I've become increasingly comfortable with Claude Code, and the productivity gains are real. But they come with a lingering concern I can't fully shake: losing control — over the produced code, over the decisions made, over the understanding of what's running in production.

These questions, as well as the methods that help me stay in control, are covered in the first article of this series. If you want to revisit the fundamentals or understand where these reflections come from, I highly recommend it: [Agentic Coding: concepts and hands-on use cases](/en/post/series/agentic_ai/ai-coding-agent/).

---

## :bookmark: References

### Official documentation
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Engineering
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code) — Official guide
- [Hooks Documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) — Hooks configuration

### Community guides
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) — Comprehensive guide by sshh
- [CC-DevOps-Skills](https://github.com/akin-ozer/cc-devops-skills) — 31 DevOps skills

### Plugins and tools
- [Code-Simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) — AI code cleanup (Anthropic)
- [Claude-Mem](https://github.com/thedotmack/claude-mem) — Persistent memory across sessions

### Previous article
- [Agentic Coding: concepts and hands-on use cases](/en/post/series/agentic_ai/ai-coding-agent/) — Part 1 of the Agentic AI series
