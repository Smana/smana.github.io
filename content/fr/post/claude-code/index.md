+++
author = "Smaine Kahlouch"
title = "Claude Code : Quand l'IA devient le copilote du Platform Engineer"
date = "2026-01-08"
summary = "Utilisation pratique d'un **coding agent** dans le quotidien du platform engineering. Au-delà du hype, des cas concrets qui démontrent comment cette nouvelle façon de travailler peut réellement **booster notre productivité**. Concepts, configuration, et retours d'expérience."
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

{{% notice info "L'IA dans notre quotidien" %}}
Impossible d'y échapper : l'IA transforme nos métiers. Fin 2025, **65% des développeurs** utilisent des outils d'IA au moins une fois par semaine selon [Stack Overflow](https://stackoverflow.com/). Mais au-delà des annonces sensationnelles, qu'en est-il **concrètement** pour nous, platform engineers ?

Dans cet article, je partage mon expérience avec Claude Code et vous montre, par des exemples réels, comment cet outil est devenu un allié précieux dans mes tâches quotidiennes.
{{% /notice %}}

Nous le voyons bien, nous assistons à un réel bouleversement provoqué par l'utilisation de l'IA. Ce domaine évolue à une vitesse vertigineuse et, honnêtement, il est difficile de mesurer aujourd'hui l'impact sur tous les aspects de notre métier. Une chose est sûre cependant : dans la tech, c'est une **révolution** !

Je ne vais pas vous faire un énième tutoriel ChatGPT. Ici, je vais vous présenter une utilisation **pratique** dans le métier du platform engineering avec une exploration de l'utilisation d'un **coding agent** — pas un simple chatbot — dans certaines tâches communes de notre quotidien.

Mais surtout, je vais tenter de vous démontrer par des cas concrets que cette nouvelle façon de travailler augmente **réellement** notre productivité. Si si !

## :dart: Objectifs de cet article

* Comprendre ce qu'est un **coding agent** et pourquoi c'est différent d'un simple chatbot
* Découvrir les concepts clés : tokens, contexte, MCPs, subagents
* Installer et configurer Claude Code efficacement
* **Cas concrets** d'utilisation dans le platform engineering
* Maîtriser les coûts et optimiser son utilisation
* Réflexions sur les limites et les pièges à éviter

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

## :brain: Comprendre les coding agents

### Qu'est-ce qui différencie un agent d'un chatbot ?

Vous utilisez probablement déjà ChatGPT ou Gemini pour poser des questions. C'est pratique, mais ça reste du **one-shot** : vous posez une question, vous obtenez une réponse, point final.

Un **coding agent** fonctionne différemment. Il opère en boucle selon le pattern **ReAct** (Reasoning + Action) :

<center><img src="react-loop.png" width="700" alt="Boucle ReAct"></center>

1. **Raisonnement** : L'agent analyse votre demande et planifie les étapes
2. **Action** : Il exécute une action (lire un fichier, exécuter une commande, chercher dans le code)
3. **Observation** : Il analyse le résultat de son action
4. **Itération** : Il décide si c'est suffisant ou s'il faut continuer

{{% notice info "La définition selon Simon Willison" %}}
[Simon Willison](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/), expert reconnu du domaine, définit un agent LLM comme : *"quelque chose qui exécute des outils en boucle pour atteindre un objectif"*. C'est simple, mais ça capture l'essentiel.
{{% /notice %}}

Concrètement, si vous demandez à un chatbot classique *"Corrige le bug dans mon auth"*, il vous donnera des suggestions génériques. Un agent, lui, va :

1. Chercher les fichiers liés à l'authentification
2. Lire le code concerné
3. Identifier le problème
4. Proposer une correction
5. L'appliquer si vous validez
6. Vérifier que ça compile/fonctionne

C'est cette capacité à **agir** sur votre environnement qui fait toute la différence.

### L'anatomie d'un agent

On peut résumer un agent avec cette formule :

```
Agent = LLM + Tools + Memory + Planning
```

| Composant | Rôle | Exemple dans Claude Code |
|-----------|------|--------------------------|
| **LLM** | Le "cerveau" qui raisonne | Claude Opus 4.5 / Sonnet 4 |
| **Tools** | Les actions possibles | Read, Write, Bash, Grep, WebFetch |
| **Memory** | Le contexte conservé | CLAUDE.md, conversation history |
| **Planning** | La stratégie d'exécution | Décomposition en sous-tâches |

### Le choix du modèle : une course effrénée

Les nouvelles versions de modèles apparaissent à une vitesse folle. Impossible de suivre ! L'efficacité (qualité de code, hallucinations, context mis à jour) peut radicalement différer selon les modèles.

Le benchmark [**SWE-bench Verified**](https://www.swebench.com/) est devenu la référence pour évaluer les capacités des modèles en développement logiciel. Il mesure la capacité à résoudre de vrais bugs issus de repositories GitHub.

<center><img src="swe-bench-leaderboard.png" width="750" alt="Leaderboard SWE-bench Verified"></center>

{{% notice warning "Ces chiffres évoluent très vite !" %}}
Consultez [swebench.com](https://www.swebench.com/) pour les derniers résultats. Au moment de la rédaction, les modèles frontier (Claude Opus 4.5, GPT-5.x, Gemini 3) se disputent la première place avec des scores autour de **75-80%**.
{{% /notice %}}

**Points clés à retenir sur SWE-bench :**

| Aspect | Impact |
|--------|--------|
| **Scaffold/Agent** | Les scores varient selon l'agent utilisé avec le modèle |
| **SWE-bench Pro** | Benchmark plus difficile (~20-25% pour les meilleurs) |
| **Parité au sommet** | Le choix dépend aussi du coût, de la latence et du context window |

La compétition est si féroce que la première place change régulièrement. En pratique, tous les modèles frontier sont suffisamment performants pour la plupart des tâches de platform engineering.

### Pourquoi Claude Code ?

Il existe de nombreuses options de coding agents : [Cursor](https://cursor.sh/), [Windsurf](https://codeium.com/windsurf), [GitHub Copilot](https://github.com/features/copilot), [Gemini CLI](https://github.com/google-gemini/gemini-cli)... Je ne suis clairement pas capable de toutes les évaluer en profondeur.

J'ai utilisé Cursor dans un premier temps, puis je suis passé à Claude Code. La raison ? Mon **background de sysadmin** plutôt porté sur le terminal. Là où d'autres préfèrent travailler exclusivement dans leur IDE, je me sens plus à l'aise avec une CLI.

Selon [Artificial Analysis](https://artificialanalysis.ai/insights/coding-agents-comparison), voici comment se positionnent les outils :

| Outil | Type | Forces | Idéal pour |
|-------|------|--------|------------|
| **Claude Code** | Terminal | Context 200K, score SWE-bench le plus élevé | Refactoring large, DevOps, automation |
| **Cursor** | IDE | Workflow visuel, Composer mode | Développement applicatif, UI |
| **GitHub Copilot** | IDE Plugin | Intégration native, entreprise-ready | Équipes Microsoft/GitHub |
| **Windsurf** | IDE | Cascade mode, UX soignée | Prototypage rapide |

{{% notice tip "Mon conseil" %}}
Beaucoup de développeurs utilisent **plusieurs outils** : Cursor pour écrire du code applicatif, Claude Code pour le refactoring et l'infrastructure. Ce n'est pas exclusif !
{{% /notice %}}

---

## :books: Les concepts essentiels

Avant de plonger dans l'utilisation concrète, il est important de comprendre quelques concepts clés. Pas de panique, je vais rester pragmatique !

### Tokens et fenêtre de contexte

#### Qu'est-ce qu'un token ?

Un **token** est l'unité de base que le modèle traite. Ce n'est pas exactement un mot — c'est plutôt un "morceau" de texte. En règle générale :

- 1 token ≈ 4 caractères en anglais
- 1 token ≈ 2-3 caractères en français (les accents comptent !)
- 100 tokens ≈ 75 mots

Pourquoi c'est important ? Parce que **tout se paye en tokens** : ce que vous envoyez (input), ce que Claude génère (output), et le contexte qu'il maintient.

#### La fenêtre de contexte

La **fenêtre de contexte** (context window) représente la quantité maximale de tokens que le modèle peut "voir" à un instant donné. Pensez-y comme sa **mémoire de travail**.

| Modèle | Context Window | Équivalent |
|--------|----------------|------------|
| Claude Opus 4.5 | 200K tokens | ~150K mots / ~300 pages |
| Claude Sonnet 4.5 | 200K tokens | ~150K mots / ~300 pages |
| Claude Sonnet 4.5 (beta) | 1M tokens | ~750K mots / ~1500 pages |
| GPT-4o | 128K tokens | ~96K mots / ~200 pages |

{{% notice note "Attention à la saturation !" %}}
Lorsque le contexte approche de sa limite, les performances se dégradent. Les recherches montrent qu'il vaut mieux **privilégier la qualité du contexte à la quantité**. C'est pour cela que Claude Code dispose d'un mécanisme d'**auto-compaction** qui résume automatiquement les conversations trop longues.
{{% /notice %}}

#### Visualiser son contexte

Claude Code fournit une commande très utile pour comprendre ce qui consomme votre contexte :

```console
/context
```

<center><img src="context-visualization.png" width="600" alt="Visualisation du contexte"></center>

Cette vue montre :
- Tokens utilisés par le système et les outils
- Tokens des fichiers CLAUDE.md (mémoire)
- Tokens de la conversation
- Espace libre disponible

### Les MCPs : connecter Claude au monde extérieur

#### En quelques mots

Le **Model Context Protocol** (MCP) est un standard ouvert créé par Anthropic qui permet aux agents IA de se connecter à des sources de données et outils externes. Pensez-y comme une **prise USB-C pour l'IA** : un connecteur universel.

{{% notice info "Un standard qui s'impose" %}}
En décembre 2025, Anthropic a [donné MCP à la Linux Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) via l'Agentic AI Foundation. OpenAI, Google, Microsoft et AWS l'ont adopté. Avec **97 millions** de téléchargements mensuels et plus de **5,800 serveurs** disponibles, c'est devenu LE standard de facto.
{{% /notice %}}

#### Architecture MCP

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Claude Code   │────▶│   MCP Client    │────▶│   MCP Server    │
│   (l'agent)     │     │   (intégré)     │     │   (externe)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
                                               ┌─────────────────┐
                                               │  GitHub, K8s,   │
                                               │  Prometheus...  │
                                               └─────────────────┘
```

#### Mes MCPs indispensables pour le platform engineering

Voici les MCPs que j'utilise au quotidien et qui constituent mon toolkit essentiel :

| MCP | Catégorie | Usage | Intérêt pour PE |
|-----|-----------|-------|-----------------|
| **[context7](https://github.com/upstash/context7)** | Documentation | Doc à jour des libs | Évite les hallucinations d'API |
| **[flux](https://fluxcd.control-plane.io/mcp/)** | GitOps | Debug Flux, reconciliation | Troubleshooting pipelines |
| **[victoriametrics](https://github.com/VictoriaMetrics-Community/mcp-victoriametrics)** | Métriques | Requêtes PromQL | Analyse cardinalité, alertes |
| **[victorialogs](https://github.com/VictoriaMetrics-Community/mcp-victorialogs)** | Logs | LogsQL | Root cause analysis |
| **[grafana](https://github.com/grafana/mcp-grafana)** | Visualisation | Dashboards, alertes | Création/modification dashboards |
| **[steampipe](https://github.com/turbot/steampipe-mcp)** | Cloud SQL | Requêtes infrastructure | Audit multi-cloud |

---

{{% notice tip "Configuration globale ou locale?" %}}

Il est possible de configurer les MCPs de manière globale ou locale. Par exemple, j'utilise quasi systématiquement `context7`, j'ai donc décidé de l'ajouter au fichier global `~/.claude/mcp.json`, ce qui m'évite de le définir pour chacun de mes projets.
Les autres MCPs sont définis au niveau du repo.

❯ /mcp

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Manage MCP servers
 5 servers

   Project MCPs (/home/smana/Sources/cloud-native-ref/.mcp.json)
 ❯ flux-operator-mcp · ✔ connected
   victorialogs · ✔ connected
   victoriametrics · ✔ connected

   User MCPs (/home/smana/.claude.json)
   context7 · ✔ connected

   Built-in MCPs (always available)
   claude-in-chrome · ✔ connected

{{% /notice %}}

#### Context7 : la documentation à jour

[Context7](https://github.com/upstash/context7) (Upstash) résout un problème majeur : les **hallucinations d'API**. Quand Claude génère du code, il peut inventer des fonctions ou utiliser des syntaxes deprecated.

Context7 injecte la documentation **versionnée** directement dans le contexte de Claude. Plus besoin de copier-coller la doc !

```console
# Installation
claude mcp add context7 -- npx -y @upstash/context7-mcp

# Ou avec une API key pour des rate limits plus élevés
claude mcp add context7 -- npx -y @upstash/context7-mcp --api-key YOUR_API_KEY
```

**Utilisation** : Ajoutez simplement "use context7" dans votre prompt :

```
> use context7 pour la doc Cilium 1.16
> Comment configurer une CiliumNetworkPolicy pour autoriser le traffic cross-namespace ?
```

---

#### Flux GitOps : debug des pipelines

Le [Flux MCP Server](https://fluxcd.control-plane.io/mcp/) (ControlPlane) connecte Claude directement à vos clusters Kubernetes via Flux Operator.

**Capacités :**
- Debug end-to-end des pipelines GitOps
- Root cause analysis des déploiements échoués
- Visualisation des dépendances (génère des diagrammes)
- Trigger reconciliations, suspend/resume

```console
# Installation
claude mcp add-json "flux-operator" '{"command":"npx","args":["-y","@controlplane/flux-mcp-server"]}'
```

**Exemple concret** :

```
> Mes Kustomizations sont bloquées. Analyse les dépendances
> et identifie pourquoi infrastructure reste en "Not Ready"
```

Claude va interroger le cluster, tracer les dépendances, et identifier les blocages (dépendances circulaires, secrets manquants, etc.).

---

#### VictoriaMetrics : analyse des métriques

Le [MCP VictoriaMetrics](https://github.com/VictoriaMetrics-Community/mcp-victoriametrics) donne à Claude un accès **read-only** à toutes les APIs VM :
- Requêtes PromQL
- Exploration de métriques, labels, séries
- Analyse de cardinalité
- Test des alerting rules

```console
# Installation (nécessite Go 1.24+)
go install github.com/VictoriaMetrics-Community/mcp-victoriametrics/cmd/mcp-victoriametrics@latest

# Configuration
claude mcp add-json "victoriametrics" '{
  "command": "/path/to/mcp-victoriametrics",
  "env": {
    "VM_INSTANCE_ENTRYPOINT": "https://vmselect.monitoring.svc:8481",
    "VM_INSTANCE_TYPE": "cluster"
  }
}'
```

**Variables d'environnement :**
- `VM_INSTANCE_ENTRYPOINT` : URL de votre instance VM
- `VM_INSTANCE_TYPE` : `single` ou `cluster`
- `VM_INSTANCE_BEARER_TOKEN` : Token d'authentification (optionnel)

---

#### VictoriaLogs : analyse des logs

Le [MCP VictoriaLogs](https://github.com/VictoriaMetrics-Community/mcp-victorialogs) permet à Claude d'interroger vos logs avec **LogsQL** pour le troubleshooting et la root cause analysis.

```console
# Installation via Smithery
npx -y @smithery/cli install @VictoriaMetrics-Community/mcp-victorialogs --client claude

# Ou manuellement
claude mcp add-json "victorialogs" '{
  "command": "/path/to/mcp-victorialogs",
  "env": {
    "VL_INSTANCE_ENTRYPOINT": "https://victorialogs.monitoring.svc:9428"
  }
}'
```

**Combiné avec VictoriaMetrics**, vous pouvez demander :

```
> Une alerte CPU élevée est déclenchée sur le namespace demo.
> Analyse les métriques des dernières 30 minutes puis corrèle avec les logs
> pour identifier la root cause.
```

---

#### Grafana : dashboards et alertes

Le [MCP Grafana](https://github.com/grafana/mcp-grafana) permet à Claude de rechercher, lire et **créer** des dashboards.

**Capacités :**
- Recherche de dashboards par titre ou metadata
- Récupération des détails d'un dashboard
- Gestion des alertes et incidents
- Accès aux datasources

```console
# Installation
claude mcp add-json "grafana" '{
  "command": "npx",
  "args": ["-y", "@grafana/mcp-server"],
  "env": {
    "GRAFANA_URL": "https://grafana.example.com",
    "GRAFANA_API_KEY": "${GRAFANA_API_KEY}"
  }
}'
```

{{% notice info "Grafana 9.0+ requis" %}}
Certaines fonctionnalités nécessitent Grafana 9.0 ou supérieur pour un accès complet aux APIs.
{{% /notice %}}

---

#### Steampipe : requêtes SQL sur le cloud

powerpipe mod install github.com/turbot/steampipe-mod-aws-insights github.com/turbot/steampipe-mod-aws-compliance github.com/turbot/steampipe-mod-aws-top-10 github.com/turbot/steampipe-mod-kubernetes-compliance github.com/turbot/steampipe-mod-kubernetes-insights

[Steampipe MCP](https://github.com/turbot/steampipe-mcp) (Turbot) permet à Claude d'interroger **100+ services cloud** avec du SQL :
- AWS, Azure, GCP
- Kubernetes, GitHub, Microsoft 365
- Plus de 2000 tables documentées

```console
# Installation (nécessite Steampipe local)
steampipe service start
claude mcp add steampipe -- npx -y @turbot/steampipe-mcp

# Ou avec Turbot Pipes (cloud)
claude mcp add steampipe -- npx -y @turbot/steampipe-mcp \
  "postgresql://user:pass@workspace.usea1.db.pipes.turbot.com:9193/db"
```

**Exemple :**

```
> Liste toutes les instances EC2 sans tags Environment
> dans les régions eu-west-*
```

Claude génère et exécute :

```sql
SELECT instance_id, region, tags
FROM aws_ec2_instance
WHERE region LIKE 'eu-west-%'
  AND tags->>'Environment' IS NULL;
```

---

#### Configuration complète : exemple de mcp.json

Voici un exemple complet de configuration pour un environment de platform engineering. Ce fichier peut être placé dans `.claude/mcp.json` à la racine de votre projet :

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"],
      "env": {
        "CONTEXT7_API_KEY": "${CONTEXT7_API_KEY}"
      }
    },
    "flux-operator": {
      "command": "npx",
      "args": ["-y", "@controlplane/flux-mcp-server"],
      "env": {
        "KUBECONFIG": "${HOME}/.kube/config"
      }
    },
    "victoriametrics": {
      "command": "${HOME}/go/bin/mcp-victoriametrics",
      "env": {
        "VM_INSTANCE_ENTRYPOINT": "https://vmselect.monitoring.svc:8481",
        "VM_INSTANCE_TYPE": "cluster",
        "VM_INSTANCE_BEARER_TOKEN": "${VM_BEARER_TOKEN}"
      }
    },
    "victorialogs": {
      "command": "${HOME}/go/bin/mcp-victorialogs",
      "env": {
        "VL_INSTANCE_ENTRYPOINT": "https://victorialogs.monitoring.svc:9428"
      }
    },
    "grafana": {
      "command": "npx",
      "args": ["-y", "@grafana/mcp-server"],
      "env": {
        "GRAFANA_URL": "https://grafana.example.com",
        "GRAFANA_API_KEY": "${GRAFANA_API_KEY}"
      }
    },
    "steampipe": {
      "command": "npx",
      "args": ["-y", "@turbot/steampipe-mcp"]
    }
  }
}
```

**Scopes de configuration :**

| Scope | Emplacement | Usage |
|-------|-------------|-------|
| **local** | `.claude/mcp.json` | Uniquement pour vous, ce projet |
| **project** | Committé dans le repo | Partagé avec l'équipe |
| **user** | `~/.claude/mcp.json` | Tous vos projets |

**Commandes utiles :**

```console
# Vérifier les MCPs configurés
claude mcp list

# Vérifier le statut dans une session
/mcp

# Voir les détails d'un MCP
claude mcp get victoriametrics
```

{{% notice tip "Mon conseil sur Kubernetes" %}}
Pour Kubernetes, j'utilise directement `kubectl` via les commandes bash plutôt qu'un MCP dédié. La CLI consomme **moins de tokens** et Claude la maîtrise parfaitement. Les MCPs Kubernetes existants sont utiles pour des cas avancés, mais `kubectl` suffit dans 90% des cas.
{{% /notice %}}

:warning: **Attention** : Chaque MCP ajouté consomme des tokens pour ses définitions d'outils. Utilisez `/context` pour identifier la consommation et désactivez les MCPs non nécessaires pour la tâche en cours avec `--disable-mcp`.

### Les subagents : déléguer intelligemment

#### Qu'est-ce qu'un subagent ?

Un **subagent** est une instance Claude séparée, lancée par l'agent principal pour effectuer une tâche spécifique. C'est comme déléguer à un "stagiaire spécialisé".

**Pourquoi c'est puissant :**

| Avantage | Explication |
|----------|-------------|
| **Contexte isolé** | Le subagent a sa propre mémoire, il ne pollue pas la conversation principale |
| **Spécialisation** | Vous pouvez lui donner un "persona" (expert sécurité, expert Terraform...) |
| **Parallélisme** | Jusqu'à 10 subagents peuvent tourner simultanément |

#### Cas d'usage concret

```
> J'ai 50 fichiers Terraform à auditer pour des problèmes de sécurité.
> Utilise des subagents pour analyser chaque module en parallèle
> et donne-moi un rapport consolidé.
```

Claude va alors :
1. Identifier les modules à analyser
2. Lancer un subagent par module (en parallèle)
3. Collecter les résultats
4. Synthétiser dans un rapport

{{% notice warning "Limitations des subagents" %}}
- Les subagents **ne peuvent pas spawner d'autres subagents** (pas de récursion infinie)
- Pas de mode "thinking" interactif dans les subagents
- Maximum **10 agents en parallèle** (les suivants sont mis en queue)
{{% /notice %}}

#### Déclencher manuellement un subagent

Vous pouvez explicitement demander l'utilisation d'un subagent :

```
> Utilise un subagent pour parcourir tous les fichiers dans /src/components
> et vérifier s'ils utilisent les derniers design tokens.
> Donne-moi juste un résumé de ceux qui doivent être corrigés.
```

---

## :hammer_and_wrench: Mise en place et configuration

### Installation

L'installation de Claude Code est simple. Vous avez besoin de Node.js 18+ :

```console
# Installation globale via npm
npm install -g @anthropic-ai/claude-code

# Vérifier l'installation
claude --version
```

Ensuite, configurez votre clé API :

```console
# Via variable d'environnement
export ANTHROPIC_API_KEY="sk-ant-..."

# Ou lors du premier lancement
claude
# Claude vous demandera votre clé
```

{{% notice tip "Avec asdf" %}}
Si vous utilisez [asdf](https://blog.ogenki.io/post/asdf/asdf/) comme moi :
```console
asdf plugin add claude-code
asdf install claude-code latest
asdf global claude-code latest
```
{{% /notice %}}

### Premiers pas

```console
# Lancer une session interactive
claude

# Mode one-shot (une seule commande)
claude -p "Liste les fichiers Terraform dans ce projet"

# Reprendre la dernière session
claude --resume

# Afficher l'aide
claude --help
```

Les actions de bases:
* modes de fonctionnement (permissions): plan, accept edit. Normal mode asking for editing
* multi modal, paste images
* thinking level
* Referencing files
* /resume
* /tasks
* keep feeding, no need to wait for the output for adding information

### Le fichier CLAUDE.md : votre contexte personnalisé

Le fichier `CLAUDE.md` est **crucial**. C'est un fichier de configuration spécial que Claude lit automatiquement au démarrage de chaque session. Il lui permet de comprendre votre projet et de fournir une assistance adaptée.

#### Où le placer ?

| Emplacement | Portée | Cas d'usage |
|-------------|--------|-------------|
| `~/.claude/CLAUDE.md` | Global (toutes les sessions) | Préférences personnelles |
| `./CLAUDE.md` | Projet (racine du repo) | Instructions spécifiques au projet |
| `./CLAUDE.local.md` | Local (gitignore) | Config personnelle non partagée |

#### Que mettre dedans ?

Selon les [best practices Anthropic](https://www.anthropic.com/engineering/claude-code-best-practices), gardez-le **concis et lisible** :

```markdown
# CLAUDE.md

## Projet
Cloud Native Ref - Plateforme Kubernetes de référence sur AWS

## Stack technique
- Infrastructure: OpenTofu + Terramate
- Kubernetes: EKS avec Cilium (sans kube-proxy)
- GitOps: Flux v2
- Observabilité: VictoriaMetrics, VictoriaLogs, Grafana
- Secrets: OpenBao (fork Vault)

## Commandes courantes
# Déployer l'infrastructure
terramate run -- tofu apply

# Vérifier Flux
flux get ks -A

# Valider les compositions KCL
./scripts/validate-kcl-compositions.sh

## Conventions
- Utiliser KCL pour les compositions Crossplane (pas YAML)
- Ne JAMAIS muter un dictionnaire après création en KCL
- Préfixer les branches: feat/, fix/, docs/

## Points d'attention
- Les secrets sensibles sont dans OpenBao, pas dans Git
- Toujours valider avec `terramate run -- tofu plan` avant apply
```

{{% notice info "Initialisation automatique" %}}
La commande `/init` analyse votre projet et génère un `CLAUDE.md` initial. C'est un excellent point de départ que vous pouvez ensuite personnaliser.
```console
claude
> /init
```
{{% /notice %}}

### Les commandes slash

Claude Code dispose de commandes intégrées accessibles via `/` :

| Commande | Description |
|----------|-------------|
| `/help` | Affiche l'aide |
| `/clear` | Efface la conversation (libère le contexte) |
| `/context` | Visualise l'utilisation du contexte |
| `/init` | Initialise CLAUDE.md pour le projet |
| `/permissions` | Gère les permissions des outils |
| `/config` | Accède à la configuration |
| `/doctor` | Diagnostique les problèmes |
| `/cost` | Affiche les coûts de la session |

### Slash commands personnalisées

Vous pouvez créer vos propres commandes slash pour des tâches récurrentes. Un fichier Markdown dans `.claude/commands/` devient une commande invocable.

#### Créer une commande

```markdown
<!-- .claude/commands/audit-terraform.md -->
# Audit de sécurité Terraform

Analyse tous les fichiers Terraform (*.tf) dans le répertoire courant et ses sous-répertoires.

Pour chaque fichier :
1. Vérifie les ressources exposées publiquement
2. Identifie les secrets en dur
3. Vérifie les tags obligatoires (Environment, Owner, Project)
4. Signale les instances sans encryption

Génère un rapport markdown avec :
- Résumé des problèmes par sévérité (Critical, High, Medium, Low)
- Détail par fichier
- Recommandations de correction
```

#### Utiliser la commande

```console
claude
> /project:audit-terraform
```

{{% notice tip "Commandes utiles pour platform engineering" %}}
Quelques idées :
- `/project:flux-status` - Rapport détaillé de l'état Flux
- `/project:cost-review` - Analyse des ressources coûteuses
- `/project:security-scan` - Audit sécurité complet
- `/project:migration-check` - Vérification pré-migration
{{% /notice %}}

---

### Skills : capacités auto-découvertes

Les **Skills** sont différents des slash commands. Ce sont des capacités que Claude **découvre et utilise automatiquement** quand elles sont pertinentes, plutôt que d'être invoquées manuellement.

#### Structure d'un Skill

Un skill est un **dossier** (pas un simple fichier) dans `.claude/skills/` :

```
.claude/skills/k8s-troubleshooter/
├── SKILL.md           # Description pour Claude (obligatoire)
├── common-errors.md   # Base de connaissances
├── runbooks/          # Procédures de diagnostic
│   ├── crashloop.md
│   └── network.md
└── scripts/           # Scripts de support
    └── diagnostic.sh
```

#### Le fichier SKILL.md

C'est la "carte d'identité" du skill. Claude le lit pour décider s'il doit l'activer :

```markdown
# Kubernetes Troubleshooter

## Description
Skill pour diagnostiquer les problèmes Kubernetes courants.
S'active automatiquement lors d'erreurs de pods, services ou déploiements.

## Activation
Ce skill s'active quand :
- L'utilisateur mentionne des erreurs Kubernetes
- Des logs de pods sont analysés
- Des problèmes de réseau sont détectés

## Capabilities
- Analyse des événements Kubernetes
- Diagnostic des CrashLoopBackOff
- Vérification des NetworkPolicies
- Recommandations basées sur les runbooks
```

#### Chargement lazy

Contrairement aux slash commands (chargées immédiatement), les skills utilisent un **chargement lazy** :
1. Seule la description (`SKILL.md`) est lue au démarrage
2. Le contenu complet n'est chargé que si Claude décide d'activer le skill
3. Économie de tokens quand le skill n'est pas pertinent

---

### Skills vs Commands : tableau comparatif

Claude Code propose deux mécanismes d'extension souvent confondus. Voici comment les distinguer :

| Aspect | Slash Commands | Skills |
|--------|----------------|--------|
| **Invocation** | Manuelle (`/project:xxx`) | Automatique (Claude décide) |
| **Emplacement** | `.claude/commands/` | `.claude/skills/` |
| **Structure** | Un fichier Markdown | Dossier avec `SKILL.md` + fichiers |
| **Chargement contexte** | Immédiat à l'invocation | Lazy (description puis contenu) |
| **Complexité** | Tâches simples, répétitives | Workflows complexes, multi-étapes |
| **Contrôle** | Total (vous décidez quand) | Délégué (Claude décide si pertinent) |

#### Quand utiliser quoi ?

| Besoin | Solution |
|--------|----------|
| Raccourci pour tâche répétitive | **Slash Command** |
| Workflow contextuel automatique | **Skill** |
| Accès à une API externe | **MCP** |
| Script de validation | **Hook** |

#### Exemple concret

**Slash Command** - Vous tapez `/project:flux-status` pour obtenir un rapport Flux.

**Skill** - Vous dites "Mon déploiement est bloqué". Claude détecte le contexte Kubernetes, active automatiquement le skill `k8s-troubleshooter`, et utilise les runbooks pour diagnostiquer.

{{% notice info "Coexistence" %}}
Skills et slash commands peuvent coexister. Utilisez des **slash commands** pour les actions que vous voulez déclencher explicitement, et des **skills** pour les capacités que Claude doit appliquer intelligemment selon le contexte.
{{% /notice %}}

### Les Hooks : être notifié quand Claude a terminé

Les **Hooks** permettent d'exécuter des commandes shell à des moments précis du cycle de vie de Claude Code. Le cas d'usage le plus pratique : **être notifié** quand Claude a fini une tâche longue.

#### Configuration de notification sonore

Ajoutez dans `.claude/settings.json` :

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.local/bin/claude-notify.sh"
          }
        ]
      }
    ]
  }
}
```

#### Script de notification (Linux)

```bash
#!/bin/bash
# ~/.local/bin/claude-notify.sh

# Jouer un son
paplay /usr/share/sounds/freedesktop/stereo/complete.oga 2>/dev/null || \
  aplay /usr/share/sounds/sound-icons/finish 2>/dev/null

# Notification desktop
notify-send "Claude Code" "Tâche terminée !" \
  --icon=dialog-information \
  --urgency=normal
```

#### Script de notification (macOS)

```bash
#!/bin/bash
# ~/.local/bin/claude-notify.sh

osascript -e 'display notification "Tâche terminée" with title "Claude Code" sound name "Glass"'
```

N'oubliez pas de rendre le script exécutable :

```console
chmod +x ~/.local/bin/claude-notify.sh
```

{{% notice tip "Ressources pour aller plus loin" %}}
- [claude-sounds](https://daveschumaker.net/claude-sounds-better-notifications-for-claude-code/) - Sons aléatoires pour plus de variété
- [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) - Exemples avancés de hooks

Les hooks permettent aussi le formatage automatique, le blocage d'actions, et bien plus. Consultez la [documentation officielle](https://code.claude.com/docs/en/hooks) pour les cas avancés.
{{% /notice %}}

---

## :rocket: Cas concrets de platform engineering

Assez de théorie ! Passons à ce qui nous intéresse vraiment : comment Claude Code peut nous aider au quotidien. Je vais vous partager deux cas concrets et détaillés qui illustrent la puissance des MCPs et du workflow avec Claude.

### :mag: Cas 1 : Troubleshooting complet d'une Network Policy

Ce cas illustre un workflow de troubleshooting de bout en bout, en utilisant plusieurs MCPs pour identifier et résoudre un problème réseau.

#### Le contexte

Après un déploiement, une application ne peut plus communiquer avec un service dans un autre namespace. Les logs montrent des timeouts mais aucune erreur explicite. Une alerte VictoriaMetrics a été déclenchée.

#### Étape 1 : Identifier les alertes actives

Avec le **MCP VictoriaMetrics**, je demande à Claude d'analyser les alertes :

```
> Utilise le MCP VictoriaMetrics pour lister les alertes actives
> sur le namespace demo. Quelles sont les alertes de type réseau ?
```

Claude interroge l'API et trouve :

```
ALERTES ACTIVES - namespace demo

| Alerte | Sévérité | Depuis | Description |
|--------|----------|--------|-------------|
| HTTPHighLatency | warning | 15min | Latence HTTP > 5s (service backend) |
| CiliumDropsHigh | critical | 12min | Drops réseau élevés (>100/min) |
| EndpointNotReady | warning | 15min | Endpoint postgres non atteignable |
```

#### Étape 2 : Analyser les logs avec VictoriaLogs

Je demande à Claude de corréler avec les logs via le **MCP VictoriaLogs** :

```
> Utilise le MCP VictoriaLogs pour chercher les erreurs de connexion
> dans le namespace demo des 30 dernières minutes.
> Filtre sur "connection refused" ou "timeout".
```

Claude exécute la requête LogsQL et trouve :

```
2026-01-08 14:32:15 [backend-7d4f9...] dial tcp 10.0.45.67:5432: i/o timeout
2026-01-08 14:32:20 [backend-7d4f9...] dial tcp 10.0.45.67:5432: i/o timeout
2026-01-08 14:32:25 [backend-7d4f9...] dial tcp 10.0.45.67:5432: i/o timeout
```

Claude identifie : les timeouts commencent exactement à 14:32, soit **2 minutes après un déploiement** d'une nouvelle NetworkPolicy.

#### Étape 3 : Investigation des NetworkPolicies

```
> Liste les CiliumNetworkPolicy et NetworkPolicy qui affectent
> la communication entre le namespace demo et le namespace database.
```

Claude trouve la policy problématique :

```yaml
# PROBLÈME IDENTIFIÉ
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: postgres-access
  namespace: database
spec:
  endpointSelector:
    matchLabels:
      app: postgres
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: backend
            # ❌ MANQUANT : le selector de namespace !
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

#### Étape 4 : Correction et application

Claude propose la correction :

```yaml {hl_lines=[12]}
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: postgres-access
  namespace: database
spec:
  endpointSelector:
    matchLabels:
      app: postgres
  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: demo  # ✅ Ajouté
            app: backend
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

Je valide et Claude applique la modification :

```console
kubectl apply -f security/base/network-policies/postgres-access.yaml
```

#### Étape 5 : Créer une VMRule de prévention

Pour éviter que ce problème ne se reproduise silencieusement, je demande à Claude de créer une alerte :

```
> Crée une VMRule qui alerte quand Cilium drop du traffic
> vers des services de type database. Utilise cilium_drop_count_total.
```

Claude génère :

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: cilium-database-drops
  namespace: observability
spec:
  groups:
    - name: cilium-network
      rules:
        - alert: CiliumDatabaseConnectionDropped
          expr: |
            sum(rate(cilium_drop_count_total{
              reason="POLICY_DENIED",
              direction="INGRESS"
            }[5m])) by (destination_namespace)
            * on(destination_namespace) group_left
            kube_namespace_labels{label_type="database"} > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Traffic vers database droppé par Cilium"
            description: "Policy réseau bloque le traffic vers {{ $labels.destination_namespace }}"
```

#### Étape 6 : Créer un dashboard Grafana

Avec le **MCP Grafana**, je demande la création d'un dashboard de monitoring :

```
> Utilise le MCP Grafana pour créer un dashboard "Network Policies Monitoring"
> avec :
> - Panel 1: Drops Cilium par namespace (time series)
> - Panel 2: Top 10 policies avec le plus de drops (table)
> - Panel 3: Latence inter-namespace (heatmap)
```

Claude utilise le MCP pour créer le dashboard. Je peux ensuite le tester visuellement.

{{% notice info "Chrome Integration (beta)" %}}
Avec la nouvelle extension Chrome de Claude, vous pouvez demander à Claude de **vérifier visuellement** que le dashboard s'affiche correctement. Claude peut interagir avec votre navigateur pour valider l'affichage.
{{% /notice %}}

#### Résumé du workflow

| Étape | MCP utilisé | Action |
|-------|-------------|--------|
| 1 | VictoriaMetrics | Identifier les alertes |
| 2 | VictoriaLogs | Analyser les logs, corréler |
| 3 | kubectl | Lister les NetworkPolicies |
| 4 | kubectl | Appliquer la correction |
| 5 | — | Créer VMRule prévention |
| 6 | Grafana | Créer dashboard monitoring |

{{% notice tip "Ce qui a fait la différence" %}}
Sans les MCPs, j'aurais dû :
- Ouvrir VMUI manuellement et copier les résultats
- Exécuter des requêtes LogsQL et reformater les logs
- Naviguer dans Grafana pour créer le dashboard

Avec les MCPs, Claude a fait tout cela en **5 minutes** au lieu de 45 minutes.
{{% /notice %}}

---

### :building_construction: Cas 2 : Feature produit avec Spec-Driven Development

Ce cas illustre comment utiliser Claude pour implémenter une vraie feature produit en suivant le framework **Spec-Driven Development** (SDD) du repository [cloud-native-ref](https://github.com/Smana/cloud-native-ref).

#### Le contexte

L'équipe produit veut pouvoir choisir entre **Kafka (Strimzi)** et **AWS SQS** pour le queuing de leurs applications. Actuellement, aucune abstraction n'existe.

#### Le workflow SDD en un coup d'œil

Le SDD de cloud-native-ref utilise un **modèle à deux documents** :

| Document | Rôle | Contenu |
|----------|------|---------|
| **GitHub Issue** | Ancre immuable | Discussion, "quoi/pourquoi" |
| **Spec File** | Design détaillé | Checklist, "comment" |

Le workflow complet : **Specify → Clarify → Tasks → Implement → Validate**

#### Étape 1 : Créer la spec avec `/specify`

La commande `/specify` est le point d'entrée du SDD. Elle crée automatiquement l'issue GitHub et le fichier de spec :

```
> /specify composition "Queue composition for Kafka and SQS backends"
```

Claude exécute la commande qui :
1. Crée une **GitHub Issue** `[SPEC] Queue composition for Kafka and SQS backends`
2. Génère un fichier de spec `docs/specs/active/0002-#42-queue-kafka-sqs.md`
3. Pré-remplit avec le template `spec-crossplane-composition.md`
4. Lit la **constitution** pour les contraintes non-négociables

```
✅ Specification created!

🔗 GitHub Issue: https://github.com/Smana/cloud-native-ref/issues/42
📄 Spec File: docs/specs/active/0002-#42-queue-kafka-sqs.md
📋 Type: composition

## Review Personas
Before implementation, self-review as:
- [ ] PM: Problem clear? User stories valid? Scope defined?
- [ ] Platform Engineer: Patterns consistent? Implementation feasible?
- [ ] Security & Compliance: Zero-trust? Least privilege? Secrets managed?
- [ ] SRE: Observable? Recoverable? Failure modes documented?
```

#### Étape 2 : Compléter la spec et clarifier

Je complète les sections du template, en ajoutant des marqueurs `[NEEDS CLARIFICATION]` pour les points à éclaircir :

```markdown
## User Stories

### US-001: Kafka Queue (P1)
**Given** une application dans le namespace `ecommerce`
**When** je déclare `queue: { enabled: true, type: kafka, size: medium }`
**Then** un cluster Kafka 3 replicas est créé avec credentials injectés

### US-002: SQS Queue (P1)
**Given** une application serverless
**When** je déclare `queue: { type: sqs }`
**Then** une queue SQS est créée avec IAM via EKS Pod Identity

[NEEDS CLARIFICATION: Faut-il supporter les FIFO queues pour SQS ?]
[NEEDS CLARIFICATION: Quelle version de Kafka par défaut ?]
```

Puis j'exécute `/clarify` pour résoudre les incertitudes interactivement :

```
> /clarify
```

Claude trouve les marqueurs et me pose les questions. Après mes réponses :

```markdown
[CLARIFIED: Non, FIFO queues hors scope pour v1 - à ajouter si demandé]
[CLARIFIED: Kafka 3.7.0 par défaut, configurable via spec.queue.kafka.version]
```

#### Étape 3 : Générer le breakdown de tâches (optionnel)

```
> /tasks
```

Claude génère un plan de tâches avec dépendances :

```markdown
## Task Breakdown - SPEC-0002

### Phase 1: Setup
| ID | Task | Dependencies |
|----|------|--------------|
| T1 | Créer le module KCL queue/main.k | - |
| T2 | Définir le XRD xqueues.cloud.ogenki.io | - |

### Phase 2: Implementation [P]
| T3 | Implémenter le backend Kafka (Strimzi) | T1 |
| T4 | Implémenter le backend SQS | T1 |
| T5 | Ajouter CiliumNetworkPolicy | T3, T4 |

### Phase 3: Validation
| T6 | Écrire les exemples (basic + complete) | T3, T4 |
| T7 | Valider avec crossplane render | T2, T6 |
```

#### Étape 4 : Implémenter en suivant la spec

Maintenant Claude peut implémenter en respectant la spec ET la constitution :

```
> Implémente la composition Queue en suivant la spec 0002
> et les règles de la constitution (préfixe xplane-*, pas de mutation)
```

Claude génère le code KCL (extrait) :

```python
# infrastructure/base/crossplane/configuration/kcl/queue/main.k

oxr = option("params").oxr
_name = oxr.metadata.name
_queueType = oxr.spec.queue.type or "kafka"
_size = oxr.spec.queue.size or "small"

# Mapping tailles vers ressources (constitution: pas de mutation)
_sizePresets = {
    kafka = {
        small = { replicas = 1, storage = "10Gi" }
        medium = { replicas = 3, storage = "50Gi" }
    }
    sqs = {
        small = { visibilityTimeout = 30 }
        medium = { visibilityTimeout = 120 }
    }
}

# Constitution: préfixe xplane-* obligatoire
_kafkaCluster = {
    metadata.name = "xplane-${_name}-kafka"
    # ...
}
```

#### Étape 5 : Valider et créer la PR

Claude exécute les validations de la constitution :

```console
kcl fmt infrastructure/base/crossplane/configuration/kcl/queue/
kcl run -Y settings-example.yaml
crossplane render examples/queue-kafka.yaml composition.yaml functions.yaml
```

Puis crée la PR avec référence automatique à la spec :

```
> /create-pr
```

La commande `/create-pr` détecte automatiquement la spec et génère :

```markdown
## Summary
Implements #42

Adds Queue composition supporting Kafka (Strimzi) and AWS SQS backends.

## Changes
- New KCL module: `queue/main.k`
- XRD: `xqueues.cloud.ogenki.io`
- Examples: basic and complete usage
```

#### Résultat : l'API utilisateur finale

Le développeur peut maintenant simplement déclarer :

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: Queue
metadata:
  name: orders-queue
  namespace: ecommerce
spec:
  queue:
    type: kafka
    size: medium
```

#### Résumé du workflow SDD

| Commande | Action | Résultat |
|----------|--------|----------|
| `/specify composition` | Créer spec | Issue GitHub + fichier spec |
| `/clarify` | Résoudre incertitudes | Marqueurs [CLARIFIED] |
| `/tasks` | Planifier | Breakdown avec dépendances |
| (implémentation) | Coder selon spec | Code conforme à la constitution |
| `/create-pr` | Créer PR | Lien automatique "Implements #XX" |

{{% notice info "Pourquoi le modèle à deux documents ?" %}}
- **GitHub Issue** : Discoverabilité, discussion, mentions, ancre immuable
- **Spec File** : Design détaillé, checklists des 4 personas (PM, PE, Security, SRE)
- **Après merge** : Spec archivée dans `docs/specs/completed/` pour référence
{{% /notice %}}

---

## :bulb: Tips pour optimiser son usage

### Git Worktrees : paralléliser les sessions Claude

Plutôt que de jongler avec des branches et du `stash`, utilisez les **git worktrees** pour travailler sur plusieurs features en parallèle avec des sessions Claude indépendantes.

```console
# Créer des worktrees pour deux features
git worktree add ../worktrees/feature-a -b feat/feature-a
git worktree add ../worktrees/feature-b -b feat/feature-b

# Lancer des sessions Claude séparées
cd ../worktrees/feature-a && claude  # Terminal 1
cd ../worktrees/feature-b && claude  # Terminal 2
```

**Pourquoi c'est puissant :**
- Chaque session a son **propre contexte** et sa propre mémoire
- Aucune interférence entre les tâches
- Merge indépendant quand c'est prêt

| Commande | Usage |
|----------|-------|
| `git worktree list` | Voir tous les worktrees actifs |
| `git worktree add <path> -b <branch>` | Créer un worktree + branche |
| `git worktree remove <path>` | Supprimer un worktree |

---

### Ma recommandation : le workflow hybride Cursor + Claude Code

Plutôt que de choisir entre IDE et terminal, combinez les deux. C'est l'approche que je recommande : **Cursor** (même en version gratuite) pour l'édition quotidienne, **Claude Code** pour les tâches agentiques.

| Besoin | Outil | Pourquoi |
|--------|-------|----------|
| Édition rapide, autocomplete | Cursor | Latence minimale, vous restez dans le flow |
| Refactoring, debugging multi-fichiers | Claude Code | Raisonnement profond, boucles autonomes |

**Les vrais gains de ce setup :**

- **Review visuelle des changements** : Claude modifie via le terminal, vous validez les diffs dans l'interface Cursor — bien plus lisible que `git diff`
- **Contexte enrichi** : L'indexation locale de Cursor complète le contexte de Claude
- **Coût maîtrisé** : Un seul abonnement (Anthropic) suffit pour la puissance agentique

{{% notice tip "En pratique" %}}
Je lance Cursor pour naviguer et éditer. Quand une tâche devient complexe (refactoring, debugging, génération de code), je bascule sur Claude Code dans le terminal intégré. Les modifications apparaissent en temps réel dans Cursor.
{{% /notice %}}

---

### Optimiser le contexte

Le contexte est précieux (et coûteux). Voici comment l'économiser :

**1. Utilisez `/clear` régulièrement**

Chaque nouvelle tâche devrait commencer par un `/clear`. Vous n'avez pas besoin de l'historique d'une conversation de debugging pour écrire de la documentation.

**2. Gardez CLAUDE.md concis**

Chaque token dans `CLAUDE.md` est chargé à **chaque conversation**. Gardez-le minimal et pertinent.

```markdown
# BON - concis et actionnable
## Commandes
make test, make lint

## Conventions
- Branches: feat/, fix/, docs/
- Pas de mutation KCL
```

**3. Préférez les CLIs aux MCPs quand possible**

Pour Kubernetes, `kubectl` consomme moins de tokens que le MCP Kubernetes. Claude connaît déjà parfaitement les CLIs.

**4. Utilisez `/context` pour auditer**

```console
/context
```

Cette commande montre ce qui consomme votre contexte. Désactivez les MCPs non utilisés pour la tâche en cours.

---

### Écrire du code "AI-readable"

Burke Holland, dans son article ["Opus 4.5 is going to change everything"](https://burkeholland.github.io/posts/opus-4-5-change-everything/), propose un paradigme intéressant : optimiser le code pour la **lisibilité par l'IA**, pas seulement humaine.

**Principes :**

| Principe | Explication |
|----------|-------------|
| **Noms descriptifs** | Variables et fonctions auto-documentées |
| **Flux linéaire** | Éviter les abstractions complexes |
| **Couplage minimal** | Permettre la régénération de fichiers entiers |
| **Structure prévisible** | Grouper par features, pas par types |

Cette approche facilite le travail de l'agent car il peut régénérer des portions de code sans casser le reste du système.

---

### Ce qui fonctionne bien vs ce qui nécessite vigilance

| ✅ Claude excelle | ⚠️ Vigilance requise |
|-------------------|----------------------|
| Debugging avec contexte | Création from scratch |
| Conversion de formats | Sécurité/PKI |
| Refactoring répétitif | Ressources cloud coûteuses |
| Analyse de dépendances | Breaking changes |

---

### Mon workflow type

```
1. Décrire le problème clairement (contexte, erreur, objectif)
2. Pointer vers les fichiers pertinents
3. Demander une analyse AVANT une solution
4. Valider la proposition
5. Appliquer par étapes (pas tout d'un coup)
6. Tester et itérer
```

{{% notice warning "Ne pas oublier" %}}
Claude est un **outil**, pas un remplaçant. J'ai appris autant en lisant ses explications qu'en appliquant ses solutions. Si vous faites juste copier-coller sans comprendre, vous perdez l'opportunité d'apprendre et vous risquez d'introduire des erreurs.
{{% /notice %}}

---

## :moneybag: Coûts et optimisation

Parlons argent. Claude Code peut vite devenir coûteux si on ne fait pas attention. Voici comment maîtriser ses dépenses.

### Comprendre le pricing

#### Les plans disponibles

| Plan | Prix | Inclut | Idéal pour |
|------|------|--------|------------|
| **Free** | 0€ | Messages limités (variable) | Découverte |
| **Pro** | 20$/mois | Usage standard | Usage personnel régulier |
| **Max 5x** | 100$/mois | 5× l'usage Pro | Power users |
| **Max 20x** | 200$/mois | 20× l'usage Pro | Usage intensif |
| **Team** | 30$/user/mois | Collaboration, admin | Petites équipes |
| **Enterprise** | Sur devis | SSO, audit, compliance | Grandes organisations |

{{% notice info "Claude Code inclus !" %}}
Depuis 2025, Claude Code est [inclus dans les plans Team et Enterprise](https://devops.com/enterprise-ai-development-gets-a-major-upgrade-claude-code-now-bundled-with-team-and-enterprise-plans/). Vous n'avez plus à choisir entre innovation et gouvernance.
{{% /notice %}}

#### Coût API (pay-per-use)

Si vous utilisez l'API directement (hors forfait) :

| Modèle | Input (par M tokens) | Output (par M tokens) |
|--------|---------------------|----------------------|
| **Opus 4.5** | 5.00$ | 25.00$ |
| **Sonnet 4.5** | 3.00$ | 15.00$ |
| **Haiku** | 1.00$ | 5.00$ |

**Prompt Caching** (réduction significative) :
- Cache read : **0.1×** le prix input (90% de réduction !)
- Cache write (5 min) : 1.25× le prix input
- Cache write (1 heure) : 2× le prix input

### Choisir le bon modèle selon la tâche

C'est **LA** clé pour optimiser ses coûts. Tous les modèles ne sont pas égaux, et Opus n'est pas toujours nécessaire.

| Tâche | Modèle recommandé | Pourquoi |
|-------|-------------------|----------|
| Debugging complexe (ex: Cilium) | **Opus 4.5** | Analyse profonde, corrélation multi-sources |
| Refactoring multi-fichiers | **Opus 4.5** | Garde le contexte complet |
| Génération Terraform/YAML | **Sonnet 4.5** | Suffisamment précis, 40% moins cher |
| Commits, petites éditions | **Haiku** | Rapide et économique |
| Lecture/synthèse de docs | **Sonnet 4.5** | Bon compromis |
| Résolution de problèmes nouveaux | **Opus 4.5** | Meilleur raisonnement |

```console
# Changer de modèle en cours de session
/model sonnet

# Ou au lancement
claude --model haiku
```

{{% notice tip "Le conseil de Boris Cherny (créateur de Claude Code)" %}}
*"J'utilise Opus 4.5 avec thinking pour tout. C'est le meilleur modèle de code que j'ai utilisé, et même s'il est plus gros et plus lent que Sonnet, comme vous avez moins besoin de le guider et qu'il est meilleur en tool use, il est presque toujours plus rapide au final."*

Mon avis : c'est vrai pour les tâches complexes, mais pour les tâches simples, Haiku reste plus économique.
{{% /notice %}}

### Surveiller sa consommation

#### Pendant la session

```console
# Voir le coût de la session actuelle
/cost

# Voir l'utilisation du contexte
/context
```

#### Historique

```console
# Statistiques d'utilisation
/stats
```

### Optimisations pratiques

#### 1. Utiliser `/clear` souvent

Chaque nouvelle tâche devrait commencer par un `/clear`. Vous n'avez pas besoin de l'historique d'une conversation de debugging pour écrire de la documentation.

```console
> /clear
Contexte effacé. Nouvelle conversation.
```

#### 2. Préférer les CLIs aux MCPs quand possible

Les MCPs ajoutent des définitions d'outils au contexte. Pour Kubernetes par exemple :

```console
# Via MCP Kubernetes (consomme des tokens pour les définitions)
> utilise le MCP k8s pour lister les pods

# Via kubectl (Claude connaît déjà la CLI)
> exécute kubectl get pods -n production
```

#### 3. Désactiver l'auto-compact si possible

L'auto-compaction résume automatiquement les conversations longues, mais elle consomme des tokens. Pour des sessions courtes et ciblées, vous pouvez la désactiver :

```console
claude --no-auto-compact
```

#### 4. Fichiers CLAUDE.md concis

Chaque token dans `CLAUDE.md` est chargé à **chaque conversation**. Gardez-le minimal et pertinent.

#### 5. Utiliser les worktrees pour paralléliser

Au lieu d'une longue session avec beaucoup de context switching, utilisez des worktrees Git pour des sessions parallèles et ciblées :

```console
# Créer des worktrees séparés
git worktree add ../wt-feature-a -b feat/feature-a
git worktree add ../wt-feature-b -b feat/feature-b

# Sessions Claude indépendantes
cd ../wt-feature-a && claude  # Session 1
cd ../wt-feature-b && claude  # Session 2
```

### Rate Limits

Depuis août 2025, Anthropic applique des **limites hebdomadaires** pour les utilisateurs intensifs :

- Affecte moins de 5% des utilisateurs
- Cible principalement l'utilisation 24/7 continue
- Les abonnés Max peuvent acheter de l'usage supplémentaire au tarif API

{{% notice warning "Si vous atteignez les limites" %}}
- Utilisez Haiku pour les tâches simples
- Évitez de laisser Claude "tourner" sans supervision
- Répartissez votre usage sur la semaine
{{% /notice %}}

---

## :thought_balloon: Réflexions et mises en garde

C'est le moment d'être honnête. Claude Code est un outil formidable, mais il n'est pas parfait. Voici mes retours après plusieurs mois d'utilisation intensive.

### Éviter la dépendance : continuer à apprendre

#### Le risque d'atrophie des compétences

C'est peut-être le point le plus important de cet article. Une [étude de Stanford](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/) a révélé que l'emploi des développeurs de 22-25 ans a chuté de **20%** entre 2022 et 2025, coïncidant avec l'essor des outils d'IA.

Les risques identifiés :
- **Capacité de résolution de problèmes** qui s'atrophie quand on ne l'exerce plus
- Difficulté à travailler **sans assistance IA**
- Transfert de connaissances aux juniors **compromis** quand les seniors délèguent tout

{{% notice warning "Le paradoxe de la productivité" %}}
Une [étude METR](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) a trouvé un résultat surprenant : les développeurs utilisant l'IA prennent **19% plus de temps** pour compléter des tâches ! Pourtant, ils *croient* avoir été plus rapides de 24%. L'écart entre perception et réalité est frappant.
{{% /notice %}}

#### Comment maintenir ses compétences ?

1. **Reviewer systématiquement** le code généré
   - Ne pas juste accepter aveuglément
   - Comprendre *pourquoi* cette solution
   - Être capable de la reproduire manuellement

2. **Se forcer à des sessions "sans IA"**
   - Une fois par semaine, débugger à l'ancienne
   - Pratiquer la lecture de code sans assistance

3. **Enseigner aux autres**
   - Expliquer le code généré force à le comprendre
   - Le pair programming reste essentiel

```
Mon workflow de review :
1. Claude génère une solution
2. Je lis TOUT le code modifié
3. Je me demande : "Aurais-je fait pareil ?"
4. Si non, pourquoi ? Qu'est-ce que j'apprends ?
5. Seulement alors, je valide
```

### Qualité du code et dette technique

Les chiffres sont parlants. Selon une [étude Qodo](https://www.qodo.ai/reports/state-of-ai-code-quality/) sur la qualité du code IA :

| Métrique | Code IA | Code humain |
|----------|---------|-------------|
| Problèmes par PR | 10.83 | 6.45 |
| Code dupliqué | **4× plus** | Baseline |
| Top frustration dev | Dette technique (62.4%) | — |

{{% notice info "Mon observation personnelle" %}}
Claude est excellent pour le **premier jet**, mais il a tendance à :
- Sur-ingénier les solutions simples
- Ajouter du code défensif inutile
- Créer des abstractions prématurées

Mon conseil : demandez explicitement la solution la plus simple possible.
{{% /notice %}}

### Hallucinations et contexte manquant

Malgré les progrès, les hallucinations persistent :
- **25%** des développeurs estiment qu'1 suggestion sur 5 contient des erreurs
- **65%** signalent que l'assistant "manque du contexte pertinent" pour le refactoring

**Cas typiques :**
- Référence à des packages qui n'existent pas
- API deprecated ou incorrecte
- Configuration incompatible avec votre version

**Comment mitiger :**
```console
# Toujours spécifier les versions
> Utilise Terraform 1.11 et le provider AWS 6.x

# Demander les sources
> Montre-moi la doc officielle qui confirme cette approche

# Utiliser context7 pour la doc à jour
> use context7 pour la doc Cilium 1.15
```

### Sécurité et confidentialité

#### Ce qui est envoyé aux serveurs

Soyons clairs sur ce que Claude Code envoie :

| Données | Envoyées ? | Notes |
|---------|------------|-------|
| Fichiers lus explicitement | Oui | Uniquement ceux que Claude lit |
| Autres fichiers du projet | Non | Restent locaux |
| Variables d'environnement | Non | Sauf si vous les partagez |
| Historique bash | Non | — |

#### Plan Enterprise vs API

| Aspect | API Standard | Enterprise |
|--------|--------------|------------|
| Rétention données | 7 jours | Configurable |
| Training sur vos données | Non | Non |
| Zero Data Retention | Option payante | Disponible |
| SSO / Audit logs | Non | Oui |
| Compliance SOC2 | Oui | Oui |

{{% notice tip "Pour les entreprises" %}}
Si vous travaillez sur du code sensible :
- Utilisez le plan **Enterprise** ou **Team**
- Demandez l'addendum **Zero-Data-Retention** (ZDR)
- Configurez des hooks pour bloquer l'envoi de fichiers sensibles
- N'utilisez **jamais** le plan Free/Consumer pour du code propriétaire
{{% /notice %}}

### Les sceptiques ont-ils tort ?

J'entends souvent des collègues dire : *"Je ne fais pas confiance à l'IA pour coder"*. Ont-ils tort ?

**Arguments des sceptiques :**
- "Je préfère comprendre mon code"
- "L'IA ne connaît pas mon contexte métier"
- "Les juniors ne vont plus apprendre"
- "C'est juste du hype"

**Ma réponse nuancée :**

Ces préoccupations sont **légitimes**. L'IA n'est pas une solution magique. Mais refuser d'utiliser ces outils, c'est comme refuser d'utiliser un IDE parce que "vim suffit".

La vraie question n'est pas *"Faut-il utiliser l'IA ?"* mais *"Comment l'utiliser intelligemment ?"*

| Utilisation | Risque | Bénéfice |
|-------------|--------|----------|
| Copier-coller aveugle | Élevé | Faible |
| Délégation avec review | Modéré | Élevé |
| Collaboration (pair programming IA) | Faible | Très élevé |

{{% notice info "Le point de vue de Burke Holland" %}}
Dans son article ["Opus 4.5 is going to change everything"](https://burkeholland.github.io/posts/opus-4-5-change-everything/), Burke Holland — qui était lui-même sceptique — admet que *"les agents IA peuvent absolument remplacer les développeurs"* pour certaines tâches. Mais il nuance immédiatement : son approche fonctionne *"50% du temps"* selon ses propres termes.

L'IA est un **amplificateur**, pas un remplacement. Un développeur médiocre avec l'IA restera médiocre. Un excellent développeur avec l'IA sera encore meilleur.
{{% /notice %}}

### Mes règles personnelles

Après plusieurs mois d'utilisation, voici les règles que je me suis fixées :

1. **Jamais de merge sans review manuelle**
   - Même si Claude dit que "ça marche"
   - Même si les tests passent

2. **Toujours comprendre avant d'appliquer**
   - Si je ne comprends pas, je ne merge pas
   - Je demande à Claude d'expliquer

3. **Une heure de "no-AI" par jour**
   - Pour garder la main
   - Pour rester capable de travailler sans

4. **Vérifier les claims techniques**
   - Consulter la doc officielle
   - Tester dans un environnement isolé

5. **Documenter ce que Claude a fait**
   - Pour les collègues
   - Pour moi-même dans 6 mois

---

## :dart: Conclusion

Au terme de cet article, j'espère vous avoir convaincu que les coding agents comme Claude Code ne sont pas un gadget, mais un **véritable changement de paradigme** dans notre façon de travailler.

### Ce que j'ai appris

| Aspect | Avant Claude Code | Avec Claude Code |
|--------|-------------------|------------------|
| Debugging Cilium | 2h de lecture de logs | 15 min avec contexte |
| Refactoring Terraform | Journée entière | 2h avec review |
| Écriture de doc | Procrastination | Généré + relu = 30 min |
| Onboarding nouveau repo | Plusieurs jours | Quelques heures |

### Les clés du succès

1. **Investir dans le contexte** : Un bon `CLAUDE.md` fait toute la différence
2. **Choisir le bon modèle** : Opus pour le complexe, Haiku pour le simple
3. **Maintenir l'esprit critique** : Review systématique, jamais d'acceptation aveugle
4. **Continuer à apprendre** : L'IA augmente, elle ne remplace pas

### Et pour la suite ?

Le domaine évolue à une vitesse folle. Dans 6 mois, cet article sera probablement en partie obsolète. Les MCPs vont se multiplier, les modèles vont s'améliorer, et de nouvelles pratiques vont émerger.

Mon conseil : **expérimentez maintenant**. Même si vous êtes sceptique, prenez une heure pour tester. Vous pourriez être surpris.

{{% notice info "Rejoignez la discussion" %}}
J'aimerais beaucoup avoir vos retours d'expérience ! N'hésitez pas à me contacter ou à ouvrir une issue sur le [repo cloud-native-ref](https://github.com/Smana/cloud-native-ref).
{{% /notice %}}

---

## :bookmark: Références

### Documentation officielle
- [Claude Code Documentation](https://code.claude.com/docs/)
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) - Anthropic Engineering
- [Model Context Protocol](https://modelcontextprotocol.io/) - Spécification officielle
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks) - Documentation des hooks
- [Claude Code Skills](https://code.claude.com/docs/en/skills) - Documentation des skills
- [Claude Pricing](https://claude.com/pricing) - Tarification à jour

### Benchmarks et comparatifs
- [SWE-bench Leaderboards](https://www.swebench.com/) - Benchmark de référence
- [SWE-bench Pro](https://scale.com/leaderboard/swe_bench_pro_public) - Benchmark plus difficile (Scale AI)
- [Artificial Analysis - Coding Agents](https://artificialanalysis.ai/insights/coding-agents-comparison) - Comparatif des agents
- [State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/) - Qodo

### MCPs pour Platform Engineering
- [Context7 MCP](https://github.com/upstash/context7) - Documentation à jour pour LLMs (Upstash)
- [Flux MCP Server](https://fluxcd.control-plane.io/mcp/) - GitOps et Flux (ControlPlane)
- [VictoriaMetrics MCP](https://github.com/VictoriaMetrics-Community/mcp-victoriametrics) - Métriques PromQL
- [VictoriaLogs MCP](https://github.com/VictoriaMetrics-Community/mcp-victorialogs) - Logs LogsQL
- [Grafana MCP](https://github.com/grafana/mcp-grafana) - Dashboards et alertes
- [Steampipe MCP](https://github.com/turbot/steampipe-mcp) - Requêtes SQL cloud (Turbot)
- [Awesome DevOps MCP Servers](https://github.com/rohitg00/awesome-devops-mcp-servers) - Liste curatée

### Articles et études
- [Opus 4.5 is going to change everything](https://burkeholland.github.io/posts/opus-4-5-change-everything/) - Burke Holland
- [AI Coding Is Everywhere](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/) - MIT Technology Review
- [METR Study on AI Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) - Étude sur la productivité
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) - Anthropic Research
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) - Guide complet

### Outils et ressources
- [Cloud Native Ref](https://github.com/Smana/cloud-native-ref) - Mon repo de référence
- [Claude Code Hooks Mastery](https://github.com/disler/claude-code-hooks-mastery) - Exemples de hooks
- [claude-sounds](https://daveschumaker.net/claude-sounds-better-notifications-for-claude-code/) - Notifications sonores
- [ClaudeLog](https://claudelog.com/) - Guides et tutoriels communautaires
