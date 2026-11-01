+++
author = "Smaine Kahlouch"
title = "Agent Factory : d'agent à agent, RunLore confie ses correctifs à la factory"
date = "2026-12-09"
summary = "Que se passe-t-il quand ce n'est plus un humain mais un autre agent qui confie une tâche à la factory ? RunLore enquête sur une alerte, propose un correctif, la factory l'implémente, et RunLore vérifie après le merge que l'alerte a bien disparu. Avec un humain aux trois points qui comptent."
featured = false
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "observability", "security", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agent Factory 🏭" %}}
1. [Vue d'ensemble : agents et humains dans la même équipe](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)
2. [Maku, le chef d'orchestre](/fr/post/series/agentic_ai/agent-factory-2-maku/)
3. [Le harnais : un environnement cloisonné pour chaque agent](/fr/post/series/agentic_ai/agent-factory-3-harnais/)
4. [Le merge gate : une autonomie qui se mérite](/fr/post/series/agentic_ai/agent-factory-4-merge-gate/)
5. [Observer et maîtriser une flotte d'agents](/fr/post/series/agentic_ai/agent-factory-5-observabilite/)
6. **D'agent à agent : RunLore confie ses correctifs à la factory** (vous êtes ici)
{{% /notice %}}

Jusqu'ici, chaque tâche partait d'un humain : un label, un message Slack, une demande à son coding agent. Mais rien n'oblige le demandeur à être une personne. Un agent qui détecte un problème peut très bien confier le correctif à d'autres agents.

C'est exactement ce que fait **RunLore**. Dans [un article de la série observabilité](/fr/post/series/observability/runlore/), il enquêtait sur les alertes et apprenait de chaque résolution, mais s'arrêtait à la proposition : la correction restait à faire à la main. Connecté à la factory, il va jusqu'au bout. Il enquête, confie le correctif, puis vérifie après le merge que l'alerte a bien disparu.

Ce dernier épisode décrit cette boucle, et surtout ce qui la rend sûre : un agent qui délègue du travail à d'autres agents, c'est une nouvelle frontière de confiance, et elle mérite autant de soin que les précédentes.

## 🎯 Objectifs de cet article

* 🩺 Comprendre ce que fait **RunLore**, et ce qui change quand il est connecté à la factory
* 🔄 Suivre la **boucle complète** sur un cas réel, de l'alerte au verdict après le merge
* 🛡️ Voir les **frontières de confiance** entre deux agents, et où l'humain reste indispensable
* 🧩 Découvrir ce qui rend ce **relais d'agent à agent** possible pour d'autres agents que RunLore

## 🩺 RunLore en deux mots

RunLore est un agent SRE. Quand une alerte se déclenche, il enquête : il lit les métriques, les logs et l'état du cluster, puis publie dans Slack une cause probable avec un **niveau de confiance**. Par défaut, il ne fait que lire. Sa seule écriture est une PR vers sa propre base de connaissances, qu'un humain relit, et qui lui permet de reconnaître immédiatement un incident déjà vu.

Connecté à la factory, RunLore devient un **client ordinaire de l'API de Maku**, ni plus ni moins. Sa documentation le résume ainsi : *« The factory decides every gate… RunLore only files, listens and reports. »* Il dépose une tâche, suit son avancement, et rend un verdict à la fin. Toutes les décisions restent du côté de la factory. Et la connexion est explicite : tant qu'aucune adresse de factory n'est configurée, RunLore reste un agent purement consultatif.

## 🔄 La boucle complète

Voici la boucle, telle qu'elle a tourné le **29 octobre** sur une vraie alerte : `AppWizardUnavailable`, qui se déclenchait parce qu'une règle d'alerte ciblait une application au nom mal orthographié, `app-wizzard`.

{{< img src="runlore-loop.png" alt="La boucle RunLore : une alerte déclenche une investigation RunLore, qui crée une tâche investigate ; un run triager en lecture seule propose un correctif dans sa room ; un mainteneur publie la proposition avec factory/publish, la relit, puis la démarre avec factory/ready ; la PR est relue par un agent, mergée par un humain ; après un soak de 30 minutes RunLore interroge Alertmanager et rend son verdict" width="1200" >}}
</br>

| Heure (UTC) | Quoi |
|---|---|
| 06:28 | RunLore conclut `action_suggested`, à 0,95 de confiance, et dépose une tâche. Maku ouvre l'issue #2657, qui ne contient qu'un résumé |
| 06:30 | Un agent **triager**, en lecture seule, propose le correctif dans sa room (217 k tokens) |
| 06:30 | Je pose `factory/publish` : Maku publie la proposition comme issue #2658 |
| 06:33 | Après mon `factory/ready`, l'implementer ouvre la PR #2659 |
| 06:40 | Le reviewer rend son verdict : **approve** |
| 06:47 | Je merge : les règles d'alerte exigent une revue humaine |
| 08:59 | Le moniteur post-merge conclut : *« healthy. main CI: healthy · follow_ups=0 · RunLore: resolved, alert AppWizardUnavailable »* |

Une vingtaine de minutes entre l'alerte et le merge, dont l'essentiel à attendre mes décisions. Voyons chaque étape.

**1. RunLore dépose une tâche, mais pas n'importe laquelle.** Il ne s'adresse à la factory que s'il recommande une action **et** que sa confiance dépasse un seuil (0,75). Sinon, la factory répond poliment « en dessous du seuil » et ne crée rien. La même alerte ne crée jamais deux tâches : l'identifiant de l'alerte sert de clé d'idempotence.

**2. Maku ouvre une tâche d'enquête.** Elle utilise le modèle `investigate` : un seul agent, en lecture seule. L'issue publique créée à cette occasion ne contient que le nom de l'alerte, la ressource concernée et le lien vers la room. Le diagnostic de RunLore, lui, reste dans la room, dont l'accès est restreint.

**3. Un triager propose le correctif.** Ce run ne peut que lire : son jeton GitHub est en lecture seule, et ses outils se limitent à consulter Flux, VictoriaMetrics et VictoriaLogs (les secrets restent refusés). Comme il manipule des données internes, il tourne sur Claude. Sa proposition prend la forme d'un passage de relais, écrit dans la room.

**4. `factory/publish` : ce qui devient public.** Je lis la proposition dans la room. Si elle me convient, je pose le label. La factory vérifie alors que le texte respecte le modèle d'issue, le **neutralise** (les liens, les mentions et les références d'issues deviennent du code, sans HTML ni caractère caché), le passe au détecteur de secrets, puis publie une nouvelle issue.

**5. `factory/ready` : ce qui coûte.** Sur l'issue publiée, un second label autorise la dépense d'un run d'implémentation. Le lien entre ce correctif et l'enquête d'origine est enregistré par la factory elle-même ; il n'est jamais lu dans le texte de l'issue, si bien qu'aucun agent ne peut le falsifier.

**6. Implémentation, revue, merge.** Le circuit est celui de n'importe quelle tâche. La classe `alert-rules` n'a pas le droit de se merger seule : c'est moi qui merge.

**7. RunLore vérifie.** Maku prévient RunLore par des webhooks signés. Trente minutes après le merge, RunLore interroge Alertmanager : l'alerte est-elle toujours active ? **Le verdict est celui d'Alertmanager**, pas l'avis de RunLore ; une courte ré-investigation ne sert qu'à joindre des éléments de preuve. RunLore transmet ensuite ce verdict à Maku, et l'annonce dans Slack : *« The alert is gone 30m after the fix: resolved. »*

**8. Le moniteur tranche.** Un verdict `resolved` compte comme sain. Un verdict `persisting` compte comme une régression : la PR est étiquetée, le disjoncteur de la classe en tient compte, et une issue de suivi est ouverte (*« RunLore: the fix in #2659 did not hold »*), sans aucun détail interne. Mais le moniteur ne revert jamais rien, et ne relance jamais de tâche de lui-même.

## 🛡️ Les frontières de confiance entre deux agents

Laisser un agent confier du travail à d'autres agents ouvre une question délicate : **que peut faire un agent compromis, ou simplement trompé, avec ce pouvoir ?** La réponse tient en quelques règles.

### RunLore est un client, avec des plafonds côté serveur

RunLore s'authentifie comme un utilisateur machine de ZITADEL. La factory le connaît comme un **principal interne**, parrainé par un mainteneur, avec des limites qu'elle applique elle-même :

| Limite | Valeur |
|---|---|
| Dépôt autorisé | Un seul, `cloud-native-ref` |
| Coût maximal par tâche | 2 $ |
| Budget quotidien | 10 $ |
| Tâches par jour | 5 |
| Tâches ouvertes en même temps | 3 |
| Confiance minimale | 0,75 |

Ces plafonds vivent dans la factory, pas dans RunLore. Le document de conception le dit en une phrase : *« a client-side gate is a promise »*. Une limite appliquée par le client n'est qu'une promesse ; seule celle du serveur est une garantie.

### Ce que RunLore ne peut pas faire

Sa requête refuse tout ce qui pourrait servir à détourner la factory : RunLore ne choisit ni le titre de l'issue, ni la branche, ni le modèle de tâche (c'est toujours une enquête), ne peut pas agir au nom de quelqu'un d'autre, et ne peut pas lancer un run directement. Il ne peut rendre un verdict que sur un correctif issu de **sa propre** enquête, et seulement dans la fenêtre prévue.

### Le texte d'un agent reste une donnée

Le diagnostic de RunLore est caviardé une première fois par RunLore, puis une seconde fois par la factory, qui retire aussi les caractères cachés. Ensuite, ce texte n'est jamais présenté à un agent comme une instruction : il est encadré et marqué comme **donnée non fiable**. Dans l'autre sens, les webhooks envoyés à RunLore sont signés et ne contiennent que des identifiants et des codes, jamais le contenu d'une room.

### Un humain aux trois points qui comptent

| Décision | Le geste | Pourquoi un humain |
|---|---|---|
| **Ce qui devient public** | `factory/publish` | Un diagnostic interne ne doit atteindre une issue publique, et donc un agent travaillant sur des données publiques, qu'après une lecture humaine |
| **Ce qui coûte** | `factory/ready` | Lancer un run d'implémentation dépense de l'argent |
| **Ce qui est mergé** | L'approbation | La classe `alert-rules` exige une revue humaine |

Deux agents, deux dépôts, et un seul humain, présent exactement là où sa décision a un sens.

Côté RunLore, la configuration tient en quelques lignes :

```yaml
# Configuration de RunLore : la factory comme délégué (extrait)
factory:
  url: http://agent-factory.agent-system.svc.cluster.local:8082
  token_url: https://auth.example.com/oauth/v2/token      # ZITADEL, client credentials
  client_id_env: FACTORY_CLIENT_ID
  client_secret_env: FACTORY_CLIENT_SECRET
  repository: Smana/cloud-native-ref
  max_cost_usd: 2
  webhook: {secret_env: FACTORY_WEBHOOK_KEY}               # webhooks signés, HMAC-SHA256
  soak: 30m
  verify:
    alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
```

## 🧩 Au-delà de RunLore

Rien dans ce mécanisme n'est propre à RunLore. Maku expose plusieurs portes d'entrée, chacune avec ses règles :

| Porte d'entrée | Pour qui | Ce qu'elle permet |
|---|---|---|
| `POST /v1/tasks`, principal **parrainé** | N'importe quel système dont un mainteneur se porte garant | Déposer des tâches sur des données publiques, dans les limites fixées par la factory |
| `POST /v1/tasks`, principal **interne** | Aujourd'hui, RunLore seulement | Déposer des enquêtes sur des données internes, avec publication contrôlée |
| Les **webhooks signés** | Tout abonné déclaré | Suivre l'avancement d'une tâche, sans contenu |
| Le serveur **MCP** et le skill `factory-handoff` | Votre propre coding agent | Rédiger, déposer et suivre des tâches depuis une session de travail |
| Les outils `room_handoff` et `room_verdict` | Les agents d'une même tâche | Se passer le relais et rendre un verdict, sans jamais se donner d'ordres |

C'est le point commun de toutes ces portes : un agent peut **proposer**, **déposer** et **rapporter** ; il ne peut ni décider à la place d'un humain, ni contourner une limite fixée par la factory.

## 💭 Dernières remarques

* **La boucle n'est pas encore éprouvée dans tous ses chemins.** Le cas `resolved` a tourné en conditions réelles ; le cas `persisting`, celui où le correctif ne tient pas, ne l'a pas encore été. Le premier essai, la veille, avait aussi révélé un trou : RunLore n'avait pas vu un merge fait par un humain, et le moniteur avait conclu `inconclusive`. C'est corrigé, mais c'est un rappel utile : une boucle se prouve sur tous ses chemins, pas seulement sur le plus heureux.

* **La preuve de RunLore reste modeste.** Lors de la vérification, sa ré-investigation a atteint sa limite d'étapes sans conclure. Ce n'est pas grave, puisque le verdict vient d'Alertmanager, mais c'est la bonne raison de ne jamais confier la décision au jugement d'un agent quand un système peut la mesurer.

Et cette série s'arrête ici. Si je ne devais retenir qu'une idée, ce serait celle-ci : **la confiance dans un agent ne se décrète pas, elle se construit dans l'architecture**. Une identité courte par run, aucun secret dans la sandbox, un réseau fermé sauf ce qui est nommé, un journal inaltérable, une autonomie gagnée classe par classe, et un humain là où sa décision a un sens. Le modèle, lui, n'est qu'une ligne de configuration.

Tout est ouvert ([`maku`](https://github.com/Smana/maku), [`ai-native-ref`](https://github.com/Smana/ai-native-ref), [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)), et la prochaine étape, **« anyone can run it »**, vise à faire tourner toute la boucle chez vous. D'ici là, les [issues](https://github.com/Smana/ai-native-ref/issues) sont le meilleur endroit pour me dire où ça casse. 🏭

## 🔖 Références

* [RunLore](https://runlore.io) · [l'article de la série observabilité](/fr/post/series/observability/runlore/)
* [RunLore et la factory dans la documentation d'`ai-native-ref`](https://airef.ogenki.io/docs/how-it-works/runlore/)
* [`maku`](https://github.com/Smana/maku) : l'API, les principaux et les webhooks
* [Standard Webhooks](https://www.standardwebhooks.com/) · [ZITADEL](https://zitadel.com/) · [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
