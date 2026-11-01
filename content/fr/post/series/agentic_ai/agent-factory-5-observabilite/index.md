+++
author = "Smaine Kahlouch"
title = "Agent Factory : observer et maîtriser une flotte d'agents"
date = "2026-12-02"
summary = "Une flotte d'agents dépense en continu. Comment savoir ce qu'elle fait sans exposer ce qu'elle lit, combien elle coûte, et comment l'arrêter ? Traces sans contenu, métriques par run et par personne, budgets à plusieurs étages, freins d'urgence, alertes, et les chiffres de sessions réelles."
featured = false
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "observability", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agent Factory 🏭" %}}
1. [Vue d'ensemble : agents et humains dans la même équipe](/fr/post/series/agentic_ai/agent-factory-1-vue-ensemble/)
2. [Maku, le chef d'orchestre](/fr/post/series/agentic_ai/agent-factory-2-maku/)
3. [Le harnais : un environnement cloisonné pour chaque agent](/fr/post/series/agentic_ai/agent-factory-3-harnais/)
4. [Le merge gate : une autonomie qui se mérite](/fr/post/series/agentic_ai/agent-factory-4-merge-gate/)
5. **Observer et maîtriser une flotte d'agents** (vous êtes ici)
6. [D'agent à agent : RunLore confie ses correctifs à la factory](/fr/post/series/agentic_ai/agent-factory-6-runlore/)
{{% /notice %}}

Un service classique consomme des ressources à peu près prévisibles. Une flotte d'agents, elle, **dépense à chaque étape** : chaque tour de boucle est un appel de modèle facturé, et un agent qui tourne en rond peut consommer en une heure ce qu'une tâche normale coûte en une semaine. Sans mesure, on le découvre sur la facture.

Trois questions guident cet épisode : **que fait la flotte** en ce moment, **combien coûte-t-elle**, et **comment l'arrêter** quand elle s'emballe. Avec une contrainte qui complique tout : les agents lisent des issues, du code et parfois des données internes, et rien de tout cela ne doit se retrouver dans un outil d'observabilité.

## 🎯 Objectifs de cet article

* 🔭 Voir ce que la plateforme **mesure**, et ce qu'elle refuse volontairement de mesurer
* 📊 Découvrir les **dashboards** et les questions auxquelles ils répondent
* 💰 Comprendre comment le **coût** d'un run est calculé et plafonné, à plusieurs étages
* 🧯 Connaître les **freins d'urgence**, du run à la flotte entière
* 🚨 Passer en revue les **alertes** qui comptent
* 📈 Lire les **chiffres** de sessions réelles, et ce qu'ils ont changé

## 🔭 Ce qu'on mesure, et ce qu'on ne mesure pas

La règle de départ est simple : **la télémétrie contient des identifiants, jamais du contenu**. Le texte d'une issue, un prompt ou une réponse de modèle ne quittent jamais la room, qui est sous contrôle d'accès. Tout le reste s'organise autour de cette règle.

{{< img src="agent-telemetry.png" alt="La télémétrie de la flotte : le run envoie ses spans à un collecteur qui tamponne l'identifiant du run et ne garde que les métadonnées, puis VictoriaTraces ; l'agent gateway envoie ses spans, ses logs d'accès vers VictoriaLogs, et les tokens et coûts par run vers VictoriaMetrics ; Maku y publie phases, dépenses et décisions ; Grafana lit les trois, et les règles d'alerte partent vers Slack. Le contenu, lui, reste dans la room" width="1200" >}}
</br>

| Signal | D'où il vient | Ce qu'il contient |
|---|---|---|
| **Traces** | Le harnais (une trace par run : la conversation, puis un span par appel de modèle) et l'agent gateway | Le modèle appelé, les tokens consommés (dont ceux lus en cache), le coût, la durée. Aucun texte |
| **Métriques** | L'agent gateway, Maku, et l'état des `AgentRun` exporté par kube-state-metrics | Tokens et coûts par run, par rôle, par modèle et par personne ; phases des tâches ; décisions du merge gate |
| **Logs** | Le journal d'étapes du harnais et les logs d'accès du gateway | Les codes de retour et les refus, pas les échanges |

Les traces passent par un collecteur OpenTelemetry dédié, qui fait trois choses avant de les envoyer à VictoriaTraces :

* il **tamponne l'identifiant du run** à partir du pod qui envoie, et non de ce que le run déclare : un run ne peut pas se faire passer pour un autre ;
* il **écarte** tout span sans identifiant de run ;
* il **ne garde que les attributs autorisés**, essentiellement les conventions `gen_ai.*` d'OpenTelemetry (modèle, tokens d'entrée et de sortie, tokens lus en cache, coût).

Côté réseau, un run ne peut joindre le collecteur que pour **envoyer des traces** : pas de logs, pas de lecture, et aucun accès direct à la base de traces.

Pour relier une dépense à une personne, il suffit d'une jointure : le gateway étiquette chaque métrique avec l'identité **vérifiée** du run, et Maku publie la correspondance entre chaque run, sa tâche, sa room et la personne qui l'a lancée.

## 📊 Les dashboards

Tout est rangé dans un dossier Grafana `agents`. Chaque dashboard répond à une question précise :

| Dashboard | La question |
|---|---|
| **Agents overview** | La factory est-elle saine, combien a-t-elle dépensé aujourd'hui par rapport à ses plafonds, qu'est-ce qui attend un humain, qu'est-ce qui a été livré ? |
| **Agent factory** | Combien de temps entre le label et la PR ? Combien de merges automatiques revertés ? Que deviennent les PR, classe par classe ? Combien d'interventions humaines (« à quel point est-ce *dark* ») ? |
| **Agent fleet** | Quels runs tournent, combien de tokens par run, le pool gVisor est-il proche de sa limite ? Un tier qui coûte autant que le tier inférieur justifie-t-il son prix ? |
| **Agent run** | Pour un run donné : tokens consommés face au budget, coût par modèle, étapes, appels d'outils, erreurs, latence, traces |
| **Agent gateway** | Requêtes par statut, refus par raison, taux d'erreur des modèles, part du cache |
| **Spend per identity** | Les dépenses d'une personne face à son plafond et à celui de la flotte, le coût par PR acceptée |

Le dernier mérite une précision, écrite dans sa description : il **ne note pas la personne**. Il sert à comprendre où part l'argent, pas à classer les gens.

Le lien vers le dashboard d'un run est affiché au démarrage du run : quand une room vous intrigue, la vue technique est à un clic.

## 💰 Budgets et coûts

### Comment un run est facturé

Chaque modèle a un prix par million de tokens, déclaré dans la configuration de Maku. Le calcul distingue les tokens d'entrée normaux, ceux **lus en cache** (beaucoup moins chers) et ceux de sortie :

| Modèle | Entrée | Entrée lue en cache | Sortie |
|---|---|---|---|
| GLM-5.3 (travail public) | 1,40 $ | 0,26 $ | 4,40 $ |
| Claude Opus (travail interne) | 4,00 $ | 0,20 $ | 20,00 $ |

*Prix par million de tokens, relevés à la mi-octobre.*

Un modèle sans prix déclaré n'est **jamais** compté à zéro : ses runs retombent sur des plafonds en tokens, et une alerte se déclenche. Et deux comptabilités tournent en parallèle, celle de Maku et celle du gateway : si elles divergent de plus d'un centime (ou de 1 %), une alerte le signale.

### Des plafonds à plusieurs étages

Un seul plafond ne suffit pas : un run peut s'emballer, une personne peut lancer trop de tâches, la flotte entière peut dériver. Les budgets s'empilent donc :

| Étage | Plafond | Qui le fait respecter |
|---|---|---|
| **Le run** | Le budget de son tier (2,55 $ en `standard`) et sa durée maximale | Maku arrête le run ; le pod a une échéance fixe |
| **La tâche** | Deux fois le budget d'un run (5,10 $ en `standard`) | Maku |
| **La personne** | 8,50 $ par jour | Maku, en mode observation : compté, pas encore refusé |
| **La flotte** | 20 tâches par jour et 4 runs en parallèle (refusés au-delà) ; 42,50 $ par jour (compté, pas encore refusé) | Maku et Kueue |
| **Le filet de sécurité** | Des seaux de tokens par run, pour la flotte et par fournisseur | L'agent gateway |

Le dernier étage est le plus brutal. Quand un seau est vide, l'appel en cours se termine, mais le suivant reçoit un refus explicite, non réessayable :

```json
{"error":{"type":"rate_limit_error","code":"agent_budget_exhausted",
 "message":"agent-router refused this call: a gateway token budget is spent (limit 5000000, resets in 12877 s). Not retryable."}}
```

## 🧯 Les freins d'urgence

| Frein | Portée | Comment |
|---|---|---|
| `factory/stop` sur une issue | Une tâche | Le label arrête la tâche au prochain passage de Maku |
| Révocation d'un run | Un run | Une annotation sur l'`AgentRun` : le run passe en `Revoked` et son pod disparaît en quelques secondes |
| **Le kill switch** | Toute la factory | Le label `factory/stop` sur une issue de contrôle, ou directement une `ConfigMap` `agent-factory-stop` |
| Kueue | Le cluster | Une file plafonnée à 16 CPU, 64 Gi et 4 pods, sans emprunt possible |

Le kill switch mérite un détail : la `ConfigMap` qui l'active est **volontairement absente de Git**, pour que Flux ne la supprime jamais en voulant « réconcilier ». Tant qu'elle existe, toutes les tâches s'arrêtent, chaque run vivant est révoqué toutes les quinze secondes, et toute nouvelle demande reçoit un refus.

Reste un cas qui n'est pas une panne, mais qui y ressemble : les runs tournent sur des **nœuds spot**, que le cloud peut reprendre à tout moment. Le harnais a alors quinze secondes pour mettre le travail en pause, pousser un commit de sauvegarde, laisser la room faire sa dernière lecture, puis révoquer son jeton. Mesurée sur EKS, la séquence complète a pris moins de cinq secondes. Le run reprend ensuite de lui-même, deux fois au plus.

## 🚨 Les alertes

Les règles sont des `VMRule` envoyées dans un canal Slack via Alertmanager. Les plus utiles, regroupées par thème :

| Thème | Exemples d'alertes | Ce qu'elles signalent |
|---|---|---|
| **Dépenses** | `AgentFleetTokenSpendHigh`, `AgentFactoryDailyBudgetHigh`, `AgentBudgetRefused` | Plus de 10 M de tokens en une heure pour la flotte, 80 % du budget du jour atteint, un refus du gateway |
| **Cache** | `AgentCacheReadZero`, `AgentCacheReadShareLow` | Le cache ne fonctionne plus, ou moins de 60 % de l'entrée lue en cache : la facture va exploser |
| **Comptabilité** | `AgentModelUnpriced`, `AgentCostCrossCheckMismatch` | Un modèle sans prix, ou deux comptabilités qui divergent |
| **Santé** | `AgentFactoryMeterBlind`, `AgentSandboxPodPending`, `AgentFactoryTaskStuck`, `PolicyBotUnavailable` | Maku ne voit plus les dépenses (et refuse alors de nouveaux runs), une sandbox ne démarre pas, une tâche est bloquée depuis deux heures |
| **Merge** | `AgentFactoryAutoMergeReverted`, `AgentFactoryMergeRegression`, `AgentFactoryClassPausedLong` | Un merge automatique reverté, une régression, une classe en pause depuis une semaine |
| **Rooms** | `RoomStalled`, `RoomApprovalPendingTooLong` | Une room inactive depuis 30 minutes, une approbation qui attend depuis un quart d'heure |

Deux d'entre elles sont critiques : une consommation anormale de la flotte, et un cache qui ne lit plus rien. La seconde peut surprendre, mais elle se comprend avec les chiffres qui suivent.

## 📈 Ce que ça coûte, ce que ça donne

Chaque chiffre porte sa date : dans un mois, les modèles, les prix et les tiers auront probablement changé.

| Mesure | Valeur | Contexte |
|---|---|---|
| Part de l'entrée lue en cache | **95,1 %** | 24 heures de travail public, jusqu'au 18 octobre |
| Coût sur cette fenêtre | **10,40 $**, contre 41,64 $ sans cache | Recalcul au tarif sans cache, pas un second run |
| Un run implementer complet | **1,65 $** pour 225 étapes | Le coût calculé par Maku correspondait exactement à celui du gateway |
| Démarrage d'une sandbox, pool vide | **une à trois minutes et demie** | Il faut un nouveau nœud gVisor |
| Démarrage avec un nœud gardé chaud | **16 secondes** (p50 et p90) | Le nœud coûte environ 125 $ par mois : désactivé par défaut |

Et une expérience plus inconfortable, menée les 14 et 15 octobre sur quinze issues volontairement difficiles, un run par issue :

| Configuration | Issues livrées | Coût |
|---|---|---|
| La factory, avec GLM-5.3 | **0 sur 15** | 9,00 $ |
| Claude Code Action, avec Claude | **13 sur 15** | 15,80 $ |
| Claude Code Action, avec GLM-5.3 | 0 sur 15 | 25,65 $ |

Deux lectures. D'abord, **le cache est le premier levier de coût** : la boucle d'un agent relit sans cesse le même contexte, et 95 % de lectures en cache divisent la facture par quatre. C'est pour ça qu'un cache qui cesse de fonctionner déclenche une alerte critique. Ensuite, **le modèle compte énormément sur les tâches difficiles** : l'écart suivait le modèle, pas l'outil. GLM fait très bien les corrections de documentation et de scripts qui constituent l'essentiel du flux, mais pas le reste. Le travail public ne pouvant pas passer sur Claude, la réponse a été d'**adapter la tâche au modèle** (un scoper qui borne, un découpage des issues trop grosses, un juge qui renvoie au travail) plutôt que de chercher le meilleur modèle unique. C'est aussi ce qui a poussé à exprimer les plafonds en dollars plutôt qu'en tokens.

## 💭 Dernières remarques

* **Observer sans lire.** Garder le contenu hors de la télémétrie a un coût : on ne peut pas chercher un prompt dans Grafana. Mais c'est ce qui permet de donner accès aux dashboards largement, sans craindre d'y exposer une donnée interne. Le transcript complet reste dans la room, pour ceux qui ont le droit de le lire.

* **Plusieurs plafonds valent mieux qu'un bon.** Le run, la tâche, la personne, la flotte, le fournisseur : chaque étage rattrape ce que le précédent a laissé passer. Les plafonds quotidiens en dollars, par personne et pour la flotte, ne font encore que compter ; ils refuseront quand j'aurai assez de recul pour les régler sans gêner personne. En attendant, ce sont les plafonds par run et par tâche, et les seaux du gateway, qui tiennent.

* **Les freins doivent se tester.** Un frein qu'on n'a jamais tiré n'est qu'une hypothèse. La révocation manuelle d'un run et la reprise après la perte d'un nœud spot ont été éprouvées en conditions réelles ; les autres le seront à leur tour, dans les runbooks.

Dans le dernier épisode, on change de point de vue : ce n'est plus un humain qui confie une tâche à la factory, mais un autre agent. 🤝

## 🔖 Références

* [L'observabilité de l'Agent Factory](https://airef.ogenki.io/docs/operate/observability/)
* [OpenTelemetry : conventions sémantiques GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
* [agentgateway](https://agentgateway.dev/) · [Kueue](https://kueue.sigs.k8s.io/)
* [VictoriaMetrics](https://victoriametrics.com/) · [VictoriaTraces](https://docs.victoriametrics.com/victoriatraces/) · [Grafana](https://grafana.com/)
