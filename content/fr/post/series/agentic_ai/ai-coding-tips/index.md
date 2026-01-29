+++
author = "Smaine Kahlouch"
title = "`Claude Code` : Optimisation et bonnes pratiques pour le Platform Engineer"
date = "2026-01-28"
summary = "Tips avancés pour tirer le meilleur de Claude Code : CLAUDE.md, hooks, gestion du contexte, workflows multi-sessions, plugins, anti-patterns à éviter et intégration CI/CD."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
aliases = ["/fr/post/ai-coding-tips/"]
tags = ["ai", "devxp", "tooling"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — Partie 2" %}}
Cet article fait suite à [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/), où nous avons exploré les fondamentaux de l'IA agentique — tokens, MCPs, Skills, Tasks — et deux cas pratiques détaillés. **Ici, on passe à la pratique avancée** : comment tirer le maximum de Claude Code au quotidien.
{{% /notice %}}

Après plusieurs mois d'utilisation intensive de Claude Code, certains patterns se dégagent clairement. Entre les sessions perdues faute de contexte, les hooks qui auraient pu tout automatiser, et les mauvaises habitudes qui font perdre un temps fou — voici les **tips et workflows qui ont transformé ma façon de travailler**.

---

## :scroll: CLAUDE.md : le cerveau persistant

Le fichier `CLAUDE.md` est le **premier levier d'optimisation**. C'est de la mémoire persistante : des instructions injectées automatiquement dans chaque conversation. Si vous n'en avez pas encore, c'est la première chose à mettre en place.

### Hiérarchie de chargement

Claude Code charge les fichiers `CLAUDE.md` selon une hiérarchie précise :

| Emplacement | Portée | Cas d'usage |
|-------------|--------|-------------|
| `~/.claude/CLAUDE.md` | Toutes les sessions, tous les projets | Préférences globales (langue, style de commit) |
| `./CLAUDE.md` | Projet (partagé via git) | Conventions d'équipe, commandes de build/test |
| `./CLAUDE.local.md` | Projet (non versionné) | Préférences personnelles sur ce projet |
| `./sous-dossier/CLAUDE.md` | Sous-arborescence | Instructions spécifiques à un module |

Les fichiers sont **cumulatifs** : Claude les charge tous du plus global au plus local. En cas de conflit, le plus local l'emporte.

### Ce que je mets dans le mien

Voici ce que contient le `CLAUDE.md` de [cloud-native-ref](https://github.com/Smana/cloud-native-ref), concrètement :

- **Commandes de build/test/lint** — les premières lignes, pour que Claude sache comment valider son travail
- **Conventions du projet** — préfixe `xplane-*` pour l'IAM, structure des compositions Crossplane, patterns KCL
- **Architecture résumée** — structure des dossiers clés, pas un roman
- **Erreurs fréquentes** — les pièges que Claude retombe dedans si on ne le prévient pas (ex: mauvais namespace par défaut, format des labels)

Ce que je n'y mets **pas** : la documentation exhaustive (c'est le rôle des [Skills](/fr/post/series/agentic_ai/ai-coding-agent/#skills--obtenir-de-nouveaux-pouvoirs)), les exemples de code longs (je référence les fichiers existants), et les instructions évidentes que Claude connaît déjà.

{{% notice warning "Règle des ~500 lignes" %}}
Chaque token de `CLAUDE.md` est chargé à **chaque conversation**. Un fichier trop long gaspille du contexte précieux. Visez ~500 lignes maximum et déplacez les instructions spécialisées vers des **Skills** qui ne se chargent qu'à la demande.
{{% /notice %}}

### Itérer sur CLAUDE.md

Traitez `CLAUDE.md` comme un **prompt de production** : itérez dessus régulièrement. Le raccourci `#` permet de demander à Claude lui-même de suggérer des améliorations à votre fichier. Vous pouvez aussi utiliser le [prompt improver](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-improver) d'Anthropic pour affiner les instructions.

---

## :hook: Hooks : mon premier automatisme

Les **hooks** sont des commandes shell exécutées automatiquement en réponse à des événements Claude Code. La différence clé avec `CLAUDE.md` : les instructions du CLAUDE.md sont *consultatives* (Claude peut les ignorer), les hooks sont **déterministes** — ils s'exécutent systématiquement.

### La notification quand Claude attend

Le premier hook que j'ai configuré — et celui que je recommande à tout le monde — c'est la **notification desktop**. Quand Claude termine une tâche ou attend votre validation, vous recevez une notification système. Fini de checker le terminal toutes les 30 secondes.

Configuration dans `~/.claude/settings.json` :

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION\" --icon=dialog-information"
          }
        ]
      }
    ]
  }
}
```

Sous Linux, `notify-send` est fourni par le paquet `libnotify`. Sur macOS, remplacez par `osascript -e 'display notification "..."'`.

### Les autres possibilités

Les hooks couvrent plusieurs événements (`PreToolUse`, `PostToolUse`, `Notification`, `Stop`, `SessionStart`) et permettent par exemple :

- **Auto-format** après chaque édition (ex: `gofmt` sur les fichiers Go)
- **Protection de fichiers sensibles** — un hook `PreToolUse` qui bloque l'écriture dans les `.env`, `.pem`, `.key` (exit code 2 = action bloquée)
- **Audit** des commandes exécutées dans un fichier de log

Je ne détaille pas ici chaque variante — la [doc officielle des hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) est très bien faite. L'essentiel c'est de comprendre que les hooks sont votre filet de sécurité **déterministe** là où `CLAUDE.md` n'est qu'un conseil.

---

## :brain: Maîtriser la fenêtre de contexte

La fenêtre de contexte (200K tokens) est **la ressource la plus critique**. Une fois saturée, les informations anciennes sont compressées et la qualité se dégrade. C'est LE sujet qui fait la différence entre un utilisateur efficace et quelqu'un qui "perd" Claude au bout de 20 minutes.

### `/compact` avec instructions custom

La commande `/compact` compresse l'historique de conversation tout en préservant les décisions clés. L'astuce : vous pouvez passer des **instructions de focus** pour garder ce qui compte :

```
/compact focus on the Crossplane composition decisions and ignore the debugging steps
```

C'est particulièrement utile après une longue session de debugging où 80% du contexte est du bruit (tentatives échouées, stack traces).

### Stratégie `/clear`

La commande `/clear` remet le contexte à zéro. Quand l'utiliser ?

- **Toujours** entre deux tâches distinctes — c'est la règle la plus importante
- Quand Claude commence à **halluciner** ou à répéter des erreurs
- Après une longue session de debugging (le contexte est pollué par les tentatives échouées)
- Quand `/context` montre un espace libre < 20%

J'ai pris l'habitude de commencer chaque nouvelle tâche par un `/clear`. Ça semble contre-intuitif (on perd le contexte), mais en pratique c'est bien plus efficace qu'un contexte pollué.

### Tool Search : quand les MCPs bouffent le contexte

Le problème : chaque MCP activé injecte ses **définitions d'outils** dans le contexte. Avec 6-7 MCPs configurés (ce qui est courant en platform engineering), ça peut représenter plus de **10% de votre fenêtre** — consommé avant même de commencer à travailler.

La solution : activer le **Tool Search** :

```bash
export ENABLE_TOOL_SEARCH=auto:10
```

Avec cette option, Claude ne charge les définitions d'outils que **quand il en a besoin**, plutôt que de toutes les garder en mémoire. Le `10` est le seuil (en pourcentage du contexte) à partir duquel le mécanisme s'active.

### Délégation aux subagents

Pour les tâches d'exploration volumineuses (parcourir un codebase, chercher dans les logs), Claude peut déléguer à des **subagents** qui ont leur propre contexte isolé. Le résultat condensé est renvoyé à la session principale — économisant ainsi de précieux tokens.

En pratique, Claude le fait automatiquement quand il juge que c'est pertinent (exploration de fichiers, recherches larges). Mais vous pouvez aussi le guider explicitement : *"utilise un subagent pour explorer la structure du module networking"*.

### Quelques règles simples

1. **`/clear` entre les tâches** : Chaque nouvelle tâche devrait commencer par un `/clear`
2. **CLAUDE.md concis** : Chaque token est chargé à chaque conversation
3. **CLIs > MCPs** : Pour les outils matures (`kubectl`, `git`, `gh`...), préférez la CLI directe — les LLMs les maîtrisent parfaitement et ça évite de charger un MCP
4. **`/context` pour auditer** : Identifiez ce qui consomme et désactivez les MCPs non utilisés

---

## :arrows_counterclockwise: Workflows multi-sessions

### Git Worktrees : paralléliser les sessions

Plutôt que de jongler avec des branches et du `stash`, j'utilise les **git worktrees** pour travailler sur plusieurs features en parallèle avec des sessions Claude indépendantes.

```console
# Créer deux features en parallèle
git worktree add ../worktrees/feature-a -b feat/feature-a
git worktree add ../worktrees/feature-b -b feat/feature-b

# Lancer deux sessions Claude distinctes
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

### Pattern Writer/Reviewer

Un pattern que j'utilise de plus en plus consiste à lancer **deux sessions en parallèle** :

| Session | Rôle | Prompt |
|---------|------|--------|
| **Writer** | Implémente le code | "Implémente la feature X selon la spec" |
| **Reviewer** | Review le code | "Review les changements sur la branche feat/X, vérifie la sécurité et les edge cases" |

Le reviewer travaille sur le même repo (via worktree ou en lecture seule) et fournit un feedback indépendant, sans le biais de l'auteur. C'est particulièrement efficace pour les changements d'infrastructure où une erreur peut coûter cher.

### Fan-out avec `-p`

Le flag `-p` (prompt non-interactif) permet d'exécuter Claude en mode **headless** et de paralléliser les tâches :

```bash
# Lancer 3 tâches en parallèle
claude -p "Ajoute des tests unitaires pour le module auth" &
claude -p "Documente l'API REST du service orders" &
claude -p "Refactore le module billing pour utiliser le nouveau SDK" &
wait
```

Chaque instance a son propre contexte. C'est idéal pour les tâches indépendantes qui ne nécessitent pas d'interaction.

---

## :desktop_computer: Workflow hybride IDE + Claude Code

C'est l'approche que j'utilise au quotidien : **Cursor** pour l'édition et la visualisation, **Claude Code** pour les tâches agentiques dans le terminal.

{{< img src="cursor+claude.png" alt="Cursor + Claude Code" width="1000" >}}

| Besoin | Outil | Pourquoi |
|--------|-------|----------|
| Édition rapide, autocomplete | Cursor | Latence minimale, vous restez dans le flow |
| Refactoring, debugging multi-fichiers | Claude Code | Raisonnement profond, boucles autonomes |

**Le vrai gain** : Claude modifie via le terminal, et je valide les diffs dans l'interface Cursor — bien plus lisible que `git diff`. Les changements apparaissent en temps réel dans l'éditeur, ce qui permet de suivre ce que Claude fait et d'intervenir rapidement si quelque chose ne va pas.

---

## :electric_plug: Plugins

Claude Code dispose d'un écosystème de **plugins** qui étendent ses capacités. Voici les deux que j'utilise au quotidien :

### Code-Simplifier : nettoyer le code généré

Le plugin **[code-simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)** est développé par Anthropic et utilisé en interne par l'équipe Claude Code. Il nettoie automatiquement le code généré par l'IA tout en préservant la fonctionnalité.

J'utilise systématiquement le code-simplifier **avant de créer une PR** après une session intensive. Il tourne sur Opus et peut significativement réduire la dette technique introduite par le code IA — code dupliqué, structures inutilement complexes, style incohérent.

### Claude-Mem : mémoire persistante entre sessions

Le plugin **[claude-mem](https://github.com/thedotmack/claude-mem)** capture automatiquement le contexte de vos sessions et le réinjecte dans les sessions futures. Plus besoin de réexpliquer votre projet à chaque nouvelle conversation.

**Ce que j'apprécie particulièrement :**
- **Capture automatique** : Enregistre les actions Claude (tool usage, décisions)
- **Recherche sémantique** : Skill `mem-search` pour retrouver des informations passées
- **Interface web** : Dashboard sur `http://localhost:37777`
- **Workflow 3-layer** : Optimise la consommation de tokens (~10x d'économie)

Exemples d'utilisation :

  - "Search my memories for when I debugged Karpenter"
  - "Find what I learned about OpenBao PKI last week"
  - "Look up my previous work on the App composition"

{{% notice warning "Considérations de confidentialité" %}}
Claude-mem stocke localement les données de session. Pour les projets sensibles, utilisez les tags `<private>` pour exclure des informations de la capture.
{{% /notice %}}

---

## :warning: Anti-patterns

Après plusieurs mois d'utilisation, voici les pièges dans lesquels je suis tombé — et que j'ai appris à éviter :

| Anti-pattern | Symptôme | Solution |
|--------------|----------|----------|
| **Kitchen sink session** | Mélanger debugging, feature, refactoring dans une même session | `/clear` entre chaque tâche distincte |
| **Spirale de corrections** | Claude corrige un bug, en crée un autre, boucle sans fin | Arrêter, `/clear`, reformuler avec plus de contexte |
| **CLAUDE.md obèse** | Context consommé dès le départ, réponses dégradées | Viser ~500 lignes, déplacer le reste vers les Skills |
| **Trust-then-verify gap** | Accepter le code sans review, découvrir les bugs en prod | Toujours lire le diff avant de commit |
| **Exploration infinie** | Claude parcourt tout le codebase au lieu d'agir | Donner des fichiers/chemins précis dans le prompt |

{{% notice tip "Le piège le plus insidieux" %}}
La **spirale de corrections** est de loin le plus dangereux. Quand Claude boucle sur un même problème après 2-3 tentatives, c'est le signe qu'il lui manque du contexte. J'ai appris à mes dépens que mieux vaut `/clear` et reformuler avec les informations manquantes plutôt que de le laisser tourner. Le coût en tokens (et en frustration) est exponentiel.
{{% /notice %}}

---

## :bulb: Ce qui fonctionne bien vs vigilance

| Ce que Claude fait bien | Vigilance requise | Pourquoi |
|------------------------|--------------------|---------:|
| Debugging avec contexte | Création from scratch | Sans contexte existant, les hallucinations augmentent |
| Conversion de formats | Sécurité / PKI | Les erreurs crypto sont silencieuses et critiques |
| Refactoring répétitif | Ressources cloud coûteuses | Un `apply` mal placé peut coûter cher |
| Analyse de dépendances | Breaking changes | Claude ne connaît pas vos contrats implicites |
| Génération de tests | Tests de performance | Les benchmarks nécessitent une validation humaine |
| Documentation technique | Choix d'architecture | Les décisions structurantes méritent un HITL |

---

## :gear: Intégration CI/CD

Claude Code s'intègre dans les pipelines CI/CD grâce au mode headless et à l'action GitHub officielle. C'est là que l'investissement dans les Skills et CLAUDE.md paye : les mêmes conventions que vous utilisez en local sont appliquées automatiquement en CI.

### GitHub Actions : `claude-code-action`

L'action [claude-code-action](https://github.com/anthropics/claude-code-action) permet d'automatiser des workflows directement dans GitHub :

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Review this PR for:
            - Security vulnerabilities
            - Performance issues
            - Kubernetes best practices
            Post your findings as PR comments.
```

### Mode headless pour scripts custom

Le flag `-p` combiné avec `--output-format json` permet d'intégrer Claude dans n'importe quel script :

```bash
# Exemple : review automatique de PR
PR_DIFF=$(gh pr diff $PR_NUMBER)
REVIEW=$(claude -p "Review ce diff pour les problèmes de sécurité : $PR_DIFF" \
  --output-format json \
  --max-turns 5)
echo "$REVIEW" | jq -r '.result' | gh pr comment $PR_NUMBER --body -
```

### Workflows automatisables

| Workflow | Déclencheur | Prompt type |
|----------|-------------|-------------|
| **PR Review** | `pull_request.opened` | "Review for security, performance, K8s best practices" |
| **Issue Triage** | `issues.opened` | "Classify this issue, suggest labels and priority" |
| **Release Notes** | `release.created` | "Generate changelog from commits since last release" |
| **Security Audit** | `schedule` (cron) | "Scan for outdated dependencies and known CVEs" |

---

## :checkered_flag: Conclusion

Si je devais résumer les tips essentiels de cet article :

- **`CLAUDE.md` concis et itéré** — c'est votre levier #1, traitez-le comme un prompt de production
- **Un hook notification** — le minimum vital pour ne pas perdre du temps à attendre
- **`/clear` entre les tâches** — la règle la plus simple et la plus impactante
- **Tool Search activé** — indispensable dès que vous avez plusieurs MCPs
- **Worktrees pour paralléliser** — une branche = une session = un contexte propre
- **Toujours lire le diff** — même quand "ça a l'air bon"

L'IA agentique est un outil puissant, mais comme tout outil, c'est la maîtrise des bonnes pratiques qui fait la différence entre un gain réel de productivité et une perte de temps déguisée.

{{% notice info "Retour à l'article 1" %}}
Pour revoir les fondamentaux (boucle agentique, tokens, MCPs, Skills, Tasks) et les cas concrets de Platform Engineering, consultez [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/).
{{% /notice %}}

---

## :bookmark: Références

### Documentation officielle
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Engineering
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code) — Guide officiel
- [Hooks Documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) — Configuration des hooks

### Guides communautaires
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) — Guide complet par sshh
- [CC-DevOps-Skills](https://github.com/akin-ozer/cc-devops-skills) — 31 skills DevOps

### Plugins et outils
- [Code-Simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier) — Nettoyage de code IA (Anthropic)
- [Claude-Mem](https://github.com/thedotmack/claude-mem) — Mémoire persistante entre sessions
- [claude-code-action](https://github.com/anthropics/claude-code-action) — GitHub Actions officielle

### Article précédent
- [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) — Partie 1 de la série Agentic AI
