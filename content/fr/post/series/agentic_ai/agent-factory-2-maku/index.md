+++
author = "Smaine Kahlouch"
title = "Agent Factory : Maku, le chef d'orchestre"
date = "2026-11-16"
summary = "De la petite interface pour lancer des agents en sandbox à une véritable usine à coder sur Kubernetes : comment Maku orchestre les tâches, s'intègre dans votre coding agent, Slack et GitHub, et pilote les workflows jusqu'au merge."
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
3. [Le harnais : un environnement cloisonné pour chaque agent](/fr/post/series/agentic_ai/agent-factory-3-harnais/)
4. [Le merge gate : une autonomie qui se mérite](/fr/post/series/agentic_ai/agent-factory-4-merge-gate/)
5. [Observer et maîtriser une flotte d'agents](/fr/post/series/agentic_ai/agent-factory-5-observabilite/)
6. [D'agent à agent : RunLore confie ses correctifs à la factory](/fr/post/series/agentic_ai/agent-factory-6-runlore/)
{{% /notice %}}

L'écosystème cloud native autour de l'IA agentique est en pleine ébullition : de nouveaux harnais, sandboxes, passerelles **MCP** (*Model Context Protocol*) et routeurs de modèles voient le jour chaque semaine. Assembler quelques briques pour faire tourner un agent dans un conteneur est aujourd'hui relativement simple.

En revanche, **piloter l'expérience globale d'une software factory** à l'échelle d'une équipe est un tout autre défi. Il faut coordonner plusieurs rôles d'agents, maîtriser les budgets, isoler chaque exécution et garantir ce qui part en production — tout en offrant **une expérience développeur aussi naturelle que possible**, sans forcer personne à quitter ses outils du quotidien.

C'est précisément le rôle de **[Maku](https://github.com/Smana/maku)**, le chef d'orchestre de l'Agent Factory. Sa promesse tient en une phrase : *« Run coding agents on your own Kubernetes cluster without having to trust them. »* Conçu comme un plan de contrôle Kubernetes natif, Maku s'appuie fortement sur le modèle des **opérateurs Kubernetes** (ses propres contrôleurs réconciliant les tâches, les rooms, les runs et les politiques de merge, aux côtés d'opérateurs comme CloudNativePG ou Envoy Gateway) et s'intègre avec **Crossplane** sur ma plateforme de référence pour composer de façon déclarative le socle qui entoure les agents.

Dans le [premier épisode](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/), Maku n'était qu'une boîte au milieu d'un schéma d'architecture. Ouvrons-la pour voir comment elle s'organise et comment on travaille avec au quotidien. 🚀

{{% notice tip "La v4 est sortie… et je cherche un partenaire pour la v5 ! 🤝" %}}
Au moment où je vous parle, je viens tout juste de sortir la **v4 de Maku**, qui a pour objectif de permettre à tout un chacun de déployer l'Agent Factory sur son propre cluster Kubernetes (*« anyone can run it »*), en local sur un laptop comme en production via Helm, Flux ou Argo CD.

Je vous avoue que j'y ai passé énormément de temps (et un certain budget cloud et tokens 😅). Pour la **v5**, je souhaite désormais **éprouver la solution en situation réelle** : je cherche donc une **entreprise partenaire** qui y verra un intérêt commun à tester la solution avec ses équipes sur de vrais cas d'usage. Si l'aventure vous intéresse, n'hésitez pas à me faire signe !
{{% /notice %}}

## 🎯 Objectifs de cet article

* 🏗️ Comprendre les **composants de Maku**, son modèle d'opérateurs Kubernetes, et pourquoi la plateforme IA vit dans un dépôt distinct (**[`ai-native-ref`](https://airef.ogenki.io/)**)
* 💻 Découvrir l'**expérience développeur** au quotidien : depuis un coding agent, depuis Slack, depuis GitHub et dans une **room**
* 🔄 Parcourir de façon synthétique le chemin **du clavier à la livraison** : les workflows et rôles, le routing des modèles et le merge gate
* 🎬 Suivre un **cas concret de bout en bout**, tiré d'une vraie tâche sur `ai-native-ref` lors de la préparation des screencasts de démo

## 🏗️ Les composants et le découpage en trois dépôts

### Pourquoi un dépôt `ai-native-ref` distinct ?

Au début du projet, toute la couche IA (le serving de modèles open-weights comme l'Agent Factory) grandissait directement à l'intérieur de [`cloud-native-ref`](https://cnref.ogenki.io/), ma plateforme cloud native de référence. En quelques semaines, l'IA a fini par représenter plus de 200 manifestes et près de 40 documents d'architecture : elle cannibalisait la CI, la documentation et la lisibilité d'un dépôt dont l'IA n'est pourtant qu'un cas d'usage parmi d'autres.

J'ai donc pris la décision ([ADR-0057](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)) de séparer clairement les responsabilités en **trois dépôts**, chacun avec son propre cycle de vie :

| Dépôt | Rôle | Ce qu'il contient |
|---|---|---|
| **[`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)** | Le socle plateforme hôte | OpenTofu, Cilium, Crossplane, ZITADEL, OpenBao, CloudNativePG, VictoriaMetrics. Déployé par défaut **sans aucune dépendance IA**. |
| **[`ai-native-ref`](https://github.com/Smana/ai-native-ref)** (*[airef.ogenki.io](https://airef.ogenki.io/)*) | L'implémentation de référence IA | Le serving GPU (vLLM via des compositions Crossplane `InferenceService`), le routage des modèles, les politiques de la factory et les runbooks de validation. |
| **[`maku`](https://github.com/Smana/maku)** | Le produit open source | Les contrôleurs Go, le broker de rooms, la Web UI, la CLI `maku`, l'application Slack, l'image du harnais et les charts Helm (`agent-factory` et `agent-factory-crds`). |

Ce découpage apporte trois avantages décisifs :

1. **Un contrat d'hôte explicite.** `cloud-native-ref` consomme `ai-native-ref` sous la forme d'un unique **artefact OCI signé par Cosign et épinglé par digest**, derrière un contrat écrit (`HOST-CONTRACT.md`). La CI de chaque dépôt valide le rendu contre l'autre.
2. **Une sécurité renforcée pour la factory.** La configuration qui gouverne les agents (les budgets, les modèles autorisés, les règles d'admission) vit dans `ai-native-ref`, **hors du dépôt `cloud-native-ref`** que les agents modifient au quotidien.
3. **Un produit Maku autonome.** Grâce au travail réalisé pour la **v4**, Maku ne dépend plus des choix spécifiques de `cloud-native-ref` : n'importe quelle équipe plateforme peut l'installer sur son propre cluster Kubernetes (avec son propre fournisseur OIDC, sa base PostgreSQL et son CNI) via un chart Helm unique et l'assistant `maku install --print`. `ai-native-ref` n'est plus qu'un jeu de valeurs et de preuves en conditions réelles au-dessus de Maku.

### Les briques de Maku

Fidèle à la philosophie Kubernetes, Maku pilote toute l'usine à travers des **CRD** (*Custom Resource Definitions* du groupe `maku.ogenki.io`) réconciliées en boucle par quelques binaires Go au périmètre volontairement étroit :

| Composant | Rôle dans le cluster | Ce qu'il réconcilie ou expose |
|---|---|---|
| **`maku-factory`** | L'orchestrateur de la factory (2 réplicas avec élection de leader) | Réconcilie les CRD `Factory`, `Repository`, `MergePolicy` et `Task`. Relève les issues GitHub, choisit le workflow et le budget, forge les jetons GitHub par run, compte les dépenses et exécute le **merge gate**. |
| **`maku-broker`** | Le journal de référence et la porte d'entrée humaine | Réconcilie la CRD `Room`. Stocke le journal *append-only* dans PostgreSQL, authentifie chaque acteur, sert la **Web UI**, le serveur MCP pour les développeurs et les outils MCP de room (`:8090`) pour les agents. |
| **`maku-run-controller`** | Le contrôleur d'exécution | Réconcilie la CRD `AgentRun` : crée pour chaque run son `ServiceAccount`, ses `NetworkPolicy` *default-deny* et son pod isolé dans une sandbox **gVisor**. |
| **`maku-bridge`** | Le sidecar de chaque sandbox | Tourne aux côtés du harnais dans le pod de run. Lit les événements de l'agent en local et les pousse vers le broker en TLS ; redescend les consignes (*steers*) et interruptions sans que l'agent ne détienne jamais le jeton de la room. |
| **`maku-gateway`** | La passerelle modèle et outils | Basée sur **agentgateway**. Porte de sortie unique des agents vers les LLM et serveurs MCP : vérifie l'identité par run et applique les plafonds. |
| **`maku-slack` & CLI `maku`** | Les interfaces du quotidien | L'application Slack transforme une discussion en tâche ; la CLI `maku` connecte le terminal et les coding agents en une commande. |

Sur `cloud-native-ref`, ces opérateurs travaillent main dans la main avec **Crossplane** : là où Maku gère le cycle de vie des tâches et des runs (`Task`, `Room`, `AgentRun`), les compositions Crossplane provisionnent l'infrastructure sous-jacente — des bases PostgreSQL (`SQLInstance` via CloudNativePG) jusqu'aux services d'inférence GPU (`InferenceService` avec vLLM et KEDA).

## 💻 L'expérience développeur : là où vous travaillez déjà

Une software factory que personne n'utilise parce qu'il faut remplir trois formulaires web a déjà perdu. Dès le départ, l'objectif était que déléguer une tâche annexe (*side task*) prenne **moins d'une minute**, depuis l'outil où l'on se trouve au moment où l'on repère le problème.

Trois portes d'entrée coexistent et convergent toutes vers le même circuit.

### 1. Depuis votre coding agent (Claude Code, OpenCode, Codex, Cursor…)

C'est mon chemin préféré au quotidien : on est en plein code dans son terminal, on tombe sur un bug annexe, une doc obsolète ou un test manquant qui n'a rien à voir avec notre branche courante. Plutôt que de perdre son fil ou de l'oublier, on le confie à la factory sans quitter sa session.

L'installation tient en une seule commande :

```bash
# Configure l'URL, ouvre le SSO, enregistre le serveur MCP `maku` et la commande `/maku`, puis lance `maku doctor`
maku setup claude      # fonctionne aussi avec opencode, codex, gemini, cursor, copilot
```

Plutôt que de noyer le modèle sous une vingtaine d'outils, Maku n'expose par défaut que **six outils MCP** (`handoff`, `start`, `inbox`, `task`, `wait`, `answer`) et trois raccourcis :

| Commande | Ce qu'elle fait |
|---|---|
| `/maku handoff <description>` | Planifie la tâche avec le broker (`handoff`), puis ouvre un formulaire local (`start`) pour créer l'issue et lancer la factory |
| `/maku task [#issue ou id]` | Affiche l'état d'une tâche, ce qu'elle attend de vous, ce qui a changé depuis votre dernier passage, le coût et la décision du gate |
| `/maku inbox` | Liste toutes vos tâches qui attendent une action humaine |

Le contrat de sécurité est strict : **votre coding agent local ne peut jamais démarrer la factory ni approuver quoi que ce soit dans votre dos**. Quand vous lancez `/maku handoff`, le *planner* du broker prépare le plan et pose au maximum **une question de clarification** s'il manque un élément (comme le fichier cible ou un critère d'acceptation vérifiable). Une fois le plan prêt, l'outil `start` ouvre un formulaire que **vous seul** pouvez valider pour créer l'issue (via votre propre CLI `gh`) et poser le label `factory/ready`.

Pendant que vous reprenez votre travail, votre agent lance `wait` en tâche de fond et vous prévient uniquement quand la tâche a une question pour vous ou que la PR est prête à être relue.

### 2. Depuis une conversation Slack

Une remarque au détour d'un canal d'équipe devient une tâche en quelques secondes grâce à **`maku-slack`**. Après avoir lié une fois son compte Slack à son identité GitHub (`@factory link`), il suffit de mentionner `@factory` avec ses propres mots — sans aucune syntaxe rigide à mémoriser :

```text
@factory le lien d'installation dans le README pointe encore vers l'ancienne page, il devrait cibler docs/install.md
```

Le planner répond dans le fil avec **une unique carte de plan** :

```text
Plan for Smana/ai-native-ref (revision 1)
docs: the install link in the README is dead
Workflow  [ standard (implementer, reviewer) ▾ ]
Budget    up to $2.00
[ Confirm ]  [ Cancel ]
```

On peut ajuster le plan en répondant simplement dans le fil, ou changer le workflow dans le menu déroulant : le message se met à jour sur place. Lorsqu'un mainteneur clique sur **Confirm** (ou ajoute une réaction ✅), l'issue est créée et la tâche démarre.

Pour éviter le bruit dans les canaux, Slack ne poste que **quatre transitions** sur toute la vie de la tâche : le démarrage (*Started*), un besoin d'intervention (*Needs you*), l'ouverture de la PR, et **une seule ligne de fin** éditée sur place lors du merge avec le bilan complet :

```text
Merged #418 · $0.16 of $2.00 · 11 min · merged automatically by policy docs-links
```

Et si un agent a besoin d'une précision en cours de route, d'une autorisation de `git push` ou qu'il faut arrêter (`stop`) ou relancer (`retry`) la tâche, tout peut se faire directement en répondant dans le fil Slack derrière un bouton de confirmation vérifiant vos droits GitHub.

### 3. Depuis GitHub et la Room (Web UI)

Bien sûr, on peut aussi rédiger une issue directement sur GitHub et la piloter avec quelques labels (`factory/ready` pour démarrer, `workflow:careful` pour forcer un cadrage approfondi, `factory/stop` pour arrêter, `factory/hold` pour geler le merge, ou `/factory retry` en commentaire). Sur l'issue, l'App de la factory poste un compte rendu clair à chaque étape : identifiant du run, branche, budget, et lien vers la **room**.

{{< img src="room.gif" alt="Trafalgar Law (One Piece) qui déploie sa « Room »" width="480" >}}

C'est là qu'intervient la **room** : à la fois boîte noire d'avion et salon de supervision pour chaque tâche (et oui, pour les amateurs de *One Piece*, le clin d'œil à la *Room* de Trafalgar Law — ce périmètre où l'on observe et maîtrise tout ce qui s'y passe — est totalement assumé 😉). Un simple log de pod disparaît avec le conteneur et mélange ce que l'agent *prétend* faire avec ce qu'il a *réellement* appelé. La room, elle, persiste et offre des garanties fortes :

* 🔒 **Inaltérable (*append-only*)** : dans PostgreSQL, le rôle du broker n'a que les droits `SELECT` et `INSERT` sur la table des événements (aucun `UPDATE` ni `DELETE`), avec un numéro de séquence sans trou et un événement de **sceau** (*seal*) à la fermeture.
* 🕵️ **Caviardée et attribuée** : chaque texte passe par les règles **gitleaks** avant d'être écrit en base, et chaque événement porte l'identité cryptographiquement vérifiée de son auteur (`human:<id>`, `agent:<run>`, `system:factory`).
* 🎛️ **Pilotable en direct** : la Web UI (protégée par SSO) affiche en haut l'encadré **NEXT** (la prochaine action attendue de vous), un **rail d'exécution** (*run rail*) avec chaque run, son modèle et son coût, ainsi que le fil complet des appels d'outils. À tout moment, vous pouvez mettre une consigne en file pour le run suivant, prendre le rôle de **pilote** (*driver*) pour réorienter l'agent à chaud, ou trancher une demande d'approbation (avec support de la règle des quatre yeux).

{{< img src="maku-room.png" alt="Une room Maku qui attend un mainteneur : l'état « Waiting — review and approve PR #42 », le budget en tokens, le dernier verdict du reviewer, l'encadré NEXT qui dit que l'approbation du commit courant vaut merge, l'action exacte sous « Needs you », puis le fil des événements de l'implementer" width="1100" caption="Une room qui attend votre approbation : la prochaine étape, l'action exacte à faire, et le fil des événements du run (vous pouvez aussi manipuler la [démo interactive](https://blog.ogenki.io/maku/))." >}}
</br>

Et lorsque plusieurs tâches tournent en parallèle, la liste des rooms fait remonter en tête celles qui attendent une action de votre part (`Needs you`) :

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku, rafraîchie toutes les 10 secondes : les tâches qui attendent un humain portent un badge « Needs you » et remontent en premier, chacune avec son dépôt, son état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms : les tâches qui attendent un humain (« Needs you ») remontent en premier." >}}
</br>

## 🔄 Du clavier à la livraison : sous le capot (en bref)

Que se passe-t-il entre le moment où la tâche démarre et celui où la PR est mergée ? Nous consacrerons un article complet à chacun de ces trois piliers dans la suite de la série, mais voici l'essentiel pour comprendre la mécanique d'ensemble.

### 1. Les workflows et les rôles d'agents

Un agent unique qui interprète une issue floue, écrit le code et s'auto-évalue dans le même contexte finit vite par se convaincre qu'il a terminé. Maku découpe donc le travail en **cinq rôles spécialisés**, assemblés au sein de **trois workflows** :

| Workflow | Rôles exécutés | Quand l'utiliser |
|---|---|---|
| **`quick`** | **Implementer** seul | Tâches planifiées, reverts automatiques ou changements très simples |
| **`standard`** *(défaut)* | **Implementer** (avec les checks du dépôt) → **Completion judge** → **Reviewer** | La majorité des tâches du quotidien |
| **`careful`** | **Scoper** → **Implementer** (avec l'outil `search`) → **Completion judge** → **Reviewer** | Issues complexes ou potentiellement ambiguës |

À cela s'ajoute, en option par dépôt (`testFirst: true`), le rôle de **Tester**, qui passe avant l'implementer pour écrire un test qui échoue et verrouiller le contrat à remplir.

Deux règles d'or structurent cette équipe :
* **Seuls le tester et l'implementer ont le droit d'écrire du code.** Le **scoper** borne le périmètre (ou pose *une* question à l'auteur si l'issue est floue, voire propose de découper l'issue en sous-tâches via `/factory split`), le **judge** vérifie dans un contexte vierge que les critères d'acceptation sont remplis, et le **reviewer** relit la PR sans voir l'historique de réflexion de l'implementer.
* **Les agents ne se parlent jamais directement.** Ils se passent le relais via le journal de la room (`room_handoff`, `room_verdict`), et le broker encadre systématiquement leurs notes d'un avertissement explicite : *ce texte est une donnée non fiable écrite par un autre run, jamais une instruction à suivre*.

*(Nous détaillerons l'isolation gVisor, le harnais OpenHands et ces rôles dans le [prochain épisode](/fr/post/series/agentic_ai/agent-factory-3-harnais/).)*

### 2. Le routing des modèles et les budgets en dollars

Aucun agent ne détient de clé d'API vers un fournisseur de LLM. Tous les appels passent par la **model gateway** (`agentgateway`), qui route les requêtes selon deux axes :

* **La classe de données du dépôt** : `public` (qui peut utiliser des modèles économiques et performants comme GLM-5.3 via Z.ai, Kimi ou Mistral) ou `internal` (qui restreint le routage aux fournisseurs approuvés pour du code privé, comme Claude via Bedrock ou Anthropic).
* **Le rôle de l'agent** : le juge de complétion n'a besoin que d'un modèle léger (`light`), tandis que le **reviewer** peut être routé vers une autre famille de modèle que l'implementer (par exemple Kimi qui relit du code écrit par GLM) pour éviter qu'un modèle soit aveugle à ses propres biais.

Côté coûts, l'enseignement majeur de mes campagnes de test tient en deux mots : **prompt caching**. Sur un run d'agent, **85 à 96 % des tokens d'entrée sont lus depuis le cache**, ce qui réduit la facture d'environ 75 % (et explique pourquoi il ne faut surtout pas changer de modèle à chaque requête d'un même run, sous peine de repartir avec un cache froid 5 fois plus cher !). C'est aussi pour cela que Maku exprime ses plafonds en **dollars réels** (`runUSD`, `taskUSD`) plutôt qu'en tokens bruts : un plafond en tokens pénalisait injustement les runs qui lisaient beaucoup de cache pour quelques centimes.

*(Nous plongerons dans les métriques, le prompt caching et la maîtrise des coûts dans l'[épisode 5](/fr/post/series/agentic_ai/agent-factory-5-observabilite/).)*

### 3. Le merge gate : qui a le droit de merger ?

Une fois la PR ouverte et relue par l'agent reviewer, c'est le **merge gate** (`maku/merge-gate`) qui entre en scène. Plutôt que de faire confiance à un label posé à l'avance, Maku **évalue le diff réel de la PR** contre un catalogue de politiques (`MergePolicy`) déclarées pour le dépôt : fichiers autorisés, nombre de lignes modifiées, forme du changement, CI verte, absence de secrets et verdict favorable du reviewer.

Il n'y a alors que deux issues possibles :
* **Le merge automatique (`live`)** : réservé aux familles de changements à faible risque qui ont fait leurs preuves en mode `shadow` (comme les corrections de liens `docs-links` ou les `revert`), plafonné par jour (`maxPerDay`) et protégé par un disjoncteur qui coupe la voie au premier incident post-merge.
* **La passe décisive à l'humain** : pour tout le reste, Maku vous indique la seule action attendue. Grâce au mode **approve-to-merge** (`approveToMerge: true`), **votre approbation GitHub sur le commit courant déclenche immédiatement le merge** par une GitHub App dédiée (`ogenki-agent-merger[bot]`), la seule identité autorisée à merger sur la branche principale.

*(Le fonctionnement complet du gate, du mode shadow et du disjoncteur sera l'objet de l'[épisode 4](/fr/post/series/agentic_ai/agent-factory-4-merge-gate/).)*

## 🎬 Un cas concret : préparer les screencasts d'`airef`

Pour voir ce que donne cette mécanique en vrai, prenons un cas qui va plus loin qu'une simple coquille dans un README.

En préparant les deux screencasts de démonstration pour la page d'accueil d'`airef.ogenki.io` ([issue #394](https://github.com/Smana/ai-native-ref/issues/394) et [runbook 24](https://github.com/Smana/ai-native-ref/blob/main/docs/runbooks/agent-factory/24-screencast.md)), je passe en revue les écrans et les dashboards Grafana de la flotte d'agents. C'est là que je repère un angle mort gênant ([issue #354](https://github.com/Smana/ai-native-ref/issues/354)) : lorsque la passerelle ou un fournisseur de modèle refuse une requête, le compteur existe en métrique, mais **les refus des fournisseurs ne sont pas exposés sur le dashboard Grafana de la flotte** (`grafana-dashboard-agent-fleet.yaml`), et aucun test de la CI ne vérifie leur présence.

Je suis au milieu de ma préparation vidéo dans mon terminal avec mon coding agent : c'est typiquement le genre de tâche — toucher un dashboard Grafana en YAML et mettre à jour deux scripts de tests Python dans la CI — que je veux déléguer à la factory sans perdre mon contexte.

### Étape 1 : Le handoff depuis le coding agent

Sans quitter ma session Claude Code, je lance :

```text
/maku handoff Dans Smana/ai-native-ref, exposer les refus des fournisseurs de modèles sur le dashboard Grafana de la flotte.
```

Le coding agent appelle l'outil MCP `handoff`. Le planner du broker analyse ma phrase et détecte une zone d'ombre : faut-il uniquement modifier le dashboard ou aussi mettre à jour les tests de conventions de la CI ? Plutôt que de deviner, il me pose **une seule question de clarification** avec sa recommandation :

```text
Question g1: Should the CI dashboard convention tests also assert the new panel and metric?
Recommendation: Yes — update test-agent-dashboard-conventions.py and test-agent-observability.py so CI guards the panel.
```

J'accepte la recommandation (`handoff` met à jour le plan stocké sur le broker), puis l'outil `start` m'affiche le formulaire de confirmation local : l'issue rédigée avec son critère d'acceptation, le workflow `standard` et le budget plafonné à 2,55 $ par run. Je valide (**File: yes**, **Start now: yes**) : `maku` crée l'[issue #354](https://github.com/Smana/ai-native-ref/issues/354) sous mon propre compte GitHub et pose le label `factory/ready`.

Mon coding agent lance aussitôt `wait` en tâche de fond : mon terminal est libre, je peux reprendre la préparation de mes screencasts.

### Étape 2 : Le relais des agents dans la Room

Moins d'une minute plus tard, `maku-factory` relève l'issue, crée la `Task` `ozdjcnhn` et ouvre sa **room**. Sur l'issue GitHub, le premier commentaire apparaît avec le lien de suivi et le bouton d'arrêt d'urgence (`factory/stop`).

Dans le namespace `agents`, `maku-run-controller` démarre le pod de l'**implementer** (`run-7lrzay4x`) dans une sandbox gVisor, avec un jeton GitHub valable une heure limité au seul dépôt `Smana/ai-native-ref`. Dans la room, on voit le run progresser étape par étape :

1. **Exploration et édition** : l'implementer inspecte le dashboard `observability/base/agent-platform/grafana-dashboard-agent-fleet.yaml`, ajoute les séries mesurant les refus par fournisseur, puis met à jour les deux suites de tests Python de la CI (`scripts/ci/tests/test-agent-dashboard-conventions.py` et `scripts/ci/tests/test-agent-observability.py`).
2. **Vérification locale dans la sandbox** : avant même de pousser son code, le harnais exécute les tests du dépôt directement dans la sandbox gVisor pour vérifier que les conventions des dashboards et les règles d'observabilité passent au vert.
3. **Ouverture de la PR et passage de relais** : l'implementer pousse la branche `agent/ozdjcnhn`, ouvre la [PR #357](https://github.com/Smana/ai-native-ref/pull/357) signée du trailer `Agent-Run: 7lrzay4x`, et appelle `room_handoff`.
4. **Jugement et revue** : le **completion judge** valide que les critères d'acceptation sont remplis (`met`). Aussitôt, un nouveau pod démarre dans la même room pour le **reviewer**, qui relit le diff de 22 lignes sur 3 fichiers avec un contexte vierge, vérifie que les tests couvrent bien le nouveau panneau et publie son verdict `approved` sur la PR.

### Étape 3 : Le verdict du merge gate et le clic final

À l'ouverture de la PR, la CI GitHub Actions s'exécute et le **merge gate** (`maku/merge-gate`) évalue le diff. Pourquoi cette PR ne se merge-t-elle pas toute seule ? Parce que le diff modifie des manifestes sous `observability/` et des tests sous `scripts/ci/tests/` : elle ne rentre pas dans l'étroite politique `docs-links` autorisée en auto-merge.

Le gate rend donc une décision explicite, visible à la fois sur la PR, dans la room et dans mon terminal lorsque le `wait` en tâche de fond me notifie :

```text
#354 dashboards: expose model provider refusals              ai-native-ref
State     waiting for a maintainer's review
Needs you review and approve PR #357
Budget    $0.48 of $5.10 (9 %)
Merge     Waits for a human: policy requires a maintainer approval
Room      https://rooms.priv.aws.ogenki.io/r/ozdjcnhn
```

Je clique sur le lien de la PR [#357](https://github.com/Smana/ai-native-ref/pull/357) : les 3 fichiers modifiés sont propres, le reviewer a déjà vérifié les conventions et la CI est verte. Je soumets une revue GitHub **Approve** sur le commit `79a51b1`. Moins d'une minute plus tard, `ogenki-agent-merger[bot]` détecte mon approbation liée à ce commit précis, merge la PR en squash et clôture l'issue avec le décompte final.

Temps humain investi : une phrase dans mon terminal au départ, trente secondes de relecture et un clic sur GitHub à l'arrivée.

## 💭 Dernières remarques

Quand j'ai écrit les premières lignes de code de Maku, l'ambition était modeste : je voulais juste une petite interface web pour lancer des agents dans des sandboxes Kubernetes et garder une trace propre de ce qu'ils faisaient. Au fil des semaines et des irritants rencontrés en conditions réelles, le projet s'est transformé en une véritable **usine à coder**.

Pour autant, la boussole n'a jamais changé : bâtir une software factory **efficace, évolutive et observable**, tout en restant **aussi naturelle que possible pour le développeur**. Qu'on parte d'une session Claude Code ou OpenCode, d'un fil Slack ou d'une issue GitHub, la machinerie de sécurité s'efface derrière des gestes simples, et l'humain garde le dernier mot là où son jugement compte vraiment.

Avec la sortie de la **v4**, un cap important vient d'être franchi : Maku est désormais découplé de mon infrastructure personnelle et peut être déployé par n'importe quelle équipe sur son propre cluster Kubernetes (ou testé en 15 minutes sur son laptop avec `task demo`). Comme je le glissais en introduction, la prochaine grande étape avec la **v5** sera de confronter cette usine au terrain au sein d'une **entreprise partenaire** sur de vrais projets d'équipe — avis aux intéressés ! 😉

Nous avons vu ici comment Maku orchestre l'expérience d'ensemble à travers un cas d'usage concret. Dans les prochains articles de la série, nous allons ouvrir le capot brique par brique, à commencer par le **harnais** : comment chaque agent est isolé dans sa sandbox gVisor, sous quelle identité il tourne, et comment les rôles coopèrent sans jamais se faire confiance. 🔒

## 🔖 Références

* [`maku`](https://github.com/Smana/maku) : le dépôt du projet · [architecture](https://github.com/Smana/maku/blob/main/docs/architecture.md) · [installer sur votre cluster](https://github.com/Smana/maku/blob/main/docs/install.md) · [quickstart sur un laptop](https://github.com/Smana/maku/blob/main/docs/quickstart.md)
* [Démo interactive d'une room Maku](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : la plateforme IA de référence · [documentation sur airef.ogenki.io](https://airef.ogenki.io/) · [guide utilisateur](https://airef.ogenki.io/docs/use/guide/) · [état et feuille de route](https://airef.ogenki.io/docs/status/)
* [ADR-0057 : pourquoi séparer `ai-native-ref` de `cloud-native-ref`](https://airef.ogenki.io/docs/decisions/0057-ai-platform-own-repository/)
* [Crossplane](https://www.crossplane.io/) · [CloudNativePG](https://cloudnative-pg.io/) · [agentgateway](https://agentgateway.dev/) · [gitleaks](https://github.com/gitleaks/gitleaks) · [Model Context Protocol](https://modelcontextprotocol.io/)
