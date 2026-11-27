+++
author = "Smaine Kahlouch"
title = "`Agent Factory` : workflows et merge policies, à chaque changement sa dose de process"
date = "2026-11-26"
summary = "Combien de process pour un changement, et quand peut-il partir sans humain ? Workflows, step-up, plan gate, merge policies en `shadow` puis en `live`… et une question plus gênante : que veut dire « succès » pour une merge policy ?"
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
2. [Maku, le chef d'orchestre](/fr/post/series/agentic_ai/agent-factory-2-maku/)
3. **Workflows et merge policies** (vous êtes ici)
4. Le harnais : une sandbox pour chaque agent (à venir)
5. Observer et maîtriser une flotte d'agents (à venir)
6. D'agent à agent : RunLore confie ses correctifs à la factory (à venir)
{{% /notice %}}

Dans l'[article précédent](/fr/post/series/agentic_ai/agent-factory-2-maku/), une correction d'une ligne dans la doc traversait toute la factory : un implementer, un reviewer, une CI verte… puis la PR m'attendait, avec ce message dans la room :

```text
Why: it would auto-merge, and the merge gate is in shadow
```

Autrement dit : « j'aurais pu la merger seule, mais on ne me fait pas encore confiance ». Ce message résume les deux décisions que la factory prend pour chaque tâche :

* **Combien de process ce changement mérite-t-il ?** Un agent seul, une relecture, un plan validé avant d'écrire la moindre ligne ?
* **Peut-il être mergé sans humain ?** Et si oui, sur quelle base ?

Dans [Maku](https://maku.ogenki.io/), la première relève des **workflows**, la seconde des **merge policies**. Je pensais que la seconde était surtout une affaire de règles bien écrites. Puis, cette semaine, je me suis posé une question toute bête à laquelle je n'avais pas de bonne réponse : **que veut dire « succès » pour une merge policy ?** On y arrive, promis. 😉

## 🎯 Objectifs de cet article

* 👥 Choisir la bonne dose de process : les **workflows** `quick`, `standard`, `careful`, et un workflow sur mesure
* ⬆️ Comprendre le **step-up** et le **plan gate** : quand la factory en fait plus, et quand elle attend votre accord avant de coder
* 🚦 Savoir ce qu'est une **merge policy**, lire une décision du gate, et suivre le chemin de `shadow` à `live`
* 🤔 Regarder en face la limite du système : le gate dit si un changement **peut** être mergé, pas s'il était **juste**

{{% notice info "D'où viennent les exemples ?" %}}
Tout ce qui suit a été vérifié sur un vrai cluster les 25 et 26 novembre, avec Maku v0.54. Les issues et PR citées existent, ce sont des exercices (*drills*) dont le [compte rendu](https://github.com/Smana/ai-native-ref/blob/main/docs/runbooks/agent-factory/results-gcp-0-2026-11-25.md) est public, échecs compris : sur 31 vérifications, 15 sont passées, 6 ont échoué, et plusieurs de ces échecs ont été corrigés dans la foulée.
{{% /notice %}}

## 🧭 Deux questions, deux mécanismes

Avant d'entrer dans le détail, une distinction qui évite bien des confusions (je me suis moi-même emmêlé plus d'une fois) :

| | **Workflow** | **Merge policy** |
|---|---|---|
| La question | Qui travaille sur la tâche, et dans quel ordre ? | Ce diff peut-il être mergé sans humain ? |
| Quand | Au démarrage de la tâche | À la fin, sur le commit exact de la PR |
| Ce qui est regardé | La tâche : sa source, un label, le défaut du dépôt | Le **diff** : fichiers, taille, forme, verdict du reviewer |
| Qui le choisit | Vous (ou le planner, ou le défaut du dépôt) | Personne : le gate le déduit du diff |

Les deux sont indépendants, à une exception près que nous verrons plus bas : une policy peut **exiger** une étape de workflow.

## 👥 Workflows : à chaque tâche sa dose de process

Corriger une faute de frappe et ajouter une fonctionnalité ne demandent pas le même encadrement. Un **workflow** décrit donc la liste des **stages** (les rôles qui se relaient) et le budget alloué.

### Les trois workflows livrés

| Workflow | Stages | Pour quoi |
|---|---|---|
| **`quick`** | implementer | Un changement qu'un check sait vérifier : un lien mort, un revert, une tâche planifiée |
| **`standard`** (défaut) | implementer → reviewer | Le cas général. Le reviewer repart de zéro et peut demander deux rounds de corrections |
| **`careful`** | scoper → (tester) → implementer → reviewer | Une tâche large ou floue. Le scoper cadre et peut découper en 4 parties au plus, un *judge* vérifie les critères d'acceptation avant la review |

Voici leur définition, telle que Maku la livre :

```yaml
# Factory : les workflows livrés par défaut (extrait)
workflows:
  quick:
    description: "One implementer. For changes a check can verify."
    budget: standard
    stepUp: standard
    stages:
      - {role: implementer, stall: true}
  standard:
    description: "An implementer and a reviewer. The default."
    budget: standard
    stepUp: careful
    stages:
      - {role: implementer, stall: true}
      - {role: reviewer, maxRounds: 2}
  careful:
    description: "Scoped first, judged before review. For wide or unclear tasks."
    budget: frontier
    stages:
      - {role: scoper, split: {maxParts: 4}, gate: {when: plan-first}}
      - {role: tester, when: repository.testFirst}
      - {role: implementer, search: true, stall: true, judge: evidence}
      - {role: reviewer, maxRounds: 2}
```

Qui choisit ? Dans l'ordre : un mainteneur qui **épingle** un workflow (le label `workflow:careful` sur l'issue, ou le menu de la carte de plan dans Slack et dans le coding agent), puis la **source** de la tâche (`quick` pour un revert ou une tâche planifiée), et enfin le **défaut du dépôt** (`standard`).

{{% notice tip "Pas de critère d'acceptation, pas de run" %}}
Avant de lancer quoi que ce soit, la factory vérifie que l'issue dit comment on saura que c'est terminé (une section `## Acceptance`). Sinon, elle pose la question et ne dépense pas un token. C'est frustrant la première fois, puis on prend le pli : c'est exactement ce dont le reviewer et le judge ont besoin pour faire leur travail.
{{% /notice %}}

### 🔧 Un workflow sur mesure

Les stages se composent. Sur ma plateforme, j'ai ajouté `thorough`, qui glisse une relecture orientée sécurité avant la review habituelle :

```yaml
# ai-native-ref : tooling/base/agent-factory/resources/factory.yaml (extrait)
workflows:
  thorough:
    description: "Tests first, then two reviews: correctness and security."
    budget: frontier
    stages:
      - {role: tester, when: repository.testFirst}
      - {role: implementer, checks: true, stall: true}
      - {role: security-reviewer}
      - {role: reviewer, maxRounds: 2}
roles:
  security-reviewer:
    capability: read
    brief: roles/security-reviewer.md
```

Un rôle sur mesure est **en lecture seule** : il reçoit le token du reviewer et ne peut rien pousser. Aucun dépôt n'utilise `thorough` par défaut, c'est le label `workflow:thorough` qui le déclenche. Je l'ai essayé sur l'[issue #501](https://github.com/Smana/ai-native-ref/issues/501) : implementer, `security-reviewer`, puis reviewer, avec 0,14 $ dépensés à l'entrée en review, sur un plafond de 13,60 $.

### ⬆️ Le step-up : monter d'un cran, une seule fois

Que se passe-t-il quand `standard` ne suffit pas ? Plutôt que d'abandonner, la factory peut **monter d'un cran** sur l'échelle `quick` → `standard` → `careful`. Le déclencheur est toujours un échec constaté, jamais une impression : budget du run épuisé, agent qui tourne en rond, rounds de review épuisés, CI rouge après correction.

{{< img src="step-up-github-comment.png" alt="Un commentaire de la factory sur une issue GitHub : « Maku stepped this task up from standard to careful: the run's budget was spent. It runs once more, scoped first and judged before review, on the same branch and pull request. New ceiling: $5.10 for the task »" width="900" caption="Le step-up annoncé sur l'issue : même branche, même PR, nouveau plafond." >}}
<br>

Trois réglages par dépôt (`workflow.stepUp`) :

* **`auto`** : la factory monte seule, une fois, et l'annonce. C'est mon réglage sur `ai-native-ref`.
* **`ask`** : elle s'arrête et propose. C'est mon réglage sur `cloud-native-ref`, où une tâche coûte plus cher.
* **`off`** : elle escalade sans rien proposer.

Quand elle propose, on accepte d'un commentaire `/factory retry deeper`, avec `maku retry <tâche> --deeper`, ou d'un clic dans la room. Sur l'[issue cnref#2912](https://github.com/Smana/cloud-native-ref/issues/2912), le bouton disait exactement ce qui allait se passer : **Run again deeper (careful, up to $13.60)**.

Deux garde-fous importants. La tâche reste dans la **même room, la même branche et la même PR**, on ne repart pas de zéro. Et le step-up ne dépasse jamais le coût que vous avez confirmé : au-delà, la factory refuse (`deeper_over_cap`) et vous demande de relancer la tâche en connaissance de cause.

### 📝 Le plan gate : valider le plan avant le code

Pour certains changements, je préfère relire **l'intention** plutôt que le diff. C'est le rôle du **plan gate** : avec le label `plan-first`, le scoper propose un plan, puis plus rien ne tourne tant qu'un humain n'a pas répondu.

{{< img src="plan-gate-github-comment.png" alt="Le commentaire de plan posté par la factory sur une issue : la tâche bornée, les fichiers attendus, trois critères d'acceptation exécutables, puis les trois réponses possibles : /factory plan approve, /factory plan revise, /factory plan split" width="850" caption="Le plan proposé sur l'[issue #593](https://github.com/Smana/ai-native-ref/issues/593) : périmètre, fichiers attendus, critères d'acceptation, et trois réponses possibles." >}}
<br>

Trois réponses, depuis l'issue ou depuis la room :

* **`/factory plan approve`** : le plan devient le périmètre de la PR.
* **`/factory plan revise <commentaire>`** : le scoper refait un tour avec votre remarque (une seule fois par tâche).
* **`/factory plan split`** : le plan est découpé en plusieurs tâches, chacune avec sa PR.

Sur l'[issue #512](https://github.com/Smana/ai-native-ref/issues/512), j'ai demandé une révision puis approuvé. Sur la [#541](https://github.com/Smana/ai-native-ref/issues/541), le scoper a proposé deux parties (coût du plan : 0,01 $) et un clic sur **Split** a ouvert la première. Tant qu'un plan attend, aucun run ne tourne ; sans réponse au bout de 72 heures, il est mis de côté, et la première réponse le relance.

### 🎁 Quatre résultats possibles

Dernière pièce : une tâche ne se termine pas forcément par une PR. Maku distingue quatre **outcomes** :

| Outcome | Comment le demander | Ce que vous obtenez |
|---|---|---|
| **`change`** (défaut) | `factory/ready` sur l'issue | Une PR |
| **`plan-first`** | Le label `plan-first` avant `factory/ready` | Un plan à valider, puis une PR |
| **`report`** | Le label `report` avant `factory/ready` | Des constats postés sur l'issue, aucune branche. `/factory fix` en fait ensuite une tâche `change` |
| **`review`** | Le label `factory/review` sur **votre** PR | Un verdict du reviewer, en lecture seule : ni push, ni merge |

Le dernier est mon préféré côté développeur : pas besoin de confier son code à la factory pour profiter d'un second regard ([PR #515](https://github.com/Smana/ai-native-ref/pull/515)).

## 🚦 Merge policies : qui a le droit de merger ?

Passons à l'autre bout de la chaîne. La PR existe, la CI est verte, le reviewer a approuvé. Qui appuie sur le bouton ?

Par défaut, un mainteneur. Une **merge policy** (la CRD `MergePolicy`) décrit une famille de changements suffisamment bornée pour s'en passer. Maku en livre trois, actives en `shadow` sur chaque dépôt dès la première PR, sans une ligne de YAML :

| Policy | Ce qu'elle couvre | Sa borne |
|---|---|---|
| **`docs-links`** | Des cibles de liens dans du Markdown, rien d'autre, 20 lignes | La **forme du diff** : `links-only` |
| **`deps-patch`** | Les mises à jour *patch* de Renovate, après leur délai de maturité, 3 fichiers, 10 lignes, 2 par jour | La forme du diff : `semver-patch` |
| **`docs`** | De la prose dans des fichiers Markdown, 5 fichiers, 100 lignes, 2 par jour | Les fichiers autorisés, et l'approbation d'un reviewer |

```yaml
# Le catalogue livré par le chart agent-factory (extrait)
kind: MergePolicy
metadata: {name: docs-links}
spec:
  description: Link targets in Markdown, nothing else.
  diffShape: links-only
  modifyOnly: true
  maxModifiedLines: 20
  evidence: low
---
kind: MergePolicy
metadata: {name: docs}
spec:
  description: Prose in Markdown files, small changes.
  allowedFiles: ['^.+\.mdx?$']
  maxPatchFiles: 5
  maxModifiedLines: 100
  maxPerDay: 2
  evidence: standard
```

Chaque dépôt **lie** ensuite les policies qu'il veut ajuster : passer l'une en `live`, en désactiver une autre, resserrer des chemins. Et au-dessus de tout, des **guardrails** centraux qu'aucune policy ne peut dépasser : par défaut 10 fichiers, 300 lignes, 5 merges par jour, et interdiction de toucher à `.github/` ou à `CODEOWNERS`. Un agent ne modifie pas la règle qui le juge.

### 🔍 Le gate, étape par étape

{{< img src="merge-policy-flow.png" alt="Le parcours d'une PR d'agent dans le merge gate : le socle fixe (CI verte, pas de secret, aucun fichier protégé), puis la recherche d'une policy qui tient sur le diff. Tout « non » envoie la PR à un mainteneur, avec la raison. Une policy en shadow consigne « would merge » sans rien merger ; une policy live merge via l'App merger, puis main est surveillée. Les décisions et ce qui suit les merges alimentent les preuves, qui servent à la promotion et au disjoncteur" width="1240" link="true" caption="Tout « non » mène à un humain, avec la raison. Seule une policy `live` qui tient merge seule." >}}
<br>

Deux points méritent qu'on s'y arrête.

**Le gate choisit la policy à partir du diff, pas d'un label.** Il évalue toutes les policies candidates et retient la plus stricte parmi celles qui **tiennent** (*hold*), c'est-à-dire dont toutes les conditions sont vraies sur ce commit. Si celle-ci est en pause ou a atteint son plafond, un humain merge : une policy plus permissive ne prend jamais le relais.

**Workflow et policy se rejoignent ici.** Une policy sans forme de diff vérifiable, comme `docs`, exige l'approbation d'un reviewer. Une tâche `quick`, sans reviewer, ne pourra donc jamais merger de la prose toute seule : le gate répond `reviewer_verdict_missing`. Moins de process en amont, c'est moins d'autonomie en aval.

### 📖 Lire une décision

Chaque décision est écrite sur la PR et consultable dans les pages **Settings** de Maku. Voici celles d'`ai-native-ref`, où tout est encore en `shadow` :

{{< img src="merge-policy-recent-decisions.png" alt="La page Merge policies de Maku pour ai-native-ref : la liste des décisions récentes. La PR #480 apparaît deux fois : « Would merge » sous la policy docs, parce que le gate est en shadow, puis « Merged automatically » après l'approbation d'un mainteneur. D'autres PR ont attendu un mainteneur, avec la raison donnée policy par policy" width="1100" caption="Les décisions récentes d'un dépôt. La [PR #480](https://github.com/Smana/ai-native-ref/pull/480) est celle de la démo de l'article précédent." >}}
<br>

On y retrouve la PR de la démo : `docs` **aurait** mergé (*Would merge*), puis mon approbation a déclenché le merge. Et pour une PR refusée, la raison est donnée **policy par policy**.

Pour m'assurer que `docs-links` ne laisse passer que ce qu'elle prétend, je lui ai soumis quatre PR pièges :

| PR | Le diff | La décision |
|---|---|---|
| [#521](https://github.com/Smana/ai-native-ref/pull/521) | Un lien relatif corrigé | ✅ `docs-links · shadow_would_arm` |
| [#522](https://github.com/Smana/ai-native-ref/pull/522) | Un fichier `icons/manifest.json` | ❌ `no_policy_matched` |
| [#520](https://github.com/Smana/ai-native-ref/pull/520) | Un lien GitHub dont le **propriétaire** change | ❌ `docs-links: class_mismatch` |
| [#524](https://github.com/Smana/ai-native-ref/pull/524) | Une URL d'archive pointant vers une **autre** page | ❌ `no_policy_matched` |

La #520 est la plus intéressante : changer `github.com/Smana/...` en `github.com/quelquun-dautre/...`, c'est techniquement « corriger un lien ». C'est aussi un excellent moyen de rediriger vos lecteurs vers un dépôt malveillant. `links-only` l'a refusée.

En ligne de commande, `maku policy list` résume les décisions d'un dépôt et `maku policy explain` détaille l'une d'elles. La même chose dans l'interface, ici pour une mise à jour de Renovate sur `cloud-native-ref` :

{{< img src="merge-policy-decision-timeline.png" alt="La chronologie d'une décision du merge gate pour la PR 2876 de cloud-native-ref sous la policy deps-patch : socle fixe passé, auteur adopté, conditions enregistrées (1 fichier sur 3, 2 lignes sur 10, forme du diff ok, délai de maturité ok), ni pause ni plafond, verdict shadow_would_arm" width="900" caption="Une décision, ligne par ligne : ce que le gate a lu, et où il se serait arrêté." >}}
<br>

### 🌗 De `shadow` à `live`

Une policy commence toujours en `shadow` : elle décide, l'écrit, et ne merge rien. Pour passer en `live`, trois conditions :

1. **Assez de décisions confirmées** : 5 pour une policy dont la forme du diff se vérifie mécaniquement, 20 sinon. Une décision ne compte que si un mainteneur a mergé la PR **telle quelle**, au commit que le gate avait jugé.
2. **Aucun mauvais merge** : ni revert, ni correctif ultérieur, ni régression.
3. **Un exercice de rollback réussi** sur le dépôt.

Ce dernier point, je l'ai joué pour de vrai : `maku policy drill Smana/ai-native-ref#505` a ouvert la [PR de revert #584](https://github.com/Smana/ai-native-ref/pull/584) via l'App *merger*, étiquetée `factory/drill`. Je l'ai mergée, et la page des preuves est passée à `drill: passed` sans que le disjoncteur ne compte un incident. Petite leçon au passage : un drill sur un vrai changement retire vraiment ce changement de `main`. Il a fallu une seconde PR pour le remettre. 😅

Quand tout est réuni, le bouton **Promote to live** ouvre une PR qui change **une ligne** (`mode: shadow` → `mode: live`), avec les décomptes dans sa description. Un humain la relit et la merge. La factory ne se promeut jamais elle-même.

{{% notice note "Où j'en suis" %}}
Soyons honnêtes : aujourd'hui, **une seule policy est `live`** chez moi, `docs-links` sur `cloud-native-ref`, plafonnée à **un merge sans humain par jour**. Tout le reste est en `shadow`, et je n'ai pas encore déroulé une promotion complète depuis l'interface.
{{% /notice %}}

### 🔌 Après le merge : surveiller, annuler, disjoncter

Un merge sans humain n'est pas la fin de l'histoire :

* ⏱️ **`main` est surveillée** pendant 30 minutes. Un check rouge sur le commit de merge déclenche un revert automatique.
* ↩️ **Un mainteneur peut annuler** pendant 7 jours avec le label `factory/revert`.
* 👀 **Un moniteur** suit ensuite certaines policies plus longtemps : Flux applique-t-il toujours correctement ? Une PR ultérieure a-t-elle dû corriger les mêmes lignes ? [RunLore](https://github.com/Smana/runlore) confirme-t-il que l'alerte est résolue ?
* 🔌 **Un disjoncteur** (*breaker*) compte les incidents parmi les derniers merges de chaque policy. Chez moi, un seul revert ou correctif parmi les dix derniers suffit : la policy repasse sous approbation humaine, et seuls des merges validés par des mainteneurs la réarment.

Et pour geler une PR précise, pas besoin d'outillage : le label `factory/hold` la rend à un humain avant toute évaluation.

### 🔗 Quand une policy impose son workflow

Voici l'exception annoncée plus haut. Une règle d'alerte, c'est un petit diff avec un gros rayon d'impact. Ma policy `alert-rules` déclare donc :

```yaml
# ai-native-ref : tooling/base/agent-factory/resources/mergepolicy-alert-rules.yaml (extrait)
kind: MergePolicy
metadata: {name: alert-rules}
spec:
  requires: [plan]
  monitor: {verification: {within: 2h}, followUpWindow: 2h}
```

Toute tâche qui touche `observability/` passe par le scoper, **quel que soit son workflow**. La première tentative a échoué : la tâche a filé en `standard` sans plan. Après correctif, sur l'[issue cnref#2922](https://github.com/Smana/cloud-native-ref/issues/2922), j'avais pourtant épinglé `workflow:standard` : dès le premier diff, la room affichait « the class alert-rules requires a plan », et le run suivant était le scoper. Une exigence de policy **ajoute** des étapes, elle n'en retire jamais.

## 🤔 Mais au fait, c'est quoi un « succès » ?

Tout ce qui précède fonctionne. Et pourtant, en relisant ma page de preuves cette semaine, quelque chose me gênait. Elle affichait fièrement des « clean merges ». *Clean* selon qui ?

Reprenons ce que chaque étage **lit** réellement :

| Étage | Ce qu'il lit | Ce qu'il peut affirmer |
|---|---|---|
| Le gate | La CI, le verdict du reviewer, les fichiers, la taille, la forme du diff | Le changement est **dans le périmètre** de la policy |
| Le moniteur | La CI de `main`, l'état de Flux, les correctifs ultérieurs | **Rien de bruyant** ne s'est produit dans la fenêtre |
| Les preuves | Les PR qu'un mainteneur a mergées telles quelles, les reverts | Un humain était d'accord, et rien de bruyant n'a suivi |

**Aucun des trois ne lit ce que le changement veut dire.** La CI et le reviewer répondent à « peut-on merger ? ». Seule la suite répond à « avait-on raison ? ». Et la suite, le moniteur ne l'entend que si elle fait du bruit.

Un exemple qui m'a convaincu. Une alerte sonne trop souvent, un agent propose de relever son seuil. Le YAML est valide, la CI est verte, le reviewer n'y voit rien d'anormal. Après le merge, l'alerte cesse de sonner et RunLore rapporte « résolu ». Tous les voyants sont au vert… alors que l'alerte vient peut-être d'être **rendue muette**. Un timeout trop court, une limite mémoire trop basse qui provoquera un OOM dans dix jours, un paragraphe de runbook devenu faux : même combat. C'est du faux silencieux, et aucun gate ne sait prouver qu'un changement arbitraire est correct.

### 🧪 La direction : des signaux de justesse

Je n'ai pas de solution miracle, mais j'ai arrêté une direction : faire dire à chaque policy **ce qui a vérifié le sens du changement**, et ne plus compter le silence comme une preuve.

{{% notice warning "Feuille de route, pas fonctionnalité" %}}
Ce qui suit est un [design validé cette semaine](https://github.com/Smana/maku/blob/main/docs/superpowers/specs/2026-11-26-merge-policy-correctness-signals-design.md), **pas encore du code**. Je le partage parce que le raisonnement me semble plus utile que la liste des options.
{{% /notice %}}

Un **signal de justesse** (*correctness signal*) est une vérification indépendante de l'auteur, qui ne peut pas réussir sans avoir lu le changement. La liste est fermée :

* une **forme de diff** déterminante (`links-only`, `semver-patch`) : chaque ligne modifiée n'a qu'un sens mécanique possible ;
* un **oracle** : un détecteur externe a nommé le défaut, et le diff retire exactement celui-là ;
* un **contrat de test** : un test écrit *avant* le changement, à partir d'un plan qu'un humain a approuvé, rouge avant et vert après, que l'implementer ne peut pas toucher.

Une CI verte, un reviewer qui approuve ou un Flux en bonne santé n'en font **pas** partie : ils disent que quelque chose fonctionne encore, pas que le changement est le bon. De là découlent les règles :

* 🚫 **Une policy sans signal ne passe pas en `live`**. Elle reste en `shadow` et continue d'accumuler des décisions.
* 📝 **Seule exception : la prose**, où aucune machine ne peut trancher. Elle pourra merger seule, mais **une décision sur cinq** sera soumise à un mainteneur, indéfiniment. L'échantillon humain ne s'arrête jamais.
* 🔐 **Les objets sensibles vont à un humain**, sous n'importe quelle policy : `NetworkPolicy`, RBAC, `Secret`, webhooks d'admission… même pour un resserrement.
* 🧾 **Le registre reste ouvert**. Un label `factory/wrong`, posé n'importe quand, transforme un merge en mauvais appel : il met la policy en pause et bloque sa promotion. Pendant 30 jours, la factory signale aussi les PR qui réécrivent des lignes mergées sans humain, et demande : « était-ce un correctif ? »

Pour mon alerte, cela donne : un plan qui dit « sonner au-dessus de 5 % pendant 10 minutes », un test qui l'écrit noir sur blanc, et un changement qui relève le seuil hors d'atteinte **échoue au test**. La policy se lit alors « merge après votre accord sur le plan, sans relecture du diff », et non plus « merge sans vous ».

Le vocabulaire change avec. Les mots qui promettent trop disparaissent des écrans :

| Le mot | Ce qu'il veut dire | Ce qu'il ne veut **pas** dire |
|---|---|---|
| **holds** (tient) | Toutes les conditions du gate passent sur ce commit | Le changement est juste |
| **signal-checked** | Un signal de justesse a jugé le changement, et l'a validé | Prouvé correct |
| **agreed** (au lieu de *clean*) | Un mainteneur a mergé la décision telle quelle | Il a lu chaque ligne |
| **quiet** (au lieu de *healthy*) | La fenêtre du moniteur s'est écoulée sans incident | Tout va bien |
| **wrong call** | Un merge reconnu mauvais, par n'importe quelle voie, n'importe quand | Un exercice |

« Correct », « sûr » et « réussi » n'apparaîtront sur aucune page : Maku ne le sait pas. Et un chiffre sera toujours suivi de son juge : « 20 mergées seules : 4 auditées et confirmées, 16 non vérifiées ».

## 👩‍💻 Et côté développeur ?

Tout cela peut sembler être un sujet de *platform engineer*. Pourtant, c'est le développeur qui le vit au quotidien. Ce que j'en retiens, de ce côté-là du clavier :

| Je veux… | Je fais… |
|---|---|
| Un simple second regard sur ma PR | Le label `factory/review` |
| Comprendre un problème avant de décider | Le label `report`, puis éventuellement `/factory fix` |
| Valider l'approche avant le code | Le label `plan-first` |
| Plus d'encadrement sur une tâche délicate | Le label `workflow:careful` |
| Savoir pourquoi ma PR attend | Lire l'encadré **NEXT** de la room, ou `maku policy explain` |
| Empêcher un merge | Le label `factory/hold` sur la PR |
| Annuler un merge de la factory | Le label `factory/revert`, dans les 7 jours |

Et surtout : **une PR qui attend dit toujours pourquoi**, avec la seule action qui vous revient. C'est sans doute ce qui compte le plus pour l'adoption. Un refus sans explication, et on contourne l'outil dès la deuxième fois.

## 💭 Dernières remarques

Quand j'ai commencé à travailler sur le merge automatique, je pensais que la difficulté serait d'écrire les bonnes règles. Elle est ailleurs : **savoir ce que les chiffres ont le droit de dire**. Vingt merges sans incident, ce n'est pas vingt merges corrects, c'est vingt merges dont personne ne s'est plaint. La nuance paraît subtile jusqu'au jour où l'on promeut une policy sur la foi de ce compteur.

Quelques limites à garder en tête :

* **L'échantillon est minuscule.** Une policy `live`, un merge par jour, un seul mainteneur (moi). Les seuils de 5 et 20 décisions sont des choix raisonnables, pas des vérités statistiques : la page des preuves affiche d'ailleurs la borne de confiance à titre indicatif, sans jamais agir dessus.
* **Les signaux de justesse sont un design.** La partie la plus délicate, le contrat de test pour de la configuration, demande aussi du travail dans chaque dépôt (savoir tester une règle d'alerte, par exemple).
* **Le code applicatif reste hors périmètre.** Aucune policy ne couvre un changement de logique métier, et je ne compte pas en ajouter tant qu'un contrat de test ne l'encadre pas.
* **Tout n'a pas marché du premier coup.** Le step-up automatique, par exemple, n'a pas encore été revalidé de bout en bout après son correctif. Le compte rendu liste ce qui reste ouvert.

Si je devais résumer en une phrase : le workflow règle **l'effort**, la policy règle **la confiance**, et ni l'un ni l'autre ne dispense de se demander, après coup, si l'on avait raison.

Dans le prochain article, on descendra d'un étage : le **harnais**, cette sandbox dans laquelle chaque agent s'exécute, et ce qui l'empêche d'en sortir. 🔒

## 🔖 Références

* [`maku`](https://github.com/Smana/maku) : [site et documentation](https://maku.ogenki.io/) · [merge policies](https://maku.ogenki.io/docs/use/merge-policies/) · [la factory : workflows, merge gate, moniteur](https://maku.ogenki.io/docs/concepts/factory/) · [guide : merge policies](https://maku.ogenki.io/docs/guide/merge-policies/) · [écrire une bonne tâche](https://maku.ogenki.io/docs/guide/writing-a-good-task/)
* [Design : les signaux de justesse des merge policies](https://github.com/Smana/maku/blob/main/docs/superpowers/specs/2026-11-26-merge-policy-correctness-signals-design.md)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) : [Maku sur Cloud Native Ref](https://airef.ogenki.io/docs/maku-deployment/) · [état du déploiement](https://airef.ogenki.io/docs/status/)
* [Compte rendu des vérifications sur cluster (25 et 26 novembre 2026)](https://github.com/Smana/ai-native-ref/blob/main/docs/runbooks/agent-factory/results-gcp-0-2026-11-25.md)
