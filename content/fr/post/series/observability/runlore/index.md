+++
author = "Smaine Kahlouch"
title = "`RunLore` : votre buddy SRE qui investigue les incidents — et apprend de chaque résolution"
date = "2026-07-04"
summary = "Souvent, la connaissance d'un incident se dissout dès qu'il est clos. RunLore, agent SRE open source, investigue et vous aide à identifier rapidement la cause racine, puis transforme chaque résolution en savoir réutilisable."
featured = false
codeMaxLines = 30
usePageBundles = true
toc = true
series = [
  "observability"
]
tags = [
    "observability",
    "ai"
]
thumbnail = "thumbnail.png"
+++

{{% notice info "Le chaînon manquant de la série 🔗" %}}
Dans cette [série d'articles](/fr/series/observability/), nous avons exploré comment **détecter** les problèmes : collecter des [métriques](/fr/post/series/observability/metrics/), centraliser des [logs](/fr/post/series/observability/logs/), puis configurer des [alertes pertinentes](/fr/post/series/observability/alerts/).

Une fois l'alerte reçue, il reste pourtant l'étape la plus coûteuse en temps : **comprendre ce qui se passe et pourquoi**. C'est précisément ce que nous abordons ici.
{{% /notice %}}

Une alerte se déclenche. Commence alors un travail de **diagnostic** : parcourir les changements récents dans Git, les dashboards Grafana, les logs, parfois un runbook, pour croiser les indices et formuler une première hypothèse.

Cette phase d'**investigation** est au cœur du métier de **SRE** (_Site Reliability Engineering_) : elle **repose sur une connaissance fine de la plateforme**, pour savoir *où* aller collecter les bons indicateurs. Le **temps** nécessaire pour remonter à la cause peut donc varier fortement selon le niveau de séniorité de la personne qui investigue. Et le savoir accumulé pendant la résolution se dissout trop souvent dès que l'incident est clos.

J'ai décidé de m'attaquer à cette problématique en créant un projet open source il y a quelques semaines : [`RunLore`](https://github.com/Smana/runlore). Au départ, l'idée était simple : un **binôme** (_buddy_) qui prend en charge la phase de collecte pour **orienter rapidement l'investigation vers la cause racine la plus probable**. Et qui, au fil du temps, **apprend le contexte dans lequel nous travaillons ensemble** : notre plateforme, nos contraintes, nos règles et standards.

Pour conserver la connaissance en mémoire, j'ai choisi l'**OKF** (_Open Knowledge Format_) : un format pensé précisément pour qu'un agent IA lise et écrive sa connaissance efficacement. J'y reviens [en détail plus loin](#okf--dune-idée-de-karpathy-à-un-standard).

Cet article en présente les idées et un premier exemple concret.

## 🎯 Objectifs de cet article

* 🔥 Comprendre le **coût caché** de l'investigation d'incident
* 🤖 Découvrir **RunLore** et son fonctionnement
* 🧠 Saisir ses **trois choix de conception** : un binaire simple, une mémoire au standard **OKF**, et l'humain à la décision
* 🛠️ **Déployer** RunLore et voir un **incident réel** investigué sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)
* 💭 Un retour **honnête** sur un projet encore très jeune

## 🔥 Le problème : le coût caché de l'investigation

L'article précédent sur les alertes se terminait sur une évidence : la configuration technique ne suffit pas, il faut aussi des **runbooks et des procédures de réponse aux incidents**. Or, dans la vraie vie, ces runbooks sont incomplets, et une grande partie de la connaissance reste **tribale** : elle vit dans la tête de quelques personnes.

Quand une alerte se déclenche, l'astreinte rejoue à chaque fois le même enchaînement manuel :

* **Qu'est-ce qui a changé ?** Un déploiement, une montée de version, une modification d'infrastructure, un certificat expiré ?
* **Qu'est-ce qui ne va pas ?** Saturation, réseau, nœuds, dépendances en échec ?
* **A-t-on déjà vu ça ?** La réponse est souvent « oui », mais personne ne retrouve où elle est documentée.

Ce pivot permanent entre Git, les métriques, les logs et les flux réseau est exactement le genre de tâche qu'un agent peut prendre en charge. À condition qu'il soit **honnête** sur ce qu'il sait et ne sait pas.

## 🤖 RunLore en bref

{{< img src="runlore-logo.png" alt="Logo RunLore — une chouette à lunettes posée sur un livre ouvert" width="120" >}}

**RunLore** est un **agent SRE** open source (licence Apache-2.0) qui s'exécute dans votre cluster Kubernetes sous la forme d'un binaire Go unique, déployé via Helm. Son principe est volontairement simple : à partir d'un événement (une alerte, un échec **GitOps** — la livraison continue pilotée par Git…), il investigue et répond à deux questions — _what changed?_ et _what's wrong?_ — puis publie dans votre messagerie une **cause racine** assortie d'un **score de confiance**, des preuves qui l'étayent, et des questions ouvertes à destination d'un humain.

La notification est _verdict-first_ : elle s'ouvre sur un **verdict d'actionnabilité** clair — _aucune action_, _action suggérée_, _action requise_ ou _non concluant_ — avant même les détails. L'astreinte sait en un coup d'œil si elle doit intervenir.

{{< img src="runlore-investigation-flow.png" alt="Flux d'investigation de RunLore : un événement déclenche une investigation qui corrèle l'historique GitOps, les métriques, les logs et les flux réseau ; la cause racine est publiée dans Slack et une PR est ouverte dans la base de connaissances" width="1080" caption="Le flux d'investigation : de l'événement à la cause racine, jusqu'à la capture dans la base de connaissances" >}}

Le fil conducteur de l'investigation est l'**historique GitOps** (Flux ou Argo CD) : c'est la colonne vertébrale du _what-changed_, qui produit un diff Git exact entre les révisions déployées. Autour de ce signal, RunLore corrèle tout le reste, chaque source étant **pluggable** : une source non configurée désactive simplement l'outil correspondant.

| Catégorie | Supporté |
|---|---|
| **Déclencheurs** _(sources)_ | Webhook Alertmanager · échecs GitOps · webhook PagerDuty |
| **GitOps** — _what changed_ | Flux · Argo CD |
| **Métriques** | VictoriaMetrics · Prometheus (_PromQL_) |
| **Logs** | VictoriaLogs (_LogsQL_) |
| **Kubernetes** | client-go — statut des pods, events, logs des contrôleurs |
| **Flux réseau** | Cilium Hubble · AWS VPC Flow Logs · GCP Firewall Logs |
| **Cloud** | AWS — CloudTrail, EC2 / ASG / EKS |
| **LLM** | Anthropic · Gemini · tout endpoint OpenAI-compatible (_vLLM, Ollama, OpenRouter…_) |
| **Notifications** | Slack · Matrix · webhook générique |
| **Base de connaissances** | GitHub (_App auth_) |

{{% notice info "Et la sécurité ? 🔒" %}}
Brancher un agent LLM sur son cluster soulève une question immédiate : que voit-il, et où cela part-il ? RunLore y répond par construction :

* **Redaction des secrets à trois frontières de confiance** : le texte de l'incident, chaque sortie d'outil avant d'atteindre le modèle, et l'investigation finale avant toute publication (PR ou chat) ;
* **RBAC minimal** : la lecture des logs de pods est restreinte à une liste de namespaces (par défaut `flux-system` uniquement) ;
* **Journal d'audit** en append-only, chaîné par hash : chaque décision est traçable et infalsifiable.
{{% /notice %}}

### Trois choix de conception

Des agents qui investiguent un incident, il en existe déjà (nous y reviendrons). Ce qui distingue RunLore tient en trois choix — et c'est leur **combinaison**, plus qu'aucun d'eux isolément, qui compte :

1. 📦 **Un simple binaire, chez vous, sur vos modèles.** Un unique binaire Go dans votre cluster ; vos données et vos LLM ne quittent jamais votre périmètre. Pas de SaaS, pas de _lock-in_.
2. 🧠 **Une mémoire qui se capitalise.** Chaque investigation nourrit un **catalogue de connaissances que vous possédez**, au standard **OKF**. Le même incident, la fois suivante, obtient une réponse instantanée plutôt qu'une nouvelle investigation.
3. ✋ **L'humain garde la main.** L'agent lit et recommande, il n'agit jamais seul (_read-only → suggest → approve_) — et surtout, **rien n'entre dans sa mémoire sans une PR relue par un humain**. Ce qu'il ne sait pas, il le délègue plutôt que de le deviner.

Les deux sections qui suivent creusent les deux choix les moins évidents : la mémoire, puis la place gardée à l'humain.

## 🧠 Une mémoire qui se capitalise

La boucle autonome _alerte → cause racine → chat_ est aujourd'hui **banalisée** : plusieurs outils savent le faire. Ce qui l'est beaucoup moins, c'est une base de connaissances qui **se consolide dans un catalogue que vous possédez**.

### La boucle d'apprentissage

RunLore implémente un cycle en quatre temps :

1. **Retrieve** — face à un incident, il commence par chercher une réponse passée _digne de confiance_ dans la base de connaissances.
2. **Capture** — si rien ne correspond, il investigue à chaud, enregistre ce qu'il a trouvé, puis **observe la suite** : l'incident s'est-il effectivement résolu ?
3. **Curate** — un finding suffisamment fiable et nouveau devient une **Pull Request** rédigée dans votre dépôt de connaissances. C'est le **point de contrôle qualité** : on relit une entrée comme on relit du code.
4. **Compound** — une fois la PR fusionnée, l'entrée est ré-indexée et devient immédiatement disponible. Le même incident, la prochaine fois, obtient une réponse instantanée — sans ré-investigation.

Tout part donc de **Retrieve**, qui tranche entre deux issues : servir une réponse connue **instantanément**, ou — faute de correspondance fiable — lancer la boucle complète, celle qui nourrit le catalogue.

{{< img src="runlore-learning-loop.png" alt="Les deux issues d'un recall dans RunLore : un hit répond instantanément sans investigation ; un miss déclenche la boucle en quatre temps — Retrieve, Capture, Curate, Compound — qui nourrit un catalogue que vous possédez et rend le même incident instantané la fois suivante" width="1080" caption="Un **hit** répond instantanément, sans investigation ; un **miss** déclenche la boucle complète — investigation, curation, PR fusionnée — qui rend le même incident instantané la fois suivante, la confiance suivant le taux de résolution constaté" >}}

Ce qui fait de ce cycle un véritable **apprentissage**, et pas un simple bloc-notes, c'est que **les résultats bouclent en retour**. RunLore tient un _outcome ledger_ : à chaque fois qu'une entrée est rappelée, il note si l'incident s'est réellement résolu ensuite. La confiance d'une entrée est **dérivée de son taux de résolution constaté**, pas affirmée par le modèle. Une entrée qui résout régulièrement gagne en confiance ; une entrée qui est rappelée mais dont l'incident ne se résout jamais **se dégrade** et finit par ne plus être proposée, jusqu'à ce qu'une nouvelle investigation la corrige. La mémoire reflète ainsi la réalité opérationnelle, et non un état figé à un instant T.

Ce calcul a une autre moitié, **humaine** : sous chaque notification (en option, sur Slack et Matrix), deux boutons **👍 / 👎** laissent l'astreinte trancher. Le clic n'est pas cosmétique — il entre dans le **même calcul de confiance** que les résolutions réelles, au même poids :

* **👍** → une observation positive : l'entrée **gagne en confiance**.
* **👎** → une observation négative : l'entrée **perd en confiance**, et un 👎 qui persiste **relance aussitôt une investigation** plutôt que de re-servir une réponse contestée.

Surtout, ces votes sont **la seule vérité terrain quand aucun signal de résolution n'arrive** : un échec GitOps ou une alerte sans `send_resolved` n'émettent jamais de « resolved » — sans retour humain, la confiance de ces entrées resterait figée. En un clic, l'humain pilote donc deux choses : **ce que l'agent croit**, et **quand il a le droit de se répéter**.

### OKF : d'une idée de Karpathy à un standard

Cette idée de mémoire n'a rien de neuf pour moi. Elle vient d'un pattern décrit par **Andrej Karpathy**, le _LLM-wiki_ : plutôt que d'entraîner ou de _fine-tuner_ un modèle, on lui donne une arborescence de fichiers markdown qu'il lit, complète et maintient **comme du code**. Le savoir vit en clair, versionné, relisible par un humain.

J'ai vécu ce pattern longtemps, à la main. D'abord dans **Obsidian**, mon coffre de notes personnel ; puis via **Tolaria**, l'outillage que j'ai construit par-dessus pour qu'un agent puisse lire et écrire dans ce coffre. Ça marchait — mais chaque base était un **format maison**, non interopérable : mes conventions, mon arborescence, mes scripts. Impossible de partager une base entre deux outils sans traduction.

C'est exactement ce vide que comble l'**OKF** (_Open Knowledge Format_), un standard publié par Google Cloud en juin 2026. Il formalise ce même pattern en quelques conventions partagées, si bien qu'un wiki produit par un outil devient lisible par n'importe quel autre.

{{% notice info "OKF, en deux mots" %}}
L'**OKF** représente la connaissance comme un répertoire de fichiers **markdown**, chacun décrivant **un concept** (un runbook, un incident, une métrique…) avec un frontmatter YAML dont le seul champ obligatoire est `type`. C'est la mise en standard du _LLM-wiki_ de Karpathy. Contrairement à une approche **RAG** (_Retrieval-Augmented Generation_) classique, il n'y a ni store propriétaire, ni schéma de compression, ni SDK imposé : du markdown versionné dans Git, lisible et modifiable autant par un humain que par un agent.
{{% /notice %}}

RunLore stocke donc sa mémoire au format **OKF-compatible**, dans un dépôt Git **que vous possédez** : indexé en BM25, relu par PR, avec une traçabilité complète. Et surtout, cette mémoire apprend **votre** contexte : vos contraintes, vos conventions, les spécificités de votre plateforme et de votre entreprise. Ce n'est pas un modèle générique, ce sont **vos** incidents, curés par **votre** équipe. Rien n'oblige d'ailleurs à démarrer d'une page blanche : le catalogue peut être **amorcé** dès le premier jour avec des connaissances _seeded_ (contraintes, architecture, conventions d'équipe), que les entrées _learned_ issues des incidents viennent ensuite enrichir.

Cette portabilité n'est pas théorique. Le binaire embarque un serveur **MCP** (_Model Context Protocol_) : `lore mcp` expose le catalogue en lecture seule (`kb_search`, `kb_get`, et le _what-changed_ GitOps) à n'importe quel client MCP — Claude Code, votre éditeur, ou même HolmesGPT. Sans cluster ni modèle : un clone du dépôt de connaissances suffit pour demander, en plein postmortem ou en revue d'une PR d'infra, « est-ce que ça nous a déjà mordus ? ».

```bash
git clone https://github.com/your-org/runlore-kb && lore mcp ./runlore-kb
```

## ✋ L'humain garde la main

La détection automatique de cause racine est un domaine ingrat : sur des incidents réalistes, même les meilleurs modèles identifient la cause racine **moins d'une fois sur deux** — c'est le constat du benchmark [ITBench](https://github.com/itbench-hub/ITBench) (IBM, ICML 2025). Sur un tel terrain, viser l'automatisation totale est un piège.

Le parti pris de RunLore est l'inverse : **l'agent n'est pas là pour remplacer l'astreinte, mais pour l'outiller — l'humain reste au point de décision**, aux deux endroits qui comptent : ce qui est *fait*, et ce qui est *appris*.

Encore faut-il que l'humain *comprenne* — sans quoi « valider » n'est qu'un clic. Tout est donc pensé pour être lu et jugé : des findings en **langage clair**, un savoir en **markdown** relu comme du code, des actions explicites marquées **réversibles** ou non.

* **Sur les actions — read-only par défaut.** La posture supportée est _read-only → suggest → approve_ : l'agent lit, corrèle et recommande ; un humain valide. Le palier `auto` (remédiation non supervisée) est expérimental, gelé, et déconseillé sur un cluster réel.
* **Sur la connaissance — rien n'est appris sans relecture.** C'est le point qui distingue vraiment : chaque finding jugé fiable et nouveau devient une **Pull Request** dans *votre* dépôt, qu'un humain relit et fusionne. L'agent propose ; l'équipe décide ce qui rejoint sa mémoire. _(D'autres agents apprennent aussi — mais enregistrent tout automatiquement, sans relecture.)_
* **Ce qu'il ne sait pas, il le délègue.** Un `unresolved` **assumé comme une réponse à part entière** : l'agent liste explicitement ce qui exige un accès ou un jugement qu'il n'a pas, au lieu de combler les trous par une supposition plausible.
* **Des signaux faits pour décider, pas pour rassurer.** Pour que la relecture humaine soit réelle et non un tampon, l'agent est calibré contre lui-même : un **score de confiance** sur chaque finding ; une passe de **vérification adverse** qui ne peut **que faire baisser** la confiance ; une mémoire **dérivée des résultats observés** et **plafonnée à 90 %**, jamais affirmée.
* **Et c'est vérifié, pas décrété.** Un **harnais d'évaluation** livré avec le projet — un _eval_ nocturne et une suite end-to-end sur k3d — rejoue des incidents connus et contrôle que rappel, vérification et décroissance se comportent comme annoncé.

Garder l'humain à la décision — sur ce qui est fait comme sur ce qui est appris — n'est pas une limite qu'on s'impose : c'est ce qui rend l'agent utilisable là où le raisonnement seul échoue une fois sur deux.

## 🔬 Les alternatives

RunLore n'arrive pas sur un terrain vierge : la **RCA** (_Root Cause Analysis_, l'identification automatisée de cause racine) est un domaine déjà bien occupé. Voici comment je le situe par rapport à l'existant :

| Outil | Ce que c'est | Ce que RunLore ajoute |
|---|---|---|
| [**k8sgpt**](https://github.com/k8sgpt-ai/k8sgpt) | Un _détecteur_ (analyzers + explication par LLM) | Une boucle d'investigation, la corrélation multi-signaux, de vrais diffs Git et l'apprentissage |
| [**HolmesGPT**](https://github.com/HolmesGPT/holmesgpt) | Le meilleur agent d'investigation open source | HolmesGPT s'appuie sur **vos** runbooks écrits à la main et n'apprend pas ; RunLore est _what-changed-first_ et s'améliore seul |
| [**OpenSRE**](https://github.com/swapnildahiphale/OpenSRE) | Le rival le plus proche : un agent multi-agents qui **apprend** réellement (mémoire épisodique + graphe de connaissances des incidents passés) | OpenSRE stocke chaque investigation **automatiquement, sans relecture** ; RunLore capitalise dans un catalogue **relu par PR, dont vous êtes propriétaire, et dont la confiance décroît selon les résultats réels** |
| [**kagent**](https://github.com/kagent-dev/kagent) | Un _framework_ d'agents in-cluster générique | Un agent SRE focalisé et opinionated (RunLore pourrait tourner _sur_ kagent) |
| **Komodor · Anyshift** (commerciaux) | RCA orientée changement, propriétaire | Le même signal, mais alimentant un catalogue **ouvert et portable** |

Soyons lucides : la **RCA orientée changement n'a rien d'unique** — des outils commerciaux calculent des diffs de changements depuis longtemps. Le vrai espace que RunLore occupe, c'est la **combinaison** que les outils ouverts n'ont pas : ce signal alimentant un **catalogue ouvert, portable et relu**, opéré par un agent qui **garde l'humain à la décision**.

## 🛠️ En pratique : de l'installation à un incident réel

Pour tester RunLore en conditions réalistes, je m'appuie sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), mon dépôt de référence : une plateforme complète sur **EKS** combinant Cilium, VictoriaMetrics, Crossplane et Flux. C'est exactement le _golden path_ de RunLore.

### 🚀 Déployer RunLore

L'installation tient en un chart Helm et un `values.yaml`. Quatre prérequis : au moins une **source de données** (ici Flux et VictoriaMetrics), un **LLM**, un **dépôt GitHub privé** pour la base de connaissances (avec une GitHub App dédiée), et une **destination de notification**. Les credentials vont dans un `Secret` Kubernetes ; le `values.yaml` fait le câblage :

```yaml
# values.yaml (extrait) — le golden path : Flux + VictoriaMetrics + Slack + GitHub
config:
  gitops:
    engine: flux                  # ou "argocd"
  sources:
    alertmanager: {}              # active le webhook Alertmanager/VMAlert
  model:
    provider: anthropic
    model: claude-sonnet-5
    api_key_env: ANTHROPIC_API_KEY
  metrics:
    url: http://vmsingle.observability.svc:8429
  notify:
    slack:
      webhook_url_env: SLACK_WEBHOOK_URL
  forge:
    kb_repo: your-org/runlore-kb  # le dépôt qui recevra les PRs de curation
```

```bash
helm install runlore deploy/helm/runlore -n runlore --create-namespace -f values.yaml
```

Il ne reste qu'à router les alertes d'Alertmanager vers `http://runlore.runlore.svc:8080/webhook/alertmanager`, et les investigations démarrent. Je m'en tiens volontairement à l'essentiel : le [guide de démarrage complet](https://github.com/Smana/runlore/blob/main/docs/getting-started.md) couvre la création de la GitHub App, la référence exhaustive du `values.yaml` et les étapes de vérification.

{{% notice tip "Démarrer sans faire flamber la facture de tokens 💸" %}}
Un agent branché sur Alertmanager peut vite déclencher beaucoup d'investigations — donc beaucoup d'appels LLM. RunLore expose plusieurs garde-fous pour cadrer le coût dès le départ :

* **`triggers.incidents.debounce`** — retient une alerte quelques instants avant d'investiguer, et l'ignore si elle se résout d'elle-même dans l'intervalle (le bruit auto-résolutif ne coûte rien).
* **`triggers.incidents.dedup.window`** — ne relance pas une alerte déjà en cours d'investigation.
* **`investigation.coalesce`** — replie une _tempête_ d'alertes corrélées en une seule investigation.
* **`investigation.rate_limit`** — plafonne le nombre d'investigations par fenêtre de temps (`max_per_window` sur une `window`, ex. 3 par heure).
* **`investigation.max_tokens_per_investigation`** — un budget de tokens strict par investigation, qui coupe court à toute dérive.
* **`model.pricing`** — renseignez les tarifs (USD par million de tokens) et chaque notification affiche le **coût estimé** de l'investigation ; la métrique `investigation_cost_usd` permet de le suivre dans le temps.

Commencez volontairement bas (rate limit serré, budget modeste), observez, puis desserrez.
{{% /notice %}}

### 🔍 L'incident : `HarborRegistryDown`

Voici un incident réellement investigué. Une alerte **`HarborRegistryDown`** se déclenche. RunLore investigue et corrèle plusieurs signaux :

* le **statut du pod** : `harbor-registry` échoue en `CreateContainerConfigError` — `couldn't find key username in Secret tooling/xplane-harbor-access-key` ;
* les **événements Kubernetes** du namespace `tooling` : un Warning persistant sur la ressource Crossplane `AccessKey/xplane-harbor` — `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2` ;
* le **lien de cause à effet** : c'est cette ressource Crossplane en échec qui doit créer le Secret consommé par le pod Harbor.

L'agent identifie alors la cause racine avec une **forte confiance** : la ressource Crossplane a atteint le quota AWS IAM `AccessKeysPerUser: 2`, ce qui l'empêche de créer les credentials du registre. Il publie le tout dans Slack, propose une remédiation (supprimer une clé d'accès inutilisée, marquée `reversible=false`), et — c'est important — **liste ce qu'il ne sait pas** (les _data gaps_).

{{< img src="runlore-investigation.png" alt="Notification Slack d'une investigation complète RunLore : un verdict « Action required », la cause racine (quota IAM), la remédiation suggérée, les hypothèses écartées, les lacunes de données, et le coût en tokens" width="760" caption="La notification _verdict-first_ d'une investigation complète : verdict d'actionnabilité, cause racine, preuves, hypothèses écartées, lacunes de données — et, en pied, le coût réel (7 appels modèle, ~58k tokens)" >}}

Une fois l'incident relu et la PR fusionnée, voici l'entrée **OKF** qui rejoint le catalogue :

```markdown
# fichier : harbor-registry-down-due-to-iam-access-key-quota-limit.md
---
type: Incident
title: Harbor Registry Down due to IAM Access Key Quota Limit
resource: tooling/harbor-registry
tags: [runlore, incident]
fingerprint: 2d6bd8279304b3e17a5d5e35a55fb0c115ffbeabde820af8cdd2494a4141a60b
---

## Decision
- **why keep:** The Crossplane resource `AccessKey/xplane-harbor` has hit an AWS
  IAM quota limit (`AccessKeysPerUser: 2`), preventing it from creating the
  credentials required by the Harbor registry.
- **confidence:** 95%

## Investigate
- pod_status shows harbor-registry failing with 'CreateContainerConfigError:
  couldn't find key username in Secret tooling/xplane-harbor-access-key'
- kube_events shows a persistent Warning on 'AccessKey/xplane-harbor':
  'LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2'
- The knowledge base article 'HarborRegistryDown' describes this exact scenario.

## Resolution
- An administrator should delete an old or unused access key, then reconcile
  the `AccessKey/xplane-harbor` resource. (reversible=false)

## Unresolved
- The name of the specific IAM user that has reached its quota.
- Which of the two existing access keys is safe to delete.
```

La section `Unresolved` illustre au passage ce principe : l'agent délègue à l'humain ce qui exige un accès ou un jugement qu'il n'a pas — ici, *quelle* clé d'accès supprimer parmi les deux.

### ⚡ La récurrence : une réponse instantanée

C'est à la **deuxième occurrence** que la boucle d'apprentissage paie. Quelque temps plus tard, le pod `harbor-registry` retombe — mais l'alerte qui se déclenche est cette fois **générique** : un simple `KubePodNotReady`, dont le texte ne mentionne ni le quota IAM, ni Crossplane, ni même Harbor autrement que par le nom du pod. C'est le cas difficile : quasiment **aucun recouvrement lexical** entre l'alerte et le runbook qui la couvre.

RunLore ne relance pourtant **aucune investigation**. Il reconnaît l'incident, remonte la réponse déjà curée depuis le catalogue et la publie **instantanément** — sans nouvel appel de raisonnement, sans nouvelle PR :

{{< img src="runlore-recall.png" alt="Notification Slack d'un rappel instantané RunLore : un bandeau « réponse instantanée depuis la base de connaissances », la cause et la résolution déjà validées, et un taux de résolution issu de l'outcome ledger" width="760" caption="Le rappel instantané : une alerte générique `KubePodNotReady` retrouve l'incident Harbor connu et remonte la réponse curée, sans ré-investigation" >}}

La notification est explicite : **⚡ réponse instantanée depuis la base de connaissances**, la cause et la résolution déjà validées, et un **taux de résolution** issu de l'_outcome ledger_ — le signal qui rend cette réponse en cache digne de confiance.

Le contraste est net — les deux pieds de notification le chiffrent. La première occurrence a coûté une **investigation complète** : une quinzaine d'outils interrogés, **7 appels modèle, ~58 000 tokens**. La récurrence ne coûte plus que **2 appels légers** — le _reranking_ qui reconnaît l'entrée, puis une brève passe de vérification, soit **~3 700 tokens** — et se résout en quelques secondes. Un ordre de grandeur de moins, pour la même réponse. C'est le quatrième temps de la boucle — **Compound** — rendu concret. Et parce que la confiance est **dérivée du taux de résolution constaté**, une entrée qui cesserait de résoudre ses incidents verrait sa confiance se dégrader et **redéclencherait une investigation fraîche** : la mémoire reste arrimée à la réalité opérationnelle.

{{% notice info "Comment une alerte générique retrouve-t-elle le bon runbook ? 🎯" %}}
`KubePodNotReady` n'a presque rien en commun, lexicalement, avec un runbook intitulé « Harbor Registry Down — IAM quota ». Le rappel ne se fie donc pas à la seule recherche **BM25** : un **pré-filtre structurel** (la ressource affectée — ici `tooling/harbor-registry`) réduit d'abord les candidats, puis un **reranker LLM** tranche, sur une confiance **calibrée** (indépendante du corpus), si le runbook candidat couvre bien *cette* ressource et *ce* symptôme. La cause précise, elle, est **re-confirmée contre l'état live du cluster** après le match — jamais supposée.
{{% /notice %}}

### 📊 Observer l'agent

Un agent qui investigue vos incidents doit lui-même être **observable**. RunLore expose des **métriques Prometheus** et livre un **dashboard Grafana** avec le chart — organisé **autour de la boucle d'apprentissage**, pas seulement de la santé technique. Sa première rangée répond à une seule question, « _la boucle fonctionne-t-elle ?_ » :

* **Fire-rate & precision du rappel** — à quelle fréquence il se déclenche, et s'il vise juste.
* **Taux de résolution** — le signal clé : les réponses rappelées résolvent-elles vraiment l'incident ?
* **Tokens économisés par le rappel** — le gain concret face à une investigation complète.
* **Entrées KB invalides** — la connaissance qui se dégrade, à surveiller.

Suivent des vues par étape (_Retrieve_, _Capture_, _Curate_), le **coût par investigation** (`investigation_cost_usd`, déjà croisé plus haut) et les classiques : durées, erreurs outils/modèle, intake d'alertes, HA.

{{< img src="runlore-dashboard.png" alt="Dashboard Grafana de RunLore organisé autour de la boucle d'apprentissage : fire-rate, taux de résolution du rappel, tokens économisés, coût par investigation" width="1080" caption="Le dashboard livré avec le chart : les SLI de la boucle d'apprentissage en tête — rappel, taux de résolution, tokens économisés, coût" >}}

C'est aussi l'instrument qui permettra de trancher, sur la durée, la question posée en clôture : cette mémoire paie-t-elle réellement ?

## 🧑‍💻 Construit avec des agents, en toute transparence

Je dois être transparent sur un point : **RunLore lui-même a été construit avec mes outils d'agentic coding** — Claude Code, des _skills_, le framework _superpowers_, et des revues multi-agents. C'est un prolongement direct de ce que j'explore dans la série [Agentic AI](/fr/series/agentic-ai/), notamment dans [l'article sur le coding agent](/fr/post/series/agentic_ai/ai-coding-agent/) et celui sur les [astuces et workflows](/fr/post/series/agentic_ai/ai-coding-tips/).

Il y a une forme de cohérence dans tout ça : la **discipline de vérification adverse** que j'utilise pour développer le projet (faire critiquer chaque finding par des agents indépendants avant de l'accepter) est exactement la philosophie embarquée dans le produit. Et puisque RunLore accepte n'importe quel endpoint OpenAI-compatible, il peut tout à fait tourner sur une stack auto-hébergée comme celle décrite dans l'article sur le [LLM self-hosted](/fr/post/series/agentic_ai/llm-self-hosted-stack/) — vLLM, Ollama, et la confidentialité qui va avec.

## 💭 Dernières remarques

Parlons franchement, parce que c'est tout l'esprit du projet : **RunLore est très jeune**. Il a quelques semaines d'existence, il est **pre-1.0**, et ses interfaces bougent encore. Le _golden path_ testé en continu (Flux ou Argo CD + VictoriaMetrics + un modèle Anthropic ou OpenAI-compatible + Slack + GitHub) est solide ; le reste — Matrix, Gemini, la source PagerDuty, les intégrations cloud, le réseau Hubble — fonctionne mais a vu moins de kilomètres.

Un point qu'il faut poser clairement : **la qualité des investigations dépend fortement de deux facteurs**. D'abord des **sources de données** branchées — sans métriques, sans logs, sans historique GitOps, l'agent raisonne à l'aveugle ; plus vous lui donnez de signaux corrélables, plus ses conclusions sont solides. Ensuite, et surtout, du **modèle** utilisé. C'est le facteur déterminant : un modèle de raisonnement récent et capable produit des causes racines d'un tout autre niveau qu'un petit modèle local. RunLore accepte n'importe quel endpoint, mais on n'obtient pas la même chose d'un modèle « pro » de dernière génération que d'un modèle contraint tournant sur un laptop.

Surtout, **la partie la plus prometteuse est aussi la moins éprouvée**. La valeur d'une mémoire qui se capitalise ne se mesure pas en une démo. Tout le mécanisme de confiance et de décroissance repose sur une hypothèse : que le taux de résolution constaté d'une entrée soit un bon signal de sa fiabilité. Le vérifier demande des **semaines d'usage réel** — voir comment le catalogue se remplit, si le rappel se déclenche au bon moment, si la confiance suit vraiment la réalité, et si la connaissance se compose. Je sais déjà qu'il y a des aspérités : par exemple, un même symptôme avec une cause différente peut s'ancrer à tort sur une investigation passée. C'est précisément le genre de comportement que seul le temps long révèle.

Je reviendrai donc avec un **retour d'expérience sur la durée**. D'ici là, j'accueille avec plaisir vos retours et vos issues.

{{% notice warning "Points d'attention pour la prod ⚠️" %}}
La posture supportée est **read-only → suggest → approve** : RunLore lit, corrèle et recommande, un humain relit et fusionne. Le palier d'autonomie `auto` (remédiation automatique sans supervision) est **expérimental, gelé, et déconseillé sur un cluster réel**. Restez sur le _golden path_, avec un humain dans la boucle de validation.
{{% /notice %}}

## 🔖 Références

* [RunLore — dépôt GitHub](https://github.com/Smana/runlore) · [guide de démarrage](https://github.com/Smana/runlore/blob/main/docs/getting-started.md)
* [Open Knowledge Format (OKF) — Knowledge Catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog)
* [Open Knowledge Format — annonce Google Cloud](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing)
* [Le pattern _LLM-wiki_ d'Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
* [ITBench — benchmark d'agents SRE (IBM)](https://github.com/itbench-hub/ITBench)
* [cloud-native-ref — plateforme de référence](https://github.com/Smana/cloud-native-ref)
* [k8sgpt](https://github.com/k8sgpt-ai/k8sgpt) · [HolmesGPT](https://github.com/HolmesGPT/holmesgpt) · [kagent](https://github.com/kagent-dev/kagent)
