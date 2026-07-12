# Design — Article « Self-hosted LLM stack, partie 4 » (FR)

**Date**: 2026-07-12
**Statut**: approuvé (design), rédaction à venir
**Langue**: français uniquement dans un premier temps (traduction EN plus tard)
**Fichier cible**: `content/fr/post/series/agentic_ai/llm-platform-abstraction/index.md`
**Source technique**: [PR #1559](https://github.com/Smana/cloud-native-ref/pull/1559) — `cloud-native-ref`, composition `InferenceService` v0.8.0
**Article précédent**: [partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/) — « poser les fondations »

---

## Thèse

> L'unité de déploiement n'est plus « un pod qui sert un modèle », c'est « **un modèle, avec sa politique de trafic, sa stratégie de rollout et sa découvrabilité** ».

L'article raconte **comment une abstraction de plateforme mûrit sous la pression du réel**. La partie 3 a posé une claim `InferenceService` qui déclare un modèle et génère son outillage Kubernetes. Deux mois d'usage ont révélé quatre manques structurels — et c'est leur résolution qui fait l'article.

L'angle est **platform engineering**, pas LLM serving : les quatre leçons (source de vérité unique, latch de readiness, échappatoire encadrée, status découvrable) sont réutilisables bien au-delà des LLMs. Le canary LoRA est la **démo phare**, pas le sujet.

## Décisions de cadrage

| Décision | Choix | Pourquoi |
|---|---|---|
| Angle | L'abstraction qui mûrit | Cœur du lectorat (platform engineers) ; leçons transposables hors LLM |
| Série | Agentic AI, **partie 4** | Suite directe de la partie 3, même notice de série en tête |
| Périmètre | SPEC-002/003/006 en profondeur ; SPEC-004 court ; SPEC-005 en encadré | Évite le changelog : chaque section doit servir la thèse |
| Preuves | Rédaction immédiate, marqueurs `TODO-e2e` | L'e2e cluster n'est pas encore passé ; on ne publie pas un chiffre non mesuré |
| LoRA | Section pédagogique + fine-tuning teasé | La partie 3 n'a jamais parlé de LoRA ; sans ça le canary est incompréhensible |

## Titre

**Titre de travail** : « Self-hosted LLM stack : quand l'abstraction grandit »

⚠️ **À retravailler avant publication** — jugé insatisfaisant. Short-list à reprendre :

- « Le modèle, pas le pod »
- « Un modèle, sa route, son canary »
- « InferenceService : ce que deux mois de production ont cassé »
- « Quatre manques, quatre champs YAML »

## Plan détaillé

### Intro + rappel

Notice `info` « Série Agentic AI — Partie 4 » renvoyant vers les parties 1 à 3.

Deux mois après la partie 3. La fondation a tenu : la claim YAML fait toujours le job. Mais en la faisant **vivre**, quatre manques se sont révélés. Ce sont eux qui structurent l'article.

Rappel court de la claim de la partie 3 (~10 lignes de YAML) pour que l'article se lise seul.

### §1 — Ce qui clochait

Le cœur de l'argument, posé tôt et sans ménagement. Auto-critique assumée d'un travail publié deux mois plus tôt.

- **Deux sources de vérité** — la claim déclare le modèle, un `ai-gateway-routes/route.yaml` **écrit à la main** déclare comment on l'atteint. Rien n'empêche un modèle sans route, ni une route qui survit à son modèle.
- **Aucune stratégie de rollout** — on bascule un modèle en tout-ou-rien.
- **Aucune échappatoire** — chaque nouveau flag vLLM exige une release de la plateforme.
- **Aucune découvrabilité** — `kubectl get isvc` ne dit pas à quels noms le modèle répond réellement.

### §2 — Le routage rejoint le modèle *(profondeur : haute)*

Source : SPEC-002 (FR-001, FR-002).

- `gateway.enabled: true` → la composition rend `Backend` + `AIServiceBackend` + `AIGatewayRoute` **par claim**. Supprimer le modèle supprime sa route.
- Le point technique intéressant : le **latch de readiness**. La route est retenue tant que le `Deployment` n'est pas `Available=True`, puis **jamais retirée** sur une indisponibilité passagère. Ni trafic vers un modèle qui ne peut pas répondre, ni flapping de route.
- Détail d'implémentation qui mérite une phrase : clés ocds de lecture et d'écriture partagent une constante unique (`_gatewayRouteSuffix`) — elles ne *peuvent pas* diverger.
- Migration : `xplane-qwen-coder` bascule, ses entrées sont retirées du `route.yaml` dans le même commit.

**Visuel** : schéma avant/après de la propriété du routage (deux sources de vérité → une).

### §3 — LoRA en deux minutes *(pédagogie)*

Prérequis absent de la partie 3.

- Un adapter LoRA est un **delta de poids de faible rang** appliqué au modèle de base, pas un modèle complet.
- vLLM sert le modèle de base **et** ses adapters **depuis le même pod, sur le même GPU** (`--enable-lora`, `--lora-modules`). C'est toute la raison pour laquelle la section suivante est gratuite.
- Honnêteté : ici les adapters viennent de HuggingFace (`jk200201/qwen2.5-coder-7b-sql-dpo`, `scthornton/qwen2.5-coder-7b-securecode`), pinés au commit SHA. **En fine-tuner un soi-même est un autre métier** — teasé en « et après ».

### §4 — Canary : 10 % du trafic, zéro GPU *(profondeur : haute)*

Source : SPEC-002 (FR-003, FR-004, CL-5, CL-6).

- Deux lignes de YAML : `gateway.canaries: [{adapter, weightPercent}]`.
- Sous le capot : `modelNameOverride` sur l'AIGatewayRoute — le trafic du modèle de base est splitté vers l'adapter, **même pod, zéro GPU supplémentaire**.
- Garde-fous CEL à l'admission : adapters distincts, somme des poids ≤ 99 %, l'adapter doit exister dans `loraAdapters[]`, poids entre 1 et 99.
- Une requête qui **nomme explicitement** l'adapter reste servie à 100 % (elle ne subit pas le split).
- Leçon de forward-compat : le champ est **un array dès le jour 1**, précisément à cause de la revue Modelplane (cf. §10).

**Visuel** : schéma du split — un pod, un GPU, base + 2 adapters, 90/10.

### §5 — L'échappatoire `engineArgs` *(profondeur : haute)*

Source : SPEC-003.

- Le problème : **vLLM ajoute des flags plus vite qu'aucune API de plateforme ne peut suivre**. Chaque flag manquant = une release de composition.
- La solution : passthrough verbatim des flags, appendus après tous les args gérés par la composition — **sauf** les ~16 flags dont l'autoscaling et le routage dépendent.
- Ces flags sont rejetés **à `kubectl apply`** (17 règles CEL dans l'XRD), avec un message qui **nomme le champ curé à utiliser à la place** : `--max-num-seqs` → `spec.model.maxNumSeqs`.
- Le lockstep XRD ↔ composition est verrouillé par un test (`test_engine_args_denylist_lockstep`, mutation-verified) : la denylist ne peut pas dériver silencieusement.
- Formule de la section : **liberté là où c'est sûr, garde-fous là où ça ne l'est pas**.

### §6 — Le modèle dit ce qu'il sert *(profondeur : moyenne)*

Source : SPEC-003 (`status.servedModels`).

- `status.servedModels` : topologie structurée (base + adapters + poids de canary), calculée à chaque réconciliation.
- Colonne `SERVED MODELS` dans un `kubectl get isvc`.
- Avant : il fallait rétro-ingénierer la config de routage pour savoir à quoi le modèle répondait.

**`TODO-e2e`** : sortie réelle de `kubectl get isvc`.

### §7 — Mesurer le canary *(profondeur : moyenne)*

Source : SPEC-006.

- Un canary qu'on ne peut pas mesurer n'est pas un canary — d'où la place de cette section **ici** et pas en annexe.
- Métriques `gen_ai.*` d'Envoy AI Gateway v1.0, scrapées dans VictoriaMetrics (VMPodScrape sur le sidecar extproc, port admin 1064).
- Dashboard Grafana `llm-gateway`, avec **attribution canary vs base**.
- Traces OTLP vers VictoriaTraces : volontairement **OFF** pour l'instant (compat de chemin non vérifiée) — à dire honnêtement.

**`TODO-e2e`** : screenshot du dashboard `llm-gateway`.

### §8 — Endpoint Picker *(profondeur : courte)*

Source : SPEC-004.

- Un `Service` ClusterIP fait du **round-robin**, ce qui est **hostile à vLLM** : il éparpille les requêtes partageant un préfixe sur des pods différents et détruit la localité du prefix cache. Il est aussi aveugle à la charge par pod.
- L'**Endpoint Picker** (GAIE v1.5.0) score chaque pod candidat : profondeur de queue, utilisation du KV cache, affinité de préfixe, présence de l'adapter LoRA.
- Complémentaire de KEDA, pas concurrent : KEDA réagit en dizaines de secondes (*combien* de replicas), l'EPP en millisecondes (*lequel*).
- Un flag → HelmRelease + InferencePool + EPP + CiliumNetworkPolicy + VMServiceScrape.
- Chiffre externe cité **comme tel** : llm-d rapporte jusqu'à 57× d'amélioration de TTFT face au round-robin. Non reproduit ici.

Notice `warning` honnête : **mutuellement exclusif avec le canary**, parce qu'un backendRef InferencePool ne supporte ni `weight` ni `modelNameOverride`. Rejeté à l'admission.

### §9 — Cold-start *(encadré)*

Source : SPEC-005.

Notice `tip` : Run:ai Model Streamer (`--load-format runai_streamer`, embarqué dans vllm-openai v0.8.5) sur le chemin PVC existant — prolonge directement la section S3 Files de la partie 3. `s3://` direct et le résolveur LoRA sont descopés (nécessitent vLLM v0.9.0).

**`TODO-e2e`** : cold-start avec / sans streamer.

### §10 — Modelplane : évolution convergente *(profondeur : moyenne)*

- Les créateurs de Crossplane ont publié **Modelplane** (`ModelService` / `ModelEndpoint`) — même intuition : le split plateforme / ML, les endpoints retenus jusqu'à Ready. Leur billet est postérieur (2026-06-23) à la v0.6.0 de cette plateforme (2026-05-09), mais leurs docs de design ont servi de **grille de revue comparative** ici.
- **Eux vont large** : la flotte — multi-cluster, multi-cloud, scheduler de placement, engine-agnostic (vLLM, SGLang). Un problème qu'on n'a délibérément pas ici.
- **Ici on va profond** sur un seul cluster : le canary pondéré est **implémenté**, pas seulement spécifié ; zero-trust / PSS restricted / secrets / GitOps sont le **plancher**, pas une remarque de fin.
- **Ce que leur design a appris** : la discipline de forward-compat — arrays plutôt qu'objets, pas de nouveau champ requis, flags fournis par l'utilisateur plutôt qu'injectés. Le champ `canaries` est un array et l'échappatoire `engineArgs` existe **à cause de cette revue**.
- Caveat honnête assumé : eux sont engine-agnostic, ici on est **couplé à vLLM avec une échappatoire** — c'est exactement ce qui rend les signaux d'autoscaling curés possibles.

### §11 — 💭 Dernières remarques

- Les quatre leçons, énoncées comme transposables hors LLM :
  1. Si deux fichiers décrivent la même chose, ils divergeront. La composition doit posséder les deux.
  2. Une route qui apparaît trop tôt envoie du trafic dans le vide ; une route qui disparaît trop vite fait du flapping. Latch.
  3. Une API sans échappatoire finit contournée. Une échappatoire sans garde-fous casse les invariants. Denylist explicite.
  4. Si `kubectl get` ne dit pas ce que la ressource fait vraiment, l'abstraction ment.
- Phrase de chute : *l'unité de déploiement n'est plus « un pod qui sert un modèle », c'est « un modèle, avec sa politique de trafic, sa stratégie de rollout et sa découvrabilité »*.
- **Et après ?** : fine-tuner son propre adapter, `s3://` direct, résolveur LoRA, traces OTLP, migration des 3 modèles restants.

### §12 — 🔖 Références

Repos, specs (`docs/specs/002` → `006`), composants (GAIE, Envoy AI Gateway, Run:ai Model Streamer, Modelplane), articles précédents de la série.

## Front matter

```toml
+++
author = "Smaine Kahlouch"
title = "<titre à retravailler>"
date = "2026-07-XX"
summary = "..."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "vllm", "crossplane", "platform-engineering"]
thumbnail = "thumbnail.png"
+++
```

## Visuels à produire

| Visuel | Type | Statut |
|---|---|---|
| Propriété du routage, avant/après | drawio → PNG | à produire |
| Split canary (1 pod, 1 GPU, base + 2 adapters, 90/10) | drawio → PNG | à produire |
| Dashboard Grafana `llm-gateway` | screenshot | `TODO-e2e` |
| `kubectl get isvc` avec colonne `SERVED MODELS` | bloc de code | `TODO-e2e` |
| Thumbnail | image | à produire |

## Contraintes de rédaction

- Style : voir la skill `ogenki-blog-style`. Ton de la partie 3 : concret, honnête, notices pour la pédagogie.
- **Aucune affirmation de perf non mesurée.** Les chiffres externes (57× TTFT de llm-d) sont cités comme externes.
- Liens internes FR : `/fr/post/...`.
- Le repo `cloud-native-ref` reste la source de vérité citée.
