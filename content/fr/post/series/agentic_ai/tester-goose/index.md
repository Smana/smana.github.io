+++
author = "Smaine Kahlouch"
title = "Tester `Goose` : un client agentique open-source comme alternative à `Claude Code`"
date = "2026-05-15"
summary = "Évaluer **Goose** sur des tâches concrètes de Platform Engineering : extensions MCP, recipes, modes d'approbation, providers multiples. Une alternative open-source à `Claude Code`, avec `Opus 4.7` aux commandes."
featured = false
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = [
    "ai",
    "devxp",
    "tooling",
    "open-source"
]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — Partie 4" %}}
Cet article fait suite à [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) (les fondamentaux), [Quelques mois avec Claude Code](/fr/post/series/agentic_ai/ai-coding-tips/) (tips et workflows quotidiens) et [Self-hosted LLM stack](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (héberger ses propres modèles open-weight). **Ici, on change de client** : et si on remplaçait `Claude Code` par un agent entièrement **open-source** ?
{{% /notice %}}

Depuis le début de cette série, j'utilise `Claude Code` au quotidien et je n'en suis globalement pas mécontent : la qualité de `Opus 4.7` est excellente, l'écosystème (`CLAUDE.md`, skills, MCP, plugins…) est mature, le rythme des releases impressionnant. Mais une question revient régulièrement quand je parle de tout cela autour de moi : *que se passe-t-il le jour où Anthropic change les règles, ferme une porte, ou simplement augmente ses prix ?*

Cette crainte n'est pas hypothétique. Début 2026, Anthropic a [bloqué sans préavis](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses) certains _harnesses_ tiers qui utilisaient Claude — un rappel utile que **l'outil et le modèle sont deux choses différentes**, et qu'on a tout intérêt à pouvoir découpler les deux.

Je me suis donc lancé un petit défi : **prendre un agent open-source sérieux, le brancher sur `Opus 4.7`, et le mettre à l'épreuve sur les mêmes tâches que je donne à `Claude Code` au quotidien**. Mon candidat : [**Goose**](https://github.com/aaif-goose/goose), un client agentique généraliste développé à l'origine par Block (Jack Dorsey), puis donné début 2026 à la **Agentic AI Foundation** de la Linux Foundation.

## :dart: Objectifs de cet article

* Comprendre **ce qu'est Goose** et en quoi il se distingue dans l'écosystème agentique
* Découvrir ses **primitives clés** : providers, extensions MCP, _recipes_, modes d'autonomie, subagents
* Installer et configurer Goose **avec Anthropic `Opus 4.7`** comme moteur de raisonnement
* Comparer **honnêtement** Goose, `Claude Code` et [`opencode`](https://opencode.ai/) sur des tâches réelles de Platform Engineering
* Identifier dans quels cas Goose **gagne**, où il **perd**, et où les outils se complètent

{{% notice tip "Le repo de référence" %}}
<table>
  <tr>
    <td><img src="repo_gift.png" style="width:80%;"></td>
    <td style="vertical-align:middle; padding-left:10px;" width="70%">
Comme dans les articles précédents de la série, les exemples sont issus de mon travail sur le repository <strong><a href="https://github.com/Smana/cloud-native-ref">Cloud Native Ref</a></strong> (EKS, Cilium, VictoriaMetrics, Crossplane, Flux, vLLM…). Je testerai Goose sur les mêmes tâches que j'ai déjà confiées à Claude Code, pour une comparaison aussi <em>apples-to-apples</em> que possible.
    </td>
  </tr>
</table>
{{% /notice %}}

{{% notice info "Déjà familier avec Goose ?" %}}
Si vous connaissez déjà les bases (installation, providers, extensions, recipes), passez directement à la [comparaison face à face](#-goose-vs-claude-code-vs-opencode--face-à-face) ou aux [cas pratiques sur cloud-native-ref](#-en-pratique-sur-cloud-native-ref).
{{% /notice %}}

---

## :goose: Qu'est-ce que Goose ?

`Goose` est un **agent IA open-source, généraliste, exécuté localement**. Pas de SaaS, pas de cloud propriétaire : un binaire que vous installez sur votre poste, qui pilote ensuite des outils en boucle pour atteindre un objectif — exactement la même **boucle agentique** que celle décrite dans la [Partie 1 de cette série](/fr/post/series/agentic_ai/ai-coding-agent/#le-fonctionnement-dun-agent).

Ce qui le distingue ?

* **Multi-provider** : 13+ fournisseurs LLM supportés (Anthropic, OpenAI, Google, Bedrock, Ollama, vLLM, OpenRouter…) — vous changez de modèle sans changer d'outil
* **Multi-interface** : un CLI (terminal), une application **Desktop** native (macOS, Linux, Windows), un serveur **REST** (`goosed`), un serveur **ACP** (Agent Client Protocol, utilisé par l'éditeur Zed)
* **Tout passe par MCP** : les extensions, y compris celles fournies en standard, sont des serveurs **Model Context Protocol**
* **Vraiment open-source** : licence **Apache 2.0**, code en Rust (~49%) + TypeScript (~45%), gouvernance neutre via la **Linux Foundation**
* **Headless natif** : un mode `--no-session` qui le rend utilisable directement dans une pipeline CI/CD, sans interaction humaine

| Caractéristique | Détail |
|---|---|
| **Repo** | [`aaif-goose/goose`](https://github.com/aaif-goose/goose) |
| **Licence** | Apache 2.0 |
| **Dernière version stable** | v1.34.0 (mai 2026) |
| **Stack** | Rust + TypeScript |
| **Packaging** | `deb`, `rpm`, `dmg`, `msi`, Flatpak, Docker |
| **Gouvernance** | Agentic AI Foundation (Linux Foundation) |
| **Précédente gouvernance** | Block — donation à la LF en début 2026 |
| **Étoiles GitHub** | ~45k |

{{% notice info "Pourquoi Block a-t-il développé un agent généraliste ?" %}}
Block (anciennement Square) a une équipe **Open Source** active qui pousse plusieurs projets infrastructurels (TBDex, Web5, etc.). Goose est né en 2025 d'un besoin interne : automatiser des tâches de _data engineering_, de _support_ et de _platform engineering_ avec un agent qu'on pouvait **auditer, étendre et déployer en interne** — autant de choses difficiles avec un client propriétaire. La donation à l'AAIF a été annoncée début 2026 comme un signal de neutralité — un détail important pour les équipes qui hésitent à dépendre d'un fournisseur unique.
{{% /notice %}}

Goose n'est pas seul sur ce créneau. Côté open-source, [`opencode`](https://github.com/sst/opencode) (TypeScript, licence MIT, développé par l'équipe SST) joue dans la même cour. Là où `opencode` mise sur un **TUI léger** et une vélocité communautaire élevée, `Goose` mise sur la **polyvalence** — CLI, Desktop, REST, ACP — et sur une **gouvernance neutre**. Nous reviendrons en détail sur cette comparaison plus loin.

## :building_construction: Architecture et composants

### Les trois visages de Goose

Là où `Claude Code` est essentiellement un CLI (avec une intégration IDE), Goose propose **trois surfaces** pour le même cœur agentique :

{{< img src="architecture-goose.png" width="950" caption="Vue d'ensemble : un cœur commun, trois clients (CLI, Desktop, REST/ACP), et des extensions MCP pour les outils." >}}

1. **`goose` (CLI)** — le binaire en ligne de commande, mode interactif (`goose session`) ou _one-shot_ (`goose run -t "..."`)
2. **Goose Desktop** — application native (historiquement Electron, en migration vers Tauri sur la v1.34+) avec une UI de chat, historique de sessions, gestion d'extensions
3. **`goosed`** — un serveur **REST** local qui expose l'agent via HTTP (utile pour intégrer Goose dans un pipeline custom, un éditeur, ou un autre outil)

Et un quatrième mode, plus récent : **ACP** — _Agent Client Protocol_, un protocole stdio bidirectionnel qui permet à un éditeur (comme [Zed](https://zed.dev)) de _piloter_ Goose en sous-processus. C'est le pendant d'**MCP** côté client : MCP standardise les outils que l'agent appelle ; ACP standardise l'agent que le client appelle.

### Tout passe par MCP

Toutes les capacités de Goose — lecture de fichiers, exécution shell, manipulation Git, contrôle de la souris, mémoire persistante — sont exposées via des **serveurs MCP**. Y compris les capacités built-in :

| Extension built-in | Rôle |
|---|---|
| `developer` | Lire / écrire des fichiers, exécuter le shell, opérations Git de base |
| `computercontroller` | Capture d'écran, contrôle clavier/souris (automation GUI) |
| `memory` | Mémoire persistante (graph Cognee) entre sessions |
| `auto-visualiser` | Rendu de résultats (Markdown → HTML, JSON _pretty-print_, graphiques) |

C'est un choix architectural assumé : pas de tools internes _spéciaux_, tout outil est un MCP. **Avantage** : vous pouvez remplacer, désactiver, ou auditer chaque capacité. **Conséquence** : la performance de Goose est très dépendante de la qualité de ses MCPs — un MCP mal écrit, lent, ou bavard pollue le contexte.

### État local et configuration

Tout l'état vit dans `~/.config/goose/` (macOS/Linux) ou `%APPDATA%\Block\goose\config\` (Windows) :

```text
~/.config/goose/
├── config.yaml          # configuration globale (provider, modèle, extensions)
├── sessions/            # historique des sessions nommées
│   ├── cnref-bootstrap.jsonl
│   └── cnref-flux.jsonl
└── recipes/             # recipes locales (workflows réutilisables)
```

Et au niveau projet, un fichier `.goosehints` joue le rôle de l'équivalent `CLAUDE.md` / `AGENTS.md` :

```text
# .goosehints — placé à la racine du projet
Ce repo est un projet Kubernetes GitOps avec Flux et Crossplane.

@README.md
@docs/architecture/README.md

Conventions :
- Toujours valider les compositions KCL avec `kcl fmt && kcl test`
- Ne jamais commiter dans `main` directement, passer par une PR
- Les ExternalSecrets vivent dans `security/base/external-secrets/`
```

Goose charge automatiquement ce fichier au démarrage de la session (ainsi que les fichiers `@référencés`).

{{% notice tip "Et le `AGENTS.md` ?" %}}
Le repo de Goose lui-même publie un fichier [`AGENTS.md`](https://github.com/aaif-goose/goose/blob/main/AGENTS.md) — sympa pour les contributeurs, mais ce n'est **pas** le fichier que Goose lit en runtime. Pour vos propres projets, utilisez `.goosehints`.
{{% /notice %}}

## :gear: Installation et premier contact

L'installation est straightforward sur les trois systèmes :

```bash
# Linux / macOS — script officiel
curl -fsSL https://github.com/aaif-goose/goose/releases/download/stable/download_cli.sh | bash

# Windows (PowerShell)
iwr -Uri "https://raw.githubusercontent.com/aaif-goose/goose/main/download_cli.ps1" -OutFile "install.ps1"
.\install.ps1

# Vérification
goose --version
# goose 1.34.0
```

{{% notice tip "Installation sans `curl | bash`" %}}
Si vous préférez éviter `curl | bash` (à raison), Goose publie des paquets natifs sur la page [Releases](https://github.com/aaif-goose/goose/releases) : `.deb`, `.rpm`, `.dmg`, `.msi`, Flatpak, et une image Docker `ghcr.io/aaif-goose/goose:latest`. Pour les utilisateurs `mise`, le binaire peut être épinglé via un alias custom.
{{% /notice %}}

Premier contact, en mode _one-shot_ pour valider la configuration :

```bash
# Hello, world version Platform Engineer
goose run -t "Lis le fichier README.md à la racine et fais-moi un résumé en 3 phrases."
```

Goose va vous demander, à la première exécution, de configurer un provider. Le mode interactif via `goose configure` est le plus simple :

```bash
goose configure
# > Configure Providers
# > Anthropic
# > Enter your API key: sk-ant-...
# > Select model: claude-opus-4-7
# > Default mode: smart_approve
```

Ensuite, une session interactive nommée — équivalent fonctionnel à une conversation persistante avec `Claude Code` :

```bash
goose session -n cnref-bootstrap
# Goose vous accueille, charge .goosehints, et attend votre prompt.
```

## :electric_plug: Providers : la liberté du multi-LLM

C'est la **principale différence de philosophie** avec `Claude Code`. Goose ne se marie à aucun modèle ; vous choisissez le vôtre, et vous pouvez en changer en une variable d'environnement.

| Provider | Auth | Modèles notables | Local ? |
|---|---|---|---|
| **Anthropic** | API key | Claude Opus 4.7, Sonnet 4.6, Haiku 4.5 | ✗ |
| **OpenAI** | API key | GPT-5.2, o1 | ✗ |
| **Google** | API key / Vertex / OAuth | Gemini 3 Pro, Gemini 3 Flash | ✗ |
| **OpenRouter** | API key | 300+ modèles routés | ✗ |
| **AWS Bedrock** | IAM | Claude, Llama, Mistral | ✗ |
| **Azure OpenAI** | Credential chain | GPT-4o, GPT-5 | ✗ |
| **Databricks** | API key | DBRX | ✗ |
| **Groq** | API key | Modèles open + inférence rapide | ✗ |
| **xAI** | API key | Grok-3 | ✗ |
| **Mistral** | API key | Mistral Large, Codestral | ✗ |
| **Ollama** | HTTP local | Qwen3-coder, DeepSeek-R1, Llama 4 | ✓ |
| **LM Studio** | HTTP local | n'importe quel GGUF | ✓ |
| **vLLM / OpenAI-compatible** | HTTP | self-hosted | ✓ |

{{% notice tip "Le pont avec la Partie 3" %}}
La stack `vLLM Production Stack` décrite dans la [Partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/) expose un endpoint **OpenAI-compatible** via l'Envoy AI Gateway. Goose peut donc s'y connecter directement comme provider — pour la première fois dans cette série, on peut **boucler la boucle** : un agent open-source piloté par un modèle open-weight servi sur notre propre infrastructure.
{{% /notice %}}

### Configurer Anthropic avec `Opus 4.7`

C'est le setup que je vais utiliser dans tout cet article — `Opus 4.7` au cerveau, sur ma machine, via API directe.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
goose configure
# > Configure Providers > Anthropic > coller la clé > claude-opus-4-7
```

L'équivalent en _config-as-code_ dans `~/.config/goose/config.yaml` :

```yaml
# ~/.config/goose/config.yaml
GOOSE_PROVIDER: "anthropic"
GOOSE_MODEL: "claude-opus-4-7"
GOOSE_TEMPERATURE: 0.7
GOOSE_MAX_TOKENS: 16000
GOOSE_MODE: "smart_approve"
```

Vérification rapide :

```bash
goose run -t "Quel modèle es-tu, et quelle est ta date de connaissance ?"
# Goose répond en utilisant Opus 4.7 et confirme sa version.
```

{{% notice warning "Pas d'abonnement Claude Max via Goose" %}}
Goose passe par l'**API directe** d'Anthropic (`api.anthropic.com`). L'abonnement **Claude Max** que vous utilisez peut-être avec `Claude Code` n'est **pas** reconnu — vous payez au token consommé, comme un usage API classique.

Tarifs `Opus 4.7` au moment de l'écriture : **~$15 / 1M tokens input**, **~$75 / 1M tokens output**. À usage intensif, ça monte vite. Bonne nouvelle : Goose supporte le **prompt caching** Anthropic, qui réduit de 50% le coût des tokens cachés — pensez à activer `GOOSE_PROMPT_CACHE_ENABLED=true` si votre `.goosehints` ou votre contexte projet sont volumineux.
{{% /notice %}}

### Lead/worker — un modèle pour planifier, un autre pour exécuter

Voici une primitive que `Claude Code` n'expose pas (du moins pas au moment où j'écris) : **utiliser deux modèles dans la même session**, un pour la planification stratégique, l'autre pour l'exécution.

```yaml
# Modèle de planification (premiers tours, raisonnement stratégique)
GOOSE_LEAD_MODEL: "claude-opus-4-7"
GOOSE_LEAD_PROVIDER: "anthropic"

# Modèle d'exécution (édition de fichiers, commandes shell)
GOOSE_MODEL: "claude-haiku-4-5"
GOOSE_PROVIDER: "anthropic"
```

Concrètement, Goose laisse `Opus 4.7` poser le plan (les premiers tours coûteux mais déterminants), puis bascule sur `Haiku 4.5` pour exécuter les actions simples (lectures, écritures, commandes). Si `Haiku` se plante, on peut re-_escalader_ sur `Opus`.

J'ai trouvé l'expérience intéressante sur des tâches longues : le coût total chute fortement, sans perte sensible de qualité tant que la phase de planification est solide. Cette option a évidemment du sens **uniquement** sur des tâches multi-étapes — pour un _one-shot_ rapide, on garde un seul modèle.

## :toolbox: Extensions et MCPs

### Trois transports possibles

Goose distingue trois types d'extensions, qui correspondent aux trois transports MCP :

1. **Built-in** — embarquées dans le binaire (`developer`, `computercontroller`, `memory`, `auto-visualiser`)
2. **Command-line (stdio)** — un binaire local lancé en sous-processus, qui parle MCP via stdin/stdout
3. **HTTP / SSE** — un serveur MCP distant, atteignable via HTTP ou Server-Sent Events

Configuration interactive :

```bash
goose configure
# > Add Extension
# > Command-Line Extension
# > Name: kubernetes
# > Command: npx
# > Args: -y @kubernetes/k8s-mcp
# > Timeout: 300
# > Environment variables: KUBECONFIG=/home/smana/.kube/config
```

L'équivalent statique dans `config.yaml` :

```yaml
extensions:
  developer:
    type: builtin
    enabled: true
    timeout: 300

  kubernetes:
    type: stdio
    cmd: npx
    args: ["-y", "@kubernetes/k8s-mcp"]
    enabled: true
    timeout: 300
    env_keys: ["KUBECONFIG"]

  github:
    type: http
    url: "https://api.githubcopilot.com/mcp/"
    enabled: true
    env_vars:
      - key: "Authorization"
        value: "Bearer ghp_xxx"
```

### Activation à la volée

Goose permet aussi d'activer des extensions **à la session**, sans modifier la config globale :

```bash
# Session avec uniquement le builtin developer + un MCP Flux
goose session -n cnref-flux \
  --with-builtin developer \
  --with-extension "npx -y @flux-operator/flux-mcp"

# Session avec un MCP HTTP distant (streamable)
goose session --with-streamable-http-extension "http://localhost:8080/mcp"
```

C'est très pratique pour _scoper_ les outils disponibles selon la tâche — par exemple, désactiver `computercontroller` quand on travaille sur un serveur sans display.

### Comparaison avec `Claude Code` : `.mcp.json` vs config Goose

C'est une vraie différence philosophique :

| Aspect | Goose | `Claude Code` |
|---|---|---|
| **Portée par défaut** | User-wide (`~/.config/goose/config.yaml`) | Projet (`.mcp.json` à la racine) |
| **Override projet** | `.goosehints` + flags `--with-*` à la session | `.mcp.json` toujours prioritaire |
| **Activation par session** | Oui (`--with-extension`, `--with-builtin`) | Non, statique au niveau du projet |
| **Transport MCP** | stdio, HTTP, SSE, builtin | stdio, HTTP |
| **Audit / liste** | `goose configure > Manage Extensions` | Lecture directe de `.mcp.json` |

**Mon avis** : la portée user-wide de Goose est très flexible (un même MCP partout, sans répéter le `.mcp.json` dans chaque repo), mais plus facile à oublier en travail d'équipe. Pour un repo partagé, `Claude Code` reste plus rigoureux : la config voyage avec le code. Le bon compromis avec Goose, c'est de versionner un `.goosehints` qui rappelle quels MCPs activer à la session.

## :scroll: Recipes : des workflows réutilisables

C'est, à mon sens, **la primitive la plus distinctive de Goose**. Une _recipe_ est un fichier YAML qui décrit un workflow agentique entier : prompt système, paramètres typés, extensions à activer, modèle à utiliser, schéma de réponse attendu.

Exemple concret — une recipe qui génère un **ADR** (Architecture Decision Record) à partir d'un répertoire technique :

```yaml
# adr-writer.yaml
version: "1.0.0"
title: "ADR Writer"
description: "Générer un Architecture Decision Record à partir d'un contexte"
instructions: |
  Tu es un architecte expérimenté. Tu écris des ADRs concis, structurés
  selon le template (Context, Decision, Consequences, Alternatives).
  Tu inspectes le contexte fourni avant de proposer une décision.
prompt: "Génère un ADR sur le sujet : {{ topic }}"

parameters:
  - key: "topic"
    input_type: "string"
    requirement: "required"
    description: "Sujet de l'ADR à rédiger"
  - key: "context_dir"
    input_type: "string"
    requirement: "optional"
    default: "."
    description: "Répertoire contenant le contexte technique à inspecter"

extensions:
  - type: "builtin"
    name: "developer"
    bundled: true

settings:
  goose_provider: "anthropic"
  goose_model: "claude-opus-4-7"
  temperature: 0.4
  max_turns: 20

response:
  json_schema:
    type: "object"
    properties:
      title:    { type: "string" }
      status:   { type: "string", enum: ["proposed", "accepted", "deprecated"] }
      context:  { type: "string" }
      decision: { type: "string" }
      consequences: { type: "string" }
      alternatives: { type: "array", items: { type: "string" } }
```

Invocation :

```bash
goose run --recipe adr-writer.yaml \
  --params topic="Migration de Prometheus vers VictoriaMetrics" \
  --params context_dir="./observability"
```

Et pour intégrer dans une pipeline (CI/CD, GitHub Action, hook Git…) :

```bash
goose run --recipe adr-writer.yaml \
  --params topic="$TOPIC" \
  --no-session \
  --output-format json \
  --max-turns 20 \
  > adr.json
```

L'output respecte le `json_schema` déclaré — pas de parsing fragile en regex, on récupère directement un objet structuré.

### Recipes vs Skills vs Subagents

Il y a une carte mentale utile à dresser entre les primitives `Claude Code`, Goose et `opencode` :

| Claude Code | Goose | opencode | Comparable ? |
|---|---|---|---|
| **Skill** (Markdown + outils) | **Recipe** (YAML déclaratif) | **Agent** (Markdown + permissions) | Oui, philosophies différentes |
| **Tool** (built-in ou MCP) | **MCP tool** | **MCP tool** | Oui, même standard |
| **Subagent** (parallèle sur invocation) | **Subagent** (langage naturel) | **Subagent** (mode subagent) | Oui |
| **CLAUDE.md / AGENTS.md** | **.goosehints** | **AGENTS.md** | Oui |
| **Plan mode** | **`smart_approve` / `chat`** | **plan agent** (lecture seule) | Approchant |

**Recipes Goose** sont plus _déclaratives_ (schéma de paramètres, JSON schema de sortie) que les skills `Claude Code`, mais moins riches : pas de composition de skills entre elles, pas de tools spécifiques liés à la skill. C'est un échange : **portabilité contre expressivité**. Une recipe peut être commitée dans un repo Git, partagée par PR, exécutée headless — un skill `Claude Code` est plus puissant mais plus lié à l'environnement Claude.

## :traffic_light: Modes et niveaux d'autonomie

Goose propose **quatre modes d'autonomie**, beaucoup plus explicites que `Claude Code` :

| Mode | Comportement | Équivalent `Claude Code` |
|---|---|---|
| `auto` | Exécute tous les outils sans confirmation | `acceptEdits` + `bypassPermissions` |
| `smart_approve` | Le LLM filtre lui-même les actions sensibles | Mode par défaut + heuristique |
| `approve` | Confirmation par outil avant exécution | Plan Mode (au plus proche) |
| `chat` | Pas d'outils, dialogue pur | Mode chat / `--dangerously-skip-permissions` désactivé |

Changement à la volée :

```bash
# Dans une session interactive
/mode auto
/mode smart_approve

# Au niveau de la session
goose session -n cnref-crossplane --mode auto

# Globalement
export GOOSE_MODE=smart_approve
```

`smart_approve` est mon mode par défaut. Le principe : le LLM lui-même classe ses actions selon leur risque (lecture vs écriture vs exécution) et n'interrompt que pour les actions potentiellement destructives. C'est l'équivalent fonctionnel du Plan Mode de `Claude Code`, mais **moins explicite** : Goose vous montre moins ses intentions avant d'agir.

Pour les workflows critiques (production, IaC, secrets), je bascule en `approve`. Pour la CI/CD ou les recipes _headless_, c'est `auto` — assumé.

## :thread: Sessions, fork, subagents, headless

### Sessions nommées et reprise

```bash
# Nouvelle session nommée
goose session -n cnref-crossplane

# Reprendre la session la plus récente
goose session --resume

# Reprendre par nom
goose session --resume -n cnref-crossplane

# Reprendre en affichant tout l'historique
goose session --resume -n cnref-crossplane --history
```

### Fork — diverger sans casser

Une primitive qui m'a séduit : **forker une session existante** pour explorer une piste alternative sans perdre l'historique original.

```bash
goose session --resume --fork --name cnref-crossplane-experiment
# Clone l'historique de cnref-crossplane, nouvelle session indépendante
```

C'est très utile quand on veut tester deux approches sur le même problème — l'équivalent d'un `git checkout -b` sur la conversation. À ma connaissance, `Claude Code` n'expose pas ce fork de manière native (on peut copier le `.claude/` à la main, c'est moins agréable).

### Export / import

```bash
# Exporter une session au format JSON portable
goose session --resume -n cnref-crossplane --export ./session.json

# Reprendre depuis un fichier (partage entre machines, par exemple)
goose session --resume --path ./session.json
```

### Subagents en langage naturel

Les subagents Goose s'invoquent **en langage naturel**, contrairement à `Claude Code` qui les expose comme des appels d'outils explicites :

```text
Crée trois tâches parallèles :
- Une qui rédige la documentation de l'API
- Une qui écrit les tests unitaires
- Une qui prépare le changelog

Chaque tâche doit pouvoir s'exécuter indépendamment, sur des fichiers
qui ne se chevauchent pas. Renvoie un résumé JSON à la fin.
```

Goose lance trois sous-agents, gère le verrouillage de fichiers pour éviter les conflits, et restitue un JSON de synthèse. **Avantage** : on planifie en français. **Inconvénient** : on perd la rigueur explicite que `Claude Code` impose avec ses subagents typés. C'est plus rapide pour prototyper, moins fiable pour des pipelines déterministes.

### Mode headless / CI

```bash
goose run \
  --recipe lint-and-test.yaml \
  --params target_dir="./infrastructure" \
  --no-session \
  --output-format json \
  --max-turns 30 \
  > result.json

# Avec sortie streamée (pour suivre dans un job CI)
goose run --recipe ci-check.yaml --output-format stream-json --quiet
```

Là où `Claude Code` est avant tout un compagnon interactif (on peut le scripter, mais ce n'est pas son terrain naturel), Goose se prête mieux à un usage **CI/CD direct** — pas d'authent interactive, sortie JSON structurée, mode headless natif, et un Dockerfile officiel pour conteneuriser tout ça.

## :robot: Goose vs `Claude Code` vs `opencode` — face à face

Voici la table que j'aurais aimé avoir en arrivant sur Goose. Trois clients agentiques sérieux, comparés ligne à ligne. **`opencode`** est l'autre grand contender open-source (TypeScript, MIT, [`sst/opencode`](https://github.com/sst/opencode), [docs](https://opencode.ai/)) — je l'avais mentionné dans la [Partie 1](/fr/post/series/agentic_ai/ai-coding-agent/#pourquoi-claude-code-) et il méritait sa place ici.

| Axe | `Claude Code` | `Goose` | `opencode` | Verdict |
|---|---|---|---|---|
| **Licence** | Propriétaire | Apache 2.0 | MIT | Match nul côté OSS |
| **Stack** | Closed (TS) | Rust + TS | TypeScript | Goose pour la perf binaire |
| **Providers** | Anthropic uniquement | 13+ providers | 75+ providers (via AI SDK) | `opencode` pour le choix brut |
| **Modèle pricing** | Sub Claude Max + API | API directe | API directe | `Claude Code` pour l'usage intensif via Max |
| **CLI / Desktop / API** | CLI + IDE | CLI + Desktop + REST + ACP | CLI + TUI client/serveur | Goose pour l'ergonomie large |
| **TUI** | TUI Ink | CLI + Desktop, pas de TUI riche | TUI dédié, client/serveur séparés | `opencode` |
| **Skills / Recipes / Agents** | Skills riches (Markdown + tools) | Recipes YAML déclaratives | Agents Markdown + permissions | Match, philosophies différentes |
| **MCP** | `.mcp.json` projet | extensions user + per-session | `opencode.json` projet, per-agent | `Claude Code` et `opencode` pour la rigueur projet |
| **Plan Mode** | Oui, explicite | `smart_approve` / `chat` | Plan agent (lecture seule) | `Claude Code` et `opencode` |
| **Subagents** | Sequential, structurés, typés | Parallèles, langage naturel | Agents en mode `subagent` | Goose pour la rapidité, `Claude Code` pour la rigueur |
| **Memory persistante** | `CLAUDE.md` projet/user | `.goosehints` + Cognee graph | `AGENTS.md` (convention partagée) | `Claude Code` plus mature au quotidien |
| **Hooks** | `PreToolUse`, `PostToolUse`, etc. | v1.34+ (jeune) | Hooks YAML (récents) | `Claude Code` aujourd'hui |
| **Headless / CI** | Possible (claude-code-action) | Natif (`--no-session`) | Natif (`opencode run`) | Goose et `opencode` |
| **Multi-modèle (lead/worker)** | Non | `GOOSE_LEAD_MODEL` + `GOOSE_MODEL` | Switch modèle par agent | Goose pour le pattern explicite |
| **Sessions / fork** | Implicite | Explicite (`--fork`, `--export`) | Sessions persistantes | Goose |
| **Communauté / vélocité** | ~12 mois, énorme | ~18 mois, croissante | ~18 mois, très active | `Claude Code` (taille), `opencode` (vélocité) |
| **Self-hosting end-to-end** | Impossible (cloud Anthropic) | Possible (Goose + Ollama/vLLM) | Possible (`opencode` + Ollama/vLLM) | Goose et `opencode` |
| **Gouvernance** | Anthropic | Linux Foundation (AAIF) | SST (entreprise) | Goose pour la neutralité |

{{% notice tip "Le verdict réel dépend de votre contexte" %}}
- **Solo dev qui ne veut rien changer aujourd'hui** → `Claude Code` reste le défaut, UX mature, écosystème immense
- **Équipe en environnement régulé / souverain** → `Goose` (multi-provider + Rust + Apache 2.0 + Linux Foundation)
- **Dev qui veut un TUI léger, open-source, ultra-réactif** → `opencode` mérite un sérieux coup d'œil
- **Plateforme team qui veut intégrer un agent dans sa CI/CD** → `Goose` ou `opencode`, les deux ont un mode headless solide
- **Quelqu'un qui dépend lourdement des skills `Claude Code`** → l'inertie est réelle, attendre encore quelques mois côté Goose
{{% /notice %}}

Goose ne prétend pas être le plus rapide ni le plus minimaliste ; il vise un **point d'équilibre** : open-source, généraliste, packagé pour CLI/Desktop/API, gouvernance neutre. C'est précisément ce qui en fait un bon candidat pour s'**inscrire dans une stratégie d'équipe** plutôt que pour un usage purement personnel.

---

## :test_tube: En pratique sur cloud-native-ref

Trois scénarios concrets, tirés directement de [cloud-native-ref](https://github.com/Smana/cloud-native-ref) — les mêmes tâches que je donnerais à `Claude Code` au quotidien.

{{% notice note "Tests à approfondir" %}}
Les blocs ci-dessous sont des **scénarios documentés** : je décris le setup, le prompt utilisé, et ce qu'on cherche à observer. Les transcripts complets et les screenshots seront ajoutés après une série de tests réels sur plusieurs sessions, sur quelques semaines d'usage. **L'objectif ici** : poser un cadre comparatif honnête, pas tirer des conclusions hâtives à partir d'une seule run.
{{% /notice %}}

### :test_tube: Scénario 1 — Étendre une composition Crossplane (KCL)

**Objectif** : ajouter un champ `observability.prometheus.enabled` (bool, défaut `true`) à la composition Crossplane `App` du repo. Quand activé, la ressource finale doit porter les labels `prometheus: "true"` et `app.kubernetes.io/component`. Mettre à jour la logique KCL, les exemples, et le README.

**Fichiers en jeu** :
* `infrastructure/base/crossplane/configuration/app-definition.yaml` (XRD)
* `infrastructure/base/crossplane/configuration/app-composition.yaml` (composition)
* `infrastructure/base/crossplane/configuration/kcl/app/main.k` (logique KCL)
* `infrastructure/base/crossplane/configuration/examples/app-*.yaml` (exemples à mettre à jour)

**Setup Goose** :

```bash
cd ~/Sources/cloud-native-ref
goose session -n cnref-crossplane \
  --with-builtin developer \
  --mode smart_approve
```

**Prompt utilisé** :

> Dans ce repo, étends la composition Crossplane `App` pour qu'elle accepte un champ optionnel `observability.prometheus.enabled` (bool, défaut `true`). Quand activé, la ressource finale doit porter les labels `prometheus: "true"` et `app.kubernetes.io/component`. Mets à jour la logique KCL dans `kcl/app/main.k`, ajuste les exemples dans `examples/`, et complète le `README.md` de la composition. Lance `kcl fmt` et `kcl test` avant de proposer un diff final.

**Ce qu'on cherche à observer** :
* Le plan que pose Goose avant d'agir (sortie de `smart_approve`)
* Le nombre de fichiers touchés (XRD, composition, KCL, exemples, README)
* La pertinence des lints KCL exécutés
* La qualité du diff comparée à ce que produit `Claude Code` (qui a déjà géré ce type de tâche dans ce repo)

{{% notice tip "Comparaison à chaud" %}}
Sur ce type de tâche multi-fichiers avec validation locale (`kcl fmt`, `kcl test`), Goose est globalement à l'aise : l'extension `developer` couvre tout ce dont il a besoin, le mode `smart_approve` filtre bien les actions destructives, et la sortie peut être capturée en JSON pour automatiser.

**Là où `Claude Code` garde une avance subjective** : les sub-skills (un `kcl-reviewer` invocable explicitement) donnent un sentiment de **rigueur méthodique** que Goose ne reproduit pas tel quel — Goose préfère l'exécution rapide en langage naturel.
{{% /notice %}}

### :test_tube: Scénario 2 — Diagnostiquer une Kustomization Flux bloquée

**Objectif** : inspecter une Kustomization `crossplane-configuration` qui n'est pas `Ready`. Identifier la cause racine en parcourant les conditions, les events, et les ressources gérées. Proposer un patch YAML correctif qui ne casse pas la chaîne `dependsOn`.

**Fichiers / live state en jeu** :
* `clusters/mycluster-0/infrastructure/crossplane-configuration.yaml` (Kustomization)
* `infrastructure/base/crossplane/configuration/kustomization.yaml` (targets)
* État live du cluster (via MCP Flux + MCP Kubernetes)

**Setup Goose avec extensions MCP** :

```bash
goose session -n cnref-flux \
  --with-builtin developer \
  --with-extension "npx -y @flux-operator/flux-mcp" \
  --with-extension "npx -y @kubernetes/k8s-mcp" \
  --mode approve
```

> Note : à défaut d'avoir un cluster sous la main, on peut reproduire l'exercice en local en cassant volontairement une Kustomization (champ `dependsOn` vers une ressource inexistante).

**Prompt utilisé** :

> La Kustomization `crossplane-configuration` du cluster `mycluster-0` n'est pas Ready depuis 15 minutes. Inspecte les conditions de la Kustomization, les events, et l'état des ressources gérées. Identifie la cause racine. Si elle est claire, propose un patch YAML correctif qui résolve le problème sans casser la chaîne `dependsOn` ni reconfigurer les autres Kustomizations du cluster.

**Ce qu'on cherche à observer** :
* La capacité de Goose à enchaîner plusieurs MCPs (Flux + Kubernetes) dans la même réflexion
* La pertinence du diagnostic (cause racine vs symptôme)
* Le mode `approve` est-il intrusif au point de ralentir, ou apporte-t-il une vraie valeur d'audit ?

L'objectif de ce scénario est aussi de **stresser le mode MCP** sur des serveurs externes (HTTP/SSE), différents des MCPs locaux du Scénario 1. C'est typiquement le genre de tâche où la qualité de la session dépend autant des MCPs que du LLM.

### :test_tube: Scénario 3 — Recipe : générer un dashboard Grafana pour vLLM

C'est le scénario qui exploite à fond les **recipes** — packager une fois, rejouer à chaque nouveau modèle déployé.

**Objectif** : générer un dashboard JSON pour les métriques vLLM (`vllm:num_running_seqs`, `vllm:gpu_cache_usage_perc`, `vllm:e2e_request_latency_seconds`) documentées dans [`docs/specs/0001-llm-platform-prometheus-autoscaling/spec.md`](https://github.com/Smana/cloud-native-ref/tree/main/docs/specs/0001-llm-platform-prometheus-autoscaling), et l'emballer dans un ConfigMap prêt à dropper dans `observability/base/victoria-metrics-k8s-stack/ogenki-grafana-provisioning.yaml`.

**La recipe** :

```yaml
# recipes/vllm-dashboard.yaml
version: "1.0.0"
title: "vLLM Grafana Dashboard"
description: "Génère un dashboard Grafana JSON pour un modèle vLLM"
instructions: |
  Tu génères un dashboard Grafana JSON valide, prêt à être chargé via le
  provisioning Grafana. Tu te bases sur les métriques vLLM listées dans
  docs/specs/0001-llm-platform-prometheus-autoscaling/spec.md.
  Tu produis un ConfigMap Kubernetes complet, namespace=monitoring.
prompt: |
  Génère un dashboard Grafana pour le modèle vLLM dans le namespace {{ namespace }}
  filtré sur le label model={{ model_label }}.

parameters:
  - key: "namespace"
    input_type: "string"
    requirement: "required"
    description: "Namespace Kubernetes du déploiement vLLM"
  - key: "model_label"
    input_type: "string"
    requirement: "required"
    description: "Valeur du label `model` à filtrer dans les requêtes"

extensions:
  - { type: "builtin", name: "developer" }

settings:
  goose_provider: "anthropic"
  goose_model: "claude-opus-4-7"
  temperature: 0.3
  max_turns: 15
```

**Invocation** :

```bash
goose run --recipe recipes/vllm-dashboard.yaml \
  --params namespace="llm" \
  --params model_label="qwen3-8b" \
  --output-format json \
  --no-session \
  > grafana-vllm-qwen3.json
```

**Ce qu'on cherche à observer** :
* La structure du dashboard JSON (panels timeseries, gauges, single-stats)
* La fidélité aux noms de métriques réels (pas d'hallucination)
* La _replay-ability_ : on doit pouvoir relancer la même recipe pour `qwen3-30b`, `deepseek-r1-7b`, etc., en changeant juste un paramètre

C'est ici que **les recipes brillent** vs les prompts ad-hoc `Claude Code` : on capitalise un workflow une fois, on le rejoue à coût zéro à chaque nouveau modèle déployé sur la plateforme. Pour une équipe Platform, c'est exactement le type de _golden path_ qu'on veut industrialiser.

---

Ces trois scénarios ne suffisent évidemment pas à conclure — mais ils donnent un aperçu honnête de ce que Goose sait faire, et de ses limites face à un outil aussi mature que `Claude Code`. **À chaque test, je documenterai en parallèle ce que `Claude Code` produit sur exactement le même prompt**, pour rester comparatif et factuel.

## :thought_balloon: Dernières remarques

### Ce qui fonctionne bien

Après quelques semaines d'usage exploratoire et la rédaction de cet article, voici les points qui m'ont **vraiment** plu chez Goose :

* **Multi-provider de bout en bout** — la même session peut basculer d'Anthropic à OpenAI à Ollama sans rien casser. Pour quiconque veut éviter le _vendor lock-in_, c'est du temps gagné
* **Headless natif et propre** — `--no-session`, `--output-format json`, `--max-turns N`, et un binaire Rust autonome. Goose se script comme un binaire Unix classique
* **Recipes versionnables** — capitaliser un workflow dans un fichier YAML, le commiter dans le repo, le rejouer en CI. La vraie boucle DevOps appliquée aux agents
* **Modes d'autonomie clairs** — `auto`, `smart_approve`, `approve`, `chat` ; on sait exactement ce qu'on autorise à un instant T
* **Vrai open-source** — Apache 2.0, gouvernance Linux Foundation, code Rust auditable. Pas de discussion sur la confidentialité du code (modulo le LLM choisi)

### Ce qui frotte encore

Soyons honnêtes — Goose n'est pas (encore ?) au niveau de `Claude Code` sur plusieurs points :

* **Maturité de l'écosystème** — `Claude Code` a un an d'avance, une communauté plus grande, des skills publiées de qualité, des plugins ([CC-DevOps-Skills](https://github.com/akin-ozer/cc-devops-skills), [Claude-Mem](https://github.com/thedotmack/claude-mem), etc.). Goose rattrape mais le delta est réel
* **`.goosehints` plus pauvre que `CLAUDE.md`** au quotidien — pas (encore) de hiérarchie aussi fine, moins de plugins qui en exploitent la richesse
* **Hooks tout jeunes** (v1.34 seulement) — pas encore le niveau de granularité des `PreToolUse` / `PostToolUse` `Claude Code`
* **Pas de Plan Mode explicite** — `smart_approve` fait le job, mais l'**explicite** de Plan Mode (un plan écrit, validable avant exécution) reste un atout `Claude Code`
* **Subagents en langage naturel** : génial pour prototyper, moins fiable pour de la prod déterministe (`Claude Code` typed-subagents font moins de promesses, en tiennent plus)

### À qui ça parle aujourd'hui ?

* **Équipes Plateforme** qui veulent intégrer un agent dans leur CI/CD ou leurs runners — Goose headless est taillé pour ça
* **Devs en environnement régulé / souverain** (banque, santé, secteur public) qui ne peuvent pas envoyer leur code à un SaaS fermé. Combinés à la stack open-weight de la [Partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/), Goose ferme la boucle de la souveraineté
* **Curieux qui veulent comprendre / hacker leur agent** — code Rust + TS, Apache 2.0, contributions ouvertes
* **Équipes qui veulent capitaliser des workflows** en _recipes_ versionnables — les ADR, les revues de PR, les migrations répétitives

### À qui ça ne parle pas (encore)

* **Devs solo qui veulent juste que ça marche** — `Claude Code` reste devant en UX et en richesse d'écosystème
* **Équipes lourdement investies dans les skills `Claude Code`** — la migration n'est ni triviale ni urgente
* **Usages très interactifs** où le côté _compagnon de codage_ prime sur l'automatisation — `Claude Code` y excelle

### Mon usage à moi

Soyons clairs : **je ne troque pas `Claude Code` contre Goose**. Pour mes sessions interactives quotidiennes, `Claude Code` + `Opus 4.7` reste mon choix par défaut — qualité, UX, écosystème, tout y est.

**Goose entre dans ma boîte à outils pour trois cas précis** :
1. **CI/CD et automatisation** — packager des _recipes_ versionnables qu'on rejoue sur chaque PR (lint, ADR, revue de spec, génération de dashboards)
2. **Branchement sur la stack self-hosted de la Partie 3** — un agent open-source qui consomme nos modèles open-weight via l'endpoint OpenAI-compatible. Bouclage de boucle assumé
3. **Exploration multi-providers** — basculer sur Gemini 3 Pro, GPT-5.2, DeepSeek-R1 sans changer d'outil, juste pour comparer leur comportement sur les mêmes tâches

C'est un **complément**, pas un remplacement — et c'est probablement le bon mode d'emploi à ce stade de maturité.

### Et après ?

* Suivre la v1.35+ : la transition Electron → Tauri du Desktop, les progrès des hooks, l'arrivée éventuelle d'un Plan Mode explicite
* Tester sérieusement l'**ACP avec Zed** — un éditeur qui pilote Goose en sous-processus, ça change pas mal de choses
* Écrire mes propres MCPs internes pour des outils que je n'ai aujourd'hui qu'en bash (Crossplane status, Karpenter NodeClaim, Cilium policies)
* Explorer **Cognee** sérieusement — la mémoire graph persistante entre sessions, c'est exactement ce que `CLAUDE.md` ne fait pas

Honnêtement, le plus intéressant avec Goose, ce n'est pas qu'il remplace `Claude Code`. C'est qu'il **prouve** qu'on peut faire un agent sérieux, ouvert, gouverné par une fondation neutre — et qu'il oblige les autres à se positionner. Que ce soit Anthropic qui ouvre certains protocoles (MCP est déjà un précédent), ou la communauté qui converge sur un standard (ACP), tout le monde gagne.

Et si Anthropic décide demain de re-fermer certaines portes ? On aura déjà l'outil pour passer à autre chose. C'est peut-être ça, la vraie valeur de Goose 😉.

---

## :bookmark: Références

### Goose — officiel
- [`aaif-goose/goose`](https://github.com/aaif-goose/goose) — le repo principal (Apache 2.0)
- [goose-docs.ai](https://goose-docs.ai/) — documentation officielle
- [Blog Goose](https://github.com/aaif-goose/goose/tree/main/documentation/blog) — articles techniques par les mainteneurs
- [Recipes officielles](https://github.com/aaif-goose/goose-recipes) — recettes communautaires
- [Releases](https://github.com/aaif-goose/goose/releases) — paquets `.deb`, `.rpm`, `.dmg`, `.msi`, Flatpak

### Goose — guides
- [Providers (Anthropic, OpenAI, Ollama…)](https://goose-docs.ai/docs/getting-started/providers)
- [Permission modes](https://goose-docs.ai/docs/guides/goose-permissions)
- [Recipes reference](https://goose-docs.ai/docs/guides/recipes/recipe-reference)
- [Subagents](https://goose-docs.ai/docs/guides/subagents)
- [Headless mode](https://goose-docs.ai/docs/tutorials/headless-goose)
- [Custom MCP extensions](https://goose-docs.ai/docs/getting-started/using-extensions)

### Écosystème agentique
- [Model Context Protocol](https://modelcontextprotocol.io/) — le standard d'interopérabilité des outils
- [Agent Client Protocol (ACP)](https://github.com/zed-industries/agent-client-protocol) — le standard côté client (utilisé par Zed)
- [Block Open Source — annonce initiale de Goose](https://block.xyz/inside/block-open-source-introduces-codename-goose)
- [Cognee](https://github.com/topoteretes/cognee) — la mémoire graph utilisée par l'extension `memory`

### Alternatives open-source comparées
- [`sst/opencode`](https://github.com/sst/opencode) — agent open-source MIT, terminal-first
- [opencode docs](https://opencode.ai/) — documentation officielle
- [`charmbracelet/crush`](https://github.com/charmbracelet/crush) — anciennement OpenCode (Charmbracelet)
- [Comparatif synthétique des outils agentiques (Partie 1)](/fr/post/series/agentic_ai/ai-coding-agent/#pourquoi-claude-code-)

### À relire dans cette série
- [Agentic Coding : concepts et cas concrets appliqués au Platform Engineering](/fr/post/series/agentic_ai/ai-coding-agent/)
- [Quelques mois avec Claude Code : tips et workflows](/fr/post/series/agentic_ai/ai-coding-tips/)
- [Self-hosted LLM stack : poser les fondations d'une plateforme open-weight](/fr/post/series/agentic_ai/llm-self-hosted-stack/)

### Ressources
- [Cloud Native Ref](https://github.com/Smana/cloud-native-ref) — mon repo de référence (EKS, Crossplane, Flux, vLLM…)
- [SWE-bench Leaderboards](https://www.swebench.com/) — benchmark agentique de référence
