+++
author = "Smaine Kahlouch"
title = "Un repo, deux clouds : une stratégie `multicloud` pragmatique"
date = "2026-09-15"
draft = true
summary = "Le **multicloud**, c'est le plus souvent une slide, rarement un repo. La stratégie d'une plateforme qui tourne réellement sur **AWS EKS et GCP GKE** depuis un seul dépôt Git : quoi abstraire, quoi laisser épouser la forme du cloud, comment acheter du GPU, et ce que la résilience exige vraiment."
featured = true
codeMaxLines = 25
usePageBundles = true
toc = true
tags = [
    "multicloud",
    "platform-engineering",
    "crossplane",
    "gitops",
    "gpu"
]
thumbnail = "thumbnail.png"
+++

Le 20 octobre 2025, une **race condition latente** dans la gestion DNS automatisée de DynamoDB [a effacé les enregistrements DNS du service](https://www.thousandeyes.com/blog/aws-outage-analysis-october-20-2025) en us-east-1. Le DNS est revenu en trois heures environ, mais le **DropletWorkflow Manager** interne d'EC2 est entré en **effondrement congestif** (_congestive collapse_) — le travail de récupération s'accumulant plus vite qu'il ne pouvait être traité — et le rétablissement complet s'est étiré sur près de quatorze heures. Et parce que us-east-1 héberge des services de control plane globaux comme **IAM** (Identity and Access Management) et **STS** (Security Token Service), le blast radius s'est étendu bien au-delà de la région.

Chaque panne de cette ampleur relance la même conversation. ❓ **Devrions-nous tourner sur plus d'un cloud ?**

Le **multicloud**, c'est le plus souvent une slide. Rarement un repo. Ce qui suit n'est ni un comparatif de fournisseurs ni une expérience de pensée : c'est une stratégie tirée d'une plateforme qui tourne réellement sur **AWS EKS** et **GCP GKE**, avec des compromis mesurés plutôt que supposés.

## 🎯 Objectifs de cet article

* Identifier les **vraies motivations du multicloud en 2026** — et les arguments pour rester sur un seul cloud
* Décider **quoi abstraire et quoi laisser épouser la forme du cloud**
* Voir comment **un seul dépôt Git** pilote **deux clusters de production**
* Définir une **stratégie de consommation GPU**, avec un **coût par million de tokens** mesuré
* Comprendre **quand une _fleet_ a besoin de plus que Flux**
* Évaluer **ce qu'exige réellement la survie à la perte d'un cloud**
* Obtenir la facture, sans détour : **ce que ça a réellement coûté**

{{% notice tip "Le repo de référence" %}}
Tout ce qui est décrit dans cet article tourne aujourd'hui dans le projet <strong><a href="https://cnref.ogenki.io">Cloud Native Ref</a></strong> (<a href="https://github.com/Smana/cloud-native-ref">sources sur GitHub</a>) : la même plateforme sur AWS EKS et GCP GKE, réconciliée depuis un seul dépôt et un seul arbre Flux — pas un fork par cloud. Les Compositions vivent dans un dépôt dédié, <a href="https://github.com/Smana/crossplane-configuration">crossplane-configuration</a>.
{{% /notice %}}

## 🌍 Pourquoi le multicloud — la version honnête

Chaque décision du reste de cet article dépend des motivations ci-dessous qui s'appliquent réellement à votre cas — classons-les donc, en commençant par celles qui ont un poids juridique.

1. **Capacité de sortie réglementaire** : pour les entités financières européennes, pouvoir quitter un fournisseur cloud n'est plus une aspiration mais une obligation. **DORA** (Digital Operational Resilience Act, le règlement européen sur la résilience opérationnelle numérique), en vigueur depuis le 17 janvier 2025, impose aux entreprises d'évaluer le risque de concentration **ICT** (Information and Communication Technology, les technologies de l'information et de la communication) et d'être capables de quitter un fournisseur critique [« sans perturbation indue »](https://schneiderdowns.com/our-thoughts-on/doras-approach-to-exit-strategy-and-termination/) (article 28), avec des clauses de sortie et de transition inscrites dans les contrats eux-mêmes (article 30). La nuance qui compte en pratique : les régulateurs attendent des plans de transition **testés**, pas des plans parfaits — un document de sortie que personne n'a jamais répété ne passe pas. Et la pression déborde désormais la finance : le **Data Act** européen, pleinement applicable depuis le 12 septembre 2025, [supprime les frais de changement de fournisseur — y compris l'egress lié à la migration — à partir du 12 janvier 2027](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/), levant la principale pénalité financière d'un départ effectif.

2. **Souveraineté** : placer les workloads par juridiction — et pouvoir les déplacer — devient un paramètre de conception plutôt qu'un élément de langage. AWS lui-même l'a reconnu en lançant son **European Sovereign Cloud** le 15 janvier 2026 : un cloud basé dans le Brandebourg, opéré par une entité juridique allemande autonome (GmbH), avec 7,8 Md€ investis. Et le [paquet de souveraineté technologique](https://www.digital-chiefs.de/en/digital-sovereignty-2026-gaia-x-delos-cloud-and-europes-response-to-the-cloud-ac/) adopté par l'UE en juin 2026 maintient le sujet sur les feuilles de route européennes.

3. **Capacité GPU** : la nouvelle motivation, plus discrète. L'offre d'accélérateurs reste tendue — le **B200** de NVIDIA devrait rester contraint jusqu'à mi-2026 — et quand le GPU dont vous avez besoin n'est pas disponible chez votre fournisseur, dans votre région, la capacité à consommer là où elle existe cesse d'être un débat philosophique. Elle sert aussi de levier tarifaire : une option crédible pour faire tourner l'inférence ailleurs change le ton des négociations GPU. Nous y consacrerons une section entière plus loin.

4. **Résilience** : l'incident us-east-1 qui ouvre cet article en est le rappel récurrent : le control plane d'un fournisseur unique est un blast radius unique, aussi bon ce fournisseur soit-il. Si cette motivation n'arrive qu'en quatrième position alors qu'elle sert d'accroche, c'est que le failover inter-cloud est la plus coûteuse de toutes à réellement construire — peu de pannes justifient un tel investissement. Une section dédiée évalue plus loin ce qu'il exige vraiment.

5. **La majorité peu glorieuse** : la plupart des entreprises n'ont jamais décidé d'être multicloud — [elles ont découvert qu'elles l'étaient déjà](https://gartsolutions.com/multi-cloud-kubernetes-the-power-and-peril/). Une opération de **M&A** (mergers and acquisitions, fusions-acquisitions) fait entrer un parc GCP chez un client tout-AWS ; une équipe data souscrit discrètement à BigQuery ; une filiale conserve ses contrats existants. La vraie question est rarement « faut-il passer au multicloud ? » mais « nous le sommes déjà — le gère-t-on délibérément ou le laisse-t-on proliférer ? »

### Le contre-argument

Rien de tout cela n'est gratuit, et prétendre le contraire est la façon dont meurent les projets multicloud. La taxe de complexité est structurelle : deux modèles IAM aux sémantiques différentes, deux stacks réseau qui ne s'accordent à peu près sur rien, deux fois plus de quotas, deux fois plus de particularités par service, et une astreinte qui doit désormais parler couramment les deux. Chaque ingénieur que vous recrutez connaît déjà les deux clouds, ou se forme au second — sur votre temps.

Vient ensuite le piège du plus petit dénominateur commun. Tout abstraire pour que ça tourne à l'identique partout, c'est renoncer précisément aux services managés qui rendaient chaque cloud attractif au départ — vous finissez par payer au prix fort une flotte de VMs génériques et par reconstruire vous-même les services différenciants.

Et l'egress coûte toujours de l'argent. Le Data Act supprime les frais de *changement de fournisseur*, pas le trafic quotidien : une architecture bavarde répartie sur deux clouds, c'est une hémorragie à chaque appel inter-cloud, dans les deux sens, indéfiniment.

Ma position, que le reste de cet article défend : le multicloud est une **capacité qu'on construit sélectivement**, pas une destination où l'on déménage. La plupart des équipes ont besoin d'une *capacité* de sortie et de portabilité à la couche API — pas d'un actif-actif symétrique généralisé. Le travail intéressant consiste à décider exactement quelle part de cette capacité construire, et où.

## 🧭 La doctrine : abstraire pour le développeur, pas pour la plateforme

Le piège du plus petit dénominateur commun a un remède structurel : cesser de se demander *combien* abstraire et se demander *pour qui*. La règle sur laquelle repose cette plateforme — découper la surface d'API par audience — tranche presque toutes les questions de conception qui suivent ; elle mérite donc d'être énoncée précisément.

Un peu de vocabulaire d'abord. Les APIs de la plateforme sont construites avec **Crossplane**, qui étend Kubernetes avec des APIs d'infrastructure sur mesure : une **XRD** (Composite Resource Definition) déclare le schéma, et une **Composition** transforme une **claim** — le manifest que l'utilisateur écrit réellement — en ressources cloud concrètes.

**Les claims côté développeur sont cloud-neutres.** `App`, `SQLInstance`, `InferenceService` : les kinds nomment l'intention, et les champs aussi — `spec.objectStore`, jamais `spec.s3Bucket`. Le même manifest s'applique tel quel sur le cluster AWS et sur le cluster GCP ; quel cloud le matérialise, c'est la Composition installée sur le cluster cible qui en décide — la claim, elle, ne porte aucun signal cloud. C'est la couche où la portabilité paie vraiment : les manifests applicatifs sont l'artefact le plus nombreux de la plateforme, et le plus coûteux à migrer à la main.

**Les APIs côté plateforme épousent délibérément la forme de leur cloud**, sous forme de XRDs sœurs, une par cloud. L'exemple le plus net : `EPI` (**EKS Pod Identity**, le mécanisme AWS qui accorde des permissions IAM aux pods). Son champ central est `spec.policyDocument` — de la policy IAM AWS en JSON, inline. Il n'existe aucune forme neutre de ce champ. Sa sœur GCP, `GCPWorkloadIdentity`, lie plutôt des rôles IAM Google à un ServiceAccount. Une API « WorkloadIdentity » fusionnée devrait porter les deux formes et en choisir une au rendu : deux APIs sous un même nom, avec pour seul ajout un semblant de portabilité. [ADR-0007](https://cnref.ogenki.io/docs/decisions/0007-cloud-abstraction-boundaries/) énonce le principe sur lequel nous ne cessons de revenir :

> Une API qui a l'air neutre sans l'être provoque des erreurs pires qu'une API qui épouse visiblement la forme de son cloud.

Les platform engineers savent déjà quel cloud ils configurent. Le leur cacher n'apporte rien, et se paie en messages d'erreur qui pointent vers la mauvaise abstraction.

**Une trappe de sortie garde la surface neutre honnête.** Les claims neutres portent des blocs optionnels `aws {}` / `gcp {}` pour les réglages spécifiques à un fournisseur, qui existent inévitablement. La portabilité reste le comportement par défaut ; atteindre une fonctionnalité propre à un cloud coûte un bloc clairement identifié, pas un fork de l'API — et un reviewer peut mesurer le couplage cloud d'une claim en grepant ces deux clés. Et quand un réglage peut rester neutre, c'est la composition qui absorbe la différence : la sauvegarde d'une `SQLInstance` demande un simple `bucketName`, et le rendu le transforme en destination `s3://` ou `gs://` selon le cloud du cluster.

La surface neutre tient-elle en pratique ? Voici le diff entre l'exemple AWS et l'exemple GCP de la même claim `InferenceService`, débarrassé des commentaires internes au repo — dans le diff complet, chaque ligne modifiée au-dessus de `apiVersion:` est un commentaire YAML :

```diff
--- inferenceservice-basic.yaml	2026-08-28 09:21:24
+++ inferenceservice-gcp.yaml	2026-08-28 09:21:24
@@ -1,23 +1,26 @@
 ---
-# Basic InferenceService — smallest viable model.
+# GCP InferenceService example — exercises the composition-gcp.yaml branch.
 [… lignes modifiées restantes élidées — toutes des commentaires YAML …]
 apiVersion: cloud.ogenki.io/v1alpha1
 kind: InferenceService
 metadata:
-  name: xplane-qwen3-8b-basic
+  name: xplane-qwen3-8b-gcp
   namespace: llm
 spec:
   model:
```

Le reste des deux fichiers — l'intégralité du `spec:` — est identique à l'octet près. La seule différence dans la surface réelle de la claim est `metadata.name`.

## 🌳 Un arbre Flux, deux clusters

La doctrine décide de la forme des APIs ; le dépôt décide qui les applique. Nous réconcilions les deux clusters avec **Flux** — le moteur GitOps qui observe un dépôt Git et applique en continu ce qu'il y trouve — depuis un seul arbre partagé. `clusters/aws-0/` et `clusters/gcp-0/` en sont les deux points d'entrée, minces à dessein : ils déclarent ce que chaque cluster exécute, en pointant vers des définitions partagées.

```text
clusters/
├── aws-0/           # entrypoint: what aws-0 runs, at which versions
└── gcp-0/           # same file layout, different paths and pins
infrastructure/
├── base/            # component definitions, written once
├── aws-0/           # AWS-only overlay
└── gcp-0/           # GCP-only overlay (ComputeClass, public DNS…)
observability/       # same base / aws-0 / gcp-0 split
security/            # same split
```

Ce découpage base/overlay est ce qui rend une pull request inter-cloud lisible. Touchez `infrastructure/base/` ou `observability/base/` et **les deux** clusters réconcilient le changement à leur prochaine synchronisation ; touchez `infrastructure/gcp-0/` et un seul le fait. Les chemins du diff annoncent le blast radius avant même que le reviewer ait lu une seule ligne de YAML.

Partagé ne veut pas dire symétrique pour autant. La base contient des *définitions*, et le point d'entrée de chaque cluster décide lesquelles il consomme : Karpenter vit dans `infrastructure/base/`, mais seul `aws-0` l'inclut — l'autoscaling des nœuds GKE passe par une `ComputeClass` dans l'overlay `gcp-0`. Ce qu'un cluster exécute se décide à son point d'entrée.

Les Compositions de la section précédente ne sont *pas* dans cet arbre. Elles sont consommées comme des paquets **Configuration** versionnés : des artefacts **OCI** (Open Container Initiative) — le même format de registre que les images de conteneurs, que Crossplane réutilise pour ses paquets — publiés depuis le dépôt dédié [crossplane-configuration](https://github.com/Smana/crossplane-configuration). Trois paquets existent : `crossplane-configuration-core` porte les contrats cloud-neutres, `-aws` et `-gcp` les Compositions par cloud. Le tag OCI *est* le tag git — la version qui tourne dans un cluster est donc toujours un commit que l'on peut checkout.

L'arbre Flux de chaque cluster épingle ensuite ce qu'il installe : `clusters/aws-0/infrastructure/crossplane-configuration.yaml` réconcilie le pin AWS, son jumeau `gcp-0` celui de GCP. Le pin lui-même tient dans un unique objet `Configuration` :

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: crossplane-configuration-aws
spec:
  package: ghcr.io/smana/crossplane-configuration-aws:v0.4.1
```

Une montée de version d'API de plateforme, ce sont donc deux petites pull requests — on bumpe le tag sur un cluster, on le laisse vivre quelques jours, puis on bumpe l'autre — jamais un big bang synchronisé entre les clouds.

La mécanique pour monter un nouveau cluster dans cet arbre — comptes, stacks OpenTofu, bootstrap Flux, ordonnancement — relève du how-to plus que de la stratégie ; le guide [Add a cloud provider](https://cnref.ogenki.io/docs/guides/add-a-cloud-provider/) la déroule de bout en bout.

{{< img src="multicloud-overview.png" alt="Un seul dépôt Git réconcilié par Flux vers EKS et GKE, avec des paquets de configuration Crossplane par cloud" width="1000" >}}

## 🔗 Les coutures : identité, DNS, secrets

Aussi propre que soit l'abstraction, il existe des endroits où les deux clouds doivent réellement se toucher — pas coexister côte à côte, mais se faire confiance, se résoudre ou se lire l'un l'autre. Garder cette liste courte est en soi un résultat de stratégie. La nôtre compte trois entrées : l'identité des workloads, le DNS, et les magasins de secrets plus l'authentification.

**L'identité des workloads.** Certains workloads sur GKE doivent appeler des APIs AWS (la couture DNS ci-dessous explique pourquoi). Ils le font en endossant un rôle IAM AWS via `AssumeRoleWithWebIdentity` : un fournisseur d'identité **OIDC** (OpenID Connect) AWS est déclaré pour faire confiance à l'émetteur de tokens de GKE, le workload présente son **bound ServiceAccount token** — de courte durée de vie, restreint à une audience, projeté dans le pod par Kubernetes — et STS l'échange contre des identifiants AWS temporaires. Zéro clé statique nulle part : pas d'access key qui traîne dans un secret GCP, aucune rotation à gérer, rien qui puisse fuiter. Et la confiance est étroite par construction — le rôle n'accorde que la modification d'enregistrements Route53 sur une seule zone hébergée (plus les lectures nécessaires pour la trouver). La couture entière tient dans une petite stack OpenTofu, `opentofu/shared/aws-gcp-federation`, rangée sous `shared/` parce que ce sont des ressources dont le seul travail est de coupler les deux clouds.

**Le DNS.** Il existe une seule zone publique faisant autorité, `cloud.ogenki.io`, hébergée dans Route53. Les services GCP vivent sous le domaine enfant `gcp.cloud.ogenki.io`, si bien qu'ils ne partagent jamais un niveau avec les enregistrements wildcard du côté AWS — l'endroit précis où deux clouds qui publient dans un même espace de noms entrent en collision. Côté GKE, chaque certificat couvre un seul hostname : une clé volée ne compromet qu'un service, pas tout le sous-domaine. Le rôle fédéré de la couture identité est ce qui fait fonctionner tout cela depuis GCP : cert-manager sur GKE prouve le contrôle du domaine en résolvant des défis **DNS-01** — écrire un enregistrement TXT que l'autorité de certification vérifie ensuite — directement dans Route53, et external-dns publie chaque hostname de `HTTPRoute` dans la même zone, de la même façon. Le raisonnement complet, y compris la collision qui a imposé le découpage en domaine enfant, est consigné dans [ADR-0019](https://cnref.ogenki.io/docs/decisions/0019-cross-cloud-dns-federation/).

**Les magasins de secrets et l'authentification.** Les manifests référencent les secrets via un seul nom de `ClusterSecretStore` portable, avec une grammaire de clés que les deux clouds acceptent ([ADR-0023](https://cnref.ogenki.io/docs/decisions/0023-portable-secret-store-names/)) ; chaque cluster adosse ce nom à son propre magasin managé — AWS Secrets Manager sur EKS, GCP Secret Manager sur GKE ([ADR-0024](https://cnref.ogenki.io/docs/decisions/0024-cloud-managed-secret-stores/)). Les manifests n'encodent jamais quel magasin répond. Au-dessus des magasins siège un fournisseur d'identité unique pour toute la plateforme : une seule instance ZITADEL, hébergée sur aws-0 et servant les deux clusters ([ADR-0022](https://cnref.ogenki.io/docs/decisions/0022-single-identity-provider-across-clouds/)) — parce que deux instances signifieraient deux annuaires d'utilisateurs non fédérés, et « se connecter deux fois, une fois par cloud » est la meilleure façon pour une plateforme de convaincre ses utilisateurs que le multicloud était une erreur.

Trois coutures, chacune volontairement réduite, chacune avec un ADR qui consigne sa raison d'être. Tout le reste demeure cantonné à son cloud — et c'est exactement ce qui rend la question GPU abordable.

## 🎮 GPUs : une stratégie de consommation, entre coût et disponibilité

La question GPU est abordable ici grâce à l'empilement qui précède : la claim `InferenceService` est cloud-neutre, et tout ce qui a la forme d'un GPU en dessous — les node pools Karpenter sur AWS, la ComputeClass sur GKE — reste propre à son cloud. Quand le même manifest tourne sur l'un ou l'autre cluster, la capacité GPU cesse d'être une contrainte héritée de votre fournisseur et devient un marché où l'on fait ses courses : entre modèles de tarification d'abord, et entre clouds quand une région est à sec. Le `spec` identique à l'octet près de la section doctrine est ce qui rend ce shopping crédible — l'option de bouger est prouvée plutôt que promise, et c'est cette preuve qui change le ton d'une négociation de renouvellement GPU.

### Adapter le modèle de tarification au workload

La tarification GPU se décline en trois formes, et la stratégie est un portefeuille, pas un choix unique. La capacité **committed-use / réservée** couvre la base 24/7 que vous êtes certain de consommer. L'**on-demand** absorbe les pics au-dessus. Le **spot/préemptible** — de la capacité fortement remisée que le fournisseur peut [récupérer avec un préavis de 30 secondes à deux minutes](https://www.thundercompute.com/blog/cloud-gpu-spot-instance-availability) — est réservé au batch interruptible : fine-tuning, campagnes d'évaluation, backfills d'embeddings. Il n'a rien à faire sous un endpoint d'inférence temps réel, où un nœud récupéré est une panne visible par l'utilisateur. Cette segmentation — [classer les workloads par tolérance à l'interruption avant de courir après les remises](https://www.cloudmagazin.com/en/2026/04/03/ai-inference-costs-cloud-finops-gpu-workloads-2026/) — représente l'essentiel du FinOps GPU ; la remise elle-même est la partie facile.

Du côté réservé, les deux clouds vendent la certitude différemment. Les **EC2 Capacity Blocks for ML** d'AWS sont des réservations à durée définie : un bloc d'accélérateurs réservé pour une fenêtre donnée, à des tarifs publiés par accélérateur-heure. Le **Dynamic Workload Scheduler** de GCP met plutôt la demande en file d'attente — le mode flex-start provisionne dès que la capacité apparaît, le mode calendar réserve une fenêtre future. Avec une offre de classe B200 contrainte jusqu'à mi-2026 (la troisième motivation de la liste d'ouverture), ces mécanismes ne servent pas à payer moins ; ils servent à obtenir des accélérateurs rares tout court.

### Dimensionner l'accélérateur au plus juste

L'erreur la plus coûteuse précède tout modèle de tarification : choisir par défaut « le GPU de l'IA ». Une carte de classe L4 (24 GB) sert confortablement les modèles de classe 8B que cette plateforme exécute ; les A100/H100 existent pour des modèles d'un ordre de grandeur plus gros, et en choisir un par défaut coûte 5 à 10x pour une marge qui tourne à vide. Notre choix sur les deux clouds est la plus petite instance mono-L4 — g6 sur AWS, g2-standard sur GCP : le même silicium à dessein, pour que la comparaison ci-dessous mesure des clouds, pas des GPUs.

### Faire du scale-to-zero votre principal levier

Le rapport 2026 State of Kubernetes Optimization de Cast AI situe l'utilisation moyenne des GPUs dans les clusters de production à [environ 5 %](https://cast.ai/blog/gpu-cloud-pricing/). C'est l'industrie qui paie à peu près vingt fois le GPU qu'elle utilise. Aucune remise ne rattrape ça — le remède est architectural. Sur cette plateforme, chaque `InferenceService` descend à zéro via **KEDA** (Kubernetes Event-driven Autoscaling), qui scale sur l'activité des requêtes plutôt que sur le CPU — le fonctionnement de cette machinerie de bout en bout (le serving vLLM, l'Envoy AI Gateway en frontal) est le sujet de l'article sur la [stack LLM self-hosted](/fr/post/series/agentic_ai/llm-self-hosted-stack/). C'est la conséquence stratégique qui compte ici : un modèle que personne n'interroge coûte le stockage objet où dorment ses poids, pas des heures de GPU.

### Comparer sur le coût par million de tokens

Les prix des instances ne répondent pas à la question qui compte — combien coûte réellement l'inférence ? L'unité comparable est le dollar par million de tokens générés, et la méthode tient en une ligne :

```text
cost_per_Mtok = (hourly_instance_cost / (throughput_tok_s * 3600)) * 1_000_000
```

Le débit vient des métriques de serving que vous collectez déjà (vLLM exporte des compteurs de tokens générés ; les tokens/s soutenus sur une fenêtre chargée sont le dénominateur qui reflète la charge réelle). La méthode est la partie durable de cette section ; les prix ci-dessous en sont les entrées périssables.

<!-- TODO(author): fill tok/s and $/Mtok using the PromQL in docs/superpowers/plans/2026-08-28-multicloud-strategy-post-inputs.md and re-verify $/h before publishing. The AWS row MUST be measured; the GKE row may ship as pending (the note below explains why) if quota still hasn't landed (if the GKE quota lands before publication, also update the Final thoughts sentence that references the pending row). Also add the missing thumbnail.png to this bundle (front matter references it; homepage card renders broken without it). Then delete this comment. -->

| Cloud | GPU | Instance | $/h (on-demand) | tok/s | $/Mtok |
|---|---|---|---|---|---|
| AWS EKS | 1x NVIDIA L4 (24 GB) | g6.2xlarge (8 vCPU / 32 GB), eu-west-3 | $1.2410 | _pending_ | _pending_ |
| GCP GKE | 1x NVIDIA L4 (24 GB) | g2-standard-8 (8 vCPU / 32 GB), europe-west4 | $0.8972 | _pending_ | _pending_ |

Quatre mises en garde avant que quiconque cite ce tableau. Les prix sont des tarifs on-demand publics relevés le 28 août 2026 — ils dériveront, et c'est exactement pourquoi la méthode compte plus que les cellules. Les lignes ne sont pas strictement comparables : les régions diffèrent parce que c'est là que chaque cluster tourne réellement, et le chiffre GCP intègre le L4 dans le prix de la machine là où le chiffre AWS est le tarif propre de l'instance. Bien que les deux lignes utilisent l'on-demand pour la comparabilité, la politique d'achat réelle de la plateforme est asymétrique — le node pool AWS mélange spot et on-demand, tandis que la ComputeClass GKE est délibérément spot-only. Et la ligne GKE pourrait rester _pending_ plus longtemps que la ligne AWS : ce cluster attend encore le quota nécessaire pour provisionner son premier nœud GPU — une illustration en direct du volet disponibilité du titre de cette section.

Voilà la stratégie pour une classe de workloads sur deux clusters ; la question suivante est ce qui se passe quand deux clusters ne suffisent plus à raconter l'histoire.

## 📈 Quand deux clusters deviennent une _fleet_

La réponse commence par ce que nous n'avons *pas* déployé. À deux clusters statiques, les points d'entrée Flux par cluster sont le bon modèle, pour les propriétés que la section sur l'arbre Flux a démontrées — un arbre lisible, un blast radius lisible dans les chemins, des montées de version en deux PRs. Un fleet manager posé au-dessus de `aws-0` et `gcp-0` ajouterait un control plane, un mode de défaillance et une pile de concepts nouveaux — pour ne rien économiser. Si deux clusters sont votre régime de croisière, considérez cette section comme un seuil que vous n'aurez peut-être jamais à franchir.

Le seuil, c'est le moment où les clusters cessent d'être une paire d'environnements nommés et pérennes pour devenir une *fleet* : un cluster par équipe, des clusters éphémères pour les previews ou les runs d'entraînement, des clusters créés à la demande par un workflow. Le pattern d'overlay par cluster survit mécaniquement à cette bascule mais échoue économiquement — chaque nouveau cluster est le copier-ajuster d'un point d'entrée, et relire la quarantième copie n'apprend rien que les trente-neuf premières n'aient déjà appris.

Ce qui remplace la copie, c'est la distribution par correspondance de labels. **Sveltos** est un contrôleur installé sur un cluster de management ; ses **ClusterProfiles** déploient add-ons et configuration vers chaque cluster enregistré qui correspond à un sélecteur de labels : déclarez une fois que les clusters étiquetés `tier=inference` reçoivent la stack GPU, et porter le label décide du reste. Son framework d'événements transforme cette commodité en capacité — le **cluster vending** : une claim Crossplane crée un cluster sur l'un ou l'autre cloud, Sveltos le détecte et l'enregistre, et les profils hydratent la plateforme complète dessus, sans humain dans la boucle. Le flux de bout en bout est documenté sous le nom de [GitOps bridge pattern](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/).

La ligne de partage des responsabilités compte plus que l'outil : Flux synchronise Git ; Sveltos distribue à travers la _fleet_. Les deux se composent — Sveltos s'intègre directement à Flux depuis la v0.23 : les manifests restent dans Git et Sveltos consomme ce que Flux a déjà synchronisé. L'adoption s'appuierait sur l'investissement Flux que cette plateforme a déjà consenti.

Ce n'est pas le seul candidat. **Karmada** (CNCF incubating) est le projet le plus mature de la catégorie — du véritable *scheduling de workloads* multi-cluster, qui place et replace des applications entre clusters : considérablement plus de machinerie qu'il n'en faut à une plateforme qui schedule par cluster. **Open Cluster Management** (**OCM**, CNCF sandbox) aborde le problème par le versant gouvernance et sert de fondation à Red Hat ACM. Les deux sont crédibles ; Sveltos est celui vers lequel cette plateforme se tournerait, parce que le besoin réel est la distribution d'add-ons plus un eventing qui se compose avec un arbre Flux déjà en place.

Quel que soit le nombre de clusters, pourtant, une question reste la même : qu'est-ce qui survit réellement quand un cloud entier disparaît ?

## 🛡️ La posture de résilience

La motivation n°4 promettait une évaluation de ce qu'exige réellement la survie à la perte d'un cloud, et elle commence par une correction : **GitOps n'est pas un backup**. Flux reconstruit tout ce qui est déclaratif — Deployments, XRDs, HTTPRoutes — parce que Git en détient la source de vérité. Il ne peut pas reconstruire ce que Git n'a jamais contenu : les octets d'un volume PostgreSQL, le contenu d'un cache, tout ce qu'un workload a écrit à l'exécution. Une posture de résilience commence donc par tracer la **frontière de l'état** (_state boundary_) : décider, pour chaque composant stateful, quel chemin le ramène.

Sur cette plateforme, la frontière compte trois zones, une par chemin de récupération : les composants stateful qui ont un chemin à eux, les workloads reconstruits par GitOps, et la perte acceptée. La première zone a deux membres, et PostgreSQL est celui qui compte le plus : **CloudNativePG** (**CNPG**), l'opérateur Postgres, archive en continu son **WAL** (Write-Ahead Log — le journal en append-only de chaque changement de la base, la primitive sur laquelle repose la récupération Postgres) vers le stockage objet. Cette archive fait plus que restaurer sur place : les [replica clusters](https://cloudnative-pg.io/docs/1.27/replica_cluster/) de CNPG peuvent amorcer un standby dans l'*autre* cloud depuis cette même archive (S3 → GCS), avec une bascule déclarative quand le standby doit prendre le relais. Le **RPO** (Recovery Point Objective — la perte de données que vous acceptez, mesurée en temps) est borné par la cadence d'archivage du WAL.

Les magasins objet sont le second membre de la première zone : accessibles depuis les deux clouds par nature, c'est là que s'écrit tout ce qui doit survivre. Une conséquence mérite d'être énoncée explicitement : il n'y a pas de **Velero** (l'outil de backup Kubernetes de facto) ici, à dessein — les snapshots de PersistentVolumes qu'il prendrait ne captureraient rien qui ne soit déjà dans Git ou dans un magasin objet.

La deuxième zone prend le chemin facile : tout ce qui est stateless est reconstruit par Flux sur le cluster survivant, depuis le même arbre, en quelques minutes. C'est la section doctrine qui paie — les manifests n'ont jamais porté de signal cloud, donc redéployer sur l'autre cloud n'est qu'une réconciliation ordinaire. La troisième zone est la perte acceptée : les caches Valkey se reconstruisent depuis leurs sources ; les perdre coûte un temps de chauffe.

{{< img src="state-boundary.png" alt="Ce qui survit à la perte d'un cloud : PostgreSQL via l'archivage du WAL vers le stockage objet, les magasins objet comme couche d'état inter-cloud, les workloads stateless reconstruits par GitOps, les caches acceptés comme perdus" width="800" >}}

L'état n'est que la moitié du failover ; le trafic est l'autre moitié : repointer les hostnames via la zone Route53 partagée de la section coutures. C'est aussi là que vit le risque résiduel. Cette zone unique est un point de concentration, et le control plane de Route53 tourne en us-east-1 — la région de l'incident d'ouverture, où c'est précisément par ces dépendances de control plane globales que l'impact s'est propagé au-delà de la région. Dans un tel événement, les enregistrements existants continuent de se résoudre, mais les *modifier* peut devenir impossible — inconfortable quand modifier des enregistrements est votre mécanisme de failover. ADR-0019 consigne le compromis : une zone unique, c'est la simplicité qui maintient les coutures au plus petit — et voilà le risque accepté en échange.

Reste la partie que l'outillage ne peut pas fournir : le failover, ici, est un **runbook testé**, pas un bouton. Promouvoir le standby CNPG, repointer le DNS, faire monter en charge le survivant — chaque étape est déclarative, mais la séquence est opérationnelle, et l'exigence est celle d'exercices périodiques : le dérouler à échéance régulière, mesurer le temps de rétablissement, corriger ce qui vous a surpris. C'est ce que les régulateurs de la motivation n°1 entendent par plans de transition *testés* — un document de sortie que personne n'a répété ne passe pas, et un runbook de failover non plus. Tant qu'il n'a pas été répété, un runbook est une hypothèse.

Ce que la posture omet délibérément, c'est l'interconnexion inter-cluster à chaud — un réseau façon Cilium Cluster Mesh, où les services s'étendent sur les deux clouds à l'exécution — un coût de possession dont la plupart des plateformes n'ont pas besoin et que celle-ci n'a pas payé : il est au backlog, et son absence fait partie de la facture que la section suivante additionne.

## 💭 Dernières remarques : ce que ça a réellement coûté

La facture, donc.

Le second cloud a été une seconde couche de fondations : une autre stack OpenTofu — réseau, GKE, secrets, identité — une seconde famille de providers Crossplane avec ses propres Compositions par cloud, et un decision record à part entière rien que pour faire tourner Cilium en mode self-managed sur GKE. Rien de tout cela n'était difficile au sens d'un problème de recherche ; c'était du volume — des semaines de soirées, pas un week-end. Et le flot de petites asymétries m'a usé davantage que les gros morceaux : des demandes de quota sans équivalent en face, une sémantique de ComputeClass qui ne recouvre celle de Karpenter qu'imparfaitement, un pool GPU spot-only sur un cloud et mixte sur l'autre — même les asymétries délibérées. Les coutures de fédération — DNS et identité — ont été l'inverse : petites en nombre de lignes, grandes en temps de réflexion.

Les abstractions ont mieux tenu que je ne l'espérais : la claim `SQLInstance` n'a jamais appris sur quel cloud elle tournait — la différence n'est jamais remontée au-dessus du rendu de sauvegarde de la composition. La fuite restée ouverte est opérationnelle : la ligne GKE du tableau GPU est toujours _pending_ parce que la demande de quota l'est aussi.

Une partie de la facture, c'est ce que j'ai choisi de ne pas construire. Il n'y a pas de Cilium Cluster Mesh, donc rien ne s'étend sur les deux clouds à l'exécution — la section résilience a signalé cette omission, et elle reste au passif, en dette reportée. Il n'y a pas d'actif-actif symétrique, ni de fleet manager au-dessus de deux clusters nommés. Ces omissions *sont* la stratégie — la capacité construite sélectivement — et chacune peut encore être ajoutée plus tard, sans en payer le coût de possession entre-temps.

Plusieurs de ces fils ouverts méritent un article à part entière, mesures à l'appui, et le backlog ressemble aujourd'hui à ceci :

* **Benchmarks GPU** — spot, Dynamic Workload Scheduler et Capacity Blocks mesurés, avec la méthode $/Mtok appliquée en entier
* **Cluster vending avec Crossplane + Sveltos** — le GitOps bridge construit de bout en bout
* **Perdre un cloud volontairement** — un replica cluster CNPG, une extinction AWS contrôlée, RTO/RPO mesurés
* **Cilium Cluster Mesh entre EKS et GKE** — le jeu en vaut-il la chandelle ? Avec les chiffres de latence et les factures d'egress

Si l'un de ces sujets peut vous être utile, dites-le en commentaires ou dans les issues GitHub du repo — les votes décideront de l'ordre.

La prochaine panne us-east-1 relancera la conversation qui a ouvert cet article ; cette fois, la réponse tourne dans un repo.

## 🔖 Références

### Plateforme
- [`cloud-native-ref`](https://cnref.ogenki.io) — La documentation complète de la plateforme · [sources sur GitHub](https://github.com/Smana/cloud-native-ref)
- [`crossplane-configuration`](https://github.com/Smana/crossplane-configuration) — Les compositions Crossplane
- [Décisions d'architecture](https://cnref.ogenki.io/docs/decisions/) — les ADR-0007, 0017–0024 sous-tendent cet article
- [Add a cloud provider](https://cnref.ogenki.io/docs/guides/add-a-cloud-provider/) — le how-to que cet article s'abstient volontairement de dupliquer

### Contexte
- [ThousandEyes — analyse de la panne AWS du 20 octobre 2025](https://www.thousandeyes.com/blog/aws-outage-analysis-october-20-2025)
- [Data Act européen : le changement de cloud et l'interdiction des frais en 2027](https://www.cloudmagazin.com/en/2026/07/06/eu-data-act-when-cloud-switching-fees-are-abolished-what-cios-need-to-examine/)
- [DORA : stratégie de sortie et risque de concentration](https://schneiderdowns.com/our-thoughts-on/doras-approach-to-exit-strategy-and-termination/)
- [FinOps pour l'inférence GPU (2026)](https://www.cloudmagazin.com/en/2026/04/03/ai-inference-costs-cloud-finops-gpu-workloads-2026/)
- [Sveltos](https://projectsveltos.io/main/) · [GitOps bridge Crossplane + Sveltos](https://projectsveltos.io/main/events/examples/sveltos_crossplane_gitops_bridge_pattern/)
- [Replica clusters CloudNativePG](https://cloudnative-pg.io/docs/1.27/replica_cluster/)
- [Cilium Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/)
