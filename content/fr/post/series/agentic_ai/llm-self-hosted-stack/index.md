+++
author = "Smaine Kahlouch"
title = "Self-hosted LLM stack : poser les fondations d'une plateforme open-weight évolutive"
date = "2026-05-13"
summary = "Une plateforme self-hosted pour héberger les modèles open-weight sur Kubernetes : `InferenceService` déclaratif, autoscaling sur signaux GPU, GitOps de bout en bout. Une fondation pensée pour évoluer avec l'écosystème."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "vllm", "self-hosted"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — Partie 3" %}}
Cet article fait suite à [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) (les fondamentaux des coding agents) et [Quelques mois avec Claude Code](/fr/post/series/agentic_ai/ai-coding-tips/) (tips et retours d'expérience au quotidien). **Ici, on prend du recul** : Que peut-on raisonnablement faire en self-hosted aujourd'hui ?
{{% /notice %}}

J'utilise désormais l'agentic coding de façon quotidienne, comme beaucoup d'entre nous. Je suis globalement satisfait de l'expérience avec Claude Code, mais je garde un œil attentif sur l'**écosystème open-weight**[^open-weight], qui est lui aussi en pleine effervescence : Qwen, Llama, Mistral, GLM, DeepSeek — et sur la bataille que se mènent les différents acteurs du secteur.

[^open-weight]: Pour la définition précise — et la distinction avec *open-source* — voir [Open Weights, par l'Open Source Initiative](https://opensource.org/ai/open-weights).

Je me suis alors lancé un petit défi : **comment pourrions-nous héberger nos propres LLMs dans notre infrastructure ?** Souveraineté des données, choix du modèle, indépendance vis-à-vis d'un fournisseur, et surtout — la curiosité technique de voir ce qui est aujourd'hui *vraiment* faisable avec les outils modernes à notre disposition.

{{% notice note "Cet article est une démonstration" %}}
Les modèles déployés ici sont des **7-8B paramètres**, servis sur un **GPU L4 24GB**. Sur les tâches agentiques complexes, leur niveau est **très en-deçà** d'un Sonnet 4.6 ou d'un Opus 4.7 — ils hallucinent davantage, bouclent sur les erreurs là où un modèle _frontier_ passe sans effort.

Les modèles open-weight qui pourraient *vraiment* s'en approcher (DeepSeek V4 Pro, Kimi K2.6) demandent un matériel hors de portée à titre personnel.

Cet article a pour objectif de **poser les fondations** d'une alternative à faire évoluer à mesure que l'écosystème rattrape son retard, et démontre ce qui est *déjà* possible aujourd'hui — y compris avec des modèles modestes.
{{% /notice %}}

## :dart: Objectifs

* Comprendre les **briques d'une stack LLM self-hosted** moderne sur Kubernetes
* Découvrir les choix techniques retenus : modèles open-weight, **vLLM Production Stack**, routage intelligent, autoscaling sur signaux GPU
* Voir comment on consomme cette plateforme depuis trois clients : **OpenWebUI**, **Continue VSCode**, **OpenCode CLI**
* Évaluer honnêtement ce qui marche, ce qui ne marche pas, et ce que ça coûte vraiment

{{% notice tip "Le repo de référence" %}}
Toute la stack est déployée par GitOps depuis [**cloud-native-ref**](https://github.com/Smana/cloud-native-ref) — au-delà de la couche LLM, le repo illustre **un écosystème cloud-native complet** :

* **GitOps** — Flux, réconciliation continue
* **Platform engineering** — Crossplane (une claim YAML génère tout l'outillage Kubernetes)
* **Observabilité** — VictoriaMetrics, VictoriaLogs, Grafana
* **Sécurité & accès** — gestion des secrets + exposition privée via Tailscale

{{% /notice %}}

---

## :world_map: Vue d'ensemble de la stack

Avant d'entrer dans le cœur de la bête, voici une vue d'ensemble :

{{< img src="architecture-vllm.png" width="1200" >}}

La stack s'organise en **trois couches**, que l'on parcourra du socle vers la porte d'entrée :

1. **Stockage** — les poids des modèles mutualisés sur un volume partagé.
2. **Inférence** — une _fleet_ de pods **vLLM** qui servent les modèles sur GPU.
3. **Accès** — l'**Envoy AI Gateway** en frontal, et le **vLLM Semantic Router** qui choisit dynamiquement le modèle quand c'est nécessaire.

L'ensemble est piloté par **Flux** (réconciliation continue depuis [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)) et exposé en privé via **Tailscale** — aucune donnée ne quitte l'infrastructure. Chaque couche est détaillée dans les sections qui suivent.

---

## :electric_plug: La pièce maîtresse : l'abstraction `InferenceService`

Si vous avez lu certains de [mes articles précédents](/fr/post/crossplane_composition_functions/), vous savez que j'apprécie particulièrement `Crossplane` pour fournir la bonne abstraction aux utilisateurs. Il fait partie de mes composants essentiels et permet d'exposer une **interface adaptée et simple** à utiliser. J'en ai déjà quelques-unes : `App`, `SQLInstance`, `EPI` — et maintenant `InferenceService`.

### Déclarer un nouveau modèle

Concrètement, exposer un nouveau LLM sur la plateforme se résume à l'ajout d'un fichier YAML dont voici un exemple :

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: InferenceService
metadata:
  name: xplane-my-new-model
  namespace: llm
spec:
  model:
    repository: Qwen/Qwen3-8B
    revision: <hf-commit-sha>
    quantization: fp8
    contextWindow: 32768
    toolCallParser: hermes
    preload:
      enabled: true
  gpu:
    count: 1
  routing:
    tier: medium
    specialty: general
  scaling:
    minReplicas: 1
    maxReplicas: 2
```

Quelques secondes après le push, Flux réconcilie, la composition Crossplane se déclenche, et **toutes les ressources Kubernetes nécessaires sont appliquées automatiquement**.

### Ce que la composition génère pour nous

À partir de cette _Claim_, une dizaine de ressources Kubernetes sont créées. Au-delà des classiques (`Service`, `ServiceAccount`, `ExternalSecret`), six d'entre elles méritent d'être pointées :

* 🧠 **[vLLM](https://github.com/vllm-project/vllm) (`Deployment`)** — moteur d'inférence : _continuous batching_ + _paged attention_ donnent un throughput **3 à 10× supérieur** à un serving naïf sur le même GPU. Image `vllm/vllm-openai`, `nodeSelector: gpu-l4`. *([on revient sur vLLM en Couche 2](#vllm-production-stack))*

* 📦 **`Job` de préchargement** — télécharge les ~15 Go de poids HuggingFace vers le PVC S3 Files. **Idempotent** (1× par couple `repository@revision`) : les pods vLLM (réplicas, redéploiements) montent ensuite le volume en quelques secondes.

* 🛡️ **Deux `CiliumNetworkPolicy`** (zero-trust) : politique restrictive pour le pod vLLM long-lived (DNS sortant + ingress AI Gateway/Iris/vmagent), politique plus permissive scopée au Job de préchargement (HF, AWS API, EKS Pod Identity Agent). Évite d'élargir les droits du serving aux besoins ponctuels du _bootstrap_.

* ⚡ **`ScaledObject` KEDA** — autoscaling déclenché **avant** que la charge ne sature un pod, plutôt qu'en réaction à la file d'attente qui se forme. *([le calcul est détaillé en Couche 2](#autoscaling-avec-keda))*

* 📊 **`VMServiceScrape` + `VMRule`** — métriques vLLM scrapées par **VictoriaMetrics** sur `/metrics`, SLOs et alertes (cold-start budget, taux d'erreur, latence) _shippés_ avec le modèle. *([approche détaillée dans la série Observabilité](/fr/post/series/observability/metrics/))*

* 🚪 **`AIGatewayRoute`** — déclare comment l'**Envoy AI Gateway** dispatche le trafic vers ce modèle : `model: xplane-qwen-coder` dans la requête OpenAI atterrit sur le bon pod, sans logique applicative.

**Pour basculer d'un modèle à un autre**, il suffit de modifier deux champs (`model.repository` et `model.revision`). Voici par exemple la PR-type pour passer du Qwen2.5-Coder-7B actuel à un Qwen3-Coder-30B le jour où il rentre sur le matériel :

```diff
# apps/base/ai/llm/qwen-coder.yaml
spec:
  model:
-    repository: Qwen/Qwen2.5-Coder-7B-Instruct
-    revision: c03e6d358207e414f1eca0bb1891e29f1db0e242
+    repository: Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8
+    revision: <hash-correspondant>
     quantization: fp8
     contextWindow: 32768
```

Quatre lignes changent. Flux réconcilie, KEDA réajuste les triggers, Karpenter provisionne le bon GPU — tout l'agencement existant suit.

{{% notice tip "KCL : versionner, tester, valider une composition" %}}
La composition n'est pas écrite en YAML patches (peu lisible, intestable) mais en [**KCL**](https://kcl-lang.io/) via la [function-kcl](https://github.com/crossplane-contrib/function-kcl) — un langage de configuration **typé**, avec **assertions natives**. Trois conséquences directes :

* **Tests unitaires** — un fichier [`main_test.k`](https://github.com/Smana/cloud-native-ref/blob/main/infrastructure/base/crossplane/configuration/kcl/inference-service/main_test.k) valide chaque comportement (`kcl test` tourne en CI sur chaque PR).
* **Packaging OCI versionné** — la composition est publiée comme image OCI (`oci://ghcr.io/smana/cloud-native-ref/crossplane-inference-service:0.6.0`), référencée par tag immuable.
* **Schéma de claim validé côté API server** — `kubectl apply` rejette les claims incohérentes (par exemple `minReplicas > maxReplicas`) **avant** que la composition ne se déclenche.

Versions, tests, schéma — pas juste "du YAML qu'on copie-colle".
{{% /notice %}}

---

## :brain: Couche 1 — Les modèles et leur stockage

Parcourons maintenant les trois couches du diagramme, en commençant par le socle. Deux décisions structurantes ici : **quel matériel est financièrement accessible** (et donc quels modèles peuvent tenir dedans), et **comment leurs poids sont stockés et chargés à la demande**.

### L'écosystème en mai 2026

Ces derniers mois, la qualité des modèles open-weight a progressé **considérablement**. Sur **SWE-bench Verified**[^swebench], les deux meilleurs modèles open-weight sont désormais **DeepSeek V4 Pro (Max)** (80.6%) et **Kimi K2.6** (80.2%) — ils se défendent plutôt bien face aux leaders propriétaires (Claude Opus 4.7 à 87.6%, GPT-5.5 à 88.7%). Côté plus léger, **Qwen2.5-Coder** et **Qwen3-Coder** (Alibaba) restent des références pour la génération de code.

[^swebench]: Verified est aujourd'hui considéré comme partiellement contaminé : OpenAI a [cessé de le reporter](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) au profit de [SWE-bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro_public), où Kimi K2.6 mène les open-weight à 58.6%.

Mais entre "pouvoir télécharger le modèle depuis HuggingFace" et "le faire tourner sur du GPU **financièrement accessible**", il y a un monde. Les meilleurs modèles open-weight pour le coding demandent des configurations **multi-H100** ou **multi-H200**, et même les modèles intermédiaires comme **Qwen3-Coder-30B-A3B-FP8** réclament au minimum un **L40S 48GB**.

Le facteur limitant ici n'est pas la solution technique mais le **coût**. Pour cette démonstration, je me suis limité à des modèles modestes.

{{% notice info "Combien coûterait un DeepSeek V4 Pro ou un Kimi K2.6 ?" %}}
Pour fixer un ordre de grandeur : faire tourner un **DeepSeek V4 Pro** (1.6T paramètres, ~49B actifs en MoE) ou un **Kimi K2.6** (~1T total, ~32B actifs) en FP8 demande typiquement **8 à 16× H100 80GB**. En 24/7 sur AWS, le budget mensuel se situe entre **40 000 et 80 000 $**. Même en spot ou en _reserved_ sur un an, on reste largement au-dessus de **15 000 $/mois**.

À titre personnel, c'est évidemment hors de portée. La démo reste donc sur du L4 24GB et on choisit des modèles qui rentrent dedans.
{{% /notice %}}

### Stockage des poids : S3 Files

Charger ~15GB de poids depuis HuggingFace à chaque cold-start de pod, c'est lent et coûteux (bande passante egress). La solution retenue : [**Amazon S3 Files**](https://aws.amazon.com/blogs/aws/launching-s3-files-making-s3-buckets-accessible-as-file-systems/), une feature AWS **toute récente** (GA en **avril 2026**) qui expose un bucket S3 comme un **filesystem NFS conforme POSIX**, montable comme un volume Kubernetes partagé entre pods.

Les avantages dans notre cas :

* **Cold-start raccourci** — un Job de préchargement télécharge les poids depuis HuggingFace une seule fois lors du premier déploiement ; chaque pod vLLM monte ensuite le volume directement, démarrage en quelques secondes
* **Multi-attach natif** — plusieurs pods (réplicas, modèles distincts) partagent le même bucket sans configuration particulière, contrairement à un PVC EBS
* **Cache intelligent** — préfetching automatique, latence ~1ms sur l'_active set_ (les poids fréquemment lus restent sur EFS sous le capot)
* **Tarification EFS** sur les données actives uniquement, et **tarif S3 standard** (~$0.023/GB/mois) sur le stockage long terme — bien moins cher qu'un PVC EBS provisionné en taille fixe
* **POSIX complet** — vLLM lit les poids comme depuis un disque local, sans adaptation

{{% notice note "Le tout premier remplissage reste lent" %}}
S3 Files n'est pas magique sur le _bootstrap_ initial : le Job de préchargement doit télécharger les ~15GB de poids depuis HuggingFace **une première fois** vers S3. Cette étape est limitée par la bande passante réseau de l'instance (~10 Gbps sur `g6.xlarge`) et prend typiquement plusieurs dizaines de secondes. Le coût initial s'amortit ensuite : tous les démarrages **suivants** (réplicas, redéploiements, autres modèles partageant le bucket) montent le volume en quelques secondes.
{{% /notice %}}

---

## :gear: Couche 2 — La couche d'inférence : vLLM sur GPU

Une fois le paysage des modèles posé et leurs poids accessibles via S3 Files, il reste à les **servir efficacement** — ce choix pèse directement sur la latence, le throughput et le coût.

### vLLM Production Stack

[**vLLM**](https://github.com/vllm-project/vllm) est un **moteur d'inférence open-source** spécialisé dans le serving de LLMs sur GPU, devenu un standard de fait. C'est le composant qui fait tourner les modèles dans cette stack, déployé via son [Helm chart Production Stack](https://github.com/vllm-project/production-stack). Les fonctionnalités qui comptent ici :

* **API OpenAI-compatible** native — tous les clients (OpenWebUI, Continue, OpenCode…) parlent à vLLM sans adaptation.
* **Support fp8 mature** au niveau hardware sur L4 / L40S / H100 / H200 — divise la VRAM par deux.
* **Parser Hermes pour le function-calling** — indispensable pour faire fonctionner les `tools[]` des clients agentiques.
* **Continuous batching** + **paged attention** — le couple qui démultiplie le throughput d'un GPU : typiquement **3 à 10× plus** qu'un serving naïf type HuggingFace Transformers ou Ollama default[^vllm-bench].

{{% notice info "Continuous batching et paged attention, en deux phrases" %}}
**Continuous batching** : un serving naïf attend qu'un lot complet de requêtes arrive avant de le traiter (batching statique). vLLM, lui, **insère chaque nouvelle requête dans le batch GPU en cours** — le GPU ne dort jamais entre deux requêtes.

**Paged attention** : un serving naïf réserve le _KV cache_ (la mémoire accumulée à chaque token, qui domine la VRAM) en un bloc contigu dimensionné sur la longueur max — un `malloc` au pire cas, gaspillé sur les requêtes courtes. vLLM le **pagine comme un OS pagine sa mémoire virtuelle** : pages de taille fixe à la demande, des séquences de longueurs très différentes cohabitent sur le même GPU sans fragmentation.
{{% /notice %}}

[^vllm-bench]: Multiplicateurs documentés dans le papier original [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180) (Kwon et al., SOSP 2023) : vLLM atteint **2-4×** le throughput de FasterTransformer et **14-24×** celui d'un baseline HuggingFace Transformers. La fourchette réelle dépend du type de charge et du baseline retenu.

Concrètement, chaque modèle est servi par un Pod vLLM avec ce genre de config (générée par la composition Crossplane) :

```yaml
servingEngineSpec:
  modelSpec:
  - name: "xplane-qwen-coder"
    repository: "vllm/vllm-openai"
    modelURL: "Qwen/Qwen2.5-Coder-7B-Instruct"
    requestGPU: 1
    vllmConfig:
      enablePrefixCaching: true
      enableChunkedPrefill: true
      maxModelLen: 32768
      dtype: "fp8"
      maxNumSeqs: 32
      extraArgs: ["--tool-call-parser", "hermes"]
```

{{% notice info "Qu'est-ce que la quantification (fp8) ?" %}}
Un modèle de langage stocke ses **poids** sous forme de nombres décimaux. En précision native (**FP16** / **BF16**), chaque poids occupe **16 bits** (2 octets) — c'est précis, mais ça consomme beaucoup de VRAM.

La **quantification** consiste à représenter ces mêmes poids sur **moins de bits**. On arrondit les valeurs avec une précision numérique réduite, ce qui **divise la mémoire** et **accélère les calculs**. La contrepartie : les sorties du modèle (probabilités sur les tokens) **dérivent légèrement** de celles du modèle non quantifié.

Le **FP8** divise la VRAM par 2 et est supporté nativement (au niveau hardware) par les GPU récents : **L4, L40S, H100, H200** — d'où son adoption généralisée en inférence.
{{% /notice %}}

Un flag du YAML ci-dessus mérite un mot supplémentaire : **`enablePrefixCaching: true`**, crucial pour le **FIM** (_Fill-In-the-Middle_, l'autocomplete dans l'IDE). Lors d'une session d'autocomplete, chaque requête envoyée par l'IDE contient le même **préfixe** (le code du fichier autour du curseur) ; seuls les derniers caractères changent à mesure que le développeur tape. Avec le _prefix caching_, vLLM **réutilise** le KV cache déjà calculé pour ce préfixe partagé d'une requête à l'autre, au lieu de tout recalculer — c'est ce qui permet de tenir la latence p95 sous les 200 ms en tab-complete intensif.

### Autoscaling avec KEDA

L'HPA standard de Kubernetes est limité à CPU/mémoire — des signaux qui ne reflètent **pas** la charge réelle d'un pod vLLM (le facteur limitant est la VRAM et le _batch_ interne du moteur, pas le CPU hôte). [**KEDA**](https://keda.sh) résout ce problème en permettant de scale sur **n'importe quel signal externe** : Prometheus, file d'attente, événement métier, etc.

#### Scaler sur des indicateurs pertinents

Pour chaque modèle, le `ScaledObject` KEDA s'appuie sur deux métriques Prometheus exposées **par vLLM** :

* `vllm:num_requests_running` — combien de requêtes vLLM traite en parallèle, rapportée à la taille de batch configurée (`maxNumSeqs`) pour mesurer la saturation du _batch_ interne.
* `vllm:gpu_cache_usage_perc` — la pression sur le KV cache GPU, qui monte vite avec les contextes longs.

```yaml
# Extrait de la composition Crossplane
triggers:
  - type: prometheus
    metadata:
      query: max(vllm:num_requests_running{model_name="xplane-qwen-coder"}) / scalar(vector(32))
      threshold: "0.7"   # 70% du batch (maxNumSeqs) occupé
  - type: prometheus
    metadata:
      query: max(vllm:gpu_cache_usage_perc{model_name="xplane-qwen-coder"})
      threshold: "0.6"   # 60% du KV cache GPU
```

L'objectif est d'**anticiper** : ces indicateurs montent *avant* qu'une file d'attente ne se forme côté utilisateur, ce qui permet à KEDA d'ajouter un réplica suffisamment tôt pour absorber la montée en charge **sans dégrader la latence des requêtes en cours**. C'est tout l'écart entre un autoscaler qui *réagit* (file qui grossit, latence qui explose) et un autoscaler qui *anticipe*.

#### Karpenter prend le relais sur les nœuds

KEDA scale les **réplicas vLLM**, mais ces réplicas ont besoin d'un GPU disponible pour être planifiés. C'est [**Karpenter**](https://karpenter.sh/) qui s'occupe de la couche en dessous : quand un pod ne trouve pas de nœud GPU libre, Karpenter provisionne automatiquement une instance fournissant du GPU (cycle ~60 s) ; et dès qu'un nœud n'héberge plus de pod GPU, il est décommissionné.

---

## :twisted_rightwards_arrows: Couche 3 — La couche d'accès : Gateway + routage

Reste à examiner la **porte d'entrée** : comment une requête arrive jusqu'au bon pod vLLM, et qui a le droit de l'envoyer.

### Envoy AI Gateway : la porte d'entrée

[**Envoy AI Gateway**](https://github.com/envoyproxy/ai-gateway) est un projet open-source bâti sur Envoy Gateway, dédié à la gestion du trafic vers les services de Generative AI. C'est lui qui agit ici comme **point d'entrée unique** de la plateforme. Ses fonctionnalités principales :

* **Compatibilité avec l’API OpenAI** — les clients adressent un endpoint unique (`https://llm.priv.cloud.ogenki.io/v1/...`) sans connaître les pods vLLM individuels.
* **Routage par modèle** — la Gateway extrait le `model` du body de la requête OpenAI (ou d'un header `x-ai-eg-model`) et dispatche vers le bon `AIServiceBackend` Kubernetes (détails dans la sous-section suivante).
* **Authentification par Bearer token** — un `SecurityPolicy` Envoy Gateway vérifie chaque requête contre une liste de tokens connus.
* **Tracking de coûts par tokens** — `llmRequestCosts` extrait input/output/total tokens depuis les réponses, base utile pour du _rate limiting_ par tenant ou par modèle (préparé mais non activé ici).
* **Multi-provider** — bien que la stack soit 100% self-hosted aujourd'hui, on pourrait demain pointer un client vers OpenAI / Anthropic / Bedrock à travers la même Gateway, sans toucher au code côté client.

### Deux modes de routage complémentaires

Un seul modèle ne fait pas tout : **coder pour le code**, **reasoner pour les maths**, **guard pour la safety**. La porte d'entrée combine **deux mécanismes complémentaires**, mobilisés selon le besoin :

* **Routage explicite** — le client cible directement un modèle (`model: xplane-qwen-coder`) via le header `x-ai-eg-model` ou le body de la requête. **[Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway)** dispatche nativement, latence négligeable. C'est ce que font Continue (autocomplete) et OpenCode quand ils savent quel modèle utiliser.
* **Routage sémantique** — le client demande le modèle virtuel **`MoM`** (_Mixture of Models_) et **[vLLM Semantic Router (Iris)](https://github.com/vllm-project/semantic-router)** analyse le prompt pour choisir le bon modèle réel (coder, reasoner, guard). Détaillé dans la sous-section suivante.

Grâce à ces possibilités, un client qui sait ce qu'il veut ne subit aucune latence supplémentaire, et un client générique (OpenWebUI) profite d'un routage intelligent automatique.

### Iris et le routage sémantique

[**Iris (vLLM Semantic Router)**](https://github.com/vllm-project/semantic-router) est le projet open-source qui implémente la logique de **_Mixture of Models_** (`MoM`). Déployé en **sidecar** à côté de l'AI Gateway, il intercepte les requêtes adressées au modèle virtuel `MoM` et choisit dynamiquement le modèle réel qui va y répondre.

Sous le capot, un classifier compact (~100M paramètres, dérivé de **mmBERT**, servi sur **CPU** — donc aucune pression sur la VRAM des pods GPU) évalue le prompt selon **plusieurs critères** : l'intention (code, raisonnement, multilingue…), la présence d'une éventuelle tentative de _jailbreak_, ou la détection de **données personnelles (PII)**. Selon le verdict, Iris route vers le modèle approprié — ou applique un _guardrail_ de sécurité dédié.

**L'intérêt** : les clients adressent un seul endpoint (`MoM`) et bénéficient d'un routage adapté à chaque prompt sans coder leur propre logique de sélection. Le _trade-off_ : ~250-300 ms de classification, payés **uniquement** sur les requêtes adressées à `MoM`. C'est acceptable pour un chat — le TTFT global reste imperceptible côté humain — mais rédhibitoire pour l'autocomplete IDE qui doit rester sous les 200 ms p95 : c'est exactement pour ça que **Continue** adresse directement le pod coder en routage explicite.

---

## :computer: Les clients

Enfin, afin de pouvoir concrètement utiliser cette plateforme, il faut évidemment des clients. Pas la peine ici de détailler la configuration de chacun — l'objectif est de montrer qu'on dispose, en open-source, **d'alternatives crédibles aux solutions propriétaires** : chat web, autocomplete IDE, agent CLI.

<div style="display:flex;gap:16px;flex-wrap:wrap;margin:1.5em 0;align-items:flex-start;">
  <figure style="flex:1 1 0;min-width:280px;margin:0;text-align:center;">
    <video class="screencast-2x" style="width:100%;border-radius:6px;" autoplay muted loop playsinline preload="metadata">
      <source src="openwebui-screencast.mp4" type="video/mp4">
      Votre navigateur ne supporte pas la lecture vidéo.
    </video>
    <figcaption style="font-size:0.9em;color:#666;margin-top:6px;"><em>OpenWebUI — chat web</em></figcaption>
  </figure>
  <figure style="flex:1 1 0;min-width:280px;margin:0;text-align:center;">
    <video class="screencast-2x" style="width:100%;border-radius:6px;" autoplay muted loop playsinline preload="metadata">
      <source src="opencode-screencast.mp4" type="video/mp4">
      Votre navigateur ne supporte pas la lecture vidéo.
    </video>
    <figcaption style="font-size:0.9em;color:#666;margin-top:6px;"><em>OpenCode — agent CLI</em></figcaption>
  </figure>
</div>

<script>document.querySelectorAll('video.screencast-2x').forEach(v => { v.playbackRate = 2.0; });</script>

### OpenWebUI — l'alternative open-source à ChatGPT / Gemini

[**OpenWebUI**](https://openwebui.com/) expose une UI de chat web standard au-dessus de notre API OpenAI-compatible. Dans le dropdown des modèles on retrouve les pods de la plateforme plus le modèle virtuel `MoM` (Iris choisit alors le routage en lisant le prompt). **Cas d'usage** : chat exploratoire, tests rapides, accès non-dev — exactement ce qu'on ferait sur chat.openai.com ou gemini.google.com.

### Continue VSCode — l'autocomplete FIM dans l'IDE

[**Continue**](https://continue.dev/) branche l'API sur VSCode (ou JetBrains). Le _killer feature_ ici, c'est le **FIM** (_Fill-In-the-Middle_) : autocomplete sous **200ms p95** grâce au pod coder dédié toujours warm et au prefix cache de vLLM. C'est ce qui fait la différence entre une autocomplete réactive et une autocomplete frustrante (cela s'apparente à ce que nous pouvons retrouver sur Cursor ou Copilot).

### OpenCode — l'alternative open-source à Claude Code

Je viens de découvrir [**OpenCode**](https://opencode.ai/) — l'agent CLI qui se rapproche le plus de l'expérience **Claude Code**, et donc le seul candidat sérieux pour envisager une migration le jour où ce serait nécessaire. Il publie un _compatibility shim_ explicite — `AGENTS.md` ↔ `CLAUDE.md`, **skills**, **MCPs**, sous-agents, slash commands — si bien que **tout le workflow construit autour de Claude Code se porte directement**. C'est ce qui le distingue d'Aider, Crush ou Continue agent mode.

---

## :bar_chart: Supervision et évaluation continue

Une plateforme LLM produit du signal **sur plusieurs axes à la fois** — santé du serving, consommation de tokens par tenant, qualité des réponses — et chacun a ses propres indicateurs (TTFT, inter-token latency, prefix cache hit, token usage par opération…) que les métriques classiques d'un monitoring web ne capturent pas.

</br>

Bonne nouvelle : la **stack d'observabilité existante** (VictoriaMetrics, VictoriaLogs, Grafana) absorbe tout ça sans nouvel outil — il restait à brancher les bonnes sources. Et l'enjeu va **au-delà de la détection d'anomalies** : comprendre *comment* la plateforme est utilisée — qui consomme quoi, sur quels modèles, pour quel coût — est tout aussi important.

### Santé de la plateforme — métriques `vLLM`

[`vLLM` expose nativement](https://docs.vllm.ai/en/stable/usage/metrics/) un endpoint Prometheus complet, chaque métrique portant un label `model_name` :

* **Charge** : `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:gpu_cache_usage_perc` (les triggers KEDA viennent d'ici)
* **Performance** : `vllm:time_to_first_token_seconds`, `vllm:inter_token_latency_seconds`, `vllm:e2e_request_latency_seconds`
* **Throughput** : `vllm:prompt_tokens`, `vllm:generation_tokens`
* **Optimisations** : `vllm:prefix_cache_hits` (essentiel pour mesurer l'efficacité du FIM)

Le dashboard Grafana `LLM Platform` (déployé via `Grafana Operator` depuis `apps/base/ai/llm/grafana-dashboard.yaml`) agrège tout ça par modèle :

{{< img src="grafana-dashboard-llm.png" alt="Dashboard Grafana — vue d'ensemble de la fleet LLM" width="1200" >}}

Sur le screenshot, la stack est au repos : 4 modèles actifs (un replica chacun), 0 requête en cours, KV cache à 0.01% — l'état "always warm" sans charge.

### Usage et FinOps — métriques de l'AI Gateway

L'**`Envoy AI Gateway`** observe le trafic **un cran plus haut** que vLLM — au niveau métier. Ses métriques suivent le standard [OpenTelemetry Gen AI Semantic Conventions](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/) : il ne compte pas des requêtes, il instrumente le **langage métier des LLMs** :

| Métrique | Mesure |
|---|---|
| `gen_ai.client.token.usage` | Tokens consommés (input / output / total) |
| `gen_ai.server.request.duration` | Latence end-to-end par requête |
| `gen_ai.server.time_to_first_token` | TTFT au niveau gateway |
| `gen_ai.server.time_per_output_token` | Inter-token latency |

Chaque métrique est automatiquement enrichie de labels `gen_ai.*` (modèle, opération, provider) — on sait déjà *qui consomme quoi* en une requête PromQL. Et on peut **injecter en labels des headers HTTP arbitraires** (`x-tenant-id`, `x-team`…) : à partir de là, on répond à :

* **Quel tenant consomme le plus de tokens** sur les 30 derniers jours ?
* **Quelle équipe a la latence p95 la plus haute** sur le coder ?
* **Quel ratio prompt/génération** par utilisateur ? (utile pour le dimensionnement de contexte)
* **Combien d'appels chat vs embedding** par client ?

> SLOs / alertes habituels (latence p95, taux d'erreur, saturation GPU) restent définis en `VMRule` à côté — rien de spécifique au LLM, j'en parle dans l'[article observability/alerting](/fr/post/series/observability/alerts/).

### Évaluation qualitative avec `Promptfoo`

Les métriques d'infra disent si la plateforme **fonctionne**. Elles ne disent pas si elle **produit des bonnes réponses** — c'est une dimension complètement orthogonale, et c'est précisément ce que [**Promptfoo**](https://www.promptfoo.dev/) apporte.

Le principe est simple : on déclare des cas de test qui ressemblent à des tests unitaires — un prompt d'entrée + des assertions sur la sortie attendue. Les types d'assertions couvrent tout l'éventail :

* **Déterministes** : `equals`, `contains`, `regex`, `is-json` (avec schéma), `javascript` / `python` — parfait pour valider une sortie structurée (tool-calling, JSON formaté)
* **Model-assisted** : `llm-rubric` (un LLM note la sortie selon une rubrique qualitative), `factuality` (adhérence aux faits fournis), `similar` (embeddings + cosine), `g-eval` (chain-of-thought avec critères custom) — la même approche que les évals propriétaires (HELM, MT-Bench), mais déclarable en YAML chez soi

Dans la stack, c'est packagé comme un **CronJob nightly** (`tooling/base/promptfoo/`) : la suite est en `ConfigMap`, le job tourne contre les modèles + le routage `MoM`, et les résultats sont **poussés en métriques Prometheus** — donc visibles dans le même Grafana que les métriques techniques.

L'intérêt concret :

* **Détection des régressions** sur un bump de modèle ou de version `vLLM` — un score qualité qui chute de 80 → 65 % du jour au lendemain, c'est plus parlant qu'un graphe de latence
* **Comparaison cascade vs modèle direct** : est-ce que le routage `MoM` dégrade la qualité par rapport à un appel direct au bon modèle ?
* **Stress-test du tool-calling séquentiel** : combien d'appels d'outils consécutifs `Qwen2.5-Coder` peut-il enchaîner avant de se planter ?

L'eval continue ne sert pas à se rassurer dans l'absolu, mais à **mesurer l'évolution dans le temps** et à éviter les régressions silencieuses — c'est ce qui me permet de quantifier ce que j'affirme dans la conclusion.

---

## :thought_balloon: Dernières remarques

### Quels objectifs a-t-on pu atteindre ?

On a réussi à bâtir un **moyen simple de servir des modèles open-weight** sur Kubernetes : une claim YAML déclenche tout l'outillage nécessaire, le swap d'un modèle se fait en quelques lignes, et l'observabilité est en place de bout en bout.

Mais soyons honnêtes sur l'expérience utilisateur : sans accès à des modèles "sérieux" (DeepSeek V4 Pro, Kimi K2.6, Qwen3-Coder-30B…), nos tests se sont essentiellement limités à **valider qu'on obtient une réponse** — pas à mesurer une vraie qualité de production. Avec les modèles 7-8B qui tiennent sur un L4, on a parfois l'impression de revenir à la préhistoire 😅.

Le point positif, c'est ce qu'il y a sous le capot : **la stack elle-même est viable et évolutive**.

### À qui ça parle aujourd'hui

L'expérience est immédiatement pertinente pour les **entreprises avec une vraie problématique de souveraineté des données** — santé, défense, finance, secteurs régulés. Et avec les modèles open-weight les plus avancés (DeepSeek V4 Pro, Kimi K2.6, à moins de dix points d'Opus sur SWE-bench), la différence de qualité avec les modèles propriétaires est devenue marginale. Pour des données qui ne peuvent de toute façon pas sortir, la question ne se pose même plus vraiment.

Pour tous les autres — moi le premier — **l'intérêt est de poser les fondations** en vue du moment où le calcul basculera vraiment. Ce moment dépend de deux facteurs qui évoluent vite :

* **Diversification du matériel** : l'hégémonie Nvidia commence à craquer. DeepSeek V4 tourne déjà sur Huawei Ascend 950 (avec _Day 0 adaptation_ confirmée par Huawei, Cambricon et Hygon). Plus de fournisseurs → plus de disponibilité → moins de pression sur les prix.
* **Accessibilité financière** : les modèles open-weight qui se rapprochent du frontier (1T+ paramètres) demandent encore 8 à 16× H100 — hors de portée à titre personnel, encore élevé pour beaucoup d'entreprises.

### Une note (volontairement courte) géopolitique

Je n'ai volontairement pas voulu rentrer dans ces considérations dans le corps de l'article — et je prends les benchmarks tels quels, en supposant qu'ils sont impartiaux. Cela dit, deux observations méritent d'être posées :

* **Les acteurs chinois privilégient l'open-weight** (DeepSeek, Kimi/Moonshot, Qwen/Alibaba). C'est un pari de stratégie industrielle qui pourrait s'avérer payant à long terme — plus l'écosystème open-weight mûrit, plus il devient compétitif face aux modèles fermés.
* **Et leurs résultats sur SWE-bench** — la référence du coding — sont **vraiment bons** : sur SWE-bench Pro, Kimi K2.6 mène les open-weight à 58,6 %, à ~6 points de Claude Opus 4.7. Sur Verified, DeepSeek V4 Pro frôle les 80 %, soit ~7 points en deçà d'Opus. L'écart se resserre, benchmark après benchmark.

### Et après ?

Soyons clairs : aujourd'hui je ne troquerais pas mon écosystème Claude. Principalement pour des raisons **financières** — pas par attachement particulier. À mon échelle d'usage, Sonnet et Opus me coûtent moins cher que de reproduire en self-hosted une qualité équivalente.

Cela dit, j'aurais aimé pousser plus loin l'usage d'**OpenCode** et migrer pour de bon ma configuration Claude (skills, MCPs, sous-agents) sur ce backend — ce sera peut-être le sujet d'un futur article dédié à cet agent coding open-source.

Mais je garde la stack vivante. Quand un Qwen3-Coder-30B-A3B tournera correctement sur un L4 quantifié — chemin documenté dans [`docs/llm-platform-future-paths.md`](https://github.com/Smana/cloud-native-ref/blob/main/docs/llm-platform-future-paths.md) — le swap sera une PR de quelques lignes. C'est ça l'**intérêt principal** de cette démo : se mettre en position d'**adopter rapidement** ce qui s'annonce, plutôt que d'avoir à tout (re)construire le jour où l'open-weight rattrapera le frontier.

Et ce rattrapage ne vient pas que des modèles : la couche serving open-source évolue tout aussi vite et ajoute régulièrement des fonctions jusque-là réservées aux solutions propriétaires. Par exemple, [**vLLM-Omni**](https://github.com/vllm-project/vllm-omni) (premier *stable* fin 2025) étend `vLLM` à l'**omni-modalité** (texte, image, audio, vidéo, en entrée *et* en sortie) avec la même API OpenAI-compatible, donc plug directement dans la plateforme décrite ici.

**Ironie de l'histoire** : l'ensemble de cette stack a été conçu et construit avec l'aide de **Claude Code** 🙃.

---

## :bookmark: Références

### Repos
- [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — La plateforme complète
- [`docs/decisions/`](https://github.com/Smana/cloud-native-ref/tree/main/docs/decisions) — ADRs (vLLM Production Stack, S3 Files…)
- [`docs/llm-platform-future-paths.md`](https://github.com/Smana/cloud-native-ref/blob/main/docs/llm-platform-future-paths.md) — Chemins d'évolution

### Composants techniques
- [vLLM Production Stack](https://github.com/vllm-project/production-stack) — Inférence LLM en production
- [vLLM-Omni](https://github.com/vllm-project/vllm-omni) — Serving omni-modalité (texte/image/audio/vidéo, in & out)
- [vLLM Semantic Router (Iris)](https://github.com/vllm-project/semantic-router) — Routage intelligent multi-modèles
- [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway) — Gateway API pour LLM
- [Promptfoo](https://www.promptfoo.dev/) — Évaluation LLM en continu
- [Continue](https://continue.dev/) — Extension VSCode IDE assist
- [OpenCode](https://opencode.ai/) — CLI agent loop OSS
- [OpenWebUI](https://openwebui.com/) — Chat web pour LLM API

### Modèles open-weight cités
- [Qwen2.5-Coder-7B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct) (Apache 2.0)
- [Qwen3-8B](https://huggingface.co/Qwen/Qwen3-8B) (Apache 2.0)
- [Llama-Guard-3-1B](https://huggingface.co/meta-llama/Llama-Guard-3-1B) (Llama 3 license)
- [DeepSeek V4 / V4-Pro](https://api-docs.deepseek.com/) (model card officielle, mai 2026)

### Sources DeepSeek V4
- [DeepSeek launches 1.6 trillion parameter V4 on Huawei chips](https://www.tomshardware.com/tech-industry/artificial-intelligence/deepseek-launches-1-6-trillion-parameter-v4-on-huawei-chips-as-us-escalates-ai-theft-accusations) — Tom's Hardware
- [Three reasons why DeepSeek's new model matters](https://www.technologyreview.com/2026/04/24/1136422/why-deepseeks-v4-matters/) — MIT Technology Review
- [DeepSeek V4 arrives with near state-of-the-art intelligence at 1/6th the cost of Opus 4.7, GPT-5.5](https://venturebeat.com/technology/deepseek-v4-arrives-with-near-state-of-the-art-intelligence-at-1-6th-the-cost-of-opus-4-7-gpt-5-5) — VentureBeat
- [Huawei Ascend, Cambricon and Hygon Completed Day 0 Adaptation to DeepSeek-V4](https://www.trendforce.com/news/2026/04/29/news-huawei-ascend-cambricon-and-hygon-completed-day-0-adaptation-to-deepseek-v4/) — TrendForce

### Articles précédents de la série
- [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) — Partie 1
- [Quelques mois avec Claude Code : tips et workflows](/fr/post/series/agentic_ai/ai-coding-tips/) — Partie 2
