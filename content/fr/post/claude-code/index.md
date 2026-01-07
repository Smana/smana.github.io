+++
author = "Smaine Kahlouch"
title = "`Agentic Coding` : Retour d'expérience d'un Platform Engineer — concepts et cas concrets"
date = "2026-01-08"
summary = "Explorer l'**agentic coding** à travers `Claude Code` : des fondamentaux (tokens, MCPs, skills) aux cas pratiques, avec un regard enthousiaste mais lucide sur cette nouvelle façon de travailler."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
tags = [
    "ai",
    "devxp",
    "tooling"
]
thumbnail = "thumbnail.png"
+++

Nous le voyons bien, nous assistons à un réel bouleversement provoqué par l'**utilisation de l'IA**. Ce domaine évolue à une telle vitesse qu'il devient presque impossible de suivre toutes les nouveautés. Quant à mesurer l'impact sur notre quotidien et notre façon de travailler, il est encore trop tôt pour le dire. Une chose est sûre cependant : dans la tech, c'est une **révolution** !

Ici, je vais vous présenter une utilisation pratique dans le **métier du "Platform Engineering"** avec une exploration de l'utilisation d'un "**coding agent**" dans certaines tâches communes de notre métier.

Mais surtout, je vais tenter de vous démontrer par des cas concrets que cette nouvelle façon de travailler augmente **réellement** notre productivité. Si si !

## :dart: Objectifs de cet article

* Comprendre ce qu'est un **coding agent** et pourquoi c'est différent d'un simple chatbot
* Découvrir les concepts clés : tokens, MCPs, skills, agents
* **Cas concrets** d'utilisation dans le platform engineering
* Optimiser son utilisation : workflow hybride, économie de contexte, choix du modèle
* Réflexions sur les limites, pièges à éviter et alternatives

{{% notice tip "Le repo de référence" %}}
<table>
  <tr>
    <td><img src="repo_gift.png" style="width:80%;"></td>
    <td style="vertical-align:middle; padding-left:10px;" width="70%">
Les exemples qui suivent sont issus de mon travail sur le repository <strong><a href="https://github.com/Smana/cloud-native-ref">Cloud Native Ref</a></strong>. Il s'agit d'une plateforme complète combinant EKS, Cilium, VictoriaMetrics, Crossplane, Flux et bien d'autres outils.
    </td>
  </tr>
</table>
{{% /notice %}}

---

## :brain: L'intérêt des _Coding Agents_?

### Qu'est-ce qui différencie un agent d'un chatbot ?

Vous utilisez probablement déjà ChatGPT, LeChat ou Gemini pour poser des questions. C'est cool, mais ça reste du **one-shot** : vous posez une question, vous obtenez une réponse dont la pertinence dépendra de la qualité de votre prompt.

Un **coding agent** fonctionne différemment. Il exécute des outils en boucle pour atteindre un objectif. C'est ce qu'on appelle une [**boucle agentique**](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/).

{{< img src="agentic-loop.png" alt="Agentic loop" width="580" >}}

Le cycle est simple : **raisonner → agir → observer → répéter**. L'agent appelle un outil, analyse le résultat, puis décide de la prochaine action. Il est donc essentiel qu'il ait accès au **retour de chaque action** — une erreur de compilation, un test qui échoue, une sortie inattendue. Cette capacité à réagir et **itérer de manière autonome** sur notre environnement local est ce qui fait toute la différence avec un simple chatbot.

Un coding agent combine plusieurs composants :

* **LLM** : Le "cerveau" qui raisonne (Claude Opus 4.5, Gemini 3 Pro, Devstral 2...)
* **Tools** : Les actions possibles (lire/écrire des fichiers, exécuter des commandes, chercher sur le web...)
* **Memory** : Le contexte conservé (fichiers `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`... selon l'outil, plus l'historique de conversation)
* **Planning** : La capacité à décomposer une tâche complexe en sous-étapes

### Bien choisir son modèle, dur de suivre la cadence 🤯

De nouveaux modèles ainsi que de nouvelles versions apparaissent à une vitesse effrénée. Il faut cependant être vigilant dans le choix du modèle car l'efficacité (qualité de code, hallucinations, contexte à jour) peut **radicalement différer**.

Le benchmark [**SWE-bench Verified**](https://www.swebench.com/) est devenu la référence pour évaluer les capacités des modèles en développement logiciel. Il mesure la capacité à résoudre de vrais bugs issus de repositories GitHub et permet de nous aider à faire notre choix.

{{< img src="swe-bench-leaderboard.png" alt="Leaderboard SWE-bench Verified" width="750" >}}

{{% notice warning "Ces chiffres évoluent très vite !" %}}
Consultez [swebench.com](https://www.swebench.com/) pour les derniers résultats. Au moment de la rédaction, Claude Opus 4.5 mène avec **74.4%**, talonné par Gemini 3 Pro (**74.2%**).
{{% /notice %}}

En pratique, les meilleurs modèles actuels sont tous suffisamment performants pour la plupart des tâches de platform engineering.

{{% notice info "Opus 4.5, un game changer" %}}
*"J'utilise Opus 4.5 avec thinking pour tout. Comme vous avez moins besoin de le guider, il est presque toujours plus rapide au final."* — Boris Cherny (créateur de Claude Code)

Je me retrouve complètement dans cette citation. En ce qui me concerne, je n'utilise plus que ce modèle et cela convient parfaitement à mon usage quotidien.
{{% /notice %}}

### Pourquoi Claude Code ?

Il existe de nombreuses options de coding agents, dont voici quelques exemples :

| Outil | Type | Forces |
|-------|------|--------|
| [**Claude Code**](https://docs.anthropic.com/en/docs/claude-code) | Terminal | Context 200K, SWE-bench élevé, hooks & MCP |
| [**opencode**](https://opencode.ai/) | Terminal | **Open source**, multi-provider, modèles locaux (Ollama) |
| [**Cursor**](https://cursor.sh/) | IDE | Workflow visuel, Composer mode |
| [**Antigravity**](https://antigravity.google/) | IDE | Agents parallèles, Manager view |

Nous pouvons aussi citer les alternatives suivantes (liste non exhaustive) : [Gemini CLI](https://github.com/google-gemini/gemini-cli), [Mistral Vibe](https://mistral.ai/news/devstral-2-vibe-cli), [GitHub Copilot](https://github.com/features/copilot)...

J'ai utilisé Cursor dans un premier temps, puis je suis passé à Claude Code. Probablement en raison de mon **background de sysadmin** plutôt porté sur le terminal. Là où d'autres préfèrent travailler exclusivement dans leur IDE, je me sens plus à l'aise avec une CLI.

---

## :books: Les concepts essentiels de Claude Code

Cette section va droit à l'essentiel : **tokens, MCPs, Skills et Tasks**. Je passe sur la config initiale (la [doc officielle](https://docs.anthropic.com/en/docs/claude-code) fait ça très bien) et sur les subagents — c'est de la mécanique interne, ce qui compte c'est ce qu'on peut *construire* avec. La plupart de ces concepts **s'appliquent aussi à d'autres coding agents**.

### Tokens et fenêtre de contexte

#### L'essentiel sur les tokens

Un **token** est l'unité de base que le modèle traite — environ 4 caractères en anglais, 2-3 en français. Pourquoi c'est important ? Parce que **tout se paye en tokens** : input, output, et contexte.

La **fenêtre de contexte** (200K tokens pour Claude) représente la "mémoire de travail" du modèle. La commande `/context` permet de visualiser comment cet espace est utilisé :

```console
/context
```

{{< img src="cmd_context.png" alt="Visualisation du contexte avec /context" width="650" >}}

Cette vue détaille la répartition du contexte entre les différents composants :

* **System prompt/tools** : Coût fixe de Claude Code (~10%)
* **MCP tools** : Définitions des MCPs activés
* **Memory files** : `CLAUDE.md`, `AGENTS.md`...
* **Messages** : Historique de conversation
* **Autocompact buffer** : Réservé pour la compression automatique
* **Free space** : Espace disponible pour continuer

<table>
  <tr>
    <td style="vertical-align:middle;" width="70%">
Une fois la limite atteinte, les informations les plus anciennes sont tout simplement <strong>oubliées</strong>. Heureusement, Claude Code dispose d'un mécanisme d'<strong>auto-compaction</strong> : quand la conversation approche des 200K tokens, il <strong>compresse intelligemment</strong> l'historique en conservant les décisions importantes tout en éliminant les échanges verbeux. Ce mécanisme permet de travailler sur des sessions longues sans perdre le fil — mais une compaction fréquente dégrade la qualité du contexte. D'où l'intérêt d'utiliser <code>/clear</code> entre les tâches distinctes.
    </td>
    <td><img src="dori_forgot.png" style="width:100%;"></td>
  </tr>
</table>

### Les MCPs : un langage universel

Le **Model Context Protocol** (MCP) est un standard ouvert créé par Anthropic qui permet aux agents IA de se connecter à des sources de données et outils externes de manière standardisée.

{{% notice info "Gouvernance ouverte" %}}
En décembre 2025, Anthropic a [confié MCP à la Linux Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) via l'Agentic AI Foundation. OpenAI, Google, Microsoft et AWS font partie des membres fondateurs.
{{% /notice %}}

Il existe une multitude de serveurs MCP. Voici ceux que j'utilise au quotidien pour interagir avec ma plateforme — **configuration, troubleshooting, analyse** :

| MCP | À quoi ça sert | Exemple concret |
|-----|----------------|-----------------|
| **[context7](https://github.com/upstash/context7)** | Doc à jour des libs/frameworks | "Utilise context7 pour la doc Cilium 1.16" → évite les hallucinations sur des APIs qui ont changé |
| **[flux](https://fluxcd.control-plane.io/mcp/)** | Debug GitOps, état des reconciliations | "Pourquoi mon HelmRelease est stuck ?" → Claude inspecte directement l'état Flux |
| **[victoriametrics](https://github.com/VictoriaMetrics-Community/mcp-victoriametrics)** | Requêtes PromQL, exploration métriques | "Quelles métriques Karpenter sont dispo ?" → liste et requête en direct |
| **[victorialogs](https://github.com/VictoriaMetrics-Community/mcp-victorialogs)** | Requêtes LogsQL, analyse logs | "Trouve les erreurs Crossplane des 2 dernières heures" → root cause analysis |
| **[grafana](https://github.com/grafana/mcp-grafana)** | Dashboards, alertes, annotations | "Crée un dashboard pour ces métriques" → génère et déploie le JSON |
| **[steampipe](https://github.com/turbot/steampipe-mcp)** | Requêtes SQL sur infra cloud | "Liste les buckets S3 publics" → audit multi-cloud en une question |

{{% notice tip "Configuration globale ou locale?" %}}
Les MCPs peuvent être configurés globalement (`~/.claude/mcp.json`) ou par projet (`.mcp.json`). J'utilise `context7` globalement car je m'en sers quasi systématiquement, les autres au niveau du repo.
{{% /notice %}}

### Skills : obtenir de nouveaux pouvoirs

{{< img src="skill-acquired-notif.png" width="450" >}}

C'est probablement la fonctionnalité qui m'a le plus emballé. Un **skill** est un fichier Markdown (`.claude/skills/*/SKILL.md`) qui permet d'injecter des **conventions**, **patterns** et **procédures** spécifiques à votre projet.

Concrètement ? Vous définissez une fois comment créer une PR propre, comment valider une composition Crossplane, ou comment débugger un problème Cilium — et Claude applique ces règles à chaque situation. C'est du **savoir-faire encapsulé** que vous pouvez partager avec votre équipe.

**Deux modes de chargement :**

* **Automatique** : Claude analyse la description du skill et le charge quand c'est pertinent
* **Explicite** : Vous invoquez directement via `/nom-du-skill`

{{% notice info "Un format qui devient standard" %}}
Les Skills ont été introduits par Anthropic et rencontrent un vrai engouement. D'autres coding agents implémentent désormais ce format : [GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills), [Google Antigravity](https://codelabs.developers.google.com/getting-started-with-antigravity-skills), Cursor... Les skills que vous créez sont un **investissement portable**.
{{% /notice %}}

#### Anatomie d'un skill

Un skill se compose d'un **frontmatter YAML** (métadonnées) et d'un **contenu Markdown** (instructions). Voici le skill `/create-pr` de [cloud-native-ref](https://github.com/Smana/cloud-native-ref/tree/main/.claude/skills) — il génère des PRs avec description structurée et diagramme Mermaid :

```markdown
<!-- .claude/skills/create-pr/SKILL.md -->
---
name: create-pr
description: Create Pull Requests with AI-generated descriptions and mermaid diagrams
allowed-tools: Bash(git:*), Bash(gh:*)
---

## Usage
/create-pr [base-branch]       # Nouvelle PR (défaut: main)
/create-pr --update <number>   # Met à jour une PR existante

## Workflow
1. Gather: git log, git diff --stat, git diff (en parallèle)
2. Detect: Type de changement (composition, infrastructure, security...)
3. Generate: Summary, diagramme Mermaid, table des fichiers
4. Create: git push + gh pr create
```

| Champ | Rôle |
|-------|------|
| `name` | Nom du skill et commande `/create-pr` |
| `description` | Aide Claude à décider quand charger automatiquement |
| `allowed-tools` | Outils autorisés sans confirmation (`git`, `gh`) |

Ce skill illustre bien la valeur ajoutée : il orchestre plusieurs commandes, analyse le diff pour détecter le type de changement, et génère une PR structurée avec diagramme — quelque chose de fastidieux à faire manuellement, et que Claude fait maintenant en quelques secondes.

### Tasks : ne jamais perdre le fil

{{< img src="tasks_notif.png" width="450" >}}

Les **Tasks** (v2.1.16+) résolvent un vrai problème des workflows autonomes : comment garder le fil sur une tâche complexe qui s'étale dans le temps ?

Les Tasks remplacent l'ancien système de "Todos" et apportent trois améliorations clés : **persistance entre sessions**, **visibilité partagée entre agents**, et **tracking des dépendances**.

Concrètement, quand Claude travaille sur une tâche longue, il peut :
- Décomposer le travail en Tasks avec dépendances
- Déléguer certaines Tasks en background
- Reprendre le travail après une interruption sans perte de contexte

{{% notice tip "Commande /tasks" %}}
Utilisez `/tasks` pour voir l'état des tâches en cours. Pratique pour suivre où en est Claude sur un workflow complexe.
{{% /notice %}}

---

## :rocket: Cas concrets de platform engineering

Assez de théorie ! Passons à ce qui nous intéresse vraiment : comment Claude Code peut nous aider au quotidien. Je vais vous partager deux cas concrets et détaillés qui illustrent la puissance des MCPs et du workflow avec Claude.

### :mag: Cas 1 : Supervision complète de Karpenter avec les MCPs

Ce cas illustre parfaitement la puissance de la **boucle agentique** présentée en introduction. Grâce aux MCPs, Claude dispose d'un contexte complet sur mon environnement (métriques disponibles, documentation à jour, état du cluster) et peut **itérer de manière autonome** : créer des ressources, les déployer, valider visuellement le résultat, puis corriger si nécessaire.

#### Le prompt

La structuration du prompt est essentielle pour guider efficacement l'agent. Un prompt bien organisé — avec contexte, objectif, étapes et contraintes — permet à Claude de comprendre non seulement *quoi* faire, mais aussi *comment* le faire. Pour aller plus loin, le [guide de prompt engineering d'Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) détaille ces bonnes pratiques.

Voici le prompt utilisé pour cette tâche :

```markdown
## Contexte
Je gère un cluster Kubernetes avec Karpenter pour l'autoscaling.
MCPs disponibles : grafana, victoriametrics, victorialogs, context7, chrome.

## Objectif
Créer un système d'observabilité complet pour Karpenter : alertes + dashboard unifié.

## Étapes
1. **Documentation** : Via context7, consulte la doc récente de Grafana
   (alerting, dashboards) et des datasources Victoria
2. **Alertes** : Crée des alertes pour :
    - Erreurs de provisioning des nodes
    - Échecs d'appels API AWS
    - Dépassement de quotas
3. **Dashboard** : Crée un dashboard Grafana unifié intégrant :
    - Métriques (temps de provisioning, coûts, capacity)
    - Logs d'erreurs Karpenter
    - Événements Kubernetes liés aux nodes
4. **Validation** : Déploie via kubectl, puis valide visuellement avec
   les MCPs grafana et chrome
5. **Finalisation** : Si le rendu est correct, applique via l'opérateur
   Grafana, commit et crée la PR

## Contraintes
- Utilise les fonctionnalités récentes de Grafana (v11+)
- Suis les bonnes pratiques : variables de dashboard, annotations,
  seuils d'alerte progressifs
```

#### Étape 1 : Planification et décomposition

Claude analyse le prompt et génère automatiquement un **plan structuré** en sous-tâches. Cette décomposition permet de suivre la progression et garantit que chaque étape est complétée avant de passer à la suivante.

{{< img src="karpenter_plan.png" alt="Plan généré par Claude Code" width="450" >}}

On voit ici les 4 tâches identifiées : création des alertes VMRule, création du dashboard unifié, validation avec kubectl et Chrome, puis finalisation avec commit et PR.

#### Étape 2 : Exploitation des MCPs pour le contexte

C'est ici que la magie opère. Claude utilise **simultanément plusieurs MCPs** pour obtenir un contexte complet :

{{< img src="karpenter_mcp.png" alt="Appels aux MCPs" width="1200" >}}

- **context7** : Récupère la documentation Grafana v11+ pour les alerting rules et le format JSON des dashboards
- **victoriametrics** : Liste toutes les métriques `karpenter_*` disponibles dans mon cluster

Cette combinaison permet à Claude de générer du code **adapté à mon environnement réel** plutôt que des exemples génériques potentiellement obsolètes.

#### Étape 3 : Validation visuelle avec Chrome MCP

Une fois le dashboard déployé via `kubectl`, Claude utilise le **MCP Chrome** pour ouvrir Grafana et valider visuellement le rendu. Il peut ainsi vérifier que les panels s'affichent correctement, que les requêtes retournent des données, et ajuster si nécessaire.

<center>
<video width="1200" autoplay loop muted playsinline>
  <source src="chrome_karpenter.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la lecture vidéo.
</video>
</center>

Cette boucle de rétroaction visuelle est un exemple concret de ce qui différencie un coding agent d'un simple chatbot : Claude **observe le résultat de ses actions** et peut itérer jusqu'à obtenir le résultat souhaité.

#### Résultat : une observabilité complète

À l'issue de ce workflow, Claude a créé une **PR complète** : 12 alertes VMRule (provisioning, API AWS, quotas, interruptions Spot) et un dashboard Grafana unifié combinant métriques, logs et événements Kubernetes.

{{< img src="karpenter_summary.png" alt="Résumé de la session" width="650" >}}

{{% notice tip "Ce qui a fait la différence" %}}
Sans les MCPs, j'aurais dû :
1. Consulter manuellement la doc Grafana v11 pour le format JSON
2. Lister les métriques Karpenter disponibles via VMUI
3. Créer les alertes et le dashboard à la main
4. Déployer, ouvrir Grafana, vérifier, corriger, re-déployer...

Avec les MCPs, Claude a accès à **tout ce contexte directement** et peut **boucler de manière autonome** jusqu'au résultat final. La PR a été créée en une seule session, avec un dashboard fonctionnel et des alertes cohérentes.
{{% /notice %}}

---

### :building_construction: Cas 2 : La spec comme source de vérité — offrir un nouveau service

Ce deuxième cas illustre la création d'une **composition Crossplane** — l'abstraction qui permet aux équipes produit de consommer des services cloud via une API Kubernetes simple. C'est le cœur du **Platform Engineering** : transformer la complexité infrastructure en self-service.

{{% notice info "Qu'est-ce que le Spec-Driven Development (SDD) ?" %}}
Le **Spec-Driven Development** est un paradigme où les spécifications — et non le code — servent d'artefact principal. À l'ère de l'IA agentique, le SDD fournit les garde-fous nécessaires pour éviter le "Vibe Coding" (prompting non structuré) et garantir que les agents produisent du code maintenable.

Pour ceux qui baignent dans Kubernetes, on peut faire un parallèle 😉 : la spec définit l'**état désiré**, et une fois validée par l'humain, l'agent IA se comporte un peu comme un *controller* — il itère en fonction des résultats (tests, validations) jusqu'à atteindre cet état. La différence : l'humain reste dans la boucle (**HITL**) pour valider la spec *avant* que l'agent ne se lance, et pour revoir le résultat final.

**Les frameworks majeurs en 2026 :**

| Framework | Force principale | Cas d'usage idéal |
|-----------|-----------------|-------------------|
| **[GitHub Spec Kit](https://github.com/github/spec-kit)** | Intégration native GitHub/Copilot | Projets greenfield, workflow structuré |
| **[BMAD](https://github.com/bmad-sim/bmad-method)** | Équipes multi-agents (PM, Architect, Dev) | Systèmes complexes multi-repos |
| **[OpenSpec](https://github.com/Fission-AI/OpenSpec)** | Léger, centré sur les changements | Projets brownfield, itération rapide |
{{% /notice %}}

{{% notice tip "Ma variante SDD pour le Platform Engineering" %}}
Pour [cloud-native-ref](https://github.com/Smana/cloud-native-ref), j'ai créé une variante inspirée de GitHub Spec Kit, adaptée aux contraintes du Platform Engineering.

**🛡️ Platform Constitution** — Les principes non-négociables sont codifiés dans une [constitution](https://github.com/Smana/cloud-native-ref/blob/main/docs/specs/constitution.md) : préfixe `xplane-*` pour le scoping IAM, zero-trust networking obligatoire, secrets via External Secrets uniquement. Claude vérifie chaque spec et implémentation contre ces règles.

**👥 4 personas de review** — Chaque spec passe par une checklist qui force à considérer plusieurs angles :

| Persona | Focus |
|---------|-------|
| **PM** | Clarté du problème, user stories alignées aux besoins réels |
| **Platform Engineer** | Cohérence API, patterns KCL respectés |
| **Security** | Zero-trust, least privilege, secrets externalisés |
| **SRE** | Health probes, observabilité, modes de failure |

**⚡ Skills Claude Code** — Le workflow est orchestré par des [skills](/post/claude-code/#skills--obtenir-de-nouveaux-pouvoirs) (voir section précédente) qui automatisent chaque étape :

| Skill | Action |
|-------|--------|
| `/spec` | Crée l'issue GitHub + le fichier spec pré-rempli |
| `/clarify` | Résout les `[NEEDS CLARIFICATION]` avec options structurées |
| `/validate` | Vérifie la complétude avant implémentation |
| `/create-pr` | Crée la PR avec référence automatique à la spec |

{{< img src="sdd_workflow.png" alt="Workflow SDD" width="700" >}}
{{% /notice %}}

#### Pourquoi le SDD pour le Platform Engineering ?

Créer une composition Crossplane n'est pas un simple script — c'est concevoir une **API pour vos utilisateurs**. Chaque décision a des implications durables :

| Décision | Impact |
|----------|--------|
| Structure de l'API (XRD) | Contrat avec les équipes produit — difficile à changer après adoption |
| Ressources créées | Coûts cloud, surface de sécurité, dépendances opérationnelles |
| Valeurs par défaut | Ce que 80% des utilisateurs obtiendront sans y penser |
| Intégrations (IAM, Network, Secrets) | Conformité, isolation, auditabilité |

Le SDD force à **réfléchir avant de coder** et à **documenter les décisions** — exactement ce dont on a besoin pour une API de plateforme.

#### Le cas d'usage : une composition Queue

L'équipe produit a besoin d'un système de queuing pour leurs applications. Selon le contexte, ils veulent pouvoir choisir entre :
- **Kafka (via Strimzi)** : pour les cas nécessitant du streaming, de la rétention longue, ou du replay
- **AWS SQS** : pour les cas simples, serverless, avec intégration native AWS

Plutôt que de leur demander de configurer Strimzi ou SQS directement (dizaines de paramètres), on va leur exposer une **API simple et unifiée**.

#### Étape 1 : Créer la spec avec `/spec`

Le skill `/spec` est le point d'entrée du workflow. Il crée automatiquement :
- Une **GitHub Issue** avec le label `spec:draft` pour le suivi et les discussions
- Un **fichier de spec** dans `docs/specs/` pré-rempli avec le template du projet

```
/spec composition "Add queuing composition supporting Strimzi (Kafka) or SQS"
```

{{< img src="sdd_spec.png" alt="Création de la spec" width="850" >}}

Claude analyse le contexte du projet (compositions existantes, constitution, ADRs) et pré-remplit la spec avec un **initial design**. Il identifie également les **points de clarification** — ici 3 questions clés sur le scope et l'authentification.

L'issue GitHub sert d'**ancre immuable** pour les discussions, tandis que le fichier spec contient le design détaillé.

#### Étape 2 : Clarifier les choix de design avec `/clarify`

La spec générée contient des marqueurs `[NEEDS CLARIFICATION]` pour les décisions que Claude ne peut pas prendre seul. Le skill `/clarify` les présente sous forme de **questions structurées avec options** :

{{< img src="sdd_clarify_1.png" alt="Clarification des choix" width="850" >}}

Chaque question propose des options analysées selon **4 perspectives** (PM, Platform Engineer, Security, SRE) avec une recommandation. Je navigue entre les onglets pour répondre à chaque clarification.

Une fois toutes les clarifications résolues, Claude met à jour la spec avec un résumé des décisions :

{{< img src="sdd_clarify_2.png" alt="Résumé des décisions" width="850" >}}

| # | Question | Décision |
|---|----------|----------|
| 1 | Kafka scope | Topics only (clusterRef required) |
| 2 | Kafka auth | SASL/SCRAM only |
| 3 | SQS scope | Same-account only |

Ces décisions sont **documentées dans la spec** — dans 6 mois, quand quelqu'un demandera "pourquoi pas de mTLS ?", la réponse sera là.

#### Étape 3 : Valider et implémenter

Avant de commencer l'implémentation, le skill `/validate` vérifie la complétude de la spec :
- Toutes les sections requises sont présentes
- Tous les marqueurs `[NEEDS CLARIFICATION]` sont résolus
- L'issue GitHub est liée
- La constitution du projet est référencée

Une fois validée, je demande à Claude d'implémenter la spec. Il entre en **plan mode** et lance des agents d'exploration en parallèle pour comprendre les patterns existants :

{{< img src="sdd_implement_1.png" alt="Exploration parallèle" width="850" >}}

Claude explore les compositions existantes (`SQLInstance`, `EKS Pod Identity`, la configuration Strimzi) pour comprendre les conventions du projet **avant d'écrire une seule ligne de code**.

L'implémentation génère les ressources appropriées selon le backend choisi :

{{< img src="sdd_implement_summary.png" alt="Ressources créées par backend" width="500" >}}

Pour **chaque backend**, la composition crée les ressources nécessaires tout en respectant les conventions du projet :
- Préfixe `xplane-*` pour toutes les ressources (convention IAM)
- `CiliumNetworkPolicy` pour le zero-trust networking
- `ExternalSecret` pour les credentials (pas de secrets en dur)
- `VMServiceScrape` pour l'observabilité

#### Étape 4 : Validation finale

Le skill `/validate` vérifie non seulement la spec mais aussi l'**implémentation** :

{{< img src="sdd_validation.png" alt="Validation finale" width="700" >}}

La validation couvre :
- **Spec** : Sections présentes, clarifications résolues, issue liée
- **Implémentation** : Phases complétées, exemples créés, CI passing
- **Review checklist** : Les 4 personas (PM, Platform Engineer, Security, SRE)

Les items "N/A" (tests E2E, documentation, failure modes) sont clairement identifiés comme optionnels pour ce type de composition.

#### Résultat : l'API utilisateur finale

Le développeur peut maintenant déclarer son besoin en quelques lignes :

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: Queue
metadata:
  name: orders-queue
  namespace: ecommerce
spec:
  # Kafka pour le streaming avec rétention
  type: kafka
  clusterRef:
    name: main-kafka
  config:
    partitions: 6
    retentionDays: 7
```

Ou pour SQS :

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: Queue
metadata:
  name: notifications-queue
  namespace: notifications
spec:
  # SQS pour les cas simples
  type: sqs
  config:
    visibilityTimeout: 30
    enableDLQ: true
```

Dans les deux cas, la plateforme gère automatiquement :
- La création des ressources (topics Kafka ou queues SQS)
- L'authentification (SASL/SCRAM ou IAM)
- Le monitoring (métriques exportées vers VictoriaMetrics)
- La sécurité réseau (CiliumNetworkPolicy)
- L'injection des credentials dans le namespace de l'application

{{% notice tip "Ce que le SDD apporte concrètement" %}}
Sans SDD, j'aurais probablement :
- Commencé à coder directement, découvert des edge cases en cours de route
- Oublié de documenter pourquoi "topics only" plutôt que "full cluster"
- Implémenté des features inutiles (mTLS, cross-account) "au cas où"
- Créé une API incohérente avec les compositions existantes

Avec SDD :
- Les **décisions sont documentées** avant l'implémentation
- Les **4 perspectives** (PM, Platform, Security, SRE) sont systématiquement couvertes
- L'API est **cohérente** avec l'existant (Claude a exploré les patterns)
- La PR référence la spec — le reviewer comprend le contexte
{{% /notice %}}

---

## :bulb: Optimiser son utilisation

### Git Worktrees : paralléliser les sessions Claude

Plutôt que de jongler avec des branches et du `stash`, utilisez les **git worktrees** pour travailler sur plusieurs features en parallèle avec des sessions Claude indépendantes.

```console
# Create two different features
git worktree add ../worktrees/feature-a -b feat/feature-a
git worktree add ../worktrees/feature-b -b feat/feature-b

# Run two distinct claude sessions
cd ../worktrees/feature-a && claude  # Terminal 1
cd ../worktrees/feature-b && claude  # Terminal 2
```

Chaque session a son **propre contexte** et sa propre mémoire — aucune interférence entre les tâches.

{{% notice tip "Worktree vs copie du repo" %}}
Contrairement à un `git clone` séparé :
- **Historique partagé** : Un seul `.git`, les commits sont immédiatement visibles partout
- **Espace disque** : ~90% d'économie (pas de duplication des objets git)
- **Branches synchronisées** : `git fetch` dans un worktree met à jour tous les autres
{{% /notice %}}

Lorsque le changement est terminé (PR mergée), il suffit de retourner dans le repo principal et de faire le ménage.

```console
cd <path_to_main_repo>
git worktree remove ../worktrees/feature-a
git branch -d feat/feature-a  # après merge de la PR
```

**Commandes utiles :**

```console
git worktree list              # Voir tous les worktrees actifs
git worktree prune             # Nettoyer les références orphelines
```

### Le workflow hybride Cursor + Claude Code

C'est l'approche que je recommande : **Cursor** pour l'édition quotidienne, **Claude Code** pour les tâches agentiques.

{{< img src="cursor+claude.png" alt="Cursor + Claude Code" width="1000" >}}

| Besoin | Outil | Pourquoi |
|--------|-------|----------|
| Édition rapide, autocomplete | Cursor | Latence minimale, vous restez dans le flow |
| Refactoring, debugging multi-fichiers | Claude Code | Raisonnement profond, boucles autonomes |

**Le vrai gain** : Claude modifie via le terminal, vous validez les diffs dans l'interface Cursor — bien plus lisible que `git diff`.

### Économiser le contexte

Le contexte est précieux (et coûteux). Quelques règles simples :

1. **`/clear` entre les tâches** : Chaque nouvelle tâche devrait commencer par un `/clear`
2. **CLAUDE.md concis** : Chaque token est chargé à chaque conversation
3. **CLIs > MCPs** : Pour les outils matures (`kubectl`, `git`, `gh`...), préférez la CLI directe — les LLMs les maîtrisent parfaitement et ça évite de charger un MCP
4. **`/context` pour auditer** : Identifiez ce qui consomme et désactivez les MCPs non utilisés

### Ce qui fonctionne bien vs ce qui nécessite vigilance

| ✅ Claude excelle | ⚠️ Vigilance requise |
|-------------------|----------------------|
| Debugging avec contexte | Création from scratch |
| Conversion de formats | Sécurité/PKI |
| Refactoring répétitif | Ressources cloud coûteuses |
| Analyse de dépendances | Breaking changes |

### Plugins recommandés

Claude Code dispose d'un écosystème de plugins qui étendent ses capacités. Voici deux plugins particulièrement utiles :

#### Code-Simplifier : nettoyer le code généré

Le plugin **[code-simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)** est développé par Anthropic et utilisé en interne par l'équipe Claude Code. Il nettoie automatiquement le code généré par l'IA tout en préservant la fonctionnalité.

**Cas d'usage** : Après une longue session de coding où Claude a implémenté plusieurs features, lancez le code-simplifier pour éliminer le code dupliqué, simplifier les structures complexes et harmoniser le style.

{{% notice tip "Mon workflow" %}}
J'utilise systématiquement le code-simplifier avant de créer une PR après une session intensive. Il tourne sur Opus et peut significativement réduire la dette technique introduite par le code IA.
{{% /notice %}}

#### Claude-Mem : mémoire persistante entre sessions

Le plugin **[claude-mem](https://github.com/thedotmack/claude-mem)** capture automatiquement le contexte de vos sessions et le réinjecte dans les sessions futures. Plus besoin de réexpliquer votre projet à chaque nouvelle conversation.

**Fonctionnalités clés :**
- **Capture automatique** : Enregistre les actions Claude (tool usage, décisions)
- **Recherche sémantique** : Skill `mem-search` pour retrouver des informations passées
- **Interface web** : Dashboard sur `http://localhost:37777`
- **Workflow 3-layer** : Optimise la consommation de tokens (~10x d'économie)

Exemples:

  - "Search my memories for when I debugged Karpenter"
  - "Find what I learned about OpenBao PKI last week"
  - "Look up my previous work on the App composition"


{{% notice warning "Considérations de confidentialité" %}}
Claude-mem stocke localement les données de session. Pour les projets sensibles, utilisez les tags `<private>` pour exclure des informations de la capture.
{{% /notice %}}

---

## :thought_balloon: Dernières remarques

Nous avons pu explorer grâce à cet article l'IA agentique et comment ses principes peuvent être utiles au quotidien. Un agent ayant accès à un contexte enrichi (`CLAUDE.md`, skills, MCPs...) peut **vraiment** être très efficace : qualité au rendez-vous et surtout, une rapidité impressionnante ! Le workflow SDD permet également, pour les projets plus complexes, de formaliser son intention et de mieux cadrer l'agent.

| Aspect | Avant Claude Code | Avec Claude Code |
|--------|-------------------|------------------|
| Debugging Cilium | 2h de lecture de logs | 15 min avec contexte |
| Refactoring Terraform | Journée entière | 2h avec review |
| Écriture de doc | Procrastination | Généré + relu = 30 min |
| Onboarding nouveau repo | Plusieurs jours | Quelques heures |

### Points de vigilance

En revanche, cela amène de nombreuses réflexions. Une [étude METR](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) a trouvé un résultat surprenant : les développeurs utilisant l'IA prennent **19% plus de temps** pour compléter des tâches, alors qu'ils *croient* avoir été plus rapides de 24%. Voici quelques leçons que j'en tire :

* **Éviter la dépendance et continuer à apprendre** — reviewer systématiquement les specs et le code généré, comprendre *pourquoi* cette solution
* **Se forcer à travailler sans IA** — je m'impose un rythme d'au moins 2 sessions par semaine "à l'ancienne"
* **Utiliser l'IA comme professeur** — lui demander d'expliquer son raisonnement et ses choix, c'est un excellent moyen d'apprendre
* **Demander la solution la plus simple** — selon une [étude Qodo](https://www.qodo.ai/reports/state-of-ai-code-quality/), le code IA génère 4× plus de code dupliqué

{{% notice warning "Confidentialité et code propriétaire" %}}
Si vous travaillez sur du code sensible ou propriétaire :
- Utilisez le plan **Team** ou **Enterprise** — vos données ne servent pas à l'entraînement
- Demandez l'option **Zero-Data-Retention** (ZDR) si nécessaire
- N'utilisez **jamais** le plan Free/Pro pour du code confidentiel

Consultez la [documentation sur la confidentialité](https://www.anthropic.com/policies/privacy) pour plus de détails.
{{% /notice %}}

### Mes prochaines étapes

C'est une préoccupation que je partage avec beaucoup de développeurs : **que se passe-t-il si Anthropic change les règles du jeu ?** Cette crainte s'est d'ailleurs matérialisée début janvier 2026, lorsqu'Anthropic a [bloqué sans préavis](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses) l'accès à Claude via des outils tiers comme [OpenCode](https://github.com/opencode-ai/opencode).

De par ma sensibilité pour l'open source, j'ai pour objectif d'explorer les alternatives ouvertes. **[Mistral Vibe](https://mistral.ai/news/devstral-2-vibe-cli)** avec Devstral 2 (72.2% SWE-bench) et **[OpenCode](https://opencode.ai/)** (multi-provider, modèles locaux via Ollama) sont en haut de ma liste. L'idée n'est pas forcément de remplacer Claude Code — qui reste excellent — mais d'avoir une solution de repli et de contribuer à un écosystème plus ouvert.

{{% notice info "Rejoignez la discussion" %}}
J'aimerais beaucoup avoir vos retours d'expérience ! N'hésitez pas à me contacter ou à ouvrir une issue sur le [repo cloud-native-ref](https://github.com/Smana/cloud-native-ref).
{{% /notice %}}

---

## :bookmark: Références

### Guides et best practices
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Engineering
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) — Guide complet par sshh

### Spec-Driven Development
- [GitHub Spec Kit](https://github.com/github/spec-kit) — Toolkit SDD de GitHub
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) — SDD léger pour projets brownfield
- [BMAD Method](https://github.com/bmad-sim/bmad-method) — Multi-agent SDD

### Plugins, Skills et MCPs
- [Code-Simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) — Nettoyage de code IA
- [Claude-Mem](https://github.com/thedotmack/claude-mem) — Mémoire persistante entre sessions
- [CC-DevOps-Skills](https://github.com/akin-ozer/cc-devops-skills) — 31 skills DevOps prêts à l'emploi
- [Awesome Claude Code Plugins](https://github.com/ccplugins/awesome-claude-code-plugins) — Liste curatée

### Études citées
- [METR Study on AI Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) — Étude sur la productivité
- [State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/) — Qodo
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Research

### Ressources
- [Cloud Native Ref](https://github.com/Smana/cloud-native-ref) — Mon repo de référence
- [SWE-bench Leaderboards](https://www.swebench.com/) — Benchmark de référence
