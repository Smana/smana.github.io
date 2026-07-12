# Design — Blog post: RunLore (FR, observability series)

**Date:** 2026-06-30
**Author:** Smaine Kahlouch (with Claude)
**Status:** Draft for review

## Goal

Publish a French blog post introducing **RunLore** (`github.com/Smana/runlore`),
the author's open-source agentic SRE incident-investigation agent. The post is the
**capstone of the `observability` series**: the series so far covered *detecting*
(metrics → logs → alerts); RunLore covers the next, most toil-heavy step —
*investigating and resolving* incidents by correlating those same signals, and
*learning* from each one.

## Constraints & conventions

- **Language:** French first (`content/fr/post/series/observability/runlore/index.md`).
  English translation is a later follow-up, out of scope for this spec.
- **Series:** `series = ["observability"]`. Page bundle (`usePageBundles = true`),
  TOML front matter (`+++`), `toc = true`, emoji-decorated H2/H3 to match the
  existing series posts (metrics / logs / alerts).
- **Tone:** follow the `ogenki-blog-style` skill + the blog-writing-style memory —
  professional + personal voice (nous = exploration, vous = instructions, je =
  personal experience/caveats); concise, theory-first, precise terminology, no weak
  analogies, no "hook" sentences that rank one section above another, honesty over
  polish; occasional self-deprecating aside and a rhetorical hook question.
- **Term handling:** every technical term **bold** on first use with a brief
  definition; acronyms spelled out on first mention — SRE (Site Reliability
  Engineering), RCA (Root Cause Analysis), OKF (Open Knowledge Format), GitOps.
  Keep English technical terms untranslated in the French post (Ingress Controller,
  GitOps, etc.).
- **Canonical structure (`ogenki-blog-style`):** Hook/Lead (problem or prior-series
  notice, never an abstract definition) → Objectifs → Prérequis / Boîte à outils →
  Concepts essentiels (lean) → En pratique → 💭 Dernières remarques → Références.
- **Tooling:** scaffold the post file with the `hugo-new-post` skill.
- **Internal links:** French posts use `/fr/post/...` paths.

## Source material (all verified locally)

- RunLore repo `~/Sources/runlore` — README sections: *How it works*, *The learning
  loop*, *Supported integrations*, *Why RunLore*, *Project status & stability*.
- Knowledge base `~/Sources/runlore-kb` — real OKF bundle (`index.md`, `README.md`,
  `log.md` + entries). **Worked example = the Harbor incident**, captured
  2026-06-28 from `cloud-native-ref`:
  - seed playbook `harborregistrydown.md`
  - learned, sharpened entry `harbor-registry-down-due-to-iam-access-key-quota-limit.md`
    (95% confidence, `Unresolved` section listing what the agent could *not* determine).
  - Other real entries available as backup: `application-airflow-degraded.md`
    (ExternalSecret wrong AWS SM path), `karpenter-ec2nodeclass-ami-not-found.md`.
- Author's wiki `~/Backup/llmwiki/wiki/runlore.md` — current honest positioning and caveats.
- OKF = Google Cloud's Open Knowledge Format, published June 2026; markdown + YAML
  frontmatter formalizing Karpathy's LLM-wiki pattern. RunLore stores OKF-compatible
  markdown in a Git repo the team owns.

## Key messages (honest positioning — from the author's own wiki)

1. **The differentiator is the open, portable, reviewable OKF knowledge catalog +
   the honesty/eval posture** — NOT the "what-changed" Git diff. Change-aware RCA
   is not unique (Komodor, Anyshift do it; it's copyable). Lead with what is genuinely
   whitespace: open/portable learning + read-only-first honesty.
2. **Honesty is a feature**: read-only by default, `unresolved` is a first-class
   answer, the adversarial *verify* pass can only ever *lower* a finding's confidence,
   honest about the sub-50% reality.
3. **It learns *your* context** — constraints, rules, company specifics — because the
   catalog is your incidents, curated by your team via PR, not a generic model.
4. **Very early stage** — pre-1.0, launched ~10 days ago, `auto` rung frozen and not
   recommended on real clusters. The memory needs *weeks* of real use to judge. One
   known quirk worth watching: recall currently keys on workload+symptom, so a
   same-symptom/different-cause incident can anchor on a stale prior RCA.

## Structure (section by section)

Maps onto the `ogenki-blog-style` canonical shape: §1 = Hook/Lead, §2 = Objectifs,
§3 = Prérequis/Boîte à outils framing within the problem, §4–§6 = Concepts essentiels
(kept lean), §8 = En pratique, §10 = Dernières remarques, §11 = Références. §7
(alternatives) and §9 (dogfooding) are series-specific additions.


1. **`{{% notice info %}}` — le chaînon manquant de la série** — compact, theory-first:
   the series covered métriques → logs → alertes (détecter) ; reste l'étape la plus
   coûteuse en *toil* : investiguer la cause racine. RunLore est cette étape.

2. **🎯 Objectifs** — bullets: le coût caché de l'investigation ; RunLore en bref ;
   la boucle d'apprentissage + OKF ; un exemple réel sur cloud-native-ref ; un retour
   honnête sur un projet très jeune.

3. **🔥 Le problème : le coût caché de l'investigation** — when an alert fires, the SRE
   manually pivots across Git (qu'est-ce qui a changé ?), metrics, logs, network flows,
   and tribal knowledge scattered in runbooks. Picks up exactly where the *alerts* post
   stopped ("runbooks et procédures de réponse aux incidents").

4. **🤖 RunLore en bref** — read-only-by-default Go binary in K8s; *what changed?
   what's wrong?* → confidence-scored RCA posted to chat. The "what-changed spine" on
   GitOps (Flux/Argo) feeding multi-signal correlation (VictoriaMetrics/Prometheus,
   VictoriaLogs, Hubble/VPC/GCP flows, CloudTrail). **Excalidraw-style architecture
   diagram** of the investigation flow (event → investigate → RCA in chat → PR to KB).
   *Diagram authored/exported by the author; placeholder `{{< img >}}` in the draft.*

5. **🧠 Ce qui le distingue : une mémoire qui se construit (OKF + boucle
   d'apprentissage)** — the heart. Compound loop **Retrieve → Capture → Curate →
   Compound** in a few sentences. Then OKF: the author already lived Karpathy's
   LLM-wiki pattern with Obsidian/Tolaria; now there's a vendor-neutral standard
   (Google Cloud, June 2026) — markdown + YAML frontmatter in a Git repo *you own*,
   BM25-indexed, PR-reviewed, not a proprietary store. It learns *your* context:
   constraints, rules, company specifics. Knowledge that resolves incidents gains
   trust; knowledge that fails decays.

6. **🪞 L'honnêteté comme fonctionnalité** — read-only-first; confidence scoring; the
   adversarial *verify* pass that can only lower confidence; `unresolved` as a
   first-class answer; honest about the sub-50% reality. Distinctive and on-brand.

7. **🔬 Analyse des alternatives** — comparison table:
   - **k8sgpt** — detector (analyzers + LLM), no investigation loop, no learning.
   - **HolmesGPT** — strongest OSS investigation agent, but hand-curated runbooks,
     does not self-improve.
   - **kagent** — generic in-cluster agent framework; has agent-memory but opaque.
   - **Komodor / Anyshift** (commercial) — change-aware RCA too, but proprietary /
     non-portable.
   Honest framing: RunLore's slice = open + portable + reviewable learning, not
   "nobody else learns".

8. **🛠️ Un exemple concret sur `cloud-native-ref`** — the real Harbor incident:
   - Symptom: `harbor-registry` pod `CreateContainerConfigError` (missing `username`
     key in Secret), kube_events Warning `LimitExceeded: AccessKeysPerUser: 2` on the
     Crossplane `AccessKey/xplane-harbor`.
   - Investigation correlates pod status + events + the Crossplane resource, and finds
     a related prior KB entry (`HarborRegistryDown`) → compound loop in action.
   - RCA posted with 95% confidence; resolution suggested (delete an unused IAM access
     key, marked `reversible=false`); honest `Unresolved`: which IAM user, which key is
     safe to delete.
   - Result: a curated OKF entry now in the catalog (show the captured markdown).
   - cloud-native-ref's stack (EKS, Cilium, VictoriaMetrics, Crossplane, Flux) *is*
     RunLore's golden path. Link to the repo.

9. **🧑‍💻 Construit en agentique, en toute transparence** *(dedicated section)* — honest
   dogfooding: RunLore itself was built with the author's agentic toolkit — Claude
   Code, skills, superpowers, multi-agent reviews. Links to the Agentic AI series:
   [`ai-coding-agent`](/fr/post/series/agentic_ai/ai-coding-agent/),
   [`ai-coding-tips`](/fr/post/series/agentic_ai/ai-coding-tips/), and the
   [self-hosted LLM stack](/fr/post/series/agentic_ai/llm-self-hosted-stack/)
   (RunLore can run against vLLM/Ollama). Note the same review discipline (adversarial
   verification) used to build it is the philosophy baked into the product.

10. **💭 Dernières remarques — un projet très jeune** — honest reflection (not a
    summary): pre-1.0, launched ~10 days ago, golden path only (Flux +
    VictoriaMetrics + Anthropic/OpenAI-compatible + Slack + GitHub). The memory angle
    needs *weeks* of real use to judge whether recall actually compounds; known quirk
    (workload+symptom recall vs. different cause). The author will report back in the
    long run. Production caveats go in a **`{{% notice warning "Points d'attention
    pour la prod" %}}`** box: read-only-by-default is the safe mode; the `auto`
    autonomy rung is experimental, frozen, and not recommended on real clusters.

11. **🔖 Références** — RunLore repo; OKF announcement + spec
    (GoogleCloudPlatform/knowledge-catalog); Karpathy LLM-wiki; k8sgpt; HolmesGPT;
    cloud-native-ref.

## Open items to confirm with the author before drafting

- Architecture diagram: author exports the Excalidraw PNG; draft uses a placeholder
  `{{< img >}}` with a descriptive `alt`/caption and a TODO note.
- Whether to include the captured OKF markdown verbatim (recommended — it's the most
  convincing artifact) or a trimmed excerpt.

## Out of scope

- English translation (follow-up).
- Any change to the RunLore codebase or the knowledge base.
