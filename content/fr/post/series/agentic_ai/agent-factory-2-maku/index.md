+++
author = "Smaine Kahlouch"
title = "Agent Factory : Maku, le chef d'orchestre"
date = "2026-11-16"
summary = "Maku orchestre l'Agent Factory sur Kubernetes : on lui confie une tâche depuis son coding agent, Slack ou GitHub, on suit le travail des agents dans une room, et on garde le dernier mot sur le merge. Tour du propriétaire."
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

Faire tourner un agent dans un conteneur, c'est devenu facile. Harnais, sandboxes, gateways MCP, routeurs de modèles : il en sort de nouveaux chaque semaine, et quelques briques bien choisies suffisent.

Faire tourner une **software factory pour une équipe**, c'est une autre histoire. Il faut coordonner plusieurs agents, tenir les budgets, isoler chaque exécution, décider de ce qui part en production… et surtout que tout ça reste **agréable à utiliser**. Si déléguer une tâche demande de quitter ses outils, personne ne le fera.

C'est le rôle de **[Maku](https://github.com/Smana/maku)**. Dans le [premier article](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/), ce n'était qu'une boîte au milieu d'un schéma. Ouvrons-la. 🚀

{{% notice tip "La v4 est sortie, et je cherche un partenaire pour la v5 🤝" %}}
La **v4** vient de sortir, avec un objectif : *« anyone can run it »*. Maku s'installe désormais sur n'importe quel cluster Kubernetes, sur un laptop comme en production, avec Helm, Flux ou Argo CD.

J'y ai passé beaucoup de temps (et pas mal de tokens 😅). Pour la **v5**, j'aimerais confronter la solution au terrain : je cherche une **entreprise partenaire** prête à l'essayer avec ses équipes, sur de vrais cas d'usage. Si ça vous tente, faites-moi signe !
{{% /notice %}}

## 🎯 Objectifs de cet article

* 🏗️ Découvrir les **composants de Maku** et ce que fait chacun
* 💻 Voir comment on **délègue une tâche** depuis son coding agent, Slack ou GitHub, et comment on la suit dans une **room**
* 🔄 Avoir un premier aperçu des **workflows** et des **merge policies** (le prochain article leur est consacré)
* 🎬 Suivre une **vraie tâche**, de la phrase tapée dans le terminal jusqu'au merge

## 🧭 Maku chez vous, Maku chez moi

Depuis la v4, **[Maku](https://github.com/Smana/maku) se suffit à lui-même**. Un cluster Kubernetes, un chart Helm, et vous branchez ce que vous avez déjà : votre fournisseur OIDC, votre PostgreSQL, votre CNI. Rien ne vous oblige à reprendre mon infrastructure.

Dans mon cas, j'ai réparti les choses sur trois repos (on en parlait dans l'[article précédent](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)) :

* **[`maku`](https://github.com/Smana/maku)** : le produit, celui que vous installez.
* **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)** : ma configuration de la factory (budgets, modèles autorisés, policies). Elle vit dans son propre repo, **hors de portée des agents**.
* **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)** : le cluster qui héberge le tout.

C'est un choix parmi d'autres, pas un prérequis. Les exemples de cet article viennent de cette installation.

## 🏗️ Les composants de Maku

Maku suit le modèle des **opérateurs Kubernetes** : tout est déclaré dans des CRDs (`Task`, `Room`, `AgentRun`, `MergePolicy`…) que quelques contrôleurs écrits en Go réconcilient en boucle. Une tâche est donc une ressource Kubernetes comme une autre : on peut faire un `kubectl get tasks`, et ça, j'aime bien. 😄

{{< img src="maku-components.png" alt="Les composants de Maku : quatre portes d'entrée (coding agent, Web UI, Slack, GitHub), le control plane avec maku-broker, maku-slack, maku-factory et maku-run-controller, la sandbox gVisor de chaque run avec maku-harness et maku-bridge, et maku-gateway comme unique porte de sortie vers les modèles et les outils MCP" width="1300" link="true" caption="Quatre portes d'entrée, un control plane, une sandbox par run." >}}
</br>

* 🏭 **`maku-factory`**, le chef d'atelier. Il transforme une issue en `Task`, choisit le workflow et le budget, lance les runs, raconte l'avancement sur l'issue et applique le **merge gate**.

* 📓 **`maku-broker`**, la mémoire. Il tient le journal de chaque **room** dans PostgreSQL, authentifie tous ceux qui y écrivent, et sert la Web UI ainsi que le serveur MCP auquel se branche votre coding agent.

* 🚀 **`maku-run-controller`**, le lanceur. Pour chaque `AgentRun`, il crée un pod isolé dans une sandbox **gVisor**, avec son propre `ServiceAccount` et un réseau fermé par défaut.

* 🤖 **`maku-harness`**, l'agent. C'est l'image qui s'exécute dans la sandbox : [OpenHands](https://github.com/OpenHands/OpenHands), `gh`, et de quoi ouvrir la PR.

* 🌉 **`maku-bridge`**, le messager. Ce sidecar remonte les événements de l'agent vers la room et redescend vos consignes (les *steers*). L'agent, lui, ne détient jamais le token de la room.

* 🚪 **`maku-gateway`**, la porte de sortie. Basée sur [agentgateway](https://agentgateway.dev/), c'est le seul chemin vers les modèles et les outils MCP. Elle vérifie l'identité de chaque run et applique les plafonds de dépense.

* 💬 **`maku-slack`** et la **CLI `maku`**, les interfaces du quotidien. On y vient tout de suite.

Un détail qui a son importance : **rien n'entre dans une sandbox**. Toutes les connexions sont ouvertes par le pod lui-même, jamais l'inverse.

## 💻 Déléguer une tâche, là où vous êtes

Mon objectif de départ : déléguer une *side task* doit prendre **moins d'une minute**, depuis l'outil où je me trouve quand je repère le problème. Trois portes d'entrée, un seul circuit derrière.

<!-- TODO(smana): intégrer le screencast complet ici dès qu'il est prêt -->
{{% notice note "🎬 Screencast à venir" %}}
Un screencast complet arrive bientôt : il montrera tout ce qui suit en conditions réelles. En attendant, vous pouvez manipuler la [démo interactive d'une room](https://blog.ogenki.io/maku/).
{{% /notice %}}

### ⌨️ Depuis votre coding agent

C'est ma porte préférée. Je suis en plein travail dans mon terminal, je tombe sur un bug sans rapport avec ma branche, une doc obsolète, un test manquant. Plutôt que de perdre le fil (ou d'oublier), je le confie à la factory sans quitter ma session.

L'installation tient en une commande :

```bash
maku setup claude      # ou opencode, codex, gemini, cursor, copilot
```

Elle ouvre le SSO, enregistre le serveur MCP `maku` et ajoute trois raccourcis :

* `/maku handoff <description>` : prépare un plan, puis me demande de confirmer avant de créer l'issue et de lancer la factory
* `/maku task [#issue]` : où en est la tâche, ce qu'elle attend de moi, combien elle a coûté
* `/maku inbox` : toutes mes tâches qui attendent une action humaine

Deux choses que j'apprécie à l'usage :

* **Mon coding agent ne peut rien lancer dans mon dos.** Il prépare le plan, mais c'est moi qui valide le formulaire. L'issue est créée avec mon propre compte GitHub.
* **Je ne surveille rien.** Une fois la tâche partie, mon agent attend en tâche de fond (`wait`) et ne me dérange que si une question se pose ou si la PR est prête.

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

On ajuste en répondant dans le thread, on change de workflow dans le menu, puis un mainteneur clique sur **Confirm** et c'est parti.

Ensuite, Slack reste discret : un message au démarrage, un si la tâche a besoin de vous, un à l'ouverture de la PR, et une ligne de bilan à la fin.

```text
Merged #418 · $0.16 of $2.00 · 11 min · merged automatically by policy docs-links
```

### 🐙 Depuis GitHub

La voie classique : on écrit une issue, on pose le label `factory/ready`, et Maku s'en charge. Quelques labels permettent de piloter la suite : `workflow:careful` pour une tâche délicate, `factory/hold` pour geler le merge, `factory/stop` pour tout arrêter.

Sur l'issue, Maku raconte chaque étape : le run en cours, la branche, le budget consommé, et le lien vers la **room**.

### 🎛️ La room : tout voir, garder la main

{{< img src="room.gif" alt="Trafalgar Law (One Piece) qui déploie sa « Room »" width="480" >}}

Les amateurs de *One Piece* auront reconnu la *Room* de Trafalgar Law : un périmètre dans lequel on voit tout et où l'on garde le contrôle. Le clin d'œil est assumé. 😉

Pourquoi ne pas se contenter des logs du pod ? Parce qu'ils disparaissent avec lui, et qu'ils mélangent ce que l'agent *dit* faire et ce qu'il fait *vraiment*. La room, elle, reste :

* 🔒 **Append-only** : le broker peut ajouter et lire des événements, jamais les modifier ni les supprimer. À la fermeture, le journal est scellé.
* 🕵️ **Attribuée et nettoyée** : chaque événement porte l'identité vérifiée de son auteur (humain, agent ou factory), et les secrets sont masqués par [gitleaks](https://github.com/gitleaks/gitleaks) avant l'écriture.
* 🎮 **Pilotable en direct** : on peut laisser une consigne pour le prochain run, prendre la main pour réorienter l'agent (*steer*), ou approuver une demande.

Dans la Web UI, l'encadré **NEXT** dit en une phrase ce qui est attendu de vous :

{{< img src="maku-room.png" alt="Une room Maku qui attend un mainteneur : l'état « Waiting — review and approve PR #42 », le budget en tokens, le dernier verdict du reviewer, l'encadré NEXT qui dit que l'approbation du commit courant vaut merge, l'action exacte sous « Needs you », puis le fil des événements de l'implementer" width="1100" caption="Une room qui attend mon approbation : la prochaine étape, l'action à faire, et le fil des événements du run." >}}
</br>

Et quand plusieurs tâches tournent en parallèle, celles qui ont besoin de vous remontent en tête de liste :

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku : les tâches qui attendent un humain portent un badge « Needs you » et remontent en premier, chacune avec son dépôt, son état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms : « Needs you » d'abord." >}}
</br>

## 🔄 Workflows et merge policies, en deux mots

Deux mécanismes décident de ce qui se passe entre le `factory/ready` et le merge. Je les survole ici : **le prochain article leur est entièrement consacré**.

### 👥 Les workflows : qui travaille sur la tâche ?

Un agent seul qui écrit le code puis évalue son propre travail finit vite par se convaincre qu'il a terminé. Un **workflow** définit donc quels rôles se relaient sur une tâche :

* **`standard`**, le workflow par défaut : un **implementer** écrit le changement et ouvre la PR, puis un **reviewer** la relit avec un regard neuf, sans voir le raisonnement du premier.
* **`quick`** : l'implementer seul, pour les changements triviaux.
* **`careful`** : un **scoper** cadre d'abord la tâche (et pose une question si l'issue est floue), puis un **judge** vérifie que les critères d'acceptation sont remplis avant la review.

Le workflow se choisit par repo, par label, ou directement dans le plan au moment du handoff.

### 🚦 Les merge policies : qui a le droit de merger ?

Par défaut, c'est vous. Maku vous indique la seule action attendue, et **votre approbation du commit vaut merge**.

Le merge automatique, lui, se mérite. Une `MergePolicy` décrit une famille de changements à faible risque (corriger des liens dans la doc, par exemple) : fichiers autorisés, taille du diff, CI verte, verdict du reviewer. Et elle évolue par étapes :

1. **`shadow`** : la policy observe. Elle note ce qu'elle *aurait* décidé, mais c'est toujours un humain qui merge.
2. **`live`** : quand elle a suffisamment de bonnes décisions à son actif, on la promeut en changeant une ligne. Elle merge alors seule, avec un plafond quotidien.
3. **Retour à la case départ** : au premier incident, un disjoncteur coupe la voie et rend la main aux humains.

L'autonomie s'étend donc à votre rythme, policy par policy, avec des règles qui se relisent comme du code.

## 🎬 Une vraie tâche, de bout en bout

En préparant les screencasts d'[airef.ogenki.io](https://airef.ogenki.io/), je remarque un angle mort sur le dashboard Grafana de la flotte d'agents : quand un fournisseur de modèles refuse une requête, la métrique existe, mais rien ne l'affiche. Ce n'est pas le moment de m'y plonger, alors je délègue :

```text
/maku handoff Dans Smana/ai-native-ref, exposer les refus des fournisseurs de modèles sur le dashboard Grafana de la flotte.
```

Voici ce qui se passe ensuite :

1. 🧐 **Une question, une seule.** Le planner hésite : faut-il aussi mettre à jour les tests de la CI ? Il me le demande, avec sa recommandation (oui). J'accepte.

2. ✅ **Je confirme.** Le formulaire me montre l'issue rédigée, le workflow `standard` et le budget. Je valide : l'[issue #354](https://github.com/Smana/ai-native-ref/issues/354) est créée, et je retourne à mes screencasts.

3. 🤖 **L'implementer s'y met.** Dans sa sandbox, il modifie le dashboard et les deux fichiers de tests, lance les checks du repo, puis ouvre la [PR #357](https://github.com/Smana/ai-native-ref/pull/357) : 22 lignes, 3 fichiers.

4. 👀 **Le reviewer relit.** Un nouveau run, un contexte vierge, et un verdict : `approved`.

5. 🚦 **Le merge gate tranche.** Le diff touche à l'observabilité et aux tests : aucune policy `live` ne le couvre, donc il m'attend. Mon terminal me prévient :

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

## 💭 Dernières remarques

Au départ, je voulais juste une petite interface web pour lancer des agents dans des sandboxes et garder une trace de ce qu'ils faisaient. Puis je me suis pris au jeu, et de fil en aiguille Maku est devenu une vraie software factory.

La boussole, elle, n'a pas bougé : que ce soit **naturel pour le développeur**. Qu'on parte de Claude Code, d'un thread Slack ou d'une issue GitHub, toute la machinerie de sécurité reste en coulisses, et l'humain garde le dernier mot là où son jugement compte.

Avec la **v4**, à vous de jouer : vous pouvez l'installer sur votre cluster, ou l'essayer en un quart d'heure sur votre laptop avec `task demo`. Et pour la v5, l'appel tient toujours : si vous voulez l'éprouver avec vos équipes, parlons-en ! 😉

Dans le prochain article, on regardera de près les **workflows** et les **merge policies** : comment les rôles se passent le relais, et comment une policy gagne (ou perd) le droit de merger toute seule.

## 🔖 Références

* [`maku`](https://github.com/Smana/maku) : le repo · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installation](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart sur un laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Démo interactive d'une room](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : la plateforme IA · [documentation](https://airef.ogenki.io/) · [guide utilisateur](https://airef.ogenki.io/docs/use/guide/) · [feuille de route](https://airef.ogenki.io/docs/status/)
* [ADR-0057 : pourquoi `ai-native-ref` a son propre repo](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [agentgateway](https://agentgateway.dev/) · [OpenHands](https://github.com/OpenHands/OpenHands) · [gVisor](https://gvisor.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
