# Agent Factory series: split design

Date: 2026-11-01. Status: approved by the author (6 articles, sub-series under `agentic_ai`, written autonomously, reviewed later).

## Goal

Split the long FR post `content/fr/post/series/agentic_ai/cloud-native-agent-factory/index.md` into six scoped
articles. **The original post stays untouched.** Each article deepens its topic from the source repositories, not
only from the original post.

## Placement and conventions

- Folders: `content/fr/post/series/agentic_ai/agent-factory-<n>-<topic>/index.md` (page bundles, images copied in).
- Front matter: TOML, `draft = true`, `series = ["Agentic AI"]`, `usePageBundles = true`, `toc = true`,
  `codeMaxLines = 30`, `featured` only on article 1. Dates one week apart from 2026-11-04 (the author adjusts).
- Title prefix `Agent Factory :` (no part count: the series length stays open). A shared `notice info` at the top of each article lists the published parts.
- French only. Diagrams in English, drawio source + PNG export (scale 2) in the bundle, icons from
  `cloud-native-ref/docs/architecture/icons` and the Maku brand assets.

## Writing rules learned while reviewing the original

- Simple words first; define a term (shadow, harness, class…) before using it.
- No ServiceAccount/projected-token internals; few YAML blocks, only when they carry the idea (the `AgentRun` claim).
- Honesty over polish: no "complete solution", no version numbers or dates that will move (no "v3"), no
  commitment to metrics in intros. Banned wording: « partis pris ».
- Keep the tone of the blog (ogenki-blog-style skill): hook, objectives, practice, « 💭 Dernières remarques ».
- Distinguish what a tool is from how the factory uses it (OpenHands lesson).

## The six articles

| # | Folder | Scope | Main sources |
|---|---|---|---|
| 1 | `agent-factory-1-vue-ensemble` | The problem (solo agent vs team), software factories and autonomy levels, the three repos/layers, one task end to end, the series map. No deep detail | Original post (intro, software factory, overview); `maku` README/architecture; `ai-native-ref` README |
| 2 | `agent-factory-2-maku` | Maku in depth: components (factory, broker, bridge, CLI, Web UI, Slack, MCP), the `Task` lifecycle, intake paths, rooms (append-only, attribution, approvals, four-eyes), the issue narration, the handoff skill | `maku` `docs/` (architecture, concepts, factory, slack, api), charts |
| 3 | `agent-factory-3-harnais` | Harness engineering, the `AgentRun` composition, OpenHands and the maku-harness image, identities, no secret in the sandbox, gVisor, egress, Kueue, and the roles (scoper, splitter, tester, implementer, search, judge, reviewer) | `crossplane-configuration/apis/agentrun`, `maku/images/maku-harness`, `ai-native-ref` runbooks 01–06 |
| 4 | `agent-factory-4-merge-gate` | Classes, the ordered gate, policy-bot, the two ways to merge, shadow and the tally, the breaker, post-merge watch and revert, drills | `maku/docs/factory.md` (merge gate, monitor, calibration), `ai-native-ref` factory config, `cloud-native-ref/.policy.yml` |
| 5 | `agent-factory-5-observabilite` | Telemetry of agents (traces, metrics, dashboards), budgets and costs, guardrails (budgets, caps, kill switch, Kueue), alerts and runbooks, real numbers | `ai-native-ref/observability`, runbooks 08, `maku` metrics docs |
| 6 | `agent-factory-6-runlore` | Agent-to-agent: RunLore investigates an alert, hands a task to the factory, the fix is verified after merge; trust boundaries between agents | `runlore` repo, `maku` verification API, original post RunLore section |

## Steps

1. Repository research per article (parallel read-only agents), each returning a fact sheet with file paths.
2. Write articles 1 → 6 from the fact sheets and the original post; copy or adapt diagrams.
3. `hugo --buildDrafts` must pass; no commit (the author reviews the six drafts later).
