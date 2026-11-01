+++
author = "Smaine Kahlouch"
title = "Agent Factory : le harnais, un environnement cloisonné pour chaque agent"
date = "2026-11-18"
summary = "Où tourne un agent, sous quelle identité, que peut-il joindre, et qui fait quoi dans une tâche ? Une claim Crossplane par run, OpenHands dans une sandbox gVisor, aucun secret à l'intérieur, un réseau fermé sauf ce qui est nommé, et des rôles qui ajoutent du jugement sans ajouter d'écrivains."
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
1. [Vue d'ensemble : agents et humains dans la même équipe](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)
2. [Maku, le chef d'orchestre](/fr/post/series/agentic_ai/agent-factory-2-maku/)
3. **Le harnais : un environnement cloisonné pour chaque agent** (vous êtes ici)
4. [Le merge gate : une autonomie qui se mérite](/fr/post/series/agentic_ai/agent-factory-4-merge-gate/)
5. [Observer et maîtriser une flotte d'agents](/fr/post/series/agentic_ai/agent-factory-5-observabilite/)
6. [D'agent à agent : RunLore confie ses correctifs à la factory](/fr/post/series/agentic_ai/agent-factory-6-runlore/)
{{% /notice %}}

Un agent qui travaille sans surveillance exécute du code que personne n'a relu. On peut lui demander d'être prudent ; on peut surtout faire en sorte qu'il n'ait **pas les moyens** de mal faire. C'est tout le sujet de cet épisode : l'environnement que la plateforme construit autour de chaque agent, et la façon dont plusieurs agents se partagent une tâche.

Dans l'[épisode précédent](/fr/post/series/agentic_ai/agent-factory-2-maku/), Maku lançait des « runs ». Voyons maintenant ce qu'est un run, concrètement : où il tourne, sous quelle identité, ce qu'il peut joindre, et quel rôle il joue.

{{% notice info "Le *harness engineering* en bref 🧰" %}}
Le terme s'est imposé début 2026 : le **harness engineering** est la conception de tout ce qui entoure le modèle, selon l'équation **Agent = Modèle + Harnais**. OpenAI a [raconté](https://openai.com/index/harness-engineering/) comment une de ses équipes a cessé d'écrire du code pour concevoir l'environnement, les contraintes et les boucles de feedback de ses agents. Birgitta Böckeler en [propose une grille de lecture](https://martinfowler.com/articles/harness-engineering.html) selon deux axes : un contrôle agit **avant** ou **après** l'agent, et il est **déterministe** ou confié à un modèle.

| | Computationnel (déterministe, rapide) | Inférentiel (un modèle évalue) |
|---|---|---|
| **Guide** (*feedforward*) : oriente l'agent avant qu'il agisse | Codemods, scripts de bootstrap | Conventions dans `AGENTS.md`, skills |
| **Senseur** (*feedback*) : vérifie ce qu'il a produit | Tests, linters, tests d'architecture | Revue de code par IA, *LLM as judge* |

Elle distingue aussi le harnais **intégré** à l'outil, conçu par ses auteurs, du harnais **extérieur** que chacun construit autour pour son propre contexte. Ici, OpenHands fournit le premier. La factory est un harnais extérieur porté par la plateforme : c'est l'objet de cet épisode.
{{% /notice %}}

## 🎯 Objectifs de cet article

* 🧩 Comprendre comment **une seule claim Crossplane** produit tout l'environnement d'un run
* 🙌 Découvrir **OpenHands**, et ce que la factory y ajoute
* 🪪 Savoir **qui agit** : une identité par intervenant, et aucun secret dans la sandbox
* 🧱 Voir comment **gVisor** et **Cilium** isolent le run, sur GKE comme sur EKS
* 🤖 Découvrir les **rôles** d'une tâche, et pourquoi on ajoute des relecteurs plutôt que des écrivains

## 🧩 Un run, une claim `AgentRun`

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
  dataClass: public                       # public | internal, sans valeur par défaut
  task:
    text: "Fix the broken link in docs/README.md."
  budget:
    maxTokens: 2000000
  egress:
    profiles: [pypi]                      # registres autorisés en plus de GitHub
```

La claim ne nomme jamais d'image : elle déclare `harness: openhands` (la valeur par défaut), et c'est la composition qui choisit le digest épinglé. De même, la classe de données n'a **pas de valeur par défaut** : classer une donnée est une décision, pas un oubli. Et seul le compte de service de la factory a le droit de créer un `AgentRun`, ce qu'une policy Kyverno garantit.

La [composition](https://github.com/Smana/crossplane-configuration/tree/main/apis/agentrun), écrite en KCL, traduit cette intention en ressources Kubernetes, toutes nommées d'après le run :

{{< img src="agentrun-composition.png" alt="La composition AgentRun : une claim AgentRun est rendue par Crossplane en ServiceAccount sans droits, ConfigMap de tâche, CiliumNetworkPolicy et Sandbox ; le pod gVisor contient le harnais OpenHands, identity-proxy et maku-bridge, et tourne sur GKE Sandbox ou sur un pool Karpenter EKS ; chaque appel sort par l'agent gateway, octo-sts délivre un jeton GitHub limité aux branches agent/**, et maku-bridge relaie les événements vers le broker de rooms" width="1200" >}}
</br>

| Ressource composée | Son rôle |
|---|---|
| `ServiceAccount` | L'identité du run, **sans aucun droit** sur l'API Kubernetes |
| `ConfigMap` | La tâche, les règles que l'agent doit suivre, et ses paramètres |
| `CiliumNetworkPolicy` | Les seules destinations que ce run peut joindre |
| `Sandbox` | Le pod gVisor : le harnais OpenHands, accompagné de deux sidecars. `identity-proxy` porte l'identité du run vers l'extérieur, `maku-bridge` relaie ses événements vers la room |

Le pod lui-même est durci au maximum : utilisateur non root, système de fichiers racine en lecture seule, toutes les *capabilities* retirées, et une annotation qui interdit à Karpenter de le déplacer en cours de route.

Au-delà du YAML économisé, ce modèle apporte quatre propriétés :

* **Des garde-fous dans l'API.** Un run `internal` qui demande un profil d'egress est refusé dès sa création, par le schéma lui-même. Et une claim est **immuable** : seul son budget de tokens peut changer. Pour modifier un run, on en lance un autre.
* **Un état au même endroit.** Le statut de la claim donne la phase du run, les tokens consommés, la PR ouverte et le digest de l'image qui a réellement tourné. La composition est le **seul** écrivain de ce statut : les contrôleurs posent des annotations, qu'elle valide avant de les reporter. Une révocation (budget épuisé, arrêt manuel) y est **définitive** : un run révoqué ne redémarre pas.
* **La même claim sur deux clouds.** Un `AgentRun` produit le même pod gVisor sur GKE et sur EKS ; seuls les nœuds diffèrent.
* **Un nettoyage garanti.** À la fin du run, le pod est supprimé sans jamais être recréé, et l'identité du run disparaît avec lui. Une policy Kyverno supprime chaque nuit les runs terminés depuis plus de 24 heures, au cas où.

Enfin, **Kueue** admet les sandboxes dans une file plafonnée (16 CPU, 64 Gi, 4 pods), sans emprunt possible : une factory qui s'emballe ne peut pas remplir le cluster.

## 🙌 Le harnais : OpenHands

[OpenHands](https://github.com/OpenHands/OpenHands) est une plateforme open source d'agents de développement, disponible en SDK Python, en CLI, en interface web et en offre cloud. Toutes ces déclinaisons reposent sur la même boucle : l'agent observe, agit avec ses outils et vérifie le résultat, jusqu'à ce que la tâche soit terminée. Ses principales fonctionnalités :

* **Des outils de développeur** : un terminal, un éditeur de fichiers et un gestionnaire de tâches, extensibles par des serveurs **MCP**.
* **L'indépendance vis-à-vis du modèle** : il s'adresse à n'importe quel fournisseur via LiteLLM.
* **Un condenseur de contexte** : à intervalles réguliers, la première moitié de l'historique est remplacée par un résumé, ce qui garde les longues sessions dans la fenêtre de contexte.
* **Un serveur d'agent** (*agent-server*) : il exécute les conversations et expose chaque action de l'agent comme un événement, via une API REST et WebSocket.

**Dans la factory**, OpenHands ne tourne qu'à un seul endroit : le conteneur du harnais, dans le pod gVisor de chaque run. Son serveur écoute uniquement en local, sans authentification, et reste donc invisible du réseau du pod. Rien ne s'y connecte depuis l'extérieur ; c'est la sandbox qui ouvre chaque connexion, vers l'agent gateway ou vers la room. Trois fonctionnalités font ce lien :

| OpenHands fournit | Dans la sandbox | Hors de la sandbox |
|---|---|---|
| Les outils MCP | L'agent appelle un outil | Les serveurs MCP tournent derrière l'agent gateway, qui vérifie l'identité du run et filtre les outils selon son rôle |
| LiteLLM | L'agent appelle un modèle, avec une fausse clé | L'agent gateway injecte la vraie clé et n'autorise que les modèles permis pour la classe de données : open-weight pour un dépôt public, Claude pour un travail interne |
| L'API d'événements | `maku-bridge`, sidecar du même pod, lit chaque événement en local | Le bridge relaie ces événements au broker, qui les inscrit dans la room |

L'image du harnais part de l'image officielle d'OpenHands, en retire `sudo` et tous les binaires *setuid*, et n'ajoute que le strict nécessaire :

* un *credential helper* Git qui obtient le jeton GitHub du run, et ne le garde qu'en mémoire ;
* un wrapper `gh` qui ajoute à chaque PR un pied de provenance (room, run, rôle, tâche, modèle) ;
* des hooks Git : l'un ajoute les trailers `Agent-Run` et `Agent-Task`, l'autre refuse toute modification du contrat de tests ;
* la boucle du juge, et les vérifications locales du dépôt, décrites plus bas.

## 🪪 Qui agit ? Une identité par intervenant

Dans une room comme sur GitHub, chaque action est attribuée à l'un de ces trois intervenants. Et une identité n'est jamais **déclarée** par celui qui écrit : elle est déduite du jeton vérifié.

| Intervenant | Identité | Comment elle est établie |
|---|---|---|
| 👤 Un humain | `human:<sub>` | Sa connexion SSO via ZITADEL (Web UI, CLI `maku`). Ses comptes GitHub et Slack sont liés à ce compte |
| 🤖 Un agent | `agent:<runId>` | L'identité Kubernetes de son pod, émise par le cluster pour ce seul run |
| ⚙️ La factory | `system:factory` | L'identité de service du contrôleur Maku |

Deux conséquences concrètes :

* Les dépenses sont **imputées à la personne** qui a lancé la tâche (le champ `principal` de la claim), pas à un compte technique anonyme.
* Ce qu'écrit un agent reste du **contenu non fiable, attribué à cet agent**. Un verdict, un changement de pilote ou une approbation ne peuvent venir que de Maku ou d'un humain.

## 🔑 Aucun secret dans la sandbox

Le harnais ne détient **aucun** des jetons du run : c'est le sidecar `identity-proxy` qui les porte et les attache à chaque appel sortant. Le namespace `agents` ne contient aucun `Secret`, à l'exception d'un certificat d'autorité, ce que Kyverno vérifie.

Les clés des fournisseurs de modèles restent dans OpenBao. C'est l'**agent gateway** ([agentgateway](https://agentgateway.dev/)) qui les injecte au passage, après avoir vérifié l'identité du run. Il compte aussi les tokens par run, par personne et pour toute la flotte : un budget épuisé se traduit par un refus explicite, pas par une facture surprise.

Même principe pour GitHub : le run ne détient **aucun** credential durable. Quand il doit pousser, son identité est échangée via [`octo-sts`](https://github.com/octo-sts/app) contre un jeton GitHub valable **au plus une heure**, pour **un dépôt** et **un rôle**, révoqué à la fin du run. Les règles de cet échange sont versionnées **dans le dépôt cible** : c'est son propriétaire qui décide quels runs peuvent y écrire.

| Rôle | Droits GitHub obtenus |
|---|---|
| Implementer | Écrire du code et ouvrir une PR |
| Tester | Écrire du code (des tests), sans ouvrir de PR |
| Reviewer, triager | Lecture seule |

Côté GitHub, des *rulesets* cantonnent les agents aux branches `agent/**` : pas de push sur `main`, pas de tag. Trois GitHub Apps se partagent le travail, avec des droits distincts : celle des **agents** (pousser sur `agent/**`, ouvrir une PR), celle de la **factory** (commenter et labelliser les issues) et celle du **merger**, la seule qui peut merger, et dont la clé ne quitte jamais le contrôleur de Maku. **Un agent ne peut ni approuver ni merger sa propre PR**, par construction. Les travaux `internal`, eux, n'obtiennent aucun jeton GitHub : ils lisent le dépôt à travers le gateway, et ne peuvent rien pousser.

## 🧱 L'isolation : gVisor

Chaque pod de run tourne sous **gVisor**. gVisor intercale un noyau en espace utilisateur entre le conteneur et le noyau de l'hôte : une tentative d'évasion par appel système doit d'abord le traverser. Je l'ai préféré aux microVMs (Kata, Firecracker) parce qu'il tourne sur des **nœuds standards** des deux clouds, sans virtualisation imbriquée ni instances *bare metal*, et que [`agent-sandbox`](https://agent-sandbox.sigs.k8s.io/), un projet des SIG Kubernetes, le prend en charge directement.

La claim ne nomme qu'une `RuntimeClass`, `gvisor` ; c'est le cluster qui fournit les nœuds :

| | GKE | EKS |
|---|---|---|
| Les nœuds | Un pool **GKE Sandbox**, la fonctionnalité gVisor intégrée de GKE | Un pool **Karpenter** dédié, où gVisor est installé au démarrage du nœud |
| Le type | Instances spot, de 0 à 2 nœuds | Instances spot, plafonnées à 16 CPU, remplacées toutes les 24 heures |

Le surcoût est réel mais discret : des I/O fichiers environ 1,7× plus lentes, presque invisibles puisqu'un agent passe l'essentiel de son temps à attendre le modèle. Le démarrage, lui, se remarque : une à trois minutes et demie quand le pool est vide et qu'il faut un nouveau nœud. Garder un nœud chaud ramène ce délai à une quinzaine de secondes, pour environ 125 $ par mois ; je l'ai désactivé par défaut.

Des policies Kyverno garantissent qu'aucun pod du namespace `agents` ne tourne autrement que sous gVisor, sans jeton d'API Kubernetes, et créé par une `Sandbox`.

## 🌐 L'egress : tout est fermé, sauf ce qui est nommé

Le namespace `agents` refuse tout trafic par défaut, et la `CiliumNetworkPolicy` de chaque run n'ouvre que le strict nécessaire : GitHub, l'agent gateway, le broker de rooms et le collecteur de traces. Les registres de paquets ne s'ajoutent que si la tâche en a besoin, par profils :

| Profil | Ce qu'il ouvre |
|---|---|
| `pypi` | `pypi.org`, `files.pythonhosted.org` |
| `npm` | `registry.npmjs.org` |
| `golang` | `proxy.golang.org`, `sum.golang.org` |
| `crates` | `index.crates.io`, `static.crates.io` |

Un nom non autorisé **ne résout même pas** : le proxy DNS de Cilium écarte la requête, l'agent reçoit `Temporary failure in name resolution`, et aucune connexion n'est jamais tentée. Pour un travail `internal`, c'est plus strict encore : aucun profil, aucun accès direct à GitHub.

{{% notice warning "Pourquoi autant d'insistance sur l'egress ? ☠️" %}}
Parce que c'est le troisième terme de la [*lethal trifecta*](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) de Simon Willison : accès à des données privées + exposition à du contenu non fiable + **capacité à communiquer vers l'extérieur**. Un agent qui lit des issues GitHub et du code tiers a forcément les deux premiers. Anthropic a documenté [un cas réel](https://www.anthropic.com/research/how-we-contain-claude) d'exfiltration **par l'endpoint d'inférence autorisé**, avec une clé fournie par l'attaquant, et conclut : *« Design for containment at the environment layer first, then steer behavior at the model layer. »* Je ne demande pas au modèle de se méfier ; je coupe le canal.
{{% /notice %}}

## 🤖 Les rôles : qui fait quoi dans une tâche

Les premières versions de la factory avaient deux agents : un qui écrit, un qui relit. Sur des issues floues, l'implementer partait dans tous les sens : l'un de nos runs a dépensé tout son budget en 319 appels à `grep` sans rien pousser. La réponse n'a pas été de chercher un meilleur modèle, mais de mieux **cadrer la tâche** : des rôles qui ajoutent du jugement **sans ajouter d'écrivains**. Seuls deux rôles écrivent ; tous les autres lisent et rapportent.

| Rôle | Ce qu'il fait | Comment il tourne | Écrit ? |
|---|---|---|---|
| **Scoper** | Transforme l'issue en tâche bornée, avec des **critères d'acceptation** vérifiables. S'il manque une information, il pose **une** question à l'auteur au lieu de deviner | Un run en lecture seule, avec un budget réduit | Non |
| **Splitter** | Propose de découper une issue trop grosse en parties, avec un ordre de merge. Un mainteneur valide avec `/factory split` | Une proposition du scoper | Non |
| **Tester** | Écrit d'abord des tests qui **échouent** sur le code actuel. Ils deviennent le contrat de l'implementer, qui n'a pas le droit de les modifier | Un run dédié | Les tests |
| **Implementer** | Écrit le changement, lance les vérifications du dépôt **dans sa sandbox** avant de pousser, et ouvre la PR | Un run | Le code |
| **Search** | Répond aux questions de l'implementer sur le dépôt, pour lui éviter de tout relire lui-même | Une conversation secondaire, dans le harnais de l'implementer, avec des outils de lecture uniquement | Non |
| **Juge** | Le changement satisfait-il les critères d'acceptation ? | Un simple appel de modèle, sans outils et avec un contexte neuf | Non |
| **Reviewer** | Relit la PR et rend un verdict. Pour une classe qui peut se merger seule, le gate exige son approbation | Un run en lecture seule | Non |

L'implementer et le reviewer tournent sur presque chaque tâche ; les autres rôles s'activent **par classe de changement**. Sur la plateforme de référence, le scoper et le juge sont actifs sur la documentation et les scripts, la recherche sur la documentation, et le tester reste un interrupteur que j'allumerai classe par classe, le temps de mesurer ce qu'il apporte.

### La boucle du juge

Le juge mérite qu'on s'y arrête, parce qu'il change la façon dont un run se termine. Sans lui, un run s'arrête quand l'agent *pense* avoir fini. Avec lui, l'implementer s'arrête quand un regard neuf le **confirme** :

* le juge lit les critères d'acceptation, le diff et le résultat de la CI ;
* **`met`** : c'est terminé, le run s'arrête ;
* **`not_yet`** : le juge explique ce qui manque, et la même conversation reprend, pour un quart d'heure au plus, deux fois au maximum ;
* **`impossible`** : la tâche vous est escaladée, avec la raison.

Deux règles complètent la boucle : un implementer qui n'a fait **aucun commit** au bout de 80 étapes est arrêté (il tourne en rond), et un run qui épuise ses reprises sans convaincre le juge s'arrête avec la raison `not_met`.

### Ce que chaque rôle a le droit d'appeler

Chaque rôle a aussi un **profil de capacités**, compilé dans les règles de l'agent gateway : la liste des modèles et des outils MCP qu'il peut appeler, selon la **classe de données** de la tâche. Un appel hors profil reçoit un refus.

| | Travail `public` | Travail `internal` |
|---|---|---|
| **Modèles** | Open-weight : GLM (Z.ai) par défaut, Kimi et Mistral en option | Claude uniquement |
| **Outils MCP** | La documentation, et les outils de la room | En plus, la lecture du cluster (Flux, VictoriaMetrics) ; les secrets restent refusés |

{{% notice info "Pourquoi pas plus d'agents ? 🤔" %}}
Les recherches de 2026 vont toutes dans le même sens : un **scoper** améliore nettement les résultats sur les issues mal spécifiées ([Ask or Assume](https://arxiv.org/abs/2603.26233) : 69,4 % contre 61,2 %), un **reviewer à contexte neuf** rattrape ce que l'auteur ne voit plus ([Cognition](https://cognition.com/blog/multi-agents-working)), et un **tester test-first** cadre l'implémentation ([TDFlow](https://arxiv.org/abs/2510.23761)). À l'inverse, les essaims d'agents qui écrivent en parallèle coûtent 7 à 15 fois plus de tokens et dégradent le travail séquentiel. D'où la règle : on ajoute des relecteurs, pas des écrivains.
{{% /notice %}}

## 💭 Dernières remarques

* **Le modèle est la partie bon marché.** Le remplacer tient dans un alias du gateway. Faire qu'un run agisse sous une identité courte, sans secret durable ni sortie réseau implicite, et laisse une trace consultable : c'est ça qui a structuré la plateforme, et c'est transposable à n'importe quel harnais.

* **Chaque compromis est écrit.** Sous gVisor, un pod ne voit pas le renouvellement de ses jetons ; leur durée de vie est donc alignée sur la durée maximale du run, et une copie volée resterait valable jusque-là. Le profil seccomp par défaut n'est pas appliqué non plus : c'est le noyau de gVisor qui tient la frontière. Ces choix sont documentés dans des ADR, pas cachés.

* **Ce qui n'est pas encore là.** Le jeton GitHub est lié au dépôt et au rôle, pas au run : n'importe quel implementer pourrait pousser sur la branche d'un autre. Le gate le rattrape par l'identité de chaque push, mais ce n'est pas une isolation. Et le reviewer tourne aujourd'hui sur le même modèle que l'implementer : les modèles par rôle sont prévus, pas encore utilisés.

Dans le prochain épisode, on passe de l'agent à la décision : qu'est-ce qui doit être vrai avant qu'une PR d'agent soit mergée ? 🚦

## 🔖 Références

* [La composition `AgentRun`](https://github.com/Smana/crossplane-configuration/tree/main/apis/agentrun) · [les rôles des agents](https://airef.ogenki.io/docs/how-it-works/roles/)
* [OpenHands](https://github.com/OpenHands/OpenHands) · [OpenAI : Harness engineering](https://openai.com/index/harness-engineering/) · [Birgitta Böckeler : Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)
* [Kubernetes agent-sandbox](https://agent-sandbox.sigs.k8s.io/) · [gVisor](https://gvisor.dev/docs/) · [Kueue](https://kueue.sigs.k8s.io/)
* [agentgateway](https://agentgateway.dev/) · [octo-sts](https://github.com/octo-sts/app) · [Cilium : politiques DNS et FQDN](https://docs.cilium.io/en/stable/security/policy/language/#dns-based)
* [Simon Willison : The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) · [Anthropic : How we contain Claude](https://www.anthropic.com/research/how-we-contain-claude)
