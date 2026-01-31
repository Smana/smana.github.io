+++
author = "Smaine Kahlouch"
title = "Quelques mois avec `Claude Code` : tips et workflows qui m'ont été utiles"
date = "2026-01-29"
summary = "CLAUDE.md, hooks, gestion du contexte, worktrees, plugins, anti-patterns : tout ce que j'aurais aimé savoir dès le début."
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

Comme pour tout outil qu'on adopte, c'est avec le temps qu'on affine sa façon de l'utiliser. À force d'itérer sur ma config et mes workflows, j'ai trouvé un rythme efficace avec Claude Code. Je partage ici ce qui fonctionne pour moi.

{{% notice info "Article vivant" %}}
Cet article sera mis à jour au fil de mes découvertes et de l'évolution des outils. N'hésitez pas à revenir de temps en temps pour y trouver de nouveaux tips.
{{% /notice %}}

---

## :scroll: CLAUDE.md : la mémoire persistante

Le fichier `CLAUDE.md` est le **premier levier d'optimisation**. Ce sont des instructions injectées automatiquement dans chaque conversation. Si vous n'en avez pas encore, c'est la première chose à mettre en place.

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

Voici un extrait condensé du `CLAUDE.md` de [cloud-native-ref](https://github.com/Smana/cloud-native-ref) pour illustrer :

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

On y retrouve les trois ingrédients essentiels : les **commandes de build/deploy** en premier (pour que Claude sache valider son travail), les **conventions de nommage**, et les **pièges spécifiques** que Claude reproduirait sans cesse si on ne le prévenait pas.

{{% notice warning "Règle des ~500 lignes" %}}
Chaque token de `CLAUDE.md` est chargé à **chaque conversation**. Un fichier trop long gaspille du contexte précieux. Visez ~500 lignes maximum et déplacez les instructions spécialisées vers des **Skills** qui ne se chargent qu'à la demande.
{{% /notice %}}

### Itérer sur CLAUDE.md

Traitez `CLAUDE.md` comme un **prompt de production** : itérez dessus régulièrement. Le raccourci `#` permet de demander à Claude lui-même de suggérer des améliorations à votre fichier. Vous pouvez aussi utiliser le [prompt improver](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-improver) d'Anthropic pour affiner les instructions.

---

## :hook: Hooks : mon premier automatisme

Les **hooks** sont des commandes shell exécutées automatiquement en réponse à des événements Claude Code. La différence clé avec `CLAUDE.md` : les instructions du CLAUDE.md sont *consultatives* (Claude peut les ignorer), les hooks sont **déterministes** — ils s'exécutent systématiquement.

### La notification quand Claude attend

Le premier hook que j'ai configuré — et celui que je recommande à tout le monde — c'est la **notification desktop**. Quand Claude termine une tâche ou attend votre validation, vous recevez une notification système accompagnée d'un son. Fini de checker le terminal toutes les 30 secondes.

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
            "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION\" --icon=dialog-information && paplay /usr/share/sounds/freedesktop/stereo/complete.oga"
          }
        ]
      }
    ]
  }
}
```

Sous Linux, `notify-send` est fourni par le paquet `libnotify` et `paplay` par `pulseaudio-utils` (ou `pipewire-pulse`). D'autres mécanismes existent pour macOS (`osascript`) ou d'autres environnements — consultez la [doc des hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) pour les alternatives.

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

### `/statusline` : un tableau de bord permanent

Plutôt que de lancer `/context` manuellement, vous pouvez configurer une **barre de statut permanente** en bas du terminal. La commande `/statusline` accepte un prompt en langage naturel pour personnaliser l'affichage. Voici ce que j'utilise :

```
/statusline show current directory, git branch, and context usage percentage.
Use ANSI colors: green for directory, cyan for git branch, yellow for context
percentage. Use unicode separators and icons like ⎇ for branch
```

{{< img src="statusline.png" width="550" >}}

D'un coup d'oeil, on voit le **répertoire courant**, la **branche git active** et le **pourcentage de contexte utilisé** — trois informations essentielles pour savoir où on en est sans interrompre son flow. Particulièrement utile quand on jongle entre plusieurs worktrees ou qu'on veut anticiper un `/compact`.

C'est ma configuration — le prompt est libre, donc vous pouvez l'adapter pour afficher ce qui compte le plus pour vous.

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

En pratique, j'alterne entre deux modes : parfois en terminal pur — vieille habitude — et parfois en mode hybride avec **Cursor** pour l'édition et **Claude Code** dans le terminal. Le workflow hybride est clairement plus confortable, et je m'y mets de plus en plus.

{{< img src="cursor+claude.png" alt="Cursor + Claude Code" width="1200" >}}

| Besoin | Outil | Pourquoi |
|--------|-------|----------|
| Édition rapide, autocomplete | Cursor | Latence minimale, vous restez dans le flow |
| Refactoring, debugging multi-fichiers | Claude Code | Raisonnement profond, boucles autonomes |

**Ce que j'apprécie dans le mode hybride** : Claude modifie via le terminal, et je valide les diffs dans l'interface Cursor — bien plus lisible que `git diff`. Les changements apparaissent en temps réel dans l'éditeur, ce qui permet de suivre ce que Claude fait et d'intervenir rapidement si besoin.

---

## :electric_plug: Plugins

Claude Code dispose d'un écosystème de **plugins** qui étendent ses capacités. Voici les deux que j'utilise au quotidien :

### Code-Simplifier : nettoyer le code généré

Le plugin **[code-simplifier](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-simplifier)** est développé par Anthropic et utilisé en interne par l'équipe Claude Code. Il nettoie automatiquement le code généré par l'IA tout en préservant la fonctionnalité.

J'ai découvert ce plugin récemment et je compte m'efforcer de l'utiliser systématiquement **avant de créer une PR** après une session intensive. Il tourne sur Opus et devrait aider à réduire la dette technique introduite par le code IA — code dupliqué, structures inutilement complexes, style incohérent.

### Claude-Mem : mémoire persistante entre sessions

Le plugin **[claude-mem](https://github.com/thedotmack/claude-mem)** capture automatiquement le contexte de vos sessions et le réinjecte dans les sessions futures. Plus besoin de réexpliquer votre projet à chaque nouvelle conversation.

**Ses deux atouts principaux :**
- **Recherche sémantique** : retrouver facilement une information d'une session passée via le skill `mem-search`
- **Optimisation de la consommation de tokens** : un workflow 3-layer qui réduit significativement l'usage (~10x d'économie)

Exemples d'utilisation :

  - "Cherche dans mes sessions quand j'ai debuggé Karpenter"
  - "Retrouve ce que j'ai appris sur OpenBao PKI la semaine dernière"
  - "Regarde mon travail précédent sur la composition App"

{{% notice warning "Considérations de confidentialité" %}}
Claude-mem stocke localement les données de session. Pour les projets sensibles, utilisez les tags `<private>` pour exclure des informations de la capture.
{{% /notice %}}

---

## :warning: Anti-patterns

Voici les pièges dans lesquels je suis tombé — et que j'ai appris à éviter :

| Anti-pattern | Symptôme | Solution |
|--------------|----------|----------|
| **Kitchen sink session** | Mélanger debugging, feature, refactoring dans une même session | `/clear` entre chaque tâche distincte |
| **Spirale de corrections** | Claude corrige un bug, en crée un autre, boucle sans fin | Arrêter, `/clear`, reformuler avec plus de contexte |
| **CLAUDE.md trop gros** | Context consommé dès le départ, réponses dégradées | Viser ~500 lignes, déplacer le reste vers les Skills |
| **Trust-then-verify gap** | Accepter le code sans review, découvrir les bugs en prod | Toujours lire le diff avant de commit |
| **Exploration infinie** | Claude parcourt tout le codebase au lieu d'agir | Donner des fichiers/chemins précis dans le prompt |

{{% notice tip "Le piège le plus insidieux" %}}
La **spirale de corrections** est de loin le plus dangereux. Exemple vécu : Claude devait ajouter une `CiliumNetworkPolicy` à une composition Crossplane. Premier essai, mauvais sélecteur d'endpoint. Il corrige, mais casse le format KCL. Il re-corrige le format, mais revient au mauvais sélecteur initial. Au bout de 5 itérations et ~40K tokens consommés, j'ai fait `/clear` et reformulé en 3 lignes en précisant le namespace cible et un exemple de policy existante. Résultat correct du premier coup. La leçon : quand Claude boucle après 2-3 tentatives, c'est le signe qu'il lui manque du **contexte**, pas de la persévérance. Mieux vaut couper et reformuler que de le laisser tourner.
{{% /notice %}}

---

## :checkered_flag: Conclusion

Au fil du temps, je me sens de plus en plus à l'aise avec Claude Code, et le gain de productivité est réel. Mais il s'accompagne d'une crainte que je n'arrive pas à évacuer complètement : celle de perdre le contrôle — sur le code produit, sur les décisions prises, sur la compréhension de ce qui tourne en production.

Ces interrogations, mais aussi les méthodes qui me permettent d'être le maître de la situation, je les aborde dans le premier article de cette série. Si vous voulez revenir aux fondamentaux ou comprendre d'où viennent ces réflexions, je vous le conseille vivement : [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/).

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

### Article précédent
- [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) — Partie 1 de la série Agentic AI
