+++
author = "Smaine Kahlouch"
title = "`RunLore` : ton buddy SRE qui investigue les incidents — et apprend de chaque résolution"
date = "2026-07-04"
summary = "Souvent, la connaissance d'un incident se dissout dès qu'il est clos. RunLore, agent SRE open source, investigue et t'aide à identifier rapidement la root cause, puis transforme chaque résolution en savoir réutilisable."
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

J'ai décidé de m'attaquer à cette problématique en créant un projet open source il y a quelques semaines : [`RunLore`](https://github.com/Smana/runlore). Au départ, l'idée était simple : un **binôme** (_buddy_) qui prend en charge la phase de collecte pour **orienter rapidement l'investigation vers la root cause la plus probable**. Et qui, au fil du temps, **apprend le contexte dans lequel nous travaillons ensemble** : notre plateforme, nos contraintes, nos règles et standards.

Pour conserver la connaissance en mémoire, j'ai choisi l'**OKF** (_Open Knowledge Format_) : un format pensé précisément pour qu'un agent IA lise et écrive sa connaissance efficacement. J'y reviens [en détail plus loin](#okf--dune-idée-de-karpathy-à-un-standard).

## 🎯 Objectifs

* 🤖 Découvrir **RunLore** et son fonctionnement
* 🧠 Ses **trois choix de conception** : un binaire simple, la **boucle d'apprentissage**, et l'**humain à la décision**
* 🛠️ Le **déployer** simplement avec Helm et le voir investiguer un **incident réel** sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)

## 🔥 Pourquoi ce projet : le coût caché de l'investigation

L'article précédent sur les alertes se terminait sur une évidence : la configuration technique ne suffit pas, il faut aussi des **runbooks et des procédures de réponse aux incidents**. Or, dans la vraie vie, ces runbooks sont incomplets, et une grande partie de la connaissance ne vit que dans la tête de quelques personnes.

Quand une alerte se déclenche, le SRE rejoue à chaque fois le même enchaînement manuel :

* **Qu'est-ce qui a changé ?** Un déploiement, une montée de version, une modification d'infrastructure, un certificat expiré ?
* **Qu'est-ce qui ne va pas ?** Saturation, réseau, nœuds, dépendances en échec ?
* **A-t-on déjà vu ça ?** La réponse est souvent « oui », mais personne ne retrouve où elle est documentée.

Ce pivot permanent entre Git, les métriques, les logs et les flux réseau est exactement le genre de tâche qu'un agent peut prendre en charge. **Plusieurs le font déjà** — mais l'enjeu, pour moi, est ailleurs : **garder l'humain au cœur des décisions**, sur ce que l'agent _fait_ comme sur ce qu'il _apprend_.

## 🤖 RunLore en bref

{{< img src="runlore-logo.png" width="120" >}}

**RunLore** est un **agent SRE** open source (licence Apache-2.0) qui s'exécute dans ton cluster Kubernetes sous la forme d'un binaire Go unique, déployé via Helm. </br>
Son principe est volontairement simple : à partir d'un événement (une alerte, un échec **GitOps** — la livraison continue pilotée par Git…), il investigue et répond à deux questions — _what changed?_ et _what's wrong?_ — puis publie dans ta messagerie une **root cause** assortie d'un **score de confiance**, des preuves qui l'étayent, et des questions ouvertes à destination d'un humain.

{{< img src="runlore-investigation-flow.png" alt="Flux d'investigation de RunLore : un événement déclenche une investigation qui corrèle l'historique GitOps, les métriques, les logs et les flux réseau ; la root cause est publiée dans Slack et une PR est ouverte dans la base de connaissances" width="1080" >}}

Le schéma se lit en trois temps :

* **Déclencheurs** — une alerte, un échec GitOps ou un webhook lancent l'investigation.
* **Sources de données** — l'**historique GitOps** (le fil rouge du _what-changed_), les métriques, les logs, les flux réseau, le cloud… _plus il y en a de branchées, plus la réponse est solide_.
* **Canaux de notification** — le verdict part vers Slack, Matrix…

Et tout ça repose sur une **base de connaissances** qui s'enrichit à chaque incident et **évolue avec le temps**.

### Trois choix de conception

Des agents qui investiguent un incident, il en existe déjà (nous y reviendrons). Ce qui distingue RunLore tient en trois choix — et c'est leur **combinaison** qui compte :

1. 📦 **Un simple binaire, chez toi, sur tes modèles.** Un unique binaire Go dans ton cluster ; tu gardes la main sur tes données et tes modèles — et avec un LLM auto-hébergé, rien ne sort de ton périmètre. Pas de SaaS, pas de _lock-in_.
2. 🧠 **Une mémoire qui s'enrichit.** Chaque investigation nourrit un **catalogue de connaissances que tu possèdes**, au standard **OKF**. Le même incident, la fois suivante, obtient une réponse instantanée plutôt qu'une nouvelle investigation.
3. ✋ **L'humain garde la main.** L'agent lit et recommande, il n'agit jamais seul (_read-only → suggest → approve_) — et surtout, **rien n'entre dans sa mémoire sans une PR relue par un humain**. Ce qu'il ne sait pas, il le délègue plutôt que de le deviner.

Les deux sections qui suivent creusent les deux choix les moins évidents : la mémoire, puis la place gardée à l'humain.

## 🧠 Une mémoire qui s'enrichit

La boucle autonome _alerte → root cause → chat_ est aujourd'hui **banalisée** : plusieurs outils savent le faire. Ce qui l'est beaucoup moins, c'est une base de connaissances qui **se consolide dans un catalogue que tu maîtrises**.

### La boucle d'apprentissage

RunLore implémente un cycle en quatre temps :

1. **Retrieve** — face à un incident, il commence par chercher une réponse passée _digne de confiance_ dans la base de connaissances.
2. **Capture** — si rien ne correspond, il investigue à chaud, enregistre ce qu'il a trouvé, puis **observe la suite** : l'incident s'est-il effectivement résolu ?
3. **Curate** — un finding suffisamment fiable et nouveau devient une **Pull Request** rédigée dans ton dépôt de connaissances. C'est le **point de contrôle qualité** : on relit une entrée comme on relit du code.
4. **Compound** — une fois la PR fusionnée, l'entrée est ré-indexée et devient immédiatement disponible. Le même incident, la prochaine fois, obtient une réponse instantanée — sans ré-investigation.

Tout part donc de **Retrieve**, qui tranche entre deux issues : servir une réponse connue **instantanément**, ou — faute de correspondance fiable — lancer la boucle complète, celle qui nourrit le catalogue.

{{< img src="runlore-learning-loop.png" width="1080" >}}
</br>

Ce qui fait de ce cycle un vrai **apprentissage** — pas un simple bloc-notes — c'est que **les résultats bouclent en retour**. RunLore tient un _outcome ledger_ : à chaque rappel d'une entrée, il note si l'incident s'est **réellement résolu** ensuite. La confiance est donc **dérivée du taux de résolution constaté**, pas affirmée par le modèle :

* une entrée qui **résout régulièrement** → elle **gagne en confiance** ;
* une entrée **rappelée sans que l'incident se résolve jamais** → elle **se dégrade** et cesse d'être proposée (jusqu'à ce qu'une nouvelle investigation la corrige).

La mémoire reste ainsi **arrimée à la réalité opérationnelle**, jamais figée à un instant T.

Ce même calcul a une **moitié humaine** : sous chaque notification (en option, sur Slack et Matrix), deux boutons **👍 / 👎**. Le clic n'est pas cosmétique — en un geste, le SRE pilote **deux leviers** :

* **La qualité du diagnostic** — un 👍 le confirme, un 👎 le déclare **erroné** ; le vote nourrit la **confiance** de l'entrée, pas son contenu. Et c'est parfois le **seul** juge : un échec GitOps ou une alerte sans `send_resolved` ne signalent jamais leur résolution.
* **Quand l'agent a le droit de se répéter** — un 👎 qui persiste **relance aussitôt une investigation** plutôt que de re-servir une réponse contestée.

### OKF : d'une idée de Karpathy à un standard

Récemment, **Google Cloud** a publié un **standard portable** pour ce qu'on appelle le _LLM-wiki_ : l'[**OKF**](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing) (_Open Knowledge Format_, juin 2026). Mais le concept, lui, n'a rien de neuf.

Il vient d'une réflexion d'**Andrej Karpathy** : plutôt que d'entraîner ou de _fine-tuner_ un modèle, on lui donne une arborescence de fichiers **markdown** qu'il lit, complète et maintient **comme du code** — un format pensé pour être **lu efficacement par l'IA**, tout en restant en clair, versionné et relisible par un humain.

J'utilisais déjà ce pattern **en local** : d'abord dans **Obsidian**, puis via **Tolaria**, l'outillage que j'ai bâti par-dessus pour qu'un agent lise et écrive dans ma base de connaissances. Il m'a donc paru naturel d'en faire la mémoire de RunLore.

{{% notice info "OKF, en deux mots" %}}
L'**OKF** représente la connaissance comme un répertoire de fichiers **markdown**, chacun décrivant **un concept** (un runbook, un incident, une métrique…) avec un frontmatter YAML dont le seul champ obligatoire est `type`. C'est la mise en standard du _LLM-wiki_ de Karpathy. Contrairement à une approche **RAG** (_Retrieval-Augmented Generation_) classique, il n'y a ni store propriétaire, ni schéma de compression, ni SDK imposé : du markdown versionné dans Git, lisible et modifiable autant par un humain que par un agent.
{{% /notice %}}

RunLore stocke donc sa mémoire au format **OKF-compatible**, dans un dépôt Git **que tu possèdes** : indexé en BM25, relu par PR, avec une traçabilité complète. Et rien n'oblige à démarrer d'une page blanche : le catalogue peut être **amorcé** dès le premier jour (contraintes, architecture, conventions d'équipe), que les incidents viennent ensuite enrichir.

## ✋ L'humain garde la main

RunLore a été construit avec un **parti pris assumé** : garder l'humain au cœur des décisions. Les réponses d'un LLM restent parfois discutables — viser l'automatisation totale peut être un piège. Ici, le SRE reste **totalement maître** :

* **de l'évolution de la base de connaissances** — rien n'y entre sans son aval : chaque finding fiable devient une **Pull Request** qu'il relit et fusionne. L'agent propose ; l'équipe décide ce qui rejoint sa mémoire.
* **du jugement de la qualité des diagnostics** — c'est lui qui confirme ou infirme (👍 / 👎), et l'agent est calibré pour ne jamais survendre : un `unresolved` assumé quand il ne sait pas, une confiance qui ne peut que **baisser** à la vérification.
* **de l'alimentation en connaissance métier** — il amorce et enrichit le catalogue avec _son_ contexte : règles d'entreprise, conventions d'équipe, habitudes, spécificités de la plateforme.

Et l'agent n'agit jamais seul : la posture est _read-only → suggest → approve_ — il lit, corrèle, recommande ; un humain valide.

Garder l'humain à la décision — sur ce qui est fait comme sur ce qui est appris — n'est pas une limite qu'on s'impose : c'est ce qui rend l'agent réellement utilisable.

## 👀 Voici ce que ça donne

Assez décrit — voici RunLore **à l'œuvre**. L'incident ci-dessous a été réellement investigué sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), mon dépôt de référence : une plateforme complète sur **EKS** combinant Cilium, VictoriaMetrics, Crossplane et Flux.

{{% notice info "Le modèle derrière la démo 🧠" %}}
Toute la démo tourne sur **GLM 5.2** (Zhipu AI, via l'API OpenAI-compatible de Z.ai) — une qualité **très proche des modèles frontière** (Claude, GPT) pour un **coût par token nettement inférieur**, décisif quand un agent branché sur Alertmanager peut lancer beaucoup d'investigations. RunLore acceptant n'importe quel endpoint OpenAI-compatible, **en changer tient en une ligne** — jusqu'à une [stack LLM auto-hébergée](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (vLLM, Ollama) si tu veux tout garder dans ton périmètre.
{{% /notice %}}

### 🔍 L'incident : `HarborRegistryDown`

Une alerte **`HarborRegistryDown`** se déclenche. RunLore investigue et corrèle plusieurs signaux :

* le **statut du pod** : `harbor-registry` échoue en `CreateContainerConfigError` — `couldn't find key username in Secret tooling/xplane-harbor-access-key` ;
* les **événements Kubernetes** du namespace `tooling` : un Warning persistant sur la ressource Crossplane `AccessKey/xplane-harbor` — `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2` ;
* le **lien de cause à effet** : c'est cette ressource Crossplane en échec qui doit créer le Secret consommé par le pod Harbor.

L'agent identifie alors la root cause avec une **forte confiance** : la ressource Crossplane a atteint le quota AWS IAM `AccessKeysPerUser: 2`, ce qui l'empêche de créer les credentials du registre. Il publie le tout dans Slack, propose une remédiation (supprimer une clé d'accès inutilisée, marquée `reversible=false`), et — c'est important — **liste ce qu'il ne sait pas** (les _data gaps_).

La notification est _verdict-first_ : elle s'ouvre sur un **verdict d'actionnabilité** clair — _aucune action_, _action suggérée_, _action requise_ ou _non concluant_ — avant même les détails. Le SRE sait en un coup d'œil s'il doit intervenir.

{{< img src="runlore-investigation.png" alt="Notification Slack d'une investigation complète RunLore : un verdict « Action required », la root cause (quota IAM), la remédiation suggérée, les hypothèses écartées, les lacunes de données, et le coût en tokens" width="760" caption="La notification _verdict-first_ d'une investigation complète : verdict d'actionnabilité, root cause, preuves, hypothèses écartées, lacunes de données — et, en pied, le coût réel (7 appels modèle, ~58k tokens)" >}}

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

## 🔬 Les alternatives

RunLore n'arrive pas sur un terrain vierge : la **RCA** (_Root Cause Analysis_, trouver automatiquement la **root cause**) est un domaine déjà bien occupé. Voici comment je le situe par rapport à l'existant :

| Outil | Ce que c'est | Ce que RunLore ajoute |
|---|---|---|
| [**k8sgpt**](https://github.com/k8sgpt-ai/k8sgpt) | Un _détecteur_ (analyzers + explication par LLM) | Une boucle d'investigation, la corrélation multi-signaux, de vrais diffs Git et l'apprentissage |
| [**HolmesGPT**](https://github.com/HolmesGPT/holmesgpt) | Le meilleur agent d'investigation open source | HolmesGPT s'appuie sur **tes** runbooks écrits à la main et n'apprend pas ; RunLore est _what-changed-first_ et s'améliore seul |
| [**OpenSRE**](https://github.com/swapnildahiphale/OpenSRE) | Le rival le plus proche : un agent multi-agents qui **apprend** réellement (mémoire épisodique + graphe de connaissances des incidents passés) | OpenSRE stocke chaque investigation **automatiquement, sans relecture** ; RunLore construit un catalogue **relu par PR, dont tu es propriétaire, et dont la confiance décroît selon les résultats réels** |
| [**kagent**](https://github.com/kagent-dev/kagent) | Un _framework_ d'agents in-cluster générique | Un agent SRE focalisé et opinionated (RunLore pourrait tourner _sur_ kagent) |

Clairement la **RCA orientée changement n'est pas nouveau** — des outils commerciaux calculent des diffs de changements depuis longtemps. Le vrai espace que RunLore occupe, c'est la **combinaison** que les outils ouverts n'ont pas : ce signal alimentant un **catalogue ouvert, portable et relu**, opéré par un agent qui **garde l'humain à la décision**.

## 🛠️ Tu peux le tester simplement

Envie de l'essayer ? L'installation tient en un **chart Helm** et un `values.yaml`. Il te faut au moins une **source de données**, un **LLM**, un **dépôt GitHub privé** pour la base de connaissances (avec une GitHub App dédiée) et une **destination de notification**. Les credentials vont dans un `Secret` Kubernetes ; le `values.yaml` fait le câblage.

```yaml
# values.yaml (extrait) — un exemple standard : Argo CD + Prometheus + VictoriaLogs + Slack
config:
  gitops:
    engine: argocd                   # ou "flux"
  sources:
    alertmanager: {}                 # webhook Alertmanager
    gitops:
      enabled: true                  # réagit aussi aux échecs Argo CD
  model:                             # n'importe quel endpoint OpenAI-compatible
    provider: openai
    model: glm-5.2
    base_url: https://api.z.ai/api/paas/v4/
    api_key_env: GLM_API_KEY
  metrics:
    url: http://kube-prometheus-stack-prometheus.monitoring.svc:9090   # Prometheus
  logs:
    url: http://victoria-logs-single-server.observability.svc:9428     # VictoriaLogs
  cloud:
    provider: aws
    region: eu-west-3
  notify:
    slack:                           # bot token : détail threadé + boutons 👍/👎
      bot_token_env: SLACK_BOT_TOKEN
      signing_secret_env: SLACK_SIGNING_SECRET
      channel: "#alerts"
      feedback_buttons: true
  forge:
    kb_repo: your-org/runlore-kb     # le dépôt qui recevra les PRs de curation
```

```bash
helm install runlore deploy/helm/runlore -n runlore --create-namespace -f values.yaml
```

Il ne reste qu'à router les alertes d'Alertmanager vers `http://runlore.runlore.svc:8080/webhook/alertmanager`, et les investigations démarrent. Je m'en tiens volontairement à l'essentiel : le [guide de démarrage complet](https://github.com/Smana/runlore/blob/main/docs/getting-started.md) couvre la création de la GitHub App, la référence exhaustive du `values.yaml` et les étapes de vérification.

{{% notice info "Et la sécurité ? 🔒" %}}
Brancher un agent LLM sur son cluster soulève une question immédiate : que voit-il, et où cela part-il ? RunLore y répond par construction :

* **Redaction des secrets à trois frontières de confiance** : le texte de l'incident, chaque sortie d'outil avant d'atteindre le modèle, et l'investigation finale avant toute publication (PR ou chat) ;
* **RBAC minimal** : la lecture des logs de pods est restreinte à une liste de namespaces (par défaut `flux-system` uniquement) ;
* **Journal d'audit** en append-only, chaîné par hash : chaque décision est traçable et infalsifiable.
{{% /notice %}}

**Côté intégrations** : l'exemple ci-dessus est un stack standard, mais tu branches ce que tu veux — **GitOps** (Flux/Argo CD), **métriques**, **logs**, **flux réseau**, **cloud**, plusieurs **LLM** et **notifieurs**, chacun _pluggable_. La **matrice complète**, qui évolue au fil des versions, vit dans le [README du dépôt](https://github.com/Smana/runlore#-supported-integrations).

{{% notice tip "Quelques conseils pour bien démarrer 💸" %}}
Un agent branché sur Alertmanager peut vite multiplier les appels LLM. Tu gardes la main : commence **volontairement bas** — par exemple **2 investigations/heure** — observe, puis desserre. Les garde-fous :

* **`investigation.rate_limit`** — le plafond d'investigations par fenêtre (commence à 2/heure) ;
* **`triggers.incidents.debounce`** + **`dedup.window`** — ignorent le bruit auto-résolutif et les alertes déjà en cours ;
* **`investigation.coalesce`** — replie une _tempête_ d'alertes corrélées en une seule investigation ;
* **`investigation.max_tokens_per_investigation`** — un budget de tokens strict ;
* **`model.pricing`** — affiche le **coût estimé** sur chaque notification (métrique `investigation_cost_usd`).
{{% /notice %}}

## 📊 Observer l'agent

Un agent qui investigue tes incidents doit lui-même être **observable**. RunLore expose des **métriques Prometheus** et livre un **dashboard Grafana** avec le chart — organisé **autour de la boucle d'apprentissage**, pas seulement de la santé technique. Sa première rangée répond à une seule question, « _la boucle fonctionne-t-elle ?_ » :

* **Fire-rate & precision du rappel** — à quelle fréquence il se déclenche, et s'il vise juste.
* **Taux de résolution** — le signal clé : les réponses rappelées résolvent-elles vraiment l'incident ?
* **Tokens économisés par le rappel** — le gain concret face à une investigation complète.
* **Entrées KB invalides** — la connaissance qui se dégrade, à surveiller.

Suivent des vues par étape (_Retrieve_, _Capture_, _Curate_), le **coût par investigation** (`investigation_cost_usd`, déjà croisé plus haut) et les classiques : durées, erreurs outils/modèle, intake d'alertes, HA.

{{< img src="runlore-dashboard.png" alt="Dashboard Grafana de RunLore organisé autour de la boucle d'apprentissage : fire-rate et taux de résolution du rappel, tokens économisés, entrées KB invalides" width="1080" caption="Le dashboard livré avec le chart : la boucle d'apprentissage en tête — fire-rate et taux de résolution du rappel, tokens économisés, entrées KB invalides" >}}

C'est aussi l'instrument qui permettra de trancher, sur la durée, la question posée en clôture : cette mémoire paie-t-elle réellement ?

## 🧑‍💻 Construit avec des agents, en toute transparence

Je dois être transparent sur un point : **RunLore lui-même a été construit avec mes outils d'agentic coding** — Claude Code, des _skills_, le framework _superpowers_, et des revues multi-agents. C'est un prolongement direct de ce que j'explore dans la série [Agentic AI](/fr/series/agentic-ai/), notamment dans [l'article sur le coding agent](/fr/post/series/agentic_ai/ai-coding-agent/) et celui sur les [astuces et workflows](/fr/post/series/agentic_ai/ai-coding-tips/).

Il y a une forme de cohérence dans tout ça : la **discipline de vérification adverse** que j'utilise pour développer le projet (faire critiquer chaque finding par des agents indépendants avant de l'accepter) est exactement la philosophie embarquée dans le produit. Et puisque RunLore accepte n'importe quel endpoint OpenAI-compatible, il peut tout à fait tourner sur une stack auto-hébergée comme celle décrite dans l'article sur le [LLM self-hosted](/fr/post/series/agentic_ai/llm-self-hosted-stack/) — vLLM, Ollama, et la confidentialité qui va avec.

## 💭 Dernières remarques

Bon, je suis conscient que le projet est **très jeune**, et comme j'en suis le **seul contributeur** pour l'instant, ça peut clairement en rebuter certains. Mais ce n'est pas un truc « vibe codé » à l'arrache : j'ai soigné la **qualité**, les **performances** et la **sécurité**, et je l'ai **testé pour de vrai**, à plusieurs reprises, sur 2 environnements. Je t'invite vraiment à l'explorer — voici les leçons que j'en tire.

* **Les diagnostics sont, à l'usage, plutôt bons.** RunLore ne renvoie pas une réponse floue : il remonte les **logs** et **événements** qui comptent et propose une **cause probable** étayée.

* **Leur qualité tient à deux choses.** Les **sources de données** branchées (sans métriques, logs ni historique GitOps, l'agent raisonne à l'aveugle) et le **modèle**. Un modèle de raisonnement solide (Opus, GLM, GPT…) produit des diagnostics bien plus solides qu'un petit modèle sans raisonnement (type Gemini Flash). Sur les incidents **ambigus, multi-signaux**, où il faut vraiment corréler, l'écart devient évident.

* **L'effort de départ n'est pas négligeable.** Sans un minimum de **discipline**, la base de connaissances ne se remplit pas toute seule : sur une prod déjà **bruyante**, compte **quelques heures** d'analyse et de **relecture de PRs** avant qu'elle devienne payante — un investissement, pas un « installe et oublie ».

* **Plus la base se remplit, plus l'agent devient utile.** Chaque investigation résolue que tu valides (ton analyse, ton contexte, une RCA précise) devient une **connaissance réutilisable** : l'incident déjà vu ressort de la KB **en quelques secondes**, sans relancer d'analyse complète, et les entrées qui **résolvent vraiment gagnent en confiance** pendant que celles qui rappellent sans jamais résoudre sont **écartées**. Reste la vraie inconnue — la **curation dans la durée** : comment les investigations s'**associent** aux bonnes entrées, ce qu'il advient quand un **même symptôme cache une cause différente** (RunLore les regroupe et réclame une vérif humaine), et si **doublons et conflits** restent gérables. Des garde-fous existent, mais seules **des semaines de trafic réel** diront s'ils tiennent.

{{% notice warning "Points d'attention pour la prod ⚠️" %}}
La posture supportée est **read-only → suggest → approve** : RunLore lit, corrèle et recommande, un humain relit et fusionne. Le palier d'autonomie `auto` (remédiation automatique sans supervision) est **expérimental, gelé, et déconseillé sur un cluster réel**. Reste sur le _golden path_, avec un humain dans la boucle de validation.
{{% /notice %}}

Je reviendrai donc avec un **retour d'expérience sur la durée**. D'ici là, le projet est **ouvert** (Apache-2.0) et c'est maintenant que ton avis compte le plus : essaie-le sur ta plateforme et dis-moi comment il se comporte, [ouvre une issue](https://github.com/Smana/runlore/issues) pour un bug, une idée ou un cas d'usage, ou envoie une PR. Chaque retour de terrain oriente la suite.

## 🔖 Références

* [RunLore — dépôt GitHub](https://github.com/Smana/runlore) · [guide de démarrage](https://github.com/Smana/runlore/blob/main/docs/getting-started.md)
* [Open Knowledge Format (OKF) — Knowledge Catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog)
* [Open Knowledge Format — annonce Google Cloud](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing)
* [Le pattern _LLM-wiki_ d'Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
* [cloud-native-ref — plateforme de référence](https://github.com/Smana/cloud-native-ref)
* [k8sgpt](https://github.com/k8sgpt-ai/k8sgpt) · [HolmesGPT](https://github.com/HolmesGPT/holmesgpt) · [kagent](https://github.com/kagent-dev/kagent)
