+++
author = "Smaine Kahlouch"
title = "`Agent Factory` : Maku, le chef d'orchestre"
date = "2026-11-21"
summary = "Un prompt, et la factory s'occupe de tout jusqu'à la PR. Une démo sert de fil conducteur pour découvrir `Maku` : ses composants, la room, les workflows et les merge policies."
featured = false
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.jpg"
+++

{{% notice info "Série Agent Factory 🏭" %}}
1. [Vue d'ensemble : agents et humains, même équipage](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)
2. **Maku, le chef d'orchestre** (vous êtes ici)
<br>(à venir)
1. Workflows et merge policies
2. Le harnais : une sandbox pour chaque agent
3. Observer et maîtriser une flotte d'agents
4. D'agent à agent : RunLore confie ses correctifs à la factory
{{% /notice %}}

Faire tourner un agent dans un pod est devenu assez courant, en témoignent les sujets abordés lors des dernières conférences (notamment la KubeCon NA). Faire tourner une **software factory pour une équipe**, c'est une autre histoire : il faut coordonner plusieurs agents, tenir les budgets, isoler chaque exécution, décider de ce qui part en production… et surtout faire en sorte que tout cela reste **agréable à utiliser**. Si déléguer une tâche demande de quitter ses outils habituels, personne ne le fera.

C'est le rôle de **[Maku](https://github.com/Smana/maku)**, le cœur de l'Agent Factory. On ne l'avait abordé que succinctement dans le [premier article](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/) ; il est temps d'ouvrir la boîte.

Cette fois, je ne vais pas commencer par l'architecture. Partons d'un cas hyper simple : **j'ai un changement à faire, et je le confie à la factory depuis mon coding agent**. On va regarder ce que ça donne pour ensuite décrire les composants qui entrent en jeu. 🚀

## 🎯 Objectifs de cet article

* 🎬 Suivre une **vraie tâche**, du prompt jusqu'au merge
* 🏗️ Comprendre ce qui se passe sous le capot : les **composants de Maku** et la **room**
* 🚪 Découvrir les autres portes d'entrée : **Slack** et **GitHub**
* 🔄 Avoir un premier aperçu des **workflows** et des **merge policies** (le prochain article leur est consacré)

## 🎬 Une tâche déléguée sans quitter son terminal

<center>
<video width="1200" controls autoplay loop muted playsinline>
  <source src="maku-demo.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la lecture vidéo.
</video>
</center>
<center><i>La version longue (2 minutes) est sur la <a href="https://blog.ogenki.io/maku/">page de démo</a>.</i></center>
<br>

Le contexte : je travaille sur [airef.ogenki.io](https://airef.ogenki.io/) et, en passant, je remarque une erreur dans la référence de configuration : le budget de l'app Slack n'affiche pas les bonnes valeurs.

Le correctif est minime, bien délimité et facile à vérifier : typiquement le genre de tâche que je confie à Maku pour passer à autre chose. Sans quitter ma session OpenCode, je délègue :

```text
/maku handoff the config reference has the wrong Slack app budget; fix it to dailyUSD 5, maxCostUSD 2
```

Et voici ce qui se passe :

1. 📝 **Le planner de Maku rédige l'issue.** À partir de ma phrase, il construit une première proposition (ce qui ne va pas, ce qui est attendu, le critère d'acceptation) que je peux ajuster avant de la valider.

2. ✅ **Je confirme.** Un formulaire, auquel mon coding agent ne peut pas répondre à ma place, me montre l'issue, le workflow (`standard`) et le budget de la tâche (5,10 $ au maximum). Je valide : l'[issue #479](https://github.com/Smana/ai-native-ref/issues/479) est créée avec mon compte GitHub, la tâche démarre, et je retourne à mon travail.

3. 🤖 **Un agent s'y met.** Quelque part sur le cluster, il corrige la page, lance les checks du dépôt (6 sur 6), puis ouvre la [PR #480](https://github.com/Smana/ai-native-ref/pull/480) : une ligne modifiée dans un fichier.

4. 👀 **Un autre agent relit.** Il repart de zéro, sans connaître le raisonnement du premier, relance les checks et rend son verdict : `approve`. S'il avait demandé des changements (`changes`), l'implementer aurait repris la PR pour traiter ses remarques : le reviewer, lui, n'a pas le droit de pousser du code.

5. 🔔 **On me prévient.** Je n'ai rien surveillé : mon coding agent suivait la tâche en arrière-plan, et la room me dit ce qu'on attend de moi :

   ```text
   Needs you · 1
   Approve the pull request
   a maintainer approves PR #480: approve the head you read on the PR page, […]; the merger merges that head
   Why: it would auto-merge, and the merge gate is in shadow
   ```

6. 🎉 **J'approuve, c'est mergé.** Je relis le diff, la CI est verte, je clique sur **Approve**. Trente secondes plus tard, la PR est mergée et l'issue fermée.

De mon côté : une phrase dans le terminal, un formulaire, la relecture d'une ligne et un clic. Douze minutes de bout en bout, pour 0,14 $.

Pour en arriver là, l'installation tient en une commande :

```bash
# Sur mon poste, une fois par coding agent
maku setup opencode      # ou claude, codex, gemini, cursor, copilot
```

Cette commande ouvre le SSO, enregistre le serveur MCP `maku` et, pour Claude Code et OpenCode, ajoute aussi la commande `/maku` :

* `/maku handoff <description>` : celle qu'on vient de voir
* `/maku task [#issue]` : où en est la tâche, ce qu'elle attend de moi, combien elle a coûté
* `/maku inbox` : toutes mes tâches qui attendent une action humaine
* `/maku cancel <tâche>` : arrêter une tâche, ou l'abandonner avec sa PR et sa branche

{{% notice info "Prérequis" %}}
`maku setup` se branche sur une factory **déjà déployée** : il demande l'URL de ses rooms. Pas encore de cluster ? `task demo` monte tout sur un laptop (voir [plus bas](#-la-factory-chez-vous-)).
{{% /notice %}}

## 🔍 L'envers du décor

Une phrase, une issue, deux agents, un verdict, un merge : qui a fait quoi ? C'est le moment de regarder sous le capot.

Maku suit le modèle des **opérateurs Kubernetes** : tout est déclaré dans des CRD (`Task`, `Room`, `AgentRun`, `MergePolicy`…) que quelques contrôleurs écrits en Go réconcilient en boucle. Une tâche est donc une ressource Kubernetes comme une autre : on peut faire un `kubectl get tasks`, et ça, je kiffe ! 😄

{{< img src="maku-components.png" alt="Les composants de Maku : quatre portes d'entrée (coding agent, Web UI, Slack, GitHub), le control plane avec maku-broker, maku-slack, maku-factory et maku-run-controller, la sandbox gVisor de chaque run avec maku-harness et maku-bridge, et maku-gateway comme unique porte de sortie vers les modèles et les outils MCP" width="1300" link="true" caption="Quatre portes d'entrée, un control plane, une sandbox par run." >}}
<br>

Reprenons la démo, composant par composant :

* 📓 **`maku-broker`**, la mémoire. Son planner a rédigé l'issue à partir de mon prompt, via le serveur MCP auquel mon coding agent est branché. Il tient aussi le journal de chaque **room** dans PostgreSQL et sert la Web UI.

* 🏭 **`maku-factory`**, l'aiguilleur. Il a repéré mon issue, créé la `Task`, choisi le workflow et le budget, lancé les agents, puis appliqué le **merge gate** à la fin.

* 🚀 **`maku-run-controller`**, le lanceur. Pour chaque agent, il crée un pod isolé dans une sandbox **gVisor**, avec son propre `ServiceAccount` et un réseau fermé par défaut. Sur mon cluster, c'est une composition Crossplane de [cloud-native-ref](https://github.com/Smana/cloud-native-ref) qui joue ce rôle.

* 🤖 **`maku-harness`**, l'agent. C'est l'image qui tourne dans la sandbox : [OpenHands](https://github.com/OpenHands/OpenHands), `gh`, et de quoi ouvrir la PR.

* 🌉 **`maku-bridge`**, le messager. Ce sidecar remonte les événements de l'agent vers la room et redescend mes consignes (les *steers*, on y revient plus bas). L'agent, lui, ne détient jamais le token de la room.

* 🚪 **`maku-gateway`**, la porte de sortie. Il s'agit en fait d'[agentgateway](https://agentgateway.dev/), que Maku se contente de configurer. C'est le seul chemin vers les modèles et les outils MCP. Elle vérifie l'identité de chaque run et applique les plafonds de dépense : c'est d'elle que viennent le budget et le coût affichés dans la room.

Côté GitHub, chez moi, trois GitHub Apps se partagent les droits : celle de la factory (issues, labels, commentaires), celle des agents (qui poussent une branche et ouvrent la PR), et celle du *merger*, la seule autorisée à merger.

## 🎛️ La room : tout voir, garder la main

Chaque tâche a sa **room** : l'espace où l'on suit tout ce qui s'y passe. Dans la démo, son lien s'affichait dans mon terminal.

{{< img src="room.gif" width="480" >}}

Dans *One Piece*, la *Room* de Trafalgar Law est une sphère hors de laquelle son pouvoir ne fonctionne pas, mais où il peut tout examiner, tout découper et tout réassembler, sans que rien ne meure. Une room Maku joue le même rôle pour une tâche : tout s'y voit, tout s'y pilote, et rien ne s'y perd.

Pourquoi ne pas se contenter des logs du pod ? Parce qu'ils disparaissent avec lui, et qu'ils mélangent ce que l'agent *dit* faire et ce qu'il fait *vraiment*. La room, elle, reste. Elle est :

* 🔒 **Append-only** : le broker peut ajouter et lire des événements, jamais les modifier ni les supprimer. À la fermeture, le journal est scellé.

* 🕵️ **Attribuée et nettoyée** : chaque événement porte l'identité vérifiée de son auteur (humain, agent ou factory), et les secrets sont masqués par [gitleaks](https://github.com/gitleaks/gitleaks) avant l'écriture.

* 🎮 **Pilotable en direct** : on peut laisser une consigne pour le prochain run, réorienter l'agent en cours de route (c'est le *steer*), ou approuver une demande.

* 👥 **Partagée** : une room accueille plusieurs humains et agents. Tout le monde suit et échange, mais un seul *driver* à la fois peut réorienter l'agent, et celui qui a guidé un run ne peut pas approuver ses demandes.

Dans la Web UI, l'encadré **NEXT** dit en une phrase ce qui est attendu de vous :

{{< img src="maku-room.png" alt="Une room Maku qui attend un mainteneur : l'état « Waiting — review and approve PR #42 », le budget en tokens, le dernier verdict du reviewer, l'encadré NEXT qui dit que l'approbation du commit courant vaut merge, l'action exacte sous « Needs you », puis le fil des événements de l'implementer" width="1100" caption="Une room qui attend mon approbation : la prochaine étape, l'action à faire, et le fil des événements du run (à manipuler dans la [démo interactive](https://blog.ogenki.io/maku/))." >}}
<br>

Et quand plusieurs tâches tournent en parallèle, celles qui ont besoin de vous remontent en tête de liste :

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku : les tâches qui attendent un humain portent un badge « Needs you » et remontent en premier, chacune avec son dépôt, son état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms : « Needs you » d'abord." >}}
<br>

## 🚪 Les autres portes d'entrée

Le coding agent est ma porte préférée, mais ce n'est pas la seule. Derrière, le circuit est exactement le même.

### 💬 Depuis Slack

Une remarque au détour d'un canal peut devenir une tâche. On mentionne `@factory`, avec ses propres mots :

```text
@factory le lien d'installation dans le README pointe encore vers l'ancienne page, il devrait cibler docs/install.md
```

Maku répond dans le thread avec une carte de plan :

```text
Plan for Smana/ai-native-ref (revision 1)
docs: the install link in the README is dead
Workflow  [ standard (implementer, reviewer) ▾ ]
Budget    up to $2.00
[ Confirm ]  [ Cancel ]
```

On ajuste en répondant dans le thread, puis un mainteneur clique sur **Confirm** et c'est parti. Ensuite, Slack reste discret : un message au démarrage, un si la tâche a besoin de vous, un à l'ouverture de la PR, et une ligne de bilan à la fin (PR mergée, coût, durée).

### 🐙 Depuis GitHub

La voie classique : on écrit une issue, on pose le label `factory/ready`, et Maku s'en charge. Quelques labels permettent de piloter la suite : `workflow:careful` pour une tâche délicate, `factory/hold` pour geler le merge, `factory/stop` pour tout arrêter.

Sur l'issue, Maku raconte chaque étape : le run en cours, la branche, le budget consommé, et le lien vers la room.

## 🔄 Workflows et merge policies, en deux mots

Dans la démo, deux agents se sont relayés, et la PR m'a attendu au lieu de se merger toute seule. Deux mécanismes en décident ; **le prochain article leur est entièrement consacré**.

### 👥 Les workflows : qui travaille sur la tâche ?

Un agent seul qui écrit le code puis évalue son propre travail finit vite par se convaincre qu'il a terminé. Un **workflow** définit donc quels rôles se relaient sur une tâche :

* **`standard`**, le workflow par défaut, celui de la démo : un **implementer** écrit le changement et ouvre la PR, puis un **reviewer** la relit avec un regard neuf.
* **`quick`** : l'implementer seul, pour les changements triviaux.
* **`careful`** : le workflow `standard`, encadré par un **scoper** qui cadre d'abord la tâche (et pose une question si l'issue est floue) et un **judge** qui vérifie les critères d'acceptation avant la review.

{{% notice tip "Et pour une tâche complexe ?" %}}
La factory ne se limite pas aux petits correctifs. Pour une nouvelle fonctionnalité, tout repose sur une **issue bien rédigée** : des specs claires et des critères d'acceptation vérifiables. Avec `careful`, le scoper s'en sert pour cadrer le travail, le judge pour vérifier le résultat, et l'équipe suit le tout dans la même room.
{{% /notice %}}

### 🚦 Les merge policies : qui a le droit de merger ?

Par défaut, c'est un mainteneur qui merge. Chez moi, j'ai activé l'option `approveToMerge` : **mon approbation du commit courant suffit**, et la GitHub App *merger* s'en charge. Dans la démo, une policy couvrait ce genre de correctif de doc, mais elle est encore en `shadow` : en `live`, elle aurait mergé seule ; là, c'est moi qui ai tranché.

Le merge automatique, lui, se mérite. Une `MergePolicy` décrit une famille de changements à faible risque (corriger la doc, par exemple) : fichiers autorisés, taille du diff, CI verte, verdict du reviewer. Et elle évolue par étapes :

1. **`shadow`** : la policy observe. Elle note ce qu'elle *aurait* décidé, mais c'est toujours un humain qui merge.
2. **`live`** : quand elle a suffisamment de bonnes décisions à son actif, on la promeut en changeant une ligne. Elle merge alors seule, avec un plafond quotidien.
3. **Retour à la case départ** : au premier incident, un disjoncteur coupe la voie et rend la main aux humains.

## 🧭 La factory chez vous ?

**Maku se suffit à lui-même.** Un cluster Kubernetes, un chart Helm, et vous branchez ce que vous avez déjà : votre fournisseur OIDC, votre PostgreSQL, votre CNI. Rien ne vous oblige à reprendre mon infrastructure. Et pour un premier essai, `task demo` monte le tout sur un laptop en un quart d'heure (sans sandbox gVisor, sur un dépôt public, avec votre propre clé de modèle).

Dans mon cas, j'ai réparti les choses sur trois dépôts (on en parlait dans l'[article précédent](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)) :

* **[`maku`](https://github.com/Smana/maku)** : le contrôleur et ses charts, ce que vous installez.
* **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)** : ma configuration de la factory (budgets, modèles autorisés, policies). Elle vit dans son propre dépôt, **hors de portée des agents**.
* **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)** : le cluster qui héberge le tout.

C'est un choix parmi d'autres, pas un prérequis.

{{% notice tip "Je cherche un partenaire 🤝" %}}
J'ai passé beaucoup de temps sur ce projet (et pas mal de tokens 😅). Pour la suite, j'aimerais confronter la solution au terrain : je cherche une **entreprise partenaire** prête à l'essayer avec ses équipes, sur de vrais cas d'usage. Si ça vous tente, faites-moi signe !
{{% /notice %}}

## 💭 Dernières remarques

Au départ, je voulais juste une petite interface web pour lancer des agents dans des sandboxes et garder une trace de ce qu'ils faisaient. Puis je me suis pris au jeu, et de fil en aiguille c'est devenu l'Agent Factory, dont Maku est le cœur.

La boussole, elle, n'a pas bougé : que ce soit **naturel pour le développeur**. Qu'on parte de Claude Code, d'OpenCode, d'un thread Slack ou d'une issue GitHub, toute la machinerie de sécurité reste en coulisses, et l'humain garde le dernier mot là où son jugement compte.

Quelques limites à connaître :

* le planner ne lit que les dépôts publics ;
* l'approbation qui vaut merge est une option, désactivée par défaut ;
* je suis pour l'instant le seul mainteneur à l'utiliser au quotidien ;
* le coût d'une tâche dépend beaucoup du modèle choisi.

Dans le prochain article, on regardera de près les **workflows** et les **merge policies** : comment les rôles se passent le relais, et comment une policy gagne (ou perd) le droit de merger toute seule.

## 🔖 Références

* [`maku`](https://github.com/Smana/maku) : le dépôt · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installation](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart sur un laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Démo : la version longue du screencast et une room à manipuler](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : la plateforme IA · [documentation](https://airef.ogenki.io/) · [guide utilisateur](https://airef.ogenki.io/docs/use/guide/) · [feuille de route](https://airef.ogenki.io/docs/status/)
* [ADR-0057 : pourquoi `ai-native-ref` a son propre dépôt](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [agentgateway](https://agentgateway.dev/) · [OpenHands](https://github.com/OpenHands/OpenHands) · [gVisor](https://gvisor.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
