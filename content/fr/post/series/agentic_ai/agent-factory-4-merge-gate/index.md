+++
author = "Smaine Kahlouch"
title = "Agent Factory : le merge gate, une autonomie qui se mérite"
date = "2026-11-25"
summary = "Quand une machine a-t-elle le droit de merger ? Classes de changements, contrôles ordonnés, policy-bot, approbation qui vaut merge, mode shadow, seuil statistique et disjoncteur : comment l'Agent Factory transforme une impression (« ça marche bien ») en décision vérifiable, et révocable."
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
3. [Le harnais : un environnement cloisonné pour chaque agent](/fr/post/series/agentic_ai/agent-factory-3-harnais/)
4. **Le merge gate : une autonomie qui se mérite** (vous êtes ici)
5. [Observer et maîtriser une flotte d'agents](/fr/post/series/agentic_ai/agent-factory-5-observabilite/)
6. [D'agent à agent : RunLore confie ses correctifs à la factory](/fr/post/series/agentic_ai/agent-factory-6-runlore/)
{{% /notice %}}

Addy Osmani le résume en une phrase : *« Verification, not generation, is the real constraint on a factory. »* Produire une PR n'est plus le problème : les agents en ouvrent autant qu'on leur en demande. La vraie question est ailleurs : **quand une machine a-t-elle le droit de merger ?**

Ma réponse tient en une idée : l'autonomie n'est pas une fonctionnalité globale qu'on active, c'est un **privilège par classe de changement**, gagné statistiquement et **révocable**. Une correction de lien dans la documentation et une mise à jour de dépendance ne présentent ni le même risque ni le même type de preuve ; elles ne méritent pas le même droit.

Cet épisode décrit le **merge gate** de l'Agent Factory : ce qui doit être vrai avant qu'une PR d'agent soit mergée, les deux façons dont elle peut l'être, et la mécanique qui permet à une classe de changements de gagner, puis de perdre, le droit de se merger seule.

## 🎯 Objectifs de cet article

* 🗂️ Comprendre ce qu'est une **classe de changement**, et lesquelles existent aujourd'hui
* 🚦 Distinguer les **deux façons de merger** : sans humain, ou sur une seule approbation
* 🔍 Suivre les **contrôles du gate**, dans l'ordre, et ce qui se passe à chaque « non »
* 📜 Voir comment **policy-bot** garde la politique de merge hors de portée des agents
* 🌗 Comprendre le **mode shadow**, le seuil statistique et le **disjoncteur**
* ↩️ Savoir exactement **ce qui déclenche un revert** après un merge

## 🗂️ Les notions de base

Quatre mots reviennent dans tout l'article :

| Notion | En une phrase |
|---|---|
| **Classe** | Une famille de PR, définie par ce qu'elle a le droit de toucher et par la forme attendue de son diff. Elle est portée par un label de l'issue, `class:<nom>` |
| **Live** | La classe a gagné le droit de se merger **sans humain** |
| **Shadow** (« ombre ») | La classe est en répétition : le gate décide comme si elle était live, mais ne merge rien |
| **App merger** | La seule identité qui merge. Ni les agents ni Maku ne détiennent sa clé |

Une classe peut aussi être simplement **surveillée** : un humain merge, mais la factory observe ce que devient le changement après le merge. Et une PR sans classe est une PR ordinaire, relue comme les autres.

Voici les classes configurées au moment où j'écris ces lignes :

| Classe | Ce qu'elle couvre | `cloud-native-ref` | `ai-native-ref` |
|---|---|---|---|
| `docs-links` | Corriger un lien dans la documentation : fichiers `.md` uniquement, moins de 21 lignes, aucun fichier ajouté ni supprimé | **live** | shadow |
| `revert` | Annuler un merge de la factory | **live** | shadow |
| `dead-links` | Remplacer un lien mort, confirmé ensuite par un scan de liens (l'*oracle*) | shadow | shadow |
| `deps-patch` | Les mises à jour de dépendances en version *patch* proposées par Renovate | shadow | — |
| `deps-minor` | Les mêmes, en version *minor* | shadow, **pour toujours** | — |
| `alert-rules` | Les règles d'alerte | surveillée, revue humaine | — |

`deps-minor` ne pourra **jamais** passer live : la configuration refuse tout simplement cette combinaison. Une version *minor* peut changer un comportement, et aucun contrôle automatique ne le prouvera mieux qu'un humain.

## 🚦 Deux façons de merger

Une PR d'agent se merge donc de deux façons, et **c'est toujours l'App merger qui appuie sur le bouton** :

| Cas | Quand | Qui merge |
|---|---|---|
| 🚀 **Sans humain** | Sa classe est live pour ce dépôt, et tous les contrôles du gate passent sur **ce** commit | L'App merger, seule |
| ✅ **Sur une approbation** | Toute autre PR d'agent, dès qu'un mainteneur approuve son commit courant | L'App merger, dans la foulée |

Le second cas change tout au quotidien. Avant, je relisais, j'approuvais, puis je devais encore contourner les règles de protection de la branche pour merger. Désormais, **mon approbation est le merge**, et elle est liée au commit que j'ai lu :

* si l'agent pousse un nouveau commit après mon approbation, policy-bot l'annule et rien ne part ;
* une approbation donnée avant la fin du dernier run qui a poussé ne compte pas ;
* une demande de changements (*request changes*) bloque tout.

Les règles de protection de `main` interdisent un merge ordinaire. Seuls l'App merger et un administrateur du dépôt peuvent les contourner, et la clé de l'App merger n'est montée que dans le contrôleur de Maku, jamais dans une sandbox. **Un agent ne peut ni approuver ni merger sa propre PR**, par construction.

## 🔍 Le gate, contrôle par contrôle

Le gate de Maku applique une liste **ordonnée** de contrôles au commit courant de la PR. Chaque « non » produit un **code de raison**, écrit sur la PR avec la seule action qui vous revient, et la PR attend un humain.

{{< img src="merge-gate.png" alt="Le merge gate : une PR d'agent passe quatre contrôles dans l'ordre (hygiène, politique, preuve, limites de la classe) ; le moindre « non » l'envoie à un mainteneur, dont l'approbation vaut merge. Si tout passe et que la classe est live, l'App merger merge sans humain, puis main est surveillée 30 minutes ; une classe en shadow écrit « would merge » et attend le mainteneur" width="760" >}}
</br>

| Étape | La question posée | Exemples de codes de raison |
|---|---|---|
| 🧼 **Hygiène** | Aucun secret détecté ? La CI requise est-elle verte ? Le contrat de tests est-il intact ? | `secret_scan_red`, `ci_red`, `contract_modified` |
| 📜 **Politique** | policy-bot est-il vert ? Aucun fichier sensible (*gate path*) n'est-il touché ? | `policy_pending`, `gate_path` |
| 🔍 **Preuve** | La PR vient-elle bien d'un agent de la factory ? Le reviewer a-t-il approuvé **ce** commit ? | `verdict_stale`, `foreign_trailer` |
| 📏 **Limites de la classe** | Les fichiers et la taille du diff restent-ils dans le périmètre de la classe ? | `file_not_allowed`, `patch_too_large` |
| 🔌 **Disjoncteur et plafonds** | La classe n'est-elle pas en pause ? Le plafond quotidien n'est-il pas atteint ? | `class_paused`, `auto_merge_cap` |

Quand la CI est rouge, le gate ne rend pas la main tout de suite : il relance d'abord un run de correction (deux au plus sur `cloud-native-ref`), puis passe la main à un humain si ça ne suffit pas. Et sur `cloud-native-ref`, le plafond est volontairement bas : **un seul merge sans humain par jour**, toutes classes confondues.

### Et après un merge sans humain ?

La factory surveille `main`. Seuls deux événements déclenchent un revert :

| Déclencheur | Pendant | Ce qui se passe |
|---|---|---|
| 🔴 Un check de CI qui tourne sur `main` (pre-commit, scan de sécurité, validation Kubernetes, scripts, liens de la doc) échoue sur le commit de merge | 30 minutes après le merge | Revert automatique : l'App merger ouvre la PR de revert |
| 🙋 Un mainteneur pose le label `factory/revert` sur la PR | 7 jours après le merge | L'App merger ouvre la PR de revert à sa demande |

Sur `cloud-native-ref`, cette PR de revert se merge elle-même, puisque la classe `revert` y est live. Une PR mergée par un humain, elle, n'est jamais revertée par la factory.

Certaines classes sont surveillées plus longtemps par un **moniteur post-merge**. Selon la classe, il vérifie que Flux applique bien le changement (les `Kustomization` et `HelmRelease` restent saines), que le scan de liens confirme le correctif, ou qu'aucune PR ne vient corriger les lignes introduites dans la semaine. Ce moniteur ne revert jamais rien : il rend un verdict (`healthy`, `inconclusive` ou `regression`), et c'est le disjoncteur qui en tient compte.

## 📜 Deux gardiens

Le gate de Maku n'est pas seul. Le premier gardien est [**policy-bot**](https://github.com/palantir/policy-bot), qui publie un statut **requis** par les règles de protection de `main`. Sa particularité : il lit `.policy.yml` **depuis `main`**, jamais depuis la PR qu'il évalue. Un agent ne peut donc pas modifier la règle qui le juge. Voici celle qui autorise la classe `docs-links` :

```yaml
# .policy.yml (cloud-native-ref), extrait
- name: "low-risk: docs-links"
  if:
    has_author_in:            {users: ["ogenki-agents[bot]"]}
    only_has_contributors_in: {users: ["ogenki-agents[bot]"]}
    from_branch:              {pattern: '^agent/[a-z2-7]{8}$'}
    only_changed_files:       {paths: ['^(docs|website/content)/.+\.md$']}
    no_changed_files:         {paths: [<les gate paths>, '^docs/(superpowers|specs)/']}
    file_not_added:           {paths: ['.*']}
    file_not_deleted:         {paths: ['.*']}
    modified_lines:           {total: "< 21"}
    has_workflow_result:      {conclusions: ["success"], workflows: [".github/workflows/ci.yaml"]}
```

Les **gate paths** sont la liste des fichiers qu'aucune PR d'agent ne peut toucher sans un humain : `.policy.yml` lui-même, `.github/`, les instructions des agents (`AGENTS.md`, `CLAUDE.md`), les manifests de la plateforme IA et de ses secrets, les compositions Crossplane, les policies OpenBao… Une trentaine de motifs au total. Une PR qui en touche un repart vers un humain, même si le reste est parfait.

Une liste de ce genre se périme facilement. Deux contrôles automatiques l'en empêchent : un script de CI échoue si une règle oublie l'un de ces motifs, et un test vérifie que le périmètre de chaque classe dans la configuration de Maku est identique à sa règle policy-bot.

Le second gardien est le gate de Maku décrit plus haut. Les deux se recouvrent volontairement : policy-bot tient la politique **versionnée dans le dépôt**, Maku tient la **preuve** (le verdict du reviewer sur ce commit, l'identité de chaque push) et l'**historique** de la classe. Une PR d'agent porte aussi un pied de provenance (`Agent-Room`, `Agent-Run`, `Agent-Task`, `Agent-Model`), mais la doc de Maku est explicite : *« Trailers and footer are provenance hints, never authorisation. »* Ce qui lie un commit à la factory, c'est l'identité qui l'a poussé, pas ce qui est écrit dedans.

## 🌗 Du shadow au live, et retour

Comment décider qu'une classe peut se merger sans humain ? En la mettant d'abord à l'épreuve, sans risque. C'est le rôle du **mode shadow**, une sorte de répétition générale :

* **En shadow**, le gate fait tout son travail mais ne merge rien. Il écrit simplement sur la PR ce qu'il aurait fait. Un mainteneur relit et merge, comme d'habitude.
* **En live**, le gate merge lui-même, sans humain.

Le commentaire laissé en shadow ressemble à ceci :

```text
would auto-merge: docs-links, checks green, verdict approve
… a maintainer merges or closes this pull request.
```

Chaque PR traitée en shadow devient un test. Quand le gate disait « j'aurais mergé » et que la PR, mergée par un humain, n'a rien cassé, c'est un bon point. Elle compte comme mauvaise si elle a été revertée, si une PR ultérieure a dû la corriger, ou si le moniteur a relevé une régression. Un merge n'est compté qu'après un délai d'observation (une semaine pour la plupart des classes), le temps de voir s'il a vraiment tenu ; les cas où l'on ne peut pas conclure sont écartés.

Pour passer une classe en live, quelques succès ne suffisent pas. Je veux pouvoir affirmer, avec 95 % de confiance, que **moins d'un merge sur dix** poserait problème. Le calcul statistique (la borne de Clopper-Pearson) donne le nombre de merges propres nécessaires :

| Mauvais merges observés | Merges concluants nécessaires |
|---|---|
| 0 | **29** |
| 1 | 46 |
| 2 | 61 |

Un script compte tout ça à partir des PR et de leurs labels, et la promotion reste une décision humaine : une PR de configuration qui cite le résultat du décompte. Pas de démo réussie trois fois qui devient une autonomie générale.

### Le disjoncteur

Dans l'autre sens, un **disjoncteur** protège les classes déjà live. Au moindre incident parmi les dix derniers merges d'une classe (un revert, ou une PR qui a dû corriger la précédente), la classe perd son autonomie et repasse sous approbation humaine. Maku l'annonce sur une issue de contrôle et l'expose dans une métrique. La classe retrouve son autonomie d'elle-même quand ses **trois merges suivants**, approuvés entre-temps par un humain, se sont révélés propres.

Ce disjoncteur, je l'ai testé exprès, lors d'exercices (des *drills*). Le principe : laisser une PR se merger seule, puis demander son annulation avec le label `factory/revert`. La factory ouvre alors la PR de revert et la merge elle-même, sans personne. Pour que l'exercice ne compte pas comme un incident, il suffit de poser le label `factory/drill` juste avant : le disjoncteur l'ignore et la classe garde son autonomie. Un revert déclenché par une CI rouge, lui, n'est **jamais** considéré comme un exercice.

## 📈 Ce que le gate mesure

Chaque décision est enregistrée deux fois : en commentaire sur la PR (avec les valeurs lues, pour pouvoir la rejouer plus tard) et en métrique. Les principales séries :

| Métrique | Ce qu'elle répond |
|---|---|
| `maku_factory_merge_decisions_total{class, verdict, reason}` | Pourquoi le gate a dit oui ou non, classe par classe |
| `maku_factory_pr_outcomes_total{class, outcome}` | Que sont devenues les PR : mergées seules, sur approbation, par un humain, revertées |
| `maku_factory_class_paused` | Quelles classes ont perdu leur autonomie |
| `maku_factory_monitor_verdicts_total` | Les verdicts du moniteur post-merge |

Des alertes s'appuient dessus : un merge automatique reverté, une régression détectée, une classe en pause depuis plus d'une semaine. Le [cinquième épisode](/fr/post/series/agentic_ai/agent-factory-5-observabilite/) y revient en détail.

## 💭 Dernières remarques

* **L'autonomie se gagne par classe, et se perd par classe.** Le shadow, le décompte et le disjoncteur transforment une opinion (« ça marche bien ») en décision (« 29 merges propres, borne sous 10 % »). Et la décision reste révocable à chaque instant.

* **Le seuil est un choix, pas une vérité.** 10 % de mauvais merges tolérés, c'est raisonnable pour des liens de documentation, et sans doute beaucoup trop pour autre chose. Le mécanisme permet de régler ce budget classe par classe ; c'est à chaque équipe de le fixer.

* **Le vrai goulot, c'est la relecture.** C'est ce que disent aussi les équipes qui font tourner des agents à grande échelle : chez Spotify, *« code generation is no longer the bottleneck. PR review is »*. Faire de l'approbation le merge était un premier pas ; une file de revue commune à tous les dépôts en sera un autre.

* **La preuve repose sur l'identité, pas sur la bonne foi.** Le gate vérifie qui a poussé chaque commit, pas ce que l'agent écrit dans son message. Une limite subsiste : le jeton GitHub d'un run est lié au dépôt et au rôle, pas au run lui-même. Le gate rattrape un run qui pousserait sur la branche d'un autre, mais ce n'est pas une isolation.

Dans le prochain épisode, on passe de la décision à la mesure : que coûte une flotte d'agents, et comment l'arrêter quand elle s'emballe ? 📈

## 🔖 Références

* [Le merge gate dans la documentation d'`ai-native-ref`](https://airef.ogenki.io/docs/how-it-works/merge-gate/)
* [`maku`](https://github.com/Smana/maku) : le contrôleur et son gate
* [policy-bot](https://github.com/palantir/policy-bot) : la politique de merge versionnée
* [Addy Osmani : Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/)
* [Intervalle de Clopper-Pearson](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Clopper%E2%80%93Pearson_interval)
