# Article « Self-hosted LLM stack, partie 4 » — Plan de rédaction

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rédiger en français la partie 4 de la série Agentic AI — comment l'abstraction `InferenceService` a mûri (PR #1559, composition v0.8.0) — publiable une fois les marqueurs `TODO-e2e` remplis.

**Architecture:** Un seul page bundle Hugo (`content/fr/post/series/agentic_ai/llm-platform-abstraction/`). Rédaction par blocs de sections, chacun vérifié par un build Hugo. Deux diagrammes drawio produits en fin de parcours, quand le texte a figé ce qu'ils doivent montrer. Les faits techniques sont **tous** figés dans ce plan (valeurs vérifiées contre la PR) : aucune tâche de rédaction ne doit aller les rechercher.

**Tech Stack:** Hugo 0.156.0 extended, thème hugo-clarity, front matter TOML, shortcodes `img` et `notice`, draw.io pour les schémas.

**Spec:** [`docs/superpowers/specs/2026-07-12-llm-platform-part4-design.md`](../specs/2026-07-12-llm-platform-part4-design.md)

## Global Constraints

- **Langue : français uniquement.** La traduction EN est hors périmètre de ce plan.
- **Style :** appliquer la skill `ogenki-blog-style`. Paragraphes de 2 à 5 phrases. Toute notion technique en **gras** à sa première occurrence, suivie immédiatement de sa définition. Tous les 3-4 paragraphes de prose : une liste, un tableau, un bloc de code ou une notice. Honnêteté sur les limites, jamais de triomphalisme.
- **Aucune affirmation de performance non mesurée.** Les chiffres externes (57× TTFT rapporté par llm-d) sont cités **comme externes et non reproduits**. Interdiction absolue d'inventer un chiffre, un screenshot ou une sortie de commande.
- **Marqueurs de données manquantes :** utiliser un commentaire HTML `<!-- TODO-e2e: <ce qui manque> -->` — invisible au rendu, greppable avant publication.
- **Liens internes FR :** `/fr/post/...` (jamais `/en/post/...`).
- **Chemin cible :** `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`
- **`draft = true`** dans le front matter jusqu'à ce que l'e2e soit passé et les `TODO-e2e` remplis.
- **Titre :** utiliser le titre de travail « Self-hosted LLM stack : quand l'abstraction grandit ». Il est **explicitement à retravailler** (tâche 9) — ne pas s'y attacher.

## Faits techniques figés (source : PR #1559, vérifiés)

Toute tâche de rédaction s'appuie **uniquement** sur ces valeurs. Ne pas en inventer d'autres.

**Versions & identités**
- Composition `InferenceService` : v0.6.0 (partie 3, commit du 2026-05-09) → **v0.8.0** (PR #1559)
- Modèle migré : `xplane-qwen-coder` (base : `Qwen/Qwen2.5-Coder-7B-Instruct`)
- Adapters LoRA déclarés : `xplane-qwen-coder-sql-dpo` (`jk200201/qwen2.5-coder-7b-sql-dpo`) et `xplane-qwen-coder-securecode` (`scthornton/qwen2.5-coder-7b-securecode`) — **tirés de HuggingFace, pinés au commit SHA**
- Preuves disponibles : `kcl test` **44/44 PASS** ; `./scripts/validate-kcl-compositions.sh` exit 0 ; `crossplane render` + Polaris contre le tag réel `0.8.0-pr1559`
- Modelplane : billet publié le **2026-06-23** ; la v0.6.0 de cette plateforme est du **2026-05-09**

**SPEC-002 — routage possédé par la composition**
- `gateway.enabled: true` → la composition rend `Backend` (FQDN du Service de la claim, port 8000), `AIServiceBackend` (schéma OpenAI), `AIGatewayRoute` (parentRef `ai-gateway/envoy-ai-gateway-system`)
- **Latch de readiness** : l'`AIGatewayRoute` est retenue tant que le `Deployment` n'est pas `Available=True`, puis **jamais retirée** en cas d'indisponibilité passagère (portail à la création + latch via `ocds`)
- Les clés `ocds` de lecture et d'écriture partagent une constante unique (`_gatewayRouteSuffix`) — elles ne peuvent pas diverger
- Canary : `gateway.canaries` est un **array, max 4 entrées**. La règle du modèle de base porte un backendRef par entrée ; le backendRef de base garde `100 - sum(weightPercent)`, chaque canary `AIServiceBackend` (`<claim>-canary-<i>`) prend son `weightPercent` et porte `modelNameOverride` = le `loraAdapters[].name` **verbatim**. Tous pointent le **même `Backend`**.
- Chaque `loraAdapters[]` a une **règle de pin** (`x-ai-eg-model: <name>` → backend de base, poids 100) : une requête nommant explicitement l'adapter est servie à 100 %, indépendamment du canary
- CEL rejette à l'admission : `canaries` sans `gateway.enabled` ; un `canaries[].adapter` absent de `loraAdapters[].name` ; des adapters dupliqués ; un `weightPercent` hors de 1–99 ; `sum(weightPercent) > 99` (le modèle de base garde toujours du trafic)
- Migration : `xplane-qwen-coder` bascule, ses entrées sont retirées de `apps/base/ai/llm/ai-gateway-routes/route.yaml` **dans le même commit**. Les 3 autres modèles suivent dans une PR ultérieure.

**SPEC-003 — échappatoire `engineArgs`**
- `spec.engineArgs` : array de string, **`maxItems: 16`**, chaque entrée est **un seul token** `--flag` ou `--flag=value` (jamais `--flag value`)
- Appendus **après** tous les args gérés par la composition (`_vllmArgs = _managedVllmArgs + _engineArgs`) : les flags gérés gagnent toujours
- **17 règles CEL** dans l'XRD : 1 règle de préfixe `--` + **16 règles de flags réservés**, une par flag, pour que chaque message nomme le champ curé à utiliser
- Table des flags réservés (à reproduire fidèlement, éventuellement abrégée) :

| Flag réservé | Message CEL (fragment) |
|---|---|
| `--model` | `use spec.model.repository` |
| `--served-model-name` | `the served model name is the claim name (metadata.name)` |
| `--max-model-len` | `use spec.model.contextWindow` |
| `--max-num-seqs` | `use spec.model.maxNumSeqs (it is the KEDA scaling denominator)` |
| `--gpu-memory-utilization` | `composition-managed (fixed at 0.92)` |
| `--quantization` | `use spec.model.quantization` |
| `--enable-prefix-caching` | `use spec.cache.prefixCache.enabled` |
| `--cpu-offload-gb` | `use spec.cache.kvOffload.{enabled,sizeGB}` |
| `--enable-auto-tool-choice` | `use spec.model.toolCallParser` |
| `--tool-call-parser` | `use spec.model.toolCallParser` |
| `--enable-lora` | `use spec.loraAdapters` |
| `--max-loras` | `use spec.loraAdapters` |
| `--max-lora-rank` | `use spec.loraAdapters (rank fixed at 64)` |
| `--lora-modules` | `use spec.loraAdapters` |
| `--port` | `serving-contract flag; the vLLM port is fixed at 8000 (Service/probes/Backend depend on it)` |
| `--host` | `serving-contract flag; managed by the composition` |

- Point de conception à souligner : **l'application vit à UN seul endroit** (le CEL de l'XRD). La composition appende verbatim et fait confiance à l'admission — pas de seconde implémentation de la denylist en KCL. Le lockstep XRD ↔ composition est verrouillé par `test_engine_args_denylist_lockstep` (mutation-verified).
- Deux flags (`--port`, `--host`) sont réservés bien que la composition **ne les émette pas** : leurs valeurs par défaut (8000, toutes interfaces) portent le contrat de service (Service, probes, FQDN du `Backend`).

**SPEC-003 — `status.servedModels`**
- Forme : liste d'objets `{name, kind (base|adapter), canaryWeightPercent (optionnel)}`, calculée **depuis le spec seul** (aucune lecture d'`ocds`) — donc peuplée dès la première réconciliation. C'est une projection de **topologie**, pas un signal de santé.
- Piège Kubernetes à raconter (CL-5) : la colonne d'affichage pointait d'abord le JSONPath wildcard `.status.servedModels[*].name`. Le convertisseur de table côté serveur ne rend que la **première** correspondance d'un wildcard — la colonne n'aurait affiché que le modèle de base, jamais les adapters. D'où l'ajout d'un scalaire `status.servedModelsSummary` (noms joints par virgule) sur lequel pointe la colonne `SERVED MODELS`.

**SPEC-006 — observabilité `gen_ai`**
- Métriques `gen_ai.*` d'Envoy AI Gateway v1.0, scrapées dans VictoriaMetrics via un `VMPodScrape` sur le **sidecar extproc** dans le namespace `envoy-gateway-system`, **port admin 1064**
- Dashboard Grafana **`llm-gateway`**, avec attribution **canary vs base**
- Traces OTLP vers VictoriaTraces : **volontairement désactivées** en attendant la vérification de compatibilité de chemin (CL-4) — à dire honnêtement

**SPEC-004 — InferencePool + Endpoint Picker**
- Problème : un `Service` ClusterIP fait du **round-robin**, hostile à vLLM — il éparpille les requêtes partageant un préfixe sur des pods différents, détruit la localité du prefix cache et force du prefill redondant. Il est aussi aveugle à la charge par pod.
- GAIE **v1.5.0** ; l'`InferencePool` a atteint **v1 GA en 2025-09**
- L'EPP scrape les **mêmes** `/metrics` vLLM que la plateforme collecte déjà, et score chaque pod : profondeur de queue + utilisation du KV cache + affinité de préfixe + présence de l'adapter LoRA
- **Complémentaire de KEDA, pas concurrent** : KEDA réagit à l'échelle de la dizaine de secondes (*combien* de replicas), l'EPP à la milliseconde (*lequel* des replicas existants). Les deux lisent les mêmes signaux de saturation vLLM.
- Chiffre externe : llm-d rapporte **jusqu'à 57× d'amélioration de TTFT** face au round-robin sous charge. **Cité comme externe, non reproduit ici.**
- Un flag (`gateway.endpointPicker.enabled`) → `HelmRelease` (chart `inferencepool` v1.5.0) + `InferencePool` + Deployment EPP + `CiliumNetworkPolicy` + `VMServiceScrape`
- **Mutuellement exclusif avec `canaries[]`** (rejeté à l'admission) : un backendRef `InferencePool` ne supporte **ni `weight` ni `modelNameOverride`**, et une seule InferencePool est admise par règle
- L'InferencePool est scopée aux replicas de **sa propre claim** (`matchLabels: {app.kubernetes.io/name: <claim>}`), jamais cross-modèle

**KEDA — troisième trigger**
- Nouveau trigger combiné en OR sur `vllm:num_requests_waiting` (`scaling.queueLengthThreshold`, défaut **8**) — le signal de pression le plus précoce. S'ajoute aux deux triggers de la partie 3 (`vllm:num_requests_running` / `maxNumSeqs`, et `vllm:gpu_cache_usage_perc`).

**SPEC-005 — cold-start**
- Run:ai Model Streamer : `--load-format runai_streamer`, embarqué dans l'image `vllm-openai` v0.8.5, activé par `model.streaming` sur le chemin PVC existant
- Les flags du streamer rejoignent la denylist `engineArgs`
- `s3://` direct et le résolveur LoRA sont **descopés** (nécessitent vLLM v0.9.0)

---

### Task 1: Squelette du post — bundle, front matter, notice de série

**Files:**
- Create: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Produces: le fichier `index.md` que toutes les tâches suivantes modifient ; les ancres de section (`##`) auxquelles les tâches suivantes se raccrochent.

- [ ] **Step 1: Créer le page bundle et le front matter**

Créer `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md` :

```toml
+++
author = "Smaine Kahlouch"
title = "Self-hosted LLM stack : quand l'abstraction grandit"
date = "2026-07-12"
summary = "Deux mois après avoir posé les fondations, l'abstraction `InferenceService` a révélé quatre manques : deux sources de vérité, aucune stratégie de rollout, aucune échappatoire, aucune découvrabilité. Voici comment on les a comblés — canary LoRA à coût GPU nul compris."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
draft = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "vllm", "crossplane", "platform-engineering"]
thumbnail = "thumbnail.png"
+++
```

- [ ] **Step 2: Poser la notice de série et le squelette des sections**

Sous le front matter, ajouter la notice de série (calquée sur celle de la partie 3) puis **uniquement les en-têtes** des sections, dans cet ordre, pour que les tâches suivantes aient des ancres stables :

```markdown
{{% notice info "Série Agentic AI — Partie 4" %}}
Cet article fait suite à [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (partie 3), qui décrit la plateforme sur laquelle tout ce qui suit s'appuie. **Ici, on la fait vivre** — et on répare ce qui a cassé.
{{% /notice %}}

## :dart: Objectifs

## :mag: Ce qui clochait

## :door: Le routage rejoint le modèle

## :dna: LoRA en deux minutes

## :hatching_chick: Canary : 10 % du trafic, zéro GPU

## :unlock: L'échappatoire `engineArgs`

## :eyes: Le modèle dit ce qu'il sert

## :bar_chart: Mesurer le canary

## :compass: Endpoint Picker : au-delà du round-robin

## :telescope: Modelplane : évolution convergente

## :thought_balloon: Dernières remarques

## :bookmark: Références
```

- [ ] **Step 3: Vérifier que Hugo construit**

Run: `hugo --minify --buildDrafts 2>&1 | tail -5`
Expected: build réussi, aucune erreur. Le post apparaît dans le décompte des pages FR.

- [ ] **Step 4: Commit**

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): scaffold part 4 — LLM platform abstraction (FR)"
```

---

### Task 2: Intro, objectifs, et §1 « Ce qui clochait »

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: le squelette de la tâche 1.
- Produces: la thèse posée en intro, à laquelle chaque section suivante doit se raccrocher.

- [ ] **Step 1: Rédiger le hook et les objectifs**

Le hook ouvre sur un problème, jamais sur une définition (règle de style). Angle : deux mois après la partie 3, la fondation a tenu — la claim YAML fait toujours le job — mais **en la faisant vivre**, quatre manques structurels sont apparus.

Rappeler en ~10 lignes de YAML la claim de la partie 3 pour que l'article se lise seul.

Puis la liste `## :dart: Objectifs` (4-5 puces) : comprendre pourquoi une abstraction de plateforme doit posséder le routage ; voir un canary LoRA à coût GPU nul ; comprendre ce qu'est une échappatoire encadrée ; en tirer quatre leçons transposables hors LLM.

- [ ] **Step 2: Rédiger §1 « Ce qui clochait »**

Auto-critique assumée du travail publié deux mois plus tôt. Quatre manques, un paragraphe court chacun :

1. **Deux sources de vérité** — la claim déclare le modèle ; un `apps/base/ai/llm/ai-gateway-routes/route.yaml` **écrit à la main** déclare comment on l'atteint. Conséquences concrètes : un modèle peut exister sans route (404 silencieux), une route peut survivre à son modèle (backend fantôme).
2. **Aucune stratégie de rollout** — la bascule d'un modèle est binaire. Les adapters LoRA (`sql-dpo`, `securecode`) ne sont joignables que par un client qui les nomme explicitement : impossible de déplacer progressivement une fraction du trafic pour valider un fine-tune sur du trafic réel.
3. **Aucune échappatoire** — chaque nouveau flag vLLM exige une release de la composition.
4. **Aucune découvrabilité** — `kubectl get isvc` ne dit pas à quels noms le modèle répond réellement ; il fallait rétro-ingénierer la config de routage.

Clore la section sur la thèse : *l'unité de déploiement n'est plus « un pod qui sert un modèle »*.

- [ ] **Step 3: Vérifier le build et relire**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

Relire contre `ogenki-blog-style` : paragraphes de 2 à 5 phrases ; rupture visuelle (liste/notice/code) tous les 3-4 paragraphes ; termes techniques en gras à la première occurrence.

- [ ] **Step 4: Commit**

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — intro, objectives and the four gaps"
```

---

### Task 3: §2 « Le routage rejoint le modèle »

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: le manque n°1 posé en §1 (deux sources de vérité).
- Produces: la notion de **latch de readiness**, réutilisée dans la section Endpoint Picker (le latch garde la route, quel que soit l'état de l'EPP).

- [ ] **Step 1: Rédiger la section**

Contenu, dans l'ordre :

1. `gateway.enabled: true` → la composition rend `Backend` + `AIServiceBackend` + `AIGatewayRoute` **par claim**. Montrer le diff de la claim (bloc `diff`). Supprimer le modèle supprime sa route : une seule source de vérité, garantie par le garbage collector Crossplane, pas par la discipline humaine.
2. **Le latch de readiness** — le point technique qui mérite le plus d'attention. Deux échecs symétriques à nommer : une route qui apparaît **trop tôt** envoie du trafic vers un modèle qui ne peut pas répondre (404/503 le temps du premier replica) ; une route qui disparaît **trop vite** fait du flapping à la moindre indisponibilité passagère. D'où : portail à la création (`Available=True`), puis **latch** — une fois créée, la route n'est plus retirée.
3. Une phrase sur le détail qui compte : les clés `ocds` de lecture et d'écriture partagent une constante unique (`_gatewayRouteSuffix`) — elles **ne peuvent pas** diverger. C'est le genre de bug qu'on ne trouve qu'en production.
4. La migration : `xplane-qwen-coder` bascule, ses entrées sont retirées de `route.yaml` dans le même commit ; les 3 autres modèles suivent. Notice `note` honnête : la double comptabilité n'est pas *encore* morte, elle est en cours de retrait.

Marqueur à poser : `<!-- TODO-e2e: schéma avant/après de la propriété du routage (tâche 8) -->`

- [ ] **Step 2: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — composition-owned gateway routing and the readiness latch"
```

---

### Task 4: §3 « LoRA en deux minutes » + §4 « Canary : 10 % du trafic, zéro GPU »

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: le latch (§2) — le canary n'existe que sur une route déjà latchée.
- Produces: la notion d'adapter LoRA, réutilisée en §6 (`servedModels`) et §7 (attribution canary vs base).

- [ ] **Step 1: Rédiger §3 (pédagogie LoRA)**

La partie 3 n'a **jamais** parlé de LoRA : sans cette section, le canary est incompréhensible. Rester court et théorique d'abord (règle de style : la théorie précède la pratique, et reste maigre).

- Un adapter **LoRA** est un **delta de poids de faible rang** appliqué au modèle de base — pas un modèle complet. Quelques dizaines de Mo contre ~15 Go.
- Conséquence décisive : **vLLM sert le modèle de base et ses adapters depuis le même pod, sur le même GPU** (`--enable-lora`, `--lora-modules`). C'est toute la raison pour laquelle la section suivante ne coûte pas un GPU de plus.
- Notice `note` d'honnêteté : ici les adapters viennent de HuggingFace (`jk200201/qwen2.5-coder-7b-sql-dpo`, `scthornton/qwen2.5-coder-7b-securecode`), pinés au commit SHA. **En fine-tuner un soi-même est un autre métier** — teasé dans « Et après ? ».

- [ ] **Step 2: Rédiger §4 (le canary)**

1. Le YAML, deux lignes utiles (`gateway.canaries: [{adapter: xplane-qwen-coder-sql-dpo, weightPercent: 10}]`).
2. Sous le capot : la règle du modèle de base porte un backendRef par entrée ; le backendRef de base garde `100 - sum(weightPercent)`, chaque canary porte son poids et un **`modelNameOverride`** valant le nom de l'adapter verbatim. **Tous pointent le même `Backend`** — c'est là que se joue le « zéro GPU » : le trafic n'est pas envoyé ailleurs, il est **réétiqueté** vers un adapter que le pod sert déjà.
3. La règle de **pin** : une requête qui nomme explicitement `xplane-qwen-coder-sql-dpo` est servie à 100 % par l'adapter, **indépendamment du split**. Les deux comportements coexistent.
4. Les garde-fous CEL, à l'admission : adapters distincts, l'adapter doit exister dans `loraAdapters[]`, poids entre 1 et 99, **`sum(weightPercent) ≤ 99`** — le modèle de base garde toujours du trafic. Souligner *pourquoi* c'est ≤ 99 et pas ≤ 100 : un canary qui absorbe 100 % du trafic n'est plus un canary, c'est un remplacement déguisé.
5. Notice `tip` — leçon de forward-compat : le champ est **un array dès le jour 1** (max 4 entrées), précisément à cause de la revue Modelplane (renvoyer vers §10). Passer d'un objet à un array plus tard aurait été un breaking change de l'API.

Marqueurs à poser :
`<!-- TODO-e2e: split réellement observé via vllm:lora_requests_info (SC-002 : entre 2% et 25% sur ≥50 requêtes, tolérance binomiale) -->`
`<!-- TODO-e2e: schéma du split canary (tâche 8) -->`

- [ ] **Step 3: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — LoRA primer and the zero-GPU weighted canary"
```

---

### Task 5: §5 « L'échappatoire `engineArgs` »

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: le manque n°3 posé en §1.
- Produces: la formule « liberté là où c'est sûr, garde-fous là où ça ne l'est pas », reprise en §11 (leçon n°3).

- [ ] **Step 1: Rédiger la section**

C'est la section la plus transposable hors LLM — la soigner particulièrement.

1. **Le problème, énoncé sans détour :** vLLM ajoute des flags plus vite qu'aucune API de plateforme ne peut suivre. Chaque flag manquant = une release de composition. Une API qui n'offre pas d'échappatoire finit **contournée** (on patche le Deployment à la main, et l'abstraction ment).
2. **La solution :** `spec.engineArgs`, un array de tokens vLLM verbatim (`maxItems: 16`), appendus **après** tous les args gérés par la composition — donc les flags gérés gagnent toujours. Contrainte de forme : **un seul token par entrée** (`--flag` ou `--flag=value`, jamais `--flag value`) — sinon un flag réservé pourrait être **passé en fraude**, éclaté sur deux entrées de la liste, et échapper à la vérification.
3. **Les garde-fous :** 16 flags sont réservés — ceux dont l'autoscaling et le routage dépendent. Rejetés **à `kubectl apply`**, par 16 règles CEL distinctes (une par flag) pour que **chaque message nomme le champ curé à utiliser à la place**. Reproduire un extrait du tableau des flags réservés (5-6 lignes suffisent, avec renvoi au README de la composition pour la liste complète) : `--max-num-seqs` → `use spec.model.maxNumSeqs (it is the KEDA scaling denominator)`, etc.
4. **Le détail qui fait la différence :** `--port` et `--host` sont réservés bien que la composition **ne les émette pas**. Leurs valeurs par défaut (8000, toutes interfaces) portent le contrat de service : le `Service`, les probes et le FQDN du `Backend` en dépendent tous. Un utilisateur pouvait casser le serving en silence.
5. **Une seule source d'application :** la denylist vit **uniquement** dans le CEL de l'XRD. La composition appende verbatim et fait confiance à l'admission — la réimplémenter en KCL aurait recréé exactement le problème que §2 vient de résoudre (deux sources de vérité). Le lockstep XRD ↔ composition est verrouillé par un test (`test_engine_args_denylist_lockstep`, mutation-verified : on a vérifié que le test **échoue** quand on casse volontairement la denylist).
6. Formule de clôture : **liberté là où c'est sûr, garde-fous là où ça ne l'est pas**.

- [ ] **Step 2: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — the engineArgs escape hatch and its CEL denylist"
```

---

### Task 6: §6 « Le modèle dit ce qu'il sert » + §7 « Mesurer le canary »

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: la topologie base + adapters + poids de canary (§3, §4).
- Produces: rien en aval ; §7 clôt le cycle « déployer → router → mesurer ».

- [ ] **Step 1: Rédiger §6 (`status.servedModels`)**

1. `status.servedModels` : une liste d'objets `{name, kind (base|adapter), canaryWeightPercent}`, plus une colonne `SERVED MODELS` dans `kubectl get isvc`. Avant, il fallait rétro-ingénierer la config de routage pour savoir à quoi le modèle répondait.
2. Le champ est calculé **depuis le spec seul** : il est peuplé dès la première réconciliation, sans attendre le Deployment ni la route. C'est une projection de **topologie**, pas un signal de santé — le dire explicitement évite un contresens.
3. Notice `note` — le piège Kubernetes, qui vaut la peine d'être raconté : la colonne pointait d'abord le JSONPath wildcard `.status.servedModels[*].name`. **Le convertisseur de table côté serveur ne rend que la première correspondance d'un wildcard** — la colonne n'aurait affiché que le modèle de base, jamais les adapters. D'où l'ajout d'un scalaire `status.servedModelsSummary` (noms joints par virgule) sur lequel pointe la colonne. Le champ structuré reste l'API machine ; le scalaire est une pure projection pour le CLI.

Marqueur : `<!-- TODO-e2e: sortie réelle de kubectl get isvc -n llm avec la colonne SERVED MODELS -->`

- [ ] **Step 2: Rédiger §7 (mesurer le canary)**

Ouvrir sur l'argument qui justifie la place de la section : **un canary qu'on ne peut pas mesurer n'est pas un canary, c'est un pari.** D'où cette section ici, et pas en annexe.

1. Les métriques `gen_ai.*` d'Envoy AI Gateway v1.0 (standard OpenTelemetry Gen AI, déjà présenté en partie 3), scrapées dans VictoriaMetrics via un `VMPodScrape` sur le sidecar extproc (namespace `envoy-gateway-system`, port admin 1064).
2. Le dashboard Grafana `llm-gateway`, avec **attribution canary vs base** — c'est précisément ce qui permet de répondre à « est-ce que les 10 % de trafic envoyés sur le fine-tune se comportent mieux ou moins bien que les 90 % restants ? ».
3. Notice `note` d'honnêteté : les traces OTLP vers VictoriaTraces sont **volontairement désactivées** pour l'instant, en attendant une vérification de compatibilité de chemin. On ne livre pas ce qu'on n'a pas vérifié.

Marqueur : `<!-- TODO-e2e: screenshot du dashboard Grafana llm-gateway avec l'attribution canary vs base -->`

- [ ] **Step 3: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — servedModels discoverability and gen_ai canary attribution"
```

---

### Task 7: §8 « Endpoint Picker » (+ trigger KEDA) et le cold-start

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: le latch (§2), le canary (§4) — la section pose leur incompatibilité.
- Produces: rien en aval.

- [ ] **Step 1: Rédiger §8 (Endpoint Picker) — section courte**

Ne pas laisser cette section gonfler : elle sert la thèse (**un flag, et la plateforme câble tout un sous-système**), elle n'est pas le sujet.

1. **Le problème :** un `Service` ClusterIP fait du round-robin. C'est **hostile à vLLM** — il éparpille sur des pods différents les requêtes qui partagent un préfixe, détruit la localité du prefix cache et force du prefill redondant. Il est aussi aveugle à la charge : une requête peut atterrir sur un replica dont le KV cache sature pendant qu'un replica tiède dort à côté.
2. **L'Endpoint Picker** (GAIE v1.5.0 ; l'`InferencePool` est GA depuis 2025-09) scrape les **mêmes** `/metrics` vLLM que la plateforme collecte déjà, et score chaque pod candidat : profondeur de queue, utilisation du KV cache, affinité de préfixe, présence de l'adapter LoRA.
3. **KEDA et l'EPP ne se concurrencent pas, ils opèrent à des échelles différentes** : KEDA décide *combien* de replicas, en dizaines de secondes ; l'EPP décide *lequel*, en millisecondes. Les deux lisent les mêmes signaux de saturation. Enchaîner ici sur le troisième trigger KEDA ajouté au passage : `vllm:num_requests_waiting` (`scaling.queueLengthThreshold`, défaut 8) — le signal de pression le plus précoce, combiné en OR avec les deux triggers de la partie 3.
4. Un flag → `HelmRelease` + `InferencePool` + Deployment EPP + `CiliumNetworkPolicy` + `VMServiceScrape`. La sécurité n'est pas une remarque de fin : le pod EPP arrive avec sa politique réseau default-deny et satisfait PSS restricted.
5. Le chiffre externe, cité **comme externe** : llm-d rapporte jusqu'à 57× d'amélioration de TTFT face au round-robin sous charge. **Non reproduit ici** — le dire.
6. Notice `warning` — la limite honnête : **EPP et canary sont mutuellement exclusifs** (rejeté à l'admission). Un backendRef `InferencePool` ne supporte ni `weight` ni `modelNameOverride`, et une seule InferencePool est admise par règle. Choisir : le routage intelligent, ou le canary pondéré. Pas les deux, aujourd'hui.

- [ ] **Step 2: Rédiger le cold-start en notice `tip`**

Court. Run:ai Model Streamer (`--load-format runai_streamer`, embarqué dans l'image `vllm-openai` v0.8.5) sur le chemin PVC existant — prolonge directement la section S3 Files de la partie 3. Ses flags rejoignent la denylist `engineArgs` (cohérence avec §5). `s3://` direct et le résolveur LoRA sont descopés : ils demandent vLLM v0.9.0.

Marqueur : `<!-- TODO-e2e: cold-start mesuré avec et sans le streamer -->`

- [ ] **Step 3: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — endpoint picker, KEDA queue trigger, cold-start streaming"
```

---

### Task 8: §10 Modelplane, §11 Dernières remarques, §12 Références

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: toutes les sections précédentes — §11 en tire les quatre leçons.
- Produces: la fin de l'article.

- [ ] **Step 1: Rédiger §10 (Modelplane)**

Section délicate : il faut être **généreux et honnête**, jamais défensif. Ordre imposé — commencer par créditer, finir par différencier.

1. Les créateurs de Crossplane ont publié **Modelplane** (`ModelService` / `ModelEndpoint`) : un plan de contrôle qui traite les **modèles**, et non les clusters, comme l'objet qu'on gère. Même intuition que celle défendue ici — split plateforme / ML, endpoints retenus jusqu'à ce que le workload soit prêt. Leur donner le crédit de la direction, franchement.
2. Chronologie honnête, énoncée sans en faire une revendication : leur billet date du 2026-06-23, la v0.6.0 de cette plateforme du 2026-05-09. Ce sont des **évolutions convergentes**, pas une course. Leurs docs de design ont surtout servi ici de **grille de revue comparative**.
3. **Eux vont large** : la flotte. Multi-cluster, multi-cloud, types de GPU hétérogènes, un scheduler qui place les replicas. Un problème qu'on n'a **délibérément pas** ici — c'est une plateforme de référence mono-cluster, prétendre le contraire serait du bruit.
4. **Ici on va profond** sur un seul cluster. Trois choses livrées ici qu'ils n'ont pas (à ce jour) : le canary pondéré **implémenté**, pas seulement spécifié (leur design décrit des endpoints pondérés ; ce billet montre le trafic effectivement splitté, validé à l'admission et testé) ; le durcissement production comme **défaut** (zero-trust, PSS restricted, secrets, autoscaling sur signaux de saturation précoces) ; le GitOps de bout en bout.
5. **Ce que leur design a appris** — le point le plus intéressant, et le plus honnête : la discipline de **forward-compatibilité**. Arrays plutôt qu'objets, aucun nouveau champ requis, flags fournis par l'utilisateur plutôt qu'injectés. Le champ `canaries` est un array et l'échappatoire `engineArgs` existe **à cause de cette revue**. Renvoyer vers §4 et §5.
6. Le caveat qui rend la comparaison honnête : eux sont **engine-agnostic** (vLLM, SGLang, topologies de parallélisme arbitraires), ici on est **couplé à vLLM avec une échappatoire**. C'est un choix de périmètre à énoncer, pas à cacher : c'est exactement ce qui rend possibles les signaux d'autoscaling curés.

- [ ] **Step 2: Rédiger §11 (Dernières remarques)**

Réflexion honnête, jamais un résumé, jamais d'auto-satisfaction (règle de style).

Les **quatre leçons**, énoncées comme transposables bien au-delà des LLMs :

1. Si deux fichiers décrivent la même chose, ils divergeront. Ce n'est pas une question de discipline, c'est une question de temps. La composition doit posséder les deux.
2. Une route qui apparaît trop tôt envoie du trafic dans le vide ; une route qui disparaît trop vite fait du flapping. Portail à la création, latch ensuite.
3. Une API sans échappatoire finit contournée ; une échappatoire sans garde-fous casse les invariants. Denylist explicite, appliquée en **un seul** endroit.
4. Si `kubectl get` ne dit pas ce que la ressource fait vraiment, l'abstraction ment.

Puis la phrase de chute : *l'unité de déploiement n'est plus « un pod qui sert un modèle », c'est « **un modèle, avec sa politique de trafic, sa stratégie de rollout et sa découvrabilité** »*.

Sous-section **« Et après ? »** : fine-tuner son propre adapter (le vrai chaînon manquant du canary), `s3://` direct, résolveur LoRA, traces OTLP, migration des 3 modèles restants et suppression définitive de `route.yaml`.

Ajouter une notice `note` sur l'état de validation : ce qui est prouvé aujourd'hui (44/44 `kcl test`, `validate-kcl-compositions.sh` exit 0, `crossplane render` + Polaris contre le tag réel `0.8.0-pr1559`) et ce qui attend le passage e2e sur le cluster.

- [ ] **Step 3: Rédiger §12 (Références)**

Liste simple, calquée sur la partie 3 : la PR #1559 ; les specs `docs/specs/002` → `006` ; les composants (Envoy AI Gateway, Gateway API Inference Extension, Run:ai Model Streamer, KEDA, Crossplane, KCL) ; Modelplane ; les articles précédents de la série (parties 1 à 3).

- [ ] **Step 4: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi.

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — Modelplane positioning, four lessons, references"
```

---

### Task 9: Les deux schémas

**Files:**
- Create: `docs/architecture/llm-gateway-routing-ownership.drawio`
- Create: `docs/architecture/llm-lora-canary-split.drawio`
- Create: `content/fr/post/series/agentic_ai/llm-platform-abstraction/routing-ownership.png`
- Create: `content/fr/post/series/agentic_ai/llm-platform-abstraction/canary-split.png`
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: les marqueurs `TODO-e2e` de schéma posés en tâches 3 et 4.
- Produces: deux `{{< img >}}` en place des marqueurs.

- [ ] **Step 1: Schéma 1 — propriété du routage, avant/après**

Utiliser la skill `drawio-skill`. Deux panneaux côte à côte.

**Avant :** la claim `InferenceService` (à gauche) génère Deployment / Service / KEDA / metrics ; **séparément**, un `route.yaml` écrit à la main (à droite, couleur distincte, icône « main ») génère `Backend` / `AIServiceBackend` / `AIGatewayRoute`. Une flèche en pointillé barrée entre les deux, annotée « aucun lien — peut diverger ». Deux dérives à annoter : « modèle sans route → 404 » et « route orpheline → backend fantôme ».

**Après :** une seule claim génère **tout**, routage compris. Le lien vers l'`AIGatewayRoute` passe par un portail annoté « Deployment Available=True (latch) ».

- [ ] **Step 2: Schéma 2 — le split canary**

**Un seul pod vLLM, un seul GPU L4.** Dans le pod : les poids du modèle de base (~15 Go) + deux adapters LoRA (quelques dizaines de Mo chacun : `sql-dpo`, `securecode`).

En amont, l'`AIGatewayRoute` avec ses règles :
- requête `model: xplane-qwen-coder` → split 90 % base / 10 % `sql-dpo` (via `modelNameOverride`)
- requête `model: xplane-qwen-coder-sql-dpo` → 100 % `sql-dpo` (règle de pin)

Annoter le point central : **les deux branches pointent le même `Backend`, donc le même pod, donc zéro GPU supplémentaire**.

- [ ] **Step 3: Exporter en PNG et insérer dans l'article**

Exporter les deux `.drawio` en PNG (largeur ~1200) dans le page bundle. Remplacer les marqueurs `TODO-e2e` de schéma par :

```markdown
{{< img src="routing-ownership.png" alt="Propriété du routage : avant, deux sources de vérité ; après, une seule" width="1200" >}}
```

```markdown
{{< img src="canary-split.png" alt="Split canary : un pod, un GPU, le modèle de base et deux adapters LoRA" width="1200" >}}
```

- [ ] **Step 4: Vérifier et commiter**

Run: `hugo --minify --buildDrafts 2>&1 | tail -3`
Expected: build réussi ; les dérivées WebP sont générées dans `resources/_gen`.

Run: `grep -c "TODO-e2e" content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`
Expected: 4 (les marqueurs de schéma ont disparu ; restent les 4 marqueurs de données cluster).

```bash
git add docs/architecture/ content/fr/post/series/agentic_ai/llm-platform-abstraction/
git commit -m "docs(blog): part 4 — routing-ownership and canary-split diagrams"
```

---

### Task 10: Passe finale — titre, relecture de style, cohérence

**Files:**
- Modify: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`

**Interfaces:**
- Consumes: l'article complet.
- Produces: un article prêt à publier, à l'exception des `TODO-e2e`.

- [ ] **Step 1: Trancher le titre**

Le titre de travail (« Self-hosted LLM stack : quand l'abstraction grandit ») est **explicitement jugé insuffisant**. Proposer une short-list à l'auteur, avec le résumé (`summary`) correspondant. Pistes de départ, à enrichir : « Le modèle, pas le pod » ; « Un modèle, sa route, son canary » ; « InferenceService : ce que deux mois d'usage ont cassé » ; « Quatre manques, quatre champs YAML ». **Ne pas trancher seul** — c'est une décision d'auteur.

- [ ] **Step 2: Relecture de style contre `ogenki-blog-style`**

Vérifier point par point :
- Paragraphes de 2 à 5 phrases ; les phrases uniques sont **délibérées**.
- Rupture visuelle (liste, tableau, code, notice) tous les 3-4 paragraphes.
- Chaque terme technique en **gras** à sa première occurrence, suivi de sa définition.
- Aucune section ne se termine sur de l'auto-satisfaction.
- Les emoji de section sont présents et cohérents avec ceux de la partie 3.

- [ ] **Step 3: Vérifier les liens internes et la cohérence des faits**

Run: `grep -n "](/" content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`
Expected: **aucun** lien `/en/post/...`. Tous les liens internes en `/fr/post/...`.

Relire l'article contre la section « Faits techniques figés » de ce plan : aucune valeur inventée, aucun chiffre de performance non mesuré, le 57× de llm-d bien attribué comme externe et non reproduit.

- [ ] **Step 4: Build de production et inventaire des `TODO-e2e`**

Run: `hugo --minify --buildDrafts 2>&1 | tail -5`
Expected: build réussi, aucun warning sur le post.

Run: `grep -n "TODO-e2e" content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`
Expected: exactement 4 marqueurs — split canary observé, sortie `kubectl get isvc`, screenshot du dashboard `llm-gateway`, cold-start avec/sans streamer. Les lister à l'auteur : ce sont les seuls bloqueurs de publication.

- [ ] **Step 5: Commit**

```bash
git add content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md
git commit -m "docs(blog): part 4 — final title, style pass and internal link check"
```

---

## Ce qui reste après ce plan

- **Le passage e2e sur le cluster** (hors périmètre de ce plan, dépend de la PR #1559) remplit les 4 marqueurs `TODO-e2e`, après quoi `draft = true` saute.
- **La thumbnail** — à produire, comme pour les articles précédents.
- **La traduction anglaise** — un plan séparé, une fois la version FR figée.
