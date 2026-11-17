+++
author = "Smaine Kahlouch"
title = "Agent Factory : Maku, le chef d'orchestre"
date = "2026-11-16"
summary = "Un changement à faire, une phrase dans le terminal, et la factory s'en occupe jusqu'à la PR. On commence par la démo, puis on ouvre le capot de Maku : ses composants, la room, les workflows et les merge policies."
featured = false
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agent Factory 🏭" %}}
1. [Vue d'ensemble : agents et humains, même équipage](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)
2. **Maku, le chef d'orchestre** (vous êtes ici)
3. Workflows et merge policies (à venir)
4. Le harnais : une sandbox pour chaque agent (à venir)
5. Observer et maîtriser une flotte d'agents (à venir)
6. D'agent à agent : RunLore confie ses correctifs à la factory (à venir)
{{% /notice %}}

Faire tourner un agent dans un conteneur, c'est devenu facile. Faire tourner une **software factory pour une équipe**, c'est une autre histoire : il faut coordonner plusieurs agents, tenir les budgets, isoler chaque exécution, décider de ce qui part en production… et surtout que tout ça reste **agréable à utiliser**. Si déléguer une tâche demande de quitter ses outils, personne ne le fera.

C'est le rôle de **[Maku](https://github.com/Smana/maku)**. Dans le [premier article](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/), ce n'était qu'une boîte au milieu d'un schéma.

Pour une fois, je ne vais pas commencer par l'architecture. Partons plutôt d'un cas tout simple : **j'ai un changement à faire, et je le confie à la factory depuis mon coding agent**. On regarde ce que ça donne, et on ouvre le capot ensuite. 🚀

## 🎯 Objectifs de cet article

* 🎬 Suivre une **vraie tâche**, de la phrase tapée dans le terminal jusqu'au merge
* 🏗️ Comprendre ce qui s'est passé en coulisses : les **composants de Maku** et la **room**
* 🚪 Découvrir les autres portes d'entrée : **Slack** et **GitHub**
* 🔄 Avoir un premier aperçu des **workflows** et des **merge policies** (le prochain article leur est consacré)

## 🎬 Une tâche déléguée sans quitter son terminal

<!-- TODO(smana): intégrer le screencast complet ici dès qu'il est prêt -->
{{% notice note "🎬 Screencast à venir" %}}
Un screencast complet arrive bientôt : il montrera cette démo en conditions réelles. En attendant, la voici pas à pas.
{{% /notice %}}

Le contexte : je travaille sur [airef.ogenki.io](https://airef.ogenki.io/) et, en passant, je remarque un angle mort sur le dashboard Grafana de la flotte d'agents. Quand un fournisseur de modèles refuse une requête, la métrique existe, mais rien ne l'affiche.

Ce n'est pas le moment de m'y plonger : j'ai autre chose en cours. Alors, sans quitter ma session Claude Code, je délègue :

```text
/maku handoff Dans Smana/ai-native-ref, exposer les refus des fournisseurs de modèles sur le dashboard Grafana de la flotte.
```

Et voici ce qui se passe :

1. 🧐 **Une question, une seule.** Faut-il aussi mettre à jour les tests de la CI ? On me le demande, avec une recommandation (oui). J'accepte.

2. ✅ **Je confirme.** Un formulaire me montre l'issue rédigée et le budget. Je valide : l'[issue #354](https://github.com/Smana/ai-native-ref/issues/354) est créée avec mon compte GitHub, et je retourne à mon travail.

3. 🤖 **Un agent s'y met.** Quelque part sur le cluster, il modifie le dashboard et deux fichiers de tests, lance les checks du repo, puis ouvre la [PR #357](https://github.com/Smana/ai-native-ref/pull/357) : 22 lignes, 3 fichiers.

4. 👀 **Un autre agent relit.** Il repart de zéro, sans connaître le raisonnement du premier, et rend son verdict : `approved`.

5. 🔔 **Mon terminal me prévient.** Je n'ai rien surveillé. Mon coding agent attendait en tâche de fond, et il me dit ce qu'on attend de moi :

```text
#354 dashboards: expose model provider refusals              ai-native-ref
State     waiting for a maintainer's review
Needs you review and approve PR #357
Budget    $0.48 of $5.10 (9 %)
Merge     Waits for a human: policy requires a maintainer approval
Room      https://rooms.priv.aws.ogenki.io/r/ozdjcnhn
```

6. 🎉 **J'approuve, c'est mergé.** Je relis les trois fichiers, la CI est verte, je clique sur **Approve**. Moins d'une minute plus tard, la PR est mergée et l'issue fermée.

De mon côté : une phrase dans le terminal, une réponse à une question, trente secondes de relecture et un clic. Coût : 0,48 $.

Pour en arriver là, l'installation tient en une commande :

```bash
maku setup claude      # ou opencode, codex, gemini, cursor, copilot
```

Elle ouvre le SSO, enregistre le serveur MCP `maku` et ajoute trois raccourcis :

* `/maku handoff <description>` : celui qu'on vient de voir
* `/maku task [#issue]` : où en est la tâche, ce qu'elle attend de moi, combien elle a coûté
* `/maku inbox` : toutes mes tâches qui attendent une action humaine

## 🔍 Que s'est-il passé en coulisses ?

Une question, une issue, deux agents, un verdict, un merge : qui a fait quoi ? C'est le moment d'ouvrir la boîte.

Maku suit le modèle des **opérateurs Kubernetes** : tout est déclaré dans des CRDs (`Task`, `Room`, `AgentRun`, `MergePolicy`…) que quelques contrôleurs écrits en Go réconcilient en boucle. Une tâche est donc une ressource Kubernetes comme une autre : on peut faire un `kubectl get tasks`, et ça, j'aime bien. 😄

{{< img src="maku-components.png" alt="Les composants de Maku : quatre portes d'entrée (coding agent, Web UI, Slack, GitHub), le control plane avec maku-broker, maku-slack, maku-factory et maku-run-controller, la sandbox gVisor de chaque run avec maku-harness et maku-bridge, et maku-gateway comme unique porte de sortie vers les modèles et les outils MCP" width="1300" link="true" caption="Quatre portes d'entrée, un control plane, une sandbox par run." >}}
</br>

Reprenons la démo, composant par composant :

* 📓 **`maku-broker`**, la mémoire. C'est lui qui m'a posé la question et préparé le plan, via le serveur MCP auquel mon coding agent est branché. Il tient aussi le journal de chaque **room** dans PostgreSQL et sert la Web UI.

* 🏭 **`maku-factory`**, le chef d'atelier. Il a repéré mon issue, créé la `Task`, choisi le workflow et le budget, lancé les agents, puis appliqué le **merge gate** à la fin.

* 🚀 **`maku-run-controller`**, le lanceur. Pour chaque agent, il a créé un pod isolé dans une sandbox **gVisor**, avec son propre `ServiceAccount` et un réseau fermé par défaut.

* 🤖 **`maku-harness`**, l'agent. C'est l'image qui tourne dans la sandbox : [OpenHands](https://github.com/OpenHands/OpenHands), `gh`, et de quoi ouvrir la PR.

* 🌉 **`maku-bridge`**, le messager. Ce sidecar remonte les événements de l'agent vers la room et redescend mes consignes (les *steers*). L'agent, lui, ne détient jamais le token de la room.

* 🚪 **`maku-gateway`**, la porte de sortie. Basée sur [agentgateway](https://agentgateway.dev/), c'est le seul chemin vers les modèles et les outils MCP. Elle vérifie l'identité de chaque run et applique les plafonds de dépense : c'est de là que vient le « $0.48 of $5.10 ».

Deux garde-fous étaient à l'œuvre sans que je m'en aperçoive :

* **Mon coding agent ne peut rien lancer dans mon dos.** Il prépare le plan, mais c'est moi qui valide le formulaire.
* **Rien n'entre dans une sandbox.** Toutes les connexions sont ouvertes par le pod lui-même, jamais l'inverse.

## 🎛️ La room : tout voir, garder la main

Dans la démo, je n'ai rien regardé. Mais j'aurais pu : le dernier message de mon terminal contenait un lien vers la **room** de la tâche.

{{< img src="room.gif" alt="Trafalgar Law (One Piece) qui déploie sa « Room »" width="480" >}}

Les amateurs de *One Piece* auront reconnu la *Room* de Trafalgar Law : un périmètre dans lequel on voit tout et où l'on garde le contrôle. Le clin d'œil est assumé. 😉

Pourquoi ne pas se contenter des logs du pod ? Parce qu'ils disparaissent avec lui, et qu'ils mélangent ce que l'agent *dit* faire et ce qu'il fait *vraiment*. La room, elle, reste :

* 🔒 **Append-only** : le broker peut ajouter et lire des événements, jamais les modifier ni les supprimer. À la fermeture, le journal est scellé.
* 🕵️ **Attribuée et nettoyée** : chaque événement porte l'identité vérifiée de son auteur (humain, agent ou factory), et les secrets sont masqués par [gitleaks](https://github.com/gitleaks/gitleaks) avant l'écriture.
* 🎮 **Pilotable en direct** : on peut laisser une consigne pour le prochain run, prendre la main pour réorienter l'agent (*steer*), ou approuver une demande.

Dans la Web UI, l'encadré **NEXT** dit en une phrase ce qui est attendu de vous :

{{< img src="maku-room.png" alt="Une room Maku qui attend un mainteneur : l'état « Waiting — review and approve PR #42 », le budget en tokens, le dernier verdict du reviewer, l'encadré NEXT qui dit que l'approbation du commit courant vaut merge, l'action exacte sous « Needs you », puis le fil des événements de l'implementer" width="1100" caption="Une room qui attend mon approbation : la prochaine étape, l'action à faire, et le fil des événements du run (à manipuler dans la [démo interactive](https://blog.ogenki.io/maku/))." >}}
</br>

Et quand plusieurs tâches tournent en parallèle, celles qui ont besoin de vous remontent en tête de liste :

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku : les tâches qui attendent un humain portent un badge « Needs you » et remontent en premier, chacune avec son dépôt, son état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms : « Needs you » d'abord." >}}
</br>

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

On ajuste en répondant dans le thread, puis un mainteneur clique sur **Confirm** et c'est parti. Ensuite, Slack reste discret : un message au démarrage, un si la tâche a besoin de vous, un à l'ouverture de la PR, et une ligne de bilan à la fin.

```text
Merged #418 · $0.16 of $2.00 · 11 min · merged automatically by policy docs-links
```

### 🐙 Depuis GitHub

La voie classique : on écrit une issue, on pose le label `factory/ready`, et Maku s'en charge. Quelques labels permettent de piloter la suite : `workflow:careful` pour une tâche délicate, `factory/hold` pour geler le merge, `factory/stop` pour tout arrêter.

Sur l'issue, Maku raconte chaque étape : le run en cours, la branche, le budget consommé, et le lien vers la room.

## 🔄 Workflows et merge policies, en deux mots

Dans la démo, deux agents se sont relayés, et la PR m'a attendu au lieu de se merger toute seule. Ce n'est pas un hasard : deux mécanismes en décident. Je les survole ici, **le prochain article leur est entièrement consacré**.

### 👥 Les workflows : qui travaille sur la tâche ?

Un agent seul qui écrit le code puis évalue son propre travail finit vite par se convaincre qu'il a terminé. Un **workflow** définit donc quels rôles se relaient sur une tâche :

* **`standard`**, le workflow par défaut, celui de la démo : un **implementer** écrit le changement et ouvre la PR, puis un **reviewer** la relit avec un regard neuf.
* **`quick`** : l'implementer seul, pour les changements triviaux.
* **`careful`** : un **scoper** cadre d'abord la tâche (et pose une question si l'issue est floue), puis un **judge** vérifie que les critères d'acceptation sont remplis avant la review.

Le workflow se choisit par repo, par label, ou directement dans le plan au moment du handoff.

### 🚦 Les merge policies : qui a le droit de merger ?

Par défaut, c'est vous. Maku vous indique la seule action attendue, et **votre approbation du commit vaut merge**. C'est ce qui s'est passé dans la démo : le diff touchait à l'observabilité et aux tests, aucune policy ne le couvrait.

Le merge automatique, lui, se mérite. Une `MergePolicy` décrit une famille de changements à faible risque (corriger des liens dans la doc, par exemple) : fichiers autorisés, taille du diff, CI verte, verdict du reviewer. Et elle évolue par étapes :

1. **`shadow`** : la policy observe. Elle note ce qu'elle *aurait* décidé, mais c'est toujours un humain qui merge.
2. **`live`** : quand elle a suffisamment de bonnes décisions à son actif, on la promeut en changeant une ligne. Elle merge alors seule, avec un plafond quotidien.
3. **Retour à la case départ** : au premier incident, un disjoncteur coupe la voie et rend la main aux humains.

L'autonomie s'étend donc à votre rythme, policy par policy, avec des règles qui se relisent comme du code.

## 🧭 Maku chez vous, Maku chez moi

Depuis la **v4**, qui vient de sortir, **Maku se suffit à lui-même**. Un cluster Kubernetes, un chart Helm, et vous branchez ce que vous avez déjà : votre fournisseur OIDC, votre PostgreSQL, votre CNI. Rien ne vous oblige à reprendre mon infrastructure. Et pour un premier essai, `task demo` monte le tout sur un laptop en un quart d'heure.

Dans mon cas, j'ai réparti les choses sur trois repos (on en parlait dans l'[article précédent](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)) :

* **[`maku`](https://github.com/Smana/maku)** : le produit, celui que vous installez.
* **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)** : ma configuration de la factory (budgets, modèles autorisés, policies). Elle vit dans son propre repo, **hors de portée des agents**.
* **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)** : le cluster qui héberge le tout.

C'est un choix parmi d'autres, pas un prérequis.

{{% notice tip "Je cherche un partenaire pour la v5 🤝" %}}
J'ai passé beaucoup de temps sur cette v4 (et pas mal de tokens 😅). Pour la **v5**, j'aimerais confronter la solution au terrain : je cherche une **entreprise partenaire** prête à l'essayer avec ses équipes, sur de vrais cas d'usage. Si ça vous tente, faites-moi signe !
{{% /notice %}}

## 💭 Dernières remarques

Au départ, je voulais juste une petite interface web pour lancer des agents dans des sandboxes et garder une trace de ce qu'ils faisaient. Puis je me suis pris au jeu, et de fil en aiguille Maku est devenu une vraie software factory.

La boussole, elle, n'a pas bougé : que ce soit **naturel pour le développeur**. Qu'on parte de Claude Code, d'un thread Slack ou d'une issue GitHub, toute la machinerie de sécurité reste en coulisses, et l'humain garde le dernier mot là où son jugement compte.

Dans le prochain article, on regardera de près les **workflows** et les **merge policies** : comment les rôles se passent le relais, et comment une policy gagne (ou perd) le droit de merger toute seule.

## 🔖 Références

* [`maku`](https://github.com/Smana/maku) : le repo · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installation](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart sur un laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Démo interactive d'une room](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : la plateforme IA · [documentation](https://airef.ogenki.io/) · [guide utilisateur](https://airef.ogenki.io/docs/use/guide/) · [feuille de route](https://airef.ogenki.io/docs/status/)
* [ADR-0057 : pourquoi `ai-native-ref` a son propre repo](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [agentgateway](https://agentgateway.dev/) · [OpenHands](https://github.com/OpenHands/OpenHands) · [gVisor](https://gvisor.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
