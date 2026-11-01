+++
author = "Smaine Kahlouch"
title = "Agents et humains dans la même équipe : une `Agent Factory` efficace et sécurisée"
date = "2026-11-04"
summary = "Gagner en autonomie et mieux collaborer à partir d'une simple issue GitHub : un label suffit à ouvrir un espace de travail cloisonné où agents et humains se relaient jusqu'à une PR relue, sécurisée et traçable."
featured = true
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — la suite logique 🏭" %}}
Dans les épisodes précédents : [les fondamentaux des coding agents](/fr/post/series/agentic_ai/ai-coding-agent/), [mes workflows Claude Code au quotidien](/fr/post/series/agentic_ai/ai-coding-tips/) et [une stack LLM self-hosted](/fr/post/series/agentic_ai/llm-self-hosted-stack/). Jusqu'ici, l'agent tournait **sur mon laptop**, avec mon propre **harnais** et moi pour seul relecteur. Il est temps de changer d'échelle : un environnement **partagé**, où l'équipe travaille avec les agents et peut faire confiance à ce qu'ils livrent.
{{% /notice %}}

Depuis quelques mois, je ne code presque plus sans agent, et je constate la même évolution autour de moi : on écrit moins de code, on pilote davantage d'agents.

Seulement, ce qui fonctionne sur mon laptop ne se transpose pas tel quel à une équipe. Chacun arrive avec ses instructions, ses outils et ses permissions, et les pratiques divergent vite d'un poste à l'autre. Dans une organisation, il ne suffit pas que mon agent soit efficace : il faut pouvoir garantir la qualité et la sécurité de ce qui part **réellement** en production, quelle que soit la personne qui a lancé l'agent.

En parallèle, les agents gagnent en fiabilité, au point qu'on peut envisager de leur confier certains changements de bout en bout, de l'issue jusqu'au merge. Encore faut-il savoir sur quels critères leur faire confiance, et pouvoir le mesurer. C'est tout l'enjeu des **software factories**, dont on entend beaucoup parler cette année.

Cet article décrit l'**Agent Factory**, ma tentative de réponse à ces questions : une chaîne qui va de l'issue à la PR mergée, construite avec des outils open source cloud native. Elle s'appuie sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), ma plateforme de référence, qui fournissait déjà une bonne partie du socle (Crossplane, Cilium, ZITADEL, OpenBao, l'observabilité). J'y ai ajouté une couche dédiée aux agents et la glue nécessaire pour faire tourner la machinerie 😉.

Je l'ai éprouvée en conditions réelles, sur **GKE** comme sur **EKS**. Toutes les briques sont open source, et rien n'oblige à tout reprendre : piochez les idées qui vous parlent, adaptez-les à votre contexte et faites-en profiter votre équipe. Entrons dans le vif du sujet ! 🚀

## 🎯 Objectifs de cet article

* 🏭 Comprendre ce qu'on appelle une **software factory**, et où se situe celle-ci sur l'échelle de l'autonomie
* 🤖 Découvrir les **rôles** d'une tâche : un agent qui cadre, un qui écrit les tests, un qui implémente, un qui juge, un qui relit
* 🧩 Voir comment **Crossplane, Cilium, gVisor, agent-sandbox, agentgateway, octo-sts et Kueue** s'assemblent pour qu'un run d'agent soit **isolé, identifié, mesuré et stoppable**
* 🔐 Suivre une **identité de bout en bout** : un humain, un run, un jeton GitHub d'une heure, et aucun secret dans la sandbox
* 🚦 Décortiquer le **merge gate** : deux façons de merger, des classes de changement, le mode *shadow* et un disjoncteur qui se teste
* 🩺 Fermer la boucle avec [RunLore](/fr/post/series/observability/runlore/) : d'une alerte à une PR mergée et vérifiée
* 📊 Lire des **chiffres réels** (coûts, cache, PRs mergées) et une liste honnête de ce qui manque encore

## 🏭 Software factory : de quoi parle-t-on ?

Le vocabulaire s'est stabilisé cet été. Addy Osmani le résume ainsi dans [Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/) : une **boucle** (*loop*) est un agent qui observe, agit et vérifie jusqu'à une condition d'arrêt ; un **harnais** (*harness*) est tout ce qui entoure cette boucle : sandbox, outils, permissions, définition du « terminé » ; une **factory**, ce sont de nombreuses boucles harnachées, alimentées par une file de travail et filtrées par un **gate de revue** avant la production. Sa phrase clé : *« Verification, not generation, is the real constraint on a factory. »*

Dan Shapiro, lui, a proposé [cinq niveaux](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/) calqués sur ceux de la conduite autonome. Au niveau 3, vous supervisez des agents et vous **lisez chaque diff** ; au niveau 4, vous écrivez des specs, vous relisez des plans et vous revenez voir si les tests passent ; au niveau 5, la *dark factory*, **plus personne ne lit le code**. L'exemple le plus commenté est celui de [StrongDM](https://simonwillison.net/2026/Feb/7/software-factory/) : *« Code must not be reviewed by humans »*, des scénarios de test tenus hors du dépôt pour que les agents ne puissent pas les voir, et un coût que Simon Willison estime à ~1 000 $ par ingénieur et par jour.

Et puis il y a le contre-exemple. Dex Horthy (HumanLayer) raconte, dans sa conférence *Harness Engineering is not Enough*, une factory entièrement automatisée qui a tenu **environ quatre mois** avant une panne majeure, longue à diagnostiquer. Osmani en tire le concept de **dette de compréhension** : *« the widening gap between how much code exists and how much any human still understands. »* La panne n'est pas spectaculaire ; elle est *« quiet and late »*.

{{% notice note "Où se situe cette factory ? 🧭" %}}
**Au niveau 4 pour presque tout, au niveau 5 pour une seule voie.** J'écris l'issue, les agents cadrent, implémentent, testent et se relisent entre eux, et **mon approbation du commit vaut merge**. Une seule famille de changements se merge sans moi : les corrections de liens dans la documentation, au plus une par jour, avec un disjoncteur qui la renvoie en revue humaine au premier revert. Pas de *dark factory*, et ce n'est pas l'objectif : l'objectif est de savoir **exactement** ce qui doit être vrai avant qu'une machine merge quoi que ce soit.
{{% /notice %}}

Ce qui m'a frappé en lisant tout ça, c'est à quel point le débat porte sur les modèles et les boucles, et si peu sur le **harnais côté plateforme** : où tourne le code, avec quels droits, et qui le constate. Un guide récent de Firecrawl le dit sans détour : *« An AI software factory is five stages with a gate at each one, and the coding agent is the cheapest piece of it. »* Le projet [Agent Baseline](https://agentbaseline.org) a même publié une architecture de référence, *The Secure Software Factory*, organisée autour de quatre frontières : la **sandbox**, le **réseau**, la **chaîne d'approvisionnement** et **l'identité**. C'est, à peu de choses près, le plan de cet article.

Un mot de vocabulaire avant d'entrer dans le concret, parce que je vais beaucoup l'utiliser : une **sandbox**, c'est l'environnement isolé dans lequel l'agent exécute du code ; l'**egress**, c'est le trafic réseau qui en *sort* ; un **run**, c'est une exécution d'un agent pour une tâche, du démarrage du pod à sa fin ; et le **merge gate**, c'est l'ensemble des conditions à remplir pour qu'une PR soit mergée, qu'un humain le fasse ou non.

## 🗺️ Vue d'ensemble : trois dépôts, trois couches

L'ensemble repose sur trois dépôts, et la séparation n'est pas cosmétique : elle dit qui possède quelle frontière de confiance.

| Couche | Dépôt | Ce qu'elle apporte |
|---|---|---|
| **Le cluster hôte** | [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) | GKE ou EKS, Cilium en *default deny*, Crossplane, Gateway API, ZITADEL, OpenBao, la stack Victoria… et le dépôt sur lequel les agents travaillent |
| **La plateforme IA** | [`ai-native-ref`](https://github.com/Smana/ai-native-ref) | Les manifests Flux de l'Agent Factory : runtime gVisor, `agent-sandbox`, l'agent gateway, `octo-sts`, Kueue, les policies Kyverno et Cilium, les dashboards et les alertes |
| **Le contrôleur** | [`maku`](https://github.com/Smana/maku) | Le contrôleur de la factory, le broker de *rooms*, le sidecar *bridge*, la CLI `maku`, la Web UI et l'app Slack |

Un détail d'architecture que j'aime bien : la plateforme IA arrive dans le cluster hôte sous la forme d'un **artefact OCI signé** (`oci://ghcr.io/smana/ai-native-ref`), épinglé par digest, vérifié par cosign et **suspendu par défaut**. Le cluster hôte publie un *contrat* (`HOST-CONTRACT.md`) qui liste ce qu'il fournit (namespaces, Gateways, mounts OpenBao, pool de nœuds gVisor) et la plateforme IA se déploie par-dessus. Le jour où il faut tout refaire, un `git revert` du pin suffit.

{{% notice tip "Agent Factory, Maku, la plateforme : qui est qui ? 🎭" %}}
L'**Agent Factory** est le produit : un assemblage d'outils open source (Maku, agentgateway, OpenHands, gVisor, `agent-sandbox`, octo-sts, Kueue, RunLore…) qui transforme une issue en PR relue. **Maku** (幕, « le rideau » en japonais) en est le contrôleur : `maku-factory` (le reconciler des tâches), `maku-broker` (le journal des rooms), `maku-bridge` (le sidecar dans chaque pod de run), la CLI `maku`, la Web UI et l'app Slack. Cette séparation est voulue : Maku ne fait **pas** confiance à l'agent et ne cherche pas à le confiner lui-même ; c'est le travail de la plateforme.
{{% /notice %}}

Voici une tâche de bout en bout, dans ses grandes lignes :

{{< img src="agent-factory-overview.png" alt="Vue d'ensemble : un mainteneur labellise une issue, Maku ouvre une room et démarre un run dans une sandbox gVisor ; chaque appel passe par l'agent gateway vers les modèles, les outils MCP et octo-sts ; le run pousse une branche et ouvre une PR que le mainteneur relit" width="1200" >}}
</br>

1. **Un mainteneur** pose le label `factory/ready` sur une issue. Le même label posé par quelqu'un d'autre ne déclenche rien : Maku revérifie que l'auteur a bien le droit `write` sur le dépôt.
2. **Maku** prend un instantané de l'issue, crée une `Task` et une `Room`, choisit un *tier* (modèle, budget en tokens et en dollars, durée max), puis lance les runs de la tâche.
3. **Crossplane** compose chaque run : un `ServiceAccount` sans aucun droit Kubernetes, une `CiliumNetworkPolicy` dédiée et une `Sandbox` qui devient un pod **gVisor** dans le namespace `agents`.
4. **Les agents** se relaient sur la branche `agent/<task>` : un *scoper* cadre la tâche, l'*implementer* (le harnais [OpenHands](https://github.com/OpenHands/OpenHands)) écrit le changement et ouvre la PR, un *juge* vérifie que c'est terminé, un *reviewer* relit. Chaque appel au modèle, à un outil MCP ou à GitHub passe par l'**agent gateway**, sous l'identité du run.
5. **Le merge gate** décide : merge automatique pour une classe `live`, « j'aurais mergé » pour une classe `shadow`, et sinon il vous rend la main avec **une seule action** à faire. Si vous approuvez le commit de la PR, c'est le merge.

Tout ce qui précède est **narré sur l'issue GitHub** par Maku, étape par étape. Voilà par exemple ce qu'il a écrit, tel quel, pendant la correction d'une règle d'alerte le 28 octobre :

```text
Agent factory task `drq5gh2i` started run `ralxknfk` (implementer) on branch `agent/drq5gh2i`.
- Budget: 1.5 M tokens, 45 minutes (tier standard)
- Watch: https://rooms.priv.gcp.ogenki.io/r/drq5gh2i (tailnet only)
- Stop: apply the label `factory/stop`

Run `ralxknfk` of task `drq5gh2i` opened #2638.

Agent factory task `drq5gh2i` waits for a maintainer's review: the policy needs a maintainer's approval.

Agent factory task `drq5gh2i` is done: the pull request was merged. Merged by @Smana.
Tokens used: 490 k.
```

Budget affiché, lien vers la room, bouton d'arrêt, identité du mergeur : l'issue **est** le compte rendu. Voyons maintenant ce qui se cache derrière chacune de ces lignes.

## 🤖 Les rôles : qui fait quoi dans une tâche

La v1 avait deux agents : un qui écrit, un qui relit. Sur des issues floues, l'implementer partait dans tous les sens : l'un de nos runs a dépensé tout son budget (2,55 $) en 319 appels à `grep` sans rien pousser. La v3 mise donc sur la **tâche** plutôt que sur le modèle : des rôles qui ajoutent du jugement **sans ajouter d'écrivains**. Le principe est simple : seuls deux rôles écrivent, tous les autres lisent et rapportent.

| Rôle | Ce qu'il fait | Écrit ? |
|---|---|---|
| **Scoper** | Lit l'issue et la transforme en tâche bornée, avec des **critères d'acceptation** vérifiables. S'il manque une information, il pose **une** question à l'auteur au lieu de deviner | Non |
| **Splitter** | Propose de découper une issue trop grosse en parties, avec un ordre de merge. Un mainteneur valide le découpage | Non |
| **Tester** | Écrit d'abord des tests qui **échouent** sur le code actuel. Ils deviennent le contrat de l'implementer, qui n'a pas le droit de les modifier | Les tests |
| **Implementer** | Écrit le changement, lance les **checks du dépôt dans sa sandbox** avant de pousser, et ouvre la PR | Le code |
| **Search** | Un assistant en lecture seule que l'implementer interroge sur le dépôt, plutôt que de tout relire lui-même | Non |
| **Juge** | Un seul appel de modèle, avec un contexte neuf : le changement satisfait-il les critères ? `not_yet` renvoie l'implementer au travail, `impossible` vous escalade la tâche | Non |
| **Reviewer** | Relit la PR avec **un autre modèle** que l'implementer et pose un verdict. Pour une classe qui peut se merger seule, le gate exige son approbation | Non |

L'implementer et le reviewer tournent sur presque chaque tâche ; les autres rôles s'activent **par classe de changement** (on y revient dans la section sur le merge gate). Sur la plateforme de référence, le scoper et le juge sont actifs sur la doc et les scripts, tandis que le tester et la recherche restent des interrupteurs que j'allume pour une classe le temps de mesurer ce qu'ils apportent. Chacun a aussi un **profil de capacités**, compilé dans les règles du gateway : la liste des modèles et des outils MCP qu'il peut appeler, selon la **classe de données** de la tâche. Pour un dépôt public, les agents tournent sur des modèles open-weight (GLM-5.3 par défaut, Kimi et Mistral en option) ; pour un travail interne, uniquement sur Claude.

{{% notice info "Pourquoi pas plus d'agents ? 🤔" %}}
Les recherches de 2026 vont toutes dans le même sens : un **scoper** améliore nettement les résultats sur les issues mal spécifiées ([Ask or Assume](https://arxiv.org/abs/2603.26233) : 69,4 % contre 61,2 %), un **reviewer à contexte neuf** rattrape ce que l'auteur ne voit plus ([Cognition](https://cognition.com/blog/multi-agents-working)), et un **tester test-first** cadre l'implémentation ([TDFlow](https://arxiv.org/abs/2510.23761)). À l'inverse, les essaims d'agents qui écrivent en parallèle coûtent 7 à 15 fois plus de tokens et dégradent le travail séquentiel. D'où la règle : on ajoute des relecteurs, pas des écrivains.
{{% /notice %}}

Depuis la v3, vous pouvez aussi lancer une tâche **sans passer par GitHub** : depuis un fil Slack (l'app rédige l'issue, un ✅ d'un mainteneur la dépose), depuis la CLI `maku`, ou directement depuis votre coding agent (Claude Code, OpenCode) grâce au serveur MCP de Maku et au skill `factory-handoff`. Le plus pratique au quotidien : une tâche annexe qui surgit en pleine session, que je confie à la factory sans quitter mon terminal.

## 🔒 Un environnement cloisonné et traçable

Un agent qui travaille sans surveillance exécute du code que personne n'a relu. Plutôt que de lui demander d'être prudent, la plateforme construit pour chaque run un environnement **éphémère et cloisonné** : une identité, un accès réseau restreint et une sandbox, créés au démarrage et détruits à la fin. Et chaque action qui s'y déroule est **attribuée** à quelqu'un.

### 🧩 Un run, une claim `AgentRun`

C'est ici que **Crossplane** montre ce qu'il sait faire. Pour lancer un run, Maku ne crée ni pod ni policy réseau : il dépose une seule ressource de haut niveau, une claim `AgentRun`, qui décrit le *quoi* sans rien dire du *comment* :

```yaml
# Une claim AgentRun, telle que Maku la crée
apiVersion: cloud.ogenki.io/v1alpha1
kind: AgentRun
metadata:
  name: xplane-run-7f3cq2xz
  namespace: agents
spec:
  role: implementer                       # implementer | reviewer | tester | triager
  repository: Smana/cloud-native-ref
  principal: "human:312345678901234567"   # la personne à qui le run est imputé
  dataClass: public                       # public | internal
  task:
    text: "Fix the broken link in docs/README.md."
  budget:
    maxTokens: 2000000
  egress:
    profiles: [pypi]                      # registres autorisés en plus de GitHub
```

La [composition](https://github.com/Smana/crossplane-configuration/tree/main/apis/agentrun), écrite en KCL, traduit cette intention en ressources Kubernetes, toutes nommées d'après le run :

{{< img src="agentrun-composition.png" alt="La composition AgentRun : une claim AgentRun est rendue par Crossplane en ServiceAccount sans droits, ConfigMap de tâche, CiliumNetworkPolicy et Sandbox ; le pod gVisor contient le harnais OpenHands, identity-proxy et maku-bridge, et tourne sur GKE Sandbox ou sur un pool Karpenter EKS ; chaque appel sort par l'agent gateway, octo-sts délivre un jeton GitHub limité aux branches agent/**, et maku-bridge relaie les événements vers le broker de rooms" width="1200" >}}
</br>

| Ressource composée | Son rôle |
|---|---|
| `ServiceAccount` | L'identité du run, **sans aucun droit** sur l'API Kubernetes |
| `ConfigMap` | La tâche et les règles que l'agent doit suivre |
| `CiliumNetworkPolicy` | Les seules destinations que ce run peut joindre |
| `Sandbox` | Le pod gVisor : le harnais OpenHands, accompagné de deux sidecars. `identity-proxy` porte l'identité du run vers l'extérieur, `maku-bridge` relaie ses événements vers la room |

Au-delà du YAML économisé, ce modèle apporte quatre propriétés :

* **Des garde-fous dans l'API.** Un run `internal` qui demande un profil d'egress est refusé dès sa création, par le schéma lui-même. Et une claim est **immuable** : seul son budget de tokens peut changer. Pour modifier un run, on en lance un autre.
* **Un état au même endroit.** Le statut de la claim donne la phase du run, les tokens consommés et la PR ouverte. Une révocation (budget épuisé, arrêt manuel) y est **définitive** : un run révoqué ne redémarre pas.
* **La même claim sur deux clouds.** Un `AgentRun` produit le même pod gVisor sur GKE et sur EKS ; seuls les nœuds diffèrent.
* **Un nettoyage garanti.** À la fin du run, le pod est supprimé sans jamais être recréé, et l'identité du run disparaît avec lui.

Enfin, **Kueue** admet les sandboxes dans une file plafonnée (16 CPU, 64 Gi, 4 pods) : une factory qui s'emballe ne peut pas remplir le cluster.

### 🙌 Le harnais : OpenHands

[OpenHands](https://github.com/OpenHands/OpenHands) est une plateforme open source d'agents de développement, disponible en SDK Python, en CLI, en interface web et en offre cloud. Toutes ces déclinaisons reposent sur la même boucle : l'agent observe, agit avec ses outils et vérifie le résultat, jusqu'à ce que la tâche soit terminée. Ses principales fonctionnalités :

* **Des outils de développeur** : un terminal, un éditeur de fichiers et un gestionnaire de tâches, extensibles par des serveurs **MCP**.
* **L'indépendance vis-à-vis du modèle** : il s'adresse à n'importe quel fournisseur via LiteLLM.
* **Un condenseur de contexte** : à intervalles réguliers, la première moitié de l'historique est remplacée par un résumé, ce qui garde les longues sessions dans la fenêtre de contexte.
* **Un serveur d'agent** (*agent-server*) : il exécute les conversations et expose chaque action de l'agent comme un événement, via une API REST et WebSocket.

**Dans la factory**, OpenHands ne tourne qu'à un seul endroit : le conteneur du harnais, dans le pod gVisor de chaque run. Rien ne s'y connecte depuis l'extérieur ; c'est la sandbox qui ouvre chaque connexion, vers l'agent gateway ou vers la room. Trois fonctionnalités font ce lien :

| OpenHands fournit | Dans la sandbox | Hors de la sandbox |
|---|---|---|
| Les outils MCP | L'agent appelle un outil | Les serveurs MCP (Flux, Victoria…) tournent derrière l'agent gateway, qui vérifie l'identité du run |
| LiteLLM | L'agent appelle un modèle | L'agent gateway injecte la clé du fournisseur et n'autorise que les modèles permis pour la classe de données : open-weight pour un dépôt public, Claude pour un travail interne |
| L'API d'événements | `maku-bridge`, sidecar du même pod, lit chaque événement en local | Le bridge relaie ces événements au broker, qui les inscrit dans la room |

L'image du harnais n'ajoute que le strict nécessaire : un *credential helper* Git qui obtient le jeton GitHub du run, un wrapper `gh` qui ajoute un pied de provenance aux PR, et la boucle du juge.

{{% notice info "Le *harness engineering* en bref 🧰" %}}
Le terme s'est imposé début 2026 : le **harness engineering** est la conception de tout ce qui entoure le modèle, selon l'équation **Agent = Modèle + Harnais**. OpenAI a [raconté](https://openai.com/index/harness-engineering/) comment une de ses équipes a cessé d'écrire du code pour concevoir l'environnement, les contraintes et les boucles de feedback de ses agents. Birgitta Böckeler en [propose une grille de lecture](https://martinfowler.com/articles/harness-engineering.html) selon deux axes : un contrôle agit **avant** ou **après** l'agent, et il est **déterministe** ou confié à un modèle.

| | Computationnel (déterministe, rapide) | Inférentiel (un modèle évalue) |
|---|---|---|
| **Guide** (*feedforward*) : oriente l'agent avant qu'il agisse | Codemods, scripts de bootstrap | Conventions dans `AGENTS.md`, skills |
| **Senseur** (*feedback*) : vérifie ce qu'il a produit | Tests, linters, tests d'architecture | Revue de code par IA, *LLM as judge* |

Elle distingue aussi le harnais **intégré** à l'outil, conçu par ses auteurs, du harnais **extérieur** que chacun construit autour pour son propre contexte. Ici, OpenHands fournit le premier. La factory est un harnais extérieur porté par la plateforme, avec ses guides (les règles du run, les critères d'acceptation du scoper) et ses senseurs (les checks du dépôt, le juge, le reviewer).
{{% /notice %}}

### 🪪 Qui agit ? Une identité par intervenant

Dans une room comme sur GitHub, chaque action est attribuée à l'un de ces trois intervenants. Et une identité n'est jamais **déclarée** par celui qui écrit : le broker la déduit du jeton qu'il vient de vérifier.

| Intervenant | Identité | Comment elle est établie |
|---|---|---|
| 👤 Un humain | `human:<sub>` | Sa connexion SSO via ZITADEL (Web UI, CLI `maku`). Ses comptes GitHub et Slack sont liés à ce compte |
| 🤖 Un agent | `agent:<runId>` | L'identité Kubernetes de son pod, émise par le cluster pour ce seul run |
| ⚙️ La factory | `system:factory` | L'identité de service du contrôleur Maku |

Deux conséquences concrètes :

* Les dépenses sont **imputées à la personne** qui a lancé la tâche (le champ `principal` de la claim), pas à un compte technique anonyme.
* Ce qu'écrit un agent reste du **contenu non fiable, attribué à cet agent**. Un verdict, un changement de pilote ou une approbation ne peuvent venir que de Maku ou d'un humain.

### 🔑 Aucun secret dans la sandbox

Les clés des fournisseurs de modèles restent dans OpenBao. C'est l'**agent gateway** ([agentgateway](https://agentgateway.dev/)) qui les injecte au passage, après avoir vérifié l'identité du run. Il compte aussi les tokens par run, par personne et pour toute la flotte : un budget épuisé se traduit par un refus explicite, pas par une facture surprise.

Même principe pour GitHub : le run ne détient **aucun** credential durable. Quand il doit pousser, son identité est échangée via [`octo-sts`](https://github.com/octo-sts/app) contre un jeton GitHub valable **au plus une heure**, pour **un dépôt** et **un rôle**. Les règles de cet échange sont versionnées **dans le dépôt cible** : c'est son propriétaire qui décide quels runs peuvent y écrire. Un reviewer ou un juge n'obtient qu'un accès en lecture, et des *rulesets* GitHub cantonnent les agents aux branches `agent/**` : pas de push sur `main`, pas de tag.

Enfin, une seule GitHub App a le droit de merger, et sa clé ne quitte jamais le contrôleur de Maku : **un agent ne peut ni approuver ni merger sa propre PR**, par construction.

### 🧱 L'isolation : gVisor

Chaque pod de run tourne sous **gVisor**, sur des nœuds dédiés : GKE Sandbox côté GCP, un pool Karpenter côté AWS. gVisor intercale un noyau en espace utilisateur entre le conteneur et le noyau de l'hôte : une tentative d'évasion par appel système doit d'abord le traverser. Je l'ai préféré aux microVMs (Kata, Firecracker) parce qu'il tourne sur des **nœuds standards** des deux clouds, et que [`agent-sandbox`](https://agent-sandbox.sigs.k8s.io/), un projet des SIG Kubernetes, le prend en charge directement.

Le surcoût est réel mais discret : des I/O fichiers environ 1,7× plus lentes, presque invisibles puisqu'un agent passe l'essentiel de son temps à attendre le modèle. Des policies Kyverno garantissent qu'aucun pod du namespace `agents` ne tourne autrement que sous gVisor, créé par une `Sandbox`.

### 🌐 L'egress : tout est fermé, sauf ce qui est nommé

Le namespace `agents` refuse tout trafic par défaut, et la `CiliumNetworkPolicy` de chaque run n'ouvre que le strict nécessaire : GitHub, l'agent gateway et le broker de rooms. Les registres de paquets (`pypi`, `npm`, `golang`, `crates`) ne s'ajoutent que si la tâche en a besoin. Un nom non autorisé **ne résout même pas** : l'agent reçoit `Temporary failure in name resolution`, et Hubble journalise la requête refusée. Pour un travail `internal`, c'est plus strict encore : aucun accès direct à GitHub, la lecture passe par le gateway et aucun push n'est possible.

{{% notice warning "Pourquoi autant d'insistance sur l'egress ? ☠️" %}}
Parce que c'est le troisième terme de la [*lethal trifecta*](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) de Simon Willison : accès à des données privées + exposition à du contenu non fiable + **capacité à communiquer vers l'extérieur**. Un agent qui lit des issues GitHub et du code tiers a forcément les deux premiers. Anthropic a documenté [un cas réel](https://www.anthropic.com/research/how-we-contain-claude) d'exfiltration **par l'endpoint d'inférence autorisé**, avec une clé fournie par l'attaquant, et conclut : *« Design for containment at the environment layer first, then steer behavior at the model layer. »* Je ne demande pas au modèle de se méfier ; je coupe le canal.
{{% /notice %}}

## 📓 La room : la boîte noire de chaque run

{{< img src="room.gif" alt="Trafalgar Law (One Piece) qui déploie sa « Room »" width="498" >}}

Quand l'agent travaille sans vous, la question suivante est : **qu'a-t-il fait, exactement ?** Un log de pod ne suffit pas : il disparaît avec le pod, n'importe qui peut y écrire n'importe quoi, et il mélange ce que l'agent *dit* avoir fait avec ce qu'il a *réellement* appelé. La réponse de Maku s'appelle une **room** : un journal partagé par tâche, à la fois boîte noire d'avion et salon de discussion. Pendant le run, vous suivez chaque étape, vous pouvez poster une consigne pour le run suivant, et vous approuvez ce que l'agent n'a pas le droit de faire seul. Après le run, la room reste, et c'est elle que vous rouvrez quand une PR vous surprend trois semaines plus tard.

{{< img src="maku-room.png" alt="Une room Maku qui attend un mainteneur : l'état « Waiting — review and approve PR #42 », le budget en tokens, le dernier verdict du reviewer, l'encadré NEXT qui dit que l'approbation du commit courant vaut merge, l'action exacte sous « Needs you », puis le fil des événements de l'implementer" width="1100" caption="Une room qui attend votre approbation : la prochaine étape, l'action exacte à faire, et le fil des événements du run (capture rendue depuis la démo de Maku)." >}}
</br>

Ce qui fait qu'on peut s'y fier :

* **Append-only.** Le rôle PostgreSQL du broker n'a que `SELECT` et `INSERT` sur la table des événements ; un trigger interdit de dé-sceller ou de re-dater. La preuve tient en une ligne :

  ```sql
  -- Tentative de réécriture d'un événement, sous le rôle du broker
  SET ROLE rooms_broker; UPDATE events SET payload = '{}' WHERE room_id = '3kq7x2ma';
  -- ERROR:  permission denied for table events
  ```

* **Séquencé sans trou.** Chaque événement reçoit un `seq` monotone sous verrou de ligne ; une clé d'idempotence `(room, client, seq d'origine)` fait qu'un rejeu renvoie l'événement déjà stocké sans en consommer un nouveau. À la fermeture, un événement de **sceau** interdit tout ajout.
* **Caviardé d'abord.** Chaque chaîne JSON passe par gitleaks **avant** d'être stockée, loguée ou comptée.
* **Attribué.** Chaque événement porte l'identité de son auteur (`human:`, `agent:` ou `system:`), établie par le broker à partir du jeton vérifié, jamais déclarée par l'émetteur.

Vous accédez aux rooms par la Web UI (derrière ZITADEL), par la CLI `maku` dans votre terminal, ou depuis votre propre coding agent via le serveur MCP. La CLI, volontairement, **ne peut rien approuver** : regarder, poster, mettre en file, forker, c'est tout. Les approbations d'actions se prennent dans la Web UI, et la carte d'approbation montre l'**action brute**, pas la prose de l'agent. La règle *four-eyes* (quand la room l'exige) interdit en plus à celui qui a soufflé la consigne d'approuver son résultat.

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku, rafraîchie toutes les 10 secondes : les tâches qui attendent un humain portent un badge « Needs you » et remontent en premier, chacune avec son dépôt, son état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms pendant la session sur EKS, le 31 octobre : ce qui attend un humain remonte en premier, et la page se rafraîchit seule." >}}
</br>

Pour vous faire une idée sans rien installer, une room réelle est rejouable dans la [démo publique](https://blog.ogenki.io/maku/).

## 🚦 Le merge gate : deux façons de merger

Voilà le point où l'on décide de ce qu'on délègue vraiment. Mon choix : l'autonomie n'est pas une fonctionnalité globale qu'on active, c'est un **privilège par classe de changement**, gagné statistiquement et **révocable**. Une *classe*, c'est une famille de PR définie par ce qu'elle a le droit de toucher et par la forme attendue de son diff : `docs-links` pour les liens de la documentation, `deps-patch` pour les mises à jour de dépendances en version patch, `alert-rules` pour les règles d'alerte, et ainsi de suite.

Une PR d'agent se merge donc de deux façons, et **c'est toujours l'App merger qui appuie sur le bouton** :

| Cas | Quand | Qui merge |
|---|---|---|
| **Seule** | Sa classe est `live` pour ce dépôt, la CI, policy-bot et le reviewer approuvent **ce** commit, le diff reste dans les limites de la classe, le disjoncteur ne l'a pas mise en pause, et le plafond quotidien n'est pas atteint | L'App merger, sans humain |
| **Sur une approbation** | Toute autre PR d'agent, dès qu'un mainteneur approuve son commit courant (*approve-to-merge*) | L'App merger, dans la foulée |
| **Jamais seule** | Une classe en `shadow` décide et enregistre, sans merger ; une PR sans classe est une PR ordinaire | Vous, via l'approbation |

Le deuxième cas est arrivé avec la v3, et il change tout au quotidien : avant, je relisais, j'approuvais, puis je devais encore contourner le ruleset pour merger. Désormais, **mon approbation est le merge**, et elle est liée au commit que j'ai lu. Si l'agent pousse après mon approbation, policy-bot la révoque et rien ne part.

Aujourd'hui, sur `cloud-native-ref`, deux classes sont `live` (`docs-links` et `revert`) ; `dead-links`, `deps-patch` et `deps-minor` décident en `shadow`. Sur `ai-native-ref`, le second dépôt, tout est en `shadow`.

### Deux gardiens

Le premier gardien est [**policy-bot**](https://github.com/palantir/policy-bot), qui lit `.policy.yml` **depuis `main`** (jamais depuis la PR) et publie un statut requis. Voici la règle qui autorise la classe `docs-links` :

```yaml
# .policy.yml (cloud-native-ref), extrait
- name: "low-risk: docs-links"
  if:
    has_author_in:            {users: ["ogenki-agents[bot]"]}
    only_has_contributors_in: {users: ["ogenki-agents[bot]"]}
    from_branch:              {pattern: '^agent/[a-z2-7]{8}$'}
    only_changed_files:       {paths: ['^(docs|website/content)/.+\.md$']}
    no_changed_files:
      paths: ['^\.policy\.yml$', '^\.github/', '(^|/)(AGENTS|CLAUDE)\.md$', ...]   # les « gate paths »
    modified_lines: {total: "< 21"}
    has_workflow_result: {conclusions: [success], workflows: [.github/workflows/ci.yaml]}
```

Une PR d'agent qui touche à `.github/`, à la politique elle-même, aux pins Crossplane ou aux mounts OpenBao ne peut **jamais** relever d'une classe automatique : ces *gate paths* repartent vers un humain, liens corrects ou pas. Un script CI échoue si une règle oublie l'un d'eux.

Le second gardien est le gate de Maku, qui applique une liste **ordonnée** de contrôles au commit courant de la PR : l'hygiène (scan de secrets, CI verte, contrat de tests intact), la politique (policy-bot vert, aucun *gate path* touché), la preuve (le reviewer a approuvé **ce** commit, le trailer `Agent-Run` nomme un run de la tâche, chaque push vient de l'App des agents), puis les limites de la classe, son disjoncteur et ses plafonds quotidiens. Le schéma ci-dessous les reprend dans l'ordre ; la liste complète, avec chaque code de raison, est dans [la documentation](https://airef.ogenki.io/docs/reference/gate-reasons/).

Chaque « non » est un **code de raison** que Maku écrit sur la PR, avec la **seule action** qui vous revient, et qu'il expose en métrique (`maku_factory_merge_decisions_total{class, verdict, reason, repository}`). Une PR d'agent porte aussi un pied de provenance (`Agent-Room`, `Agent-Run`, `Agent-Task`, `Agent-Model`), mais la doc de Maku est explicite : *« Trailers and footer are provenance hints, never authorisation. »* Ce qui lie un commit à la factory, c'est l'identité qui a poussé, pas ce qui est écrit dedans.

{{< img src="merge-gate.png" alt="Le merge gate : une PR d'agent passe quatre contrôles dans l'ordre (hygiène, politique, preuve, limites de la classe) ; le moindre « non » l'envoie à un mainteneur, dont l'approbation vaut merge. Si tout passe et que la classe est live, l'App merger merge sans humain, puis main est surveillée 30 minutes ; une classe en shadow écrit « would merge » et attend le mainteneur" width="760" >}}
</br>

**Et après un merge sans humain ?** Seuls deux événements déclenchent un revert :

| Déclencheur | Pendant | Ce qui se passe |
|---|---|---|
| 🔴 Un check de CI qui tourne sur `main` (pre-commit, scan de sécurité, validation Kubernetes, scripts, liens de la doc) échoue sur le commit de merge | 30 minutes après le merge | Revert automatique : l'App merger ouvre la PR de revert |
| 🙋 Un mainteneur pose le label `factory/revert` sur la PR | 7 jours après le merge | L'App merger ouvre la PR de revert à sa demande |

Sur `cloud-native-ref`, cette PR de revert se merge elle-même sans humain, puisque la classe `revert` y est `live`. Chaque revert compte pour le disjoncteur décrit plus bas. Une PR mergée par un humain, elle, n'est jamais revertée par la factory.

Certaines classes sont surveillées plus longtemps : un moniteur vérifie par exemple que Flux applique bien le changement, ou qu'aucune PR ne vient le corriger dans la semaine. Ce moniteur ne revert jamais rien. Il note le résultat, et c'est le disjoncteur qui en tient compte.

### Du shadow au live, et retour

Comment décider qu'une classe peut se merger sans humain ? En la mettant d'abord à l'épreuve, sans risque. C'est le rôle du **mode shadow** (« ombre »), une sorte de répétition générale :

* **En shadow**, le gate fait tout son travail mais ne merge rien. Il écrit simplement sur la PR ce qu'il aurait fait (*« would auto-merge »*). Un mainteneur relit et merge, comme d'habitude.
* **En live**, le gate merge lui-même, sans humain.

Chaque PR traitée en shadow devient un test. Quand le gate disait « j'aurais mergé » et que la PR, mergée par un humain, n'a rien cassé (ni revert, ni correctif, ni régression), c'est un bon point. Sinon, c'en est un mauvais. Un merge n'est compté qu'après un délai d'observation, le temps de voir s'il a vraiment tenu.

Pour passer une classe en live, quelques succès ne suffisent pas. Je veux pouvoir affirmer, avec 95 % de confiance, que **moins d'un merge sur dix** poserait problème. Le calcul statistique (la borne de Clopper-Pearson) donne le nombre de merges propres nécessaires : **29** s'il n'y a eu aucun mauvais merge, 46 s'il y en a eu un, 61 s'il y en a eu deux. Pas de démo réussie trois fois qui devient une autonomie générale.

Dans l'autre sens, un **disjoncteur** protège les classes déjà en live. Au moindre incident parmi les dix derniers merges d'une classe (un revert, ou une PR qui a dû corriger la précédente), la classe perd son autonomie et repasse sous approbation humaine. Elle la retrouve d'elle-même quand ses trois merges suivants, approuvés entre-temps par un humain, se sont révélés propres.

Ce disjoncteur, je l'ai testé exprès, lors de deux exercices (des *drills*). Le principe : laisser une PR se merger seule, puis demander son annulation avec le label `factory/revert`. La factory ouvre alors la PR de revert et la merge elle-même, sans personne. Pour que l'exercice ne compte pas comme un incident, il suffit de poser le label `factory/drill` juste avant : le disjoncteur l'ignore et la classe garde son autonomie. Le premier de ces exercices, sur GKE, a d'ailleurs été l'occasion du tout premier merge entièrement autonome de la factory.

## 🩺 RunLore ferme la boucle : de l'alerte à la PR vérifiée

Dans [l'article précédent](/fr/post/series/observability/runlore/), RunLore investiguait les alertes et apprenait de chaque résolution, mais s'arrêtait à la proposition. L'Agent Factory lui donne une suite : RunLore est un **client API ordinaire** de Maku, sous son propre principal machine (`system:runlore`, via ZITADEL), avec un plafond de **2 $ par tâche** et **10 $ par jour**.

{{< img src="runlore-loop.png" alt="La boucle RunLore : une alerte déclenche une investigation RunLore, qui crée une tâche investigate ; un run triager en lecture seule propose un correctif dans sa room ; un mainteneur publie la proposition avec factory/publish, la relit, puis la démarre avec factory/ready ; la PR est relue par un agent, mergée par un humain ; après un soak de 30 minutes RunLore interroge Alertmanager et rend son verdict" width="1200" >}}
</br>

Le point de design qui compte, ce sont les **deux labels** :

* `factory/publish` : la proposition de l'agent reste **dans sa room** tant qu'un mainteneur ne l'a pas relue. L'issue créée par Maku ne contient qu'un *stub* (alerte, ressource, confiance) ; aucun finding interne ne devient public sans ce geste.
* `factory/ready` : sur l'issue publiée, un **second** label autorise la dépense d'un run d'implémentation. Entre les deux, un humain a lu ce qui va devenir public **et** ce qui va coûter.

Voilà ce que ça a donné le **28 octobre**, sur une vraie alerte (`PodinfoNoAvailableReplica`, qui se déclenchait en permanence à cause d'une faute de frappe, `podnfo`, dans un sélecteur de `VMRule`) :

| Heure (UTC) | Quoi |
|---|---|
| 17:29 | RunLore rend un verdict `action_suggested` à 0,92 de confiance et crée la tâche `hvavktlo` ; Maku ouvre l'issue #2630 (stub) |
| 17:33 → 17:36 | Un run **triager**, en lecture seule, propose le correctif dans sa room : 206 k tokens, ~3 minutes |
| 17:36 | Je pose `factory/publish` ; Maku publie la proposition comme issue #2631 |
| 18:14 | Je pose `factory/ready` ; run implementer `ralxknfk` (tier standard, 1,5 M tokens / 45 min) |
| 18:16 | PR #2638 ouverte |
| 18:25 | Verdict du reviewer : **approve**, avec sa propre vérification (`kustomize build`, `grep` du nom fautif) |
| 19:20 | Je merge (classe `alert-rules`, revue humaine obligatoire) ; 490 k tokens au total |
| 21:32 | Le moniteur post-merge rend son verdict : CI de `main` saine, zéro PR de suivi |

Le lendemain, même scénario sur `AppWizardUnavailable` (#2657 → #2658 → #2659), et cette fois le moniteur a pu écrire : *« RunLore: resolved, alert AppWizardUnavailable »*. Le **soak** de 30 minutes après le merge interroge Alertmanager ; le verdict `resolved` ou `persisting` est celui d'Alertmanager, la ré-investigation ne sert que de preuve. Un `persisting` ouvre un suivi et peut faire sauter le disjoncteur de la classe.

Côté RunLore, la configuration tient en quelques lignes :

```yaml
# Configuration de RunLore : Maku comme délégué (extrait)
factory:
  url: http://agent-factory.agent-system.svc.cluster.local:8082
  token_url: https://auth.example.com/oauth/v2/token      # ZITADEL, client credentials
  client_id_env: FACTORY_CLIENT_ID
  client_secret_env: FACTORY_CLIENT_SECRET
  repository: Smana/cloud-native-ref
  max_cost_usd: 2
  webhook: {secret_env: FACTORY_WEBHOOK_KEY}               # Standard Webhooks, HMAC-SHA256
  soak: 30m
  verify:
    alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
```

Deux agents, deux dépôts, un seul humain aux trois points qui comptent : ce qui devient public, ce qui coûte, et ce qui est mergé.

## 📊 Ce que ça coûte, ce que ça donne

Assez d'architecture, place aux chiffres. Je tiens à ce que chacun porte sa date, parce que tout bouge vite : dans un mois, le modèle, les prix et les tiers auront probablement changé. Les agents du dépôt public tournent sur **GLM-5.3** (Z.ai) : une qualité proche des modèles frontière pour un coût par token bien inférieur, ce qui compte quand un agent boucle des centaines de fois.

| Mesure | Valeur | Contexte |
|---|---|---|
| PRs d'agents mergées | **58** du 27 septembre au 31 octobre | 37 sur `cloud-native-ref`, 21 sur `ai-native-ref` : docs, scripts, règles d'alerte, runbooks |
| …dont mergées sans humain | **3** (#2520, #2697, et une PR de docs sur `ai-native-ref`) | Plus le revert #2698, mergé par l'App merger pendant le drill |
| …dont mergées sur une approbation | **1** (`ai-native-ref` #271) | L'App merger a mergé **14 s** après mon approbation, au commit que j'avais lu |
| Part des tokens d'entrée lus en cache | **95,1 %** | 27,4 M sur 28,8 M, fenêtre de 24 h au 18 octobre |
| Coût mesuré sur cette fenêtre | **10,40 $** contre 41,64 $ sans cache | Recalcul tarifaire, pas un second run |
| Un run implementer complet (PR #2508, 225 étapes) | **1,65 $** | 4,43 M tokens d'entrée, dont 4,16 M en cache |
| Tier `standard` | 1,5 M tokens / 45 min / **2,55 $** par run, 5,10 $ par tâche | Plafond de la flotte : 42,50 $ par jour |
| Démarrage d'une sandbox, pool vide | **1,5 à 3,5 min** (205 s sur un nœud neuf) | gVisor + image du harnais |
| Démarrage avec un nœud gardé chaud | **p50 16 s, p90 17 s** sur 10 runs | Le nœud coûte 125,80 $/mois : **désactivé** par défaut |
| Cohorte « issues difficiles » (15 issues, 14–15 octobre) | factory + GLM-5.3 : **0/15** (9 $) · Claude Code Action + Claude : **13/15** (15,80 $) | Un dépôt, un run par issue |

Deux lectures. D'abord, **le cache est le premier levier de coût** : la boucle d'un agent relit sans cesse le même contexte, et 95 % de hits divisent la facture par quatre. Ensuite, et c'est plus inconfortable, **le modèle compte énormément sur les tâches difficiles**. GLM-5.3 fait très bien les corrections de docs et de scripts qui constituent l'essentiel du flux ; sur quinze issues volontairement dures, il n'en a livré aucune là où Claude en livrait treize. Le travail public ne pouvant pas passer sur Claude, c'est ce qui a orienté la v3 : **adapter la tâche au modèle** (le scoper qui borne, le splitter qui découpe, le juge qui renvoie au travail) plutôt que chercher le meilleur modèle unique.

Tout est instrumenté : les séries `maku_factory_*` couvrent les phases, le délai label → PR, le devenir des PRs par classe (`auto_merged`, `approved_merged`, `human_merged`, `reverted`, `follow_up`), les décisions du gate avec leur raison, les budgets et les coûts par run, par rôle, par personne et par modèle. Les traces ne contiennent **que des identifiants, le tier, la phase et un code de raison**, jamais le texte d'une issue ni un prompt. Le transcript complet reste dans la room, sous contrôle d'accès.

## 🔬 Les alternatives

L'Agent Factory n'arrive pas sur un terrain vide. Pour situer :

| Solution | Ce que c'est | Ce que l'Agent Factory fait autrement |
|---|---|---|
| [**GitHub Copilot cloud agent**](https://docs.github.com/copilot/concepts/agents/cloud-agent/about-cloud-agent) | L'agent cloud de GitHub, avec plan à approuver et, depuis le 28 octobre, [Agent Merge](https://github.blog/changelog/2026-10-28-agent-merge-watches-and-fixes-pull-requests-in-the-cloud/), qui corrige la CI et les conflits d'une PR | Agent Merge ne merge pas lui-même : il s'en remet à l'auto-merge natif de GitHub. Pas de classes, pas de shadow, pas de disjoncteur |
| [**Codex cloud**](https://learn.chatgpt.com/docs/cloud) | Un environnement par tâche chez OpenAI, accès réseau configurable par domaines | Vous n'opérez rien, mais vous ne possédez ni le journal ni la politique de merge |
| [**OpenHands**](https://github.com/OpenHands/OpenHands) | Le harnais que j'utilise, qui propose aussi son *resolver* GitHub (issue labellisée → PR) et un cloud | Le resolver fait l'intake et la PR ; l'identité par run, l'egress, les rôles, le gate et la room sont à construire autour |
| [**Optio**](https://github.com/jonwiggins/optio) | Orchestrateur self-hosted : tickets → agents dans des pods Kubernetes → PR, reprise sur échec CI | Le plus proche dans l'esprit ; un pod par dépôt plutôt qu'une sandbox par run, et pas de gate statistique par classe |
| **StrongDM** | La *dark factory* : aucun humain n'écrit ni ne relit, scénarios hors dépôt, clones de SaaS tiers | L'autre bout du spectre, et un budget d'un autre ordre |

Un point a changé en 2026, et il m'a obligé à revoir mon discours : **« ça tourne chez vous » n'est plus un argument**. Claude Code [peut exécuter ses sessions cloud](https://code.claude.com/docs/en/claude-code-on-the-web) dans l'environnement d'une organisation, et Cursor, Amp ou Devin proposent eux aussi des runners auto-hébergés. Les produits gagnent toujours sur le **temps jusqu'à la première PR**. Ce qui reste propre à cet assemblage, après une étude du paysage d'octobre, c'est plutôt :

* le **merge autonome par classe**, avec shadow, plafonds et disjoncteur : aucun des produits étudiés ne le documente ;
* la **politique de merge versionnée dans le dépôt** (`.policy.yml`, lue depuis `main`), et un mergeur unique dont aucun agent ne détient la clé ;
* le **routage des modèles par classe de données** au niveau du gateway : le travail public sur des modèles open-weight, l'interne sur Claude, et un run qui ne peut pas tricher ;
* un **plan de contrôle open source et GitOps**, de bout en bout.

Ce n'est pas une victoire universelle ; c'est un arbitrage, et il demande un cluster et de l'exploitation.

## 💭 Dernières remarques

Soyons clairs : c'est un chantier de cinq semaines, sur deux dépôts et deux clouds, et les clusters de référence sont démontés entre deux sessions pour des raisons de coût. Voilà ce que j'en retiens, et ce qui manque.

* **Le modèle est la partie bon marché.** Le remplacer tient dans un alias du gateway. Faire qu'un run agisse sous une identité courte, sans secret durable ni sortie réseau implicite, et laisse une preuve consultable : c'est ça qui a structuré la plateforme, et c'est transposable à n'importe quel harnais.

* **L'autonomie se gagne par classe, et se perd par classe.** Un correctif de liens et une mise à jour de dépendance ne présentent ni le même risque ni la même preuve ; ils ne méritent pas le même droit. Le gate en shadow, le tally et le disjoncteur transforment une opinion (« ça marche bien ») en décision (« 29 merges propres, borne sous 10 % »). Et `deps-minor` ne pourra **jamais** passer live, par construction.

* **Le vrai goulot, c'est la relecture.** C'est ce que disent aussi les équipes qui font tourner des agents à grande échelle : chez Spotify, *« code generation is no longer the bottleneck. PR review is »*. Faire de l'approbation le merge était un premier pas ; une file de revue commune à tous les dépôts en sera un autre.

* **Ce qui n'est pas encore là.** Aucun classifieur de complexité : chaque tâche est au tier `standard`. Plusieurs étapes du chemin humain n'ont pas encore tourné en live, comme le rejeu d'une politique ou le retour d'une tâche le lendemain. Les budgets par personne comptent sans refuser. Et le token GitHub est lié au dépôt et au rôle, pas au run : n'importe quel run implementer peut pousser sur n'importe quelle branche `agent/**`. Le gate le rattrape par le trailer et par l'identité de push, mais ce n'est pas une isolation.

* **La dette de compréhension reste le vrai risque.** Les rooms gardent la trace de *comment* un changement a été produit, et le fait que ce soit moi qui écrive l'issue et qui approuve garde la décision humaine. Mais quatre mois, c'est aussi l'horizon de la factory de Dex Horthy. Je verrai dans quelques mois si relire des PR d'agents sur un dépôt que je connais bien suffit à ne pas décrocher.

La suite est écrite, et elle a un nom : la v4, **« anyone can run it »**. Aujourd'hui, l'ensemble tourne bien… sur ma plateforme. L'objectif est que vous puissiez faire tourner toute la boucle sur un laptop (kind) en une demi-heure, puis l'installer sur votre propre cluster avec votre fournisseur d'identité, vos secrets et votre outil GitOps. Viendront ensuite une file de revue commune, la vérification sur de vrais services avant d'ouvrir la PR, et les campagnes sur plusieurs dépôts ([la feuille de route](https://airef.ogenki.io/docs/status/)). Tout est ouvert ([`maku`](https://github.com/Smana/maku), [`ai-native-ref`](https://github.com/Smana/ai-native-ref), [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)), et les [issues](https://github.com/Smana/ai-native-ref/issues) sont le meilleur endroit pour me dire où ça casse chez vous.

## 🔖 Références

### Dépôts, docs et démo

* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : la plateforme IA déployable · [documentation](https://airef.ogenki.io) · [la release v3.0.0](https://github.com/Smana/ai-native-ref/releases/tag/v3.0.0)
* [Les rôles des agents](https://airef.ogenki.io/docs/how-it-works/roles/) · [le merge gate](https://airef.ogenki.io/docs/how-it-works/merge-gate/) · [le guide utilisateur](https://airef.ogenki.io/docs/use/guide/)
* [`maku`](https://github.com/Smana/maku) : le contrôleur (factory, rooms, CLI, Slack) · [démo publique d'une room](https://blog.ogenki.io/maku/)
* [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) : le cluster hôte · [documentation](https://cnref.ogenki.io)
* [RunLore](https://runlore.io) · [l'article de la série observabilité](/fr/post/series/observability/runlore/)

### Composants

* [Kubernetes agent-sandbox](https://agent-sandbox.sigs.k8s.io/) · [gVisor](https://gvisor.dev/docs/) · [Kueue](https://kueue.sigs.k8s.io/)
* [agentgateway](https://agentgateway.dev/) · [octo-sts](https://github.com/octo-sts/app) · [policy-bot](https://github.com/palantir/policy-bot)
* [OpenHands](https://github.com/OpenHands/OpenHands) · [Cilium : politiques DNS et FQDN](https://docs.cilium.io/en/stable/security/policy/language/#dns-based)

### Software factories

* [Addy Osmani : Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/)
* [OpenAI : Harness engineering](https://openai.com/index/harness-engineering/)
* [Birgitta Böckeler : Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)
* [Dan Shapiro : The five levels, from spicy autocomplete to the software factory](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/)
* [Simon Willison : la software factory de StrongDM](https://simonwillison.net/2026/Feb/7/software-factory/)
* [Firecrawl : How to build an AI software factory](https://www.firecrawl.dev/blog/ai-software-factory)
* [Agent Baseline : The Secure Software Factory, architecture de référence](https://agentbaseline.org/software-factory.pdf)
* [awesome-software-factories](https://github.com/varun1505/awesome-software-factories) : le paysage, régulièrement mis à jour

### Sécurité des agents

* [Simon Willison : The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
* [Anthropic : How we contain Claude](https://www.anthropic.com/research/how-we-contain-claude) · [Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
* [Configurer l'accès Internet du Copilot coding agent](https://github.blog/changelog/2025-07-15-configure-internet-access-for-copilot-coding-agent)
