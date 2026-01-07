+++
author = "Smaine Kahlouch"
title = "Claude Code : Quand l'IA devient le copilote du Platform Engineer"
date = "2026-01-15"
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

**Scores actuels (janvier 2026) :**

| Modèle | SWE-bench Verified | Éditeur |
|--------|-------------------|---------|
| **Claude Opus 4.5** | 80.9% | Anthropic |
| GPT-5.2 | 80.0% | OpenAI |
| Gemini 3 Flash | 78.0% | Google |
| Claude Sonnet 4.5 | 72.4% | Anthropic |
| Gemini 2.5 Pro | 63.8% | Google |

{{% notice warning "Ces chiffres évoluent vite !" %}}
Au moment où vous lirez cet article, ces scores auront probablement changé. La compétition est féroce ! Consultez [swebench.com](https://www.swebench.com/) pour les derniers résultats.
{{% /notice %}}

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

Voici ceux que j'utilise au quotidien :

| MCP | Usage | Pourquoi l'utiliser |
|-----|-------|---------------------|
| **[context7](https://github.com/upstash/context7)** | Documentation à jour | Injecte la doc des libs directement dans le contexte |
| **[victoriametrics](https://github.com/VictoriaMetrics/mcp-victoriametrics)** | Requêtes métriques | Analyse directe depuis Claude |
| **[github](https://github.com/anthropics/claude-code/tree/main/packages/github-mcp-server)** | Interactions Git | Issues, PRs, reviews |

{{% notice tip "Mon conseil sur Kubernetes" %}}
Pour Kubernetes, j'utilise directement `kubectl` via les commandes bash plutôt qu'un MCP dédié. La CLI consomme **moins de tokens** et Claude la maîtrise parfaitement. Les MCPs Kubernetes existants ([kubectl-mcp](https://github.com/rohitg00/awesome-devops-mcp-servers)) sont utiles, mais ajoutent de la complexité.
{{% /notice %}}

#### Installation d'un MCP

```console
# Ajouter context7 pour la documentation
claude mcp add context7 -- npx -y @upstash/context7-mcp

# Lister les MCPs configurés
claude mcp list

# Supprimer un MCP
claude mcp remove context7
```

:warning: **Attention** : Chaque MCP ajouté consomme des tokens pour ses définitions d'outils. Selon [Anthropic](https://www.anthropic.com/engineering/claude-code-best-practices), utilisez `/context` pour identifier la consommation et désactivez les MCPs non nécessaires pour la tâche en cours.

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

### Skills : commandes personnalisées

Les **Skills** sont des commandes personnalisées que vous pouvez créer pour des tâches récurrentes.

#### Créer un skill

Créez un fichier markdown dans `.claude/commands/` :

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

#### Utiliser le skill

```console
claude
> /project:audit-terraform
```

{{% notice tip "Skills pour platform engineering" %}}
Quelques idées de skills utiles :
- `/project:flux-status` - Rapport détaillé de l'état Flux
- `/project:cost-review` - Analyse des ressources coûteuses
- `/project:security-scan` - Audit sécurité complet
- `/project:migration-check` - Vérification pré-migration
{{% /notice %}}

### Les Hooks : automatisation avancée

Les **Hooks** (introduits en juin 2025) permettent d'exécuter des commandes shell à des moments précis du cycle de vie de Claude Code.

#### Types de hooks disponibles

| Hook | Déclencheur | Cas d'usage |
|------|-------------|-------------|
| `PreToolUse` | Avant l'exécution d'un outil | Validation, blocage |
| `PostToolUse` | Après l'exécution d'un outil | Formatage, logging |
| `UserPromptSubmit` | Avant traitement du prompt | Enrichissement |
| `SessionStart` | Début de session | Chargement de contexte |
| `SessionEnd` | Fin de session | Nettoyage, stats |

#### Exemple : Auto-format après édition

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true"
      }
    ]
  }
}
```

#### Exemple : Bloquer les commits sans review

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git commit:*)",
        "command": "echo 'Avez-vous reviewé les changements ?' && exit 2"
      }
    ]
  }
}
```

{{% notice warning "Codes de sortie des hooks" %}}
- `0` : OK, continuer
- `2` : Bloquer l'action (PreToolUse uniquement), le message stderr est montré à Claude
- Autre : Erreur non bloquante, affichée à l'utilisateur
{{% /notice %}}

---

## :rocket: Cas concrets de platform engineering

Assez de théorie ! Passons à ce qui nous intéresse vraiment : comment Claude Code peut nous aider au quotidien. Je vais vous partager des situations réelles que j'ai rencontrées et comment l'agent m'a fait gagner un temps précieux.

### :mag: Cas 1 : Debugging d'un problème de connectivité Cilium

#### Le contexte

Situation classique : après un déploiement, des pods n'arrivent plus à communiquer entre eux. Les logs applicatifs indiquent des timeouts, mais rien de plus. Où commencer ?

#### Mon approche avec Claude Code

```console
claude
```

```
> J'ai des pods dans le namespace demo qui n'arrivent pas à joindre
> le service postgres dans le namespace database.
> Les logs montrent "connection timed out".
> Peux-tu m'aider à diagnostiquer ?
```

Claude va alors :
1. Lister les `CiliumNetworkPolicy` et `NetworkPolicy` en place
2. Vérifier les endpoints Cilium avec `cilium endpoint list`
3. Analyser les flows avec Hubble si disponible
4. Identifier les règles manquantes ou mal configurées

#### Ce que Claude a trouvé

```yaml
# Claude a identifié que la NetworkPolicy était trop restrictive
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
            # Manquant : io.kubernetes.pod.namespace: demo
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

Claude m'a proposé la correction :

```yaml {hl_lines=[12,13]}
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
            k8s:io.kubernetes.pod.namespace: demo
            app: backend
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

{{% notice info "Ce qui m'a fait gagner du temps" %}}
Sans Claude, j'aurais passé 30 minutes à éplucher les policies une par une, tester avec `kubectl exec`, et relire la doc Cilium sur les labels cross-namespace. Là, en 2 minutes, le problème était identifié et corrigé.
{{% /notice %}}

---

### :wrench: Cas 2 : Refactoring de compositions Crossplane en KCL

#### Le contexte

J'utilise Crossplane avec des compositions écrites en KCL pour abstraire la création de ressources AWS. Le problème ? Certaines compositions sont devenues difficiles à maintenir et génèrent des erreurs obscures.

#### La situation initiale

Une composition RDS qui crashait avec une erreur peu explicite :

```console
cannot compose resources: cannot render composed resource "securitygroup":
invalid function result: duplicate entry for managed resource
```

#### Mon prompt à Claude

```
> J'ai une composition Crossplane KCL pour RDS qui génère une erreur
> de "duplicate entry". Voici le fichier :
> infrastructure/crossplane/configuration/kcl/rds-composition.k
>
> Analyse le code et identifie le problème.
> Rappel : on ne doit pas muter les dictionnaires après création en KCL.
```

#### L'analyse de Claude

Claude a immédiatement identifié le pattern problématique :

```python
# MAUVAIS - Mutation post-création (bug connu KCL #285)
_sg = securitygroup.SecurityGroup {
    metadata.name = "rds-sg-${appName}"
}
_sg.metadata.annotations["crossplane.io/external-name"] = externalName  # BOOM!
```

Et proposé la correction :

```python {hl_lines=[3,4,5,6]}
# BON - Définition complète à la création
_sg = securitygroup.SecurityGroup {
    metadata = {
        name = "rds-sg-${appName}"
        annotations = {
            "crossplane.io/external-name" = externalName
        }
    }
    spec = {
        # ... reste de la spec
    }
}
```

#### Bonus : validation automatique

Claude m'a aussi rappelé l'existence du script de validation :

```console
./scripts/validate-kcl-compositions.sh
```

{{% notice warning "Attention aux mutations KCL" %}}
C'est un piège classique quand on vient de Python ou d'autres langages. En KCL, une fois un objet créé, **on ne le modifie plus**. Si vous devez ajouter des annotations conditionnelles, utilisez les expressions inline :

```python
annotations = {
    if _ready: "krm.kcl.dev/ready" = "True"
    "crossplane.io/external-name" = externalName
}
```
{{% /notice %}}

---

### :bar_chart: Cas 3 : Analyse de métriques avec VictoriaMetrics

#### Le contexte

Une alerte VictoriaMetrics indique une cardinalité explosive sur le cluster. Il faut identifier rapidement quelles métriques sont responsables avant que le stockage ne sature.

#### Investigation avec Claude et le MCP VictoriaMetrics

J'utilise le MCP VictoriaMetrics pour permettre à Claude d'interroger directement VMUI :

```console
claude mcp add victoriametrics -- npx @anthropic-ai/mcp-victoriametrics
```

Puis :

```
> La cardinalité a explosé ces dernières 24h.
> Utilise le MCP VictoriaMetrics pour :
> 1. Identifier les métriques avec le plus de séries temporelles
> 2. Trouver les labels qui causent l'explosion
> 3. Proposer des relabel configs pour VMAgent
```

#### Ce que Claude a découvert

```
Top métriques par cardinalité :

| Métrique                          | Séries   | Problème identifié        |
|-----------------------------------|----------|---------------------------|
| http_requests_total               | 847,293  | Label "request_id" unique |
| kubernetes_pod_info               | 234,891  | Churn rate pods élevé     |
| custom_business_metric            | 156,234  | Timestamp dans label      |
```

#### Solution proposée par Claude

```yaml
# VMAgent relabel config pour supprimer les labels problématiques
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAgent
metadata:
  name: vmagent
spec:
  # ...
  inlineScrapeConfig: |
    - job_name: 'problematic-app'
      relabel_configs:
        # Supprimer request_id qui est unique par requête
        - action: labeldrop
          regex: 'request_id'
        # Supprimer tout label contenant un timestamp
        - action: labeldrop
          regex: '.*timestamp.*'
      metric_relabel_configs:
        # Aggréger au lieu de garder chaque instance
        - source_labels: [__name__]
          regex: 'http_requests_total'
          action: keep
```

{{% notice note "L'importance du contexte MCP" %}}
Sans le MCP VictoriaMetrics, Claude aurait dû me demander d'exécuter les requêtes manuellement. Avec le MCP, il interroge directement l'API et me donne des résultats en temps réel. C'est **la** différence entre un chatbot et un agent.
{{% /notice %}}

---

### :memo: Cas 4 : Génération de VMRules depuis des PrometheusRules

#### Le contexte

Migration de Prometheus vers VictoriaMetrics : j'ai 47 fichiers PrometheusRule à convertir en VMRule. Le format est similaire mais pas identique.

#### Le prompt

```
> J'ai des PrometheusRules dans observability/base/prometheus-rules/
> Je veux les convertir en VMRules pour VictoriaMetrics.
>
> Pour chaque fichier :
> 1. Convertis le format (apiVersion, kind)
> 2. Adapte les annotations si nécessaire
> 3. Crée le fichier correspondant dans observability/base/vmrules/
>
> Commence par me montrer un exemple avant de tout convertir.
```

#### Résultat

Claude a d'abord montré la différence :

```yaml
# AVANT - PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-alerts
spec:
  groups:
    - name: node
      rules:
        - alert: NodeHighCPU
          expr: node_cpu_seconds_total > 80
          for: 5m
          annotations:
            summary: "High CPU on {{ $labels.instance }}"
```

```yaml
# APRÈS - VMRule
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: node-alerts
spec:
  groups:
    - name: node
      rules:
        - alert: NodeHighCPU
          expr: node_cpu_seconds_total > 80
          for: 5m
          annotations:
            summary: "High CPU on {{ $labels.instance }}"
```

Après validation, Claude a converti les 47 fichiers en quelques minutes, en conservant la structure et en adaptant uniquement ce qui devait l'être.

---

### :evergreen_tree: Cas 5 : Travail parallèle avec les Git Worktrees

#### Le contexte

Je dois travailler sur deux features en parallèle :
- Mise à jour de Cilium vers 1.15
- Ajout d'un nouveau cluster de test

Plutôt que de jongler avec des branches et du `stash`, j'utilise les worktrees.

#### Mise en place

```console
# Depuis le repo principal
cd ~/Sources/cloud-native-ref

# Créer les worktrees
git worktree add ../worktrees/cilium-upgrade -b feat/cilium-1.15
git worktree add ../worktrees/test-cluster -b feat/test-cluster
```

#### Sessions Claude parallèles

Terminal 1 :
```console
cd ../worktrees/cilium-upgrade
claude

> Je veux mettre à jour Cilium de 1.14.2 vers 1.15.
> Analyse les breaking changes et prépare la migration.
```

Terminal 2 :
```console
cd ../worktrees/test-cluster
claude

> Crée la configuration Flux pour un nouveau cluster "test-cluster-0"
> basé sur le pattern de mycluster-0 mais avec des ressources réduites.
```

{{% notice tip "Pourquoi c'est puissant" %}}
Chaque session Claude a son propre contexte, sa propre mémoire. Pendant que l'un analyse les breaking changes Cilium, l'autre génère les manifests Flux. **Aucune interférence**, et je peux merger indépendamment quand c'est prêt.
{{% /notice %}}

#### Nettoyage

```console
# Une fois la feature mergée
git worktree remove ../worktrees/cilium-upgrade
git branch -d feat/cilium-1.15
```

| Commande | Usage |
|----------|-------|
| `git worktree list` | Voir tous les worktrees actifs |
| `git worktree add <path> -b <branch>` | Créer un worktree + branche |
| `git worktree remove <path>` | Supprimer un worktree |
| `git worktree prune` | Nettoyer les références obsolètes |

---

### :arrows_counterclockwise: Cas 6 : Dépannage d'une chaîne de dépendances Flux

#### Le contexte

Après un `flux reconcile`, certaines Kustomizations restent bloquées en `Not Ready` avec des messages de dépendances non satisfaites.

#### Le prompt

```
> Mes Kustomizations Flux sont bloquées. Voici la sortie de flux get ks :
>
> NAME                    READY   STATUS
> flux-system             True    Applied
> crds                    True    Applied
> infrastructure          False   dependency 'security' not ready
> security                False   dependency 'crds' not ready
> observability           False   dependency 'infrastructure' not ready
>
> Analyse ma config dans clusters/mycluster-0/ et identifie le problème.
```

#### Diagnostic de Claude

Après analyse des fichiers, Claude a trouvé une **dépendance circulaire** :

```yaml
# clusters/mycluster-0/infrastructure.yaml
spec:
  dependsOn:
    - name: security  # security dépend de crds...

# clusters/mycluster-0/security.yaml
spec:
  dependsOn:
    - name: crds
    - name: infrastructure  # ...mais infrastructure dépend de security !
```

#### Solution

```yaml
# Corriger security.yaml - retirer la dépendance circulaire
spec:
  dependsOn:
    - name: crds
    # Retirer: - name: infrastructure
```

Claude a aussi suggéré de visualiser les dépendances :

```console
flux tree kustomization flux-system
```

Et m'a rappelé l'importance du paramètre `timeout` pour éviter les blocages trop longs :

```yaml
spec:
  timeout: 5m  # Par défaut c'est 10m, parfois trop long pour debugger
  retryInterval: 1m
```

---

### :building_construction: Cas 7 : Création d'un self-service pour bases de données

#### Le contexte

L'équipe de développement veut pouvoir créer des bases PostgreSQL sans passer par l'équipe plateforme. Je dois créer une abstraction Crossplane simple.

#### Le prompt structuré

```
> Je veux créer un Claim Crossplane simple pour que les devs puissent
> demander une base de données PostgreSQL managée.
>
> Contraintes :
> - Utiliser CloudNativePG (pas RDS)
> - Namespace dédié par base
> - Backup automatique sur S3
> - Secret généré pour les credentials
>
> Le dev doit juste spécifier : nom, taille stockage, version PG.
>
> Base-toi sur les patterns existants dans infrastructure/crossplane/
```

#### Ce que Claude a produit

**1. Le Claim (ce que le dev utilise) :**

```yaml
apiVersion: database.ogenki.io/v1alpha1
kind: PostgreSQLClaim
metadata:
  name: my-app-db
  namespace: my-app
spec:
  storageSize: 10Gi
  postgresVersion: "16"
```

**2. La Composition (ce que la plateforme gère) :**

```python
# infrastructure/crossplane/configuration/kcl/postgresql-composition.k
import crossplane.v1 as xp
import kubernetes.api.core.v1 as k8s

_params = option("params")
_appName = _params.metadata.name

# Namespace dédié
_namespace = k8s.Namespace {
    metadata.name = "db-${_appName}"
}

# Cluster CloudNativePG
_cluster = {
    apiVersion = "postgresql.cnpg.io/v1"
    kind = "Cluster"
    metadata = {
        name = _appName
        namespace = "db-${_appName}"
    }
    spec = {
        instances = 2
        imageName = "ghcr.io/cloudnative-pg/postgresql:${_params.spec.postgresVersion}"
        storage = {
            size = _params.spec.storageSize
            storageClass = "gp3"
        }
        backup = {
            barmanObjectStore = {
                destinationPath = "s3://cnpg-backups-ogenki/${_appName}"
                s3Credentials = {
                    accessKeyId.name = "backup-creds"
                    secretAccessKey.name = "backup-creds"
                }
            }
            retentionPolicy = "30d"
        }
    }
}

# Export des ressources
items = [_namespace, _cluster]
```

{{% notice info "Platform as a Product" %}}
C'est exactement ça, traiter la plateforme comme un produit. Le dev a une interface simple (`PostgreSQLClaim`), et toute la complexité (namespace, backup, réplication) est masquée. Claude m'a aidé à construire cette abstraction en 15 minutes au lieu de 2 heures.
{{% /notice %}}

---

## :bulb: Conseils issus de ces expériences

### Ce qui fonctionne bien

| Situation | Pourquoi Claude excelle |
|-----------|------------------------|
| Debugging avec contexte | Lit les logs, configs, et corrèle |
| Conversion de formats | YAML → YAML, HCL → KCL, etc. |
| Refactoring répétitif | 47 fichiers ? Pas de problème |
| Documentation | Génère des diagrammes, des README |
| Analyse de dépendances | Graphes Flux, imports Terraform |

### Ce qui nécessite plus de vigilance

| Situation | Point d'attention |
|-----------|-------------------|
| Création from scratch | Vérifier les best practices récentes |
| Sécurité/PKI | Toujours valider manuellement |
| Ressources cloud coûteuses | Bien reviewer avant `apply` |
| Breaking changes | Consulter la doc officielle en plus |

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
- [Claude Pricing](https://claude.com/pricing) - Tarification à jour

### Benchmarks et comparatifs
- [SWE-bench Leaderboards](https://www.swebench.com/) - Benchmark de référence
- [Artificial Analysis - Coding Agents](https://artificialanalysis.ai/insights/coding-agents-comparison) - Comparatif des agents
- [State of AI Code Quality 2025](https://www.qodo.ai/reports/state-of-ai-code-quality/) - Qodo

### MCPs pour Platform Engineering
- [Awesome DevOps MCP Servers](https://github.com/rohitg00/awesome-devops-mcp-servers) - Liste curatée
- [Context7 MCP](https://github.com/upstash/context7) - Documentation à jour pour LLMs
- [10 Best MCP Servers for Platform Engineers](https://stackgen.com/blog/the-10-best-mcp-servers-for-platform-engineers-in-2025) - StackGen

### Articles et études
- [AI Coding Is Everywhere](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/) - MIT Technology Review
- [METR Study on AI Productivity](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) - Étude sur la productivité
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) - Anthropic Research

### Outils et ressources
- [Cloud Native Ref](https://github.com/Smana/cloud-native-ref) - Mon repo de référence
- [Claude Code Hooks Mastery](https://github.com/disler/claude-code-hooks-mastery) - Exemples de hooks
- [ClaudeLog](https://claudelog.com/) - Guides et tutoriels communautaires
