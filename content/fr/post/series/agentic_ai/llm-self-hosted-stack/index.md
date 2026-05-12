+++
author = "Smaine Kahlouch"
title = "`vLLM` : faire tourner ses propres modèles chez soi, ce qui est concrètement possible"
date = "2026-05-07"
summary = "Démonstration d'une stack LLM self-hosted complète sur Kubernetes : vLLM, semantic-router, OpenCode. Ce qui est faisable aujourd'hui avec des modèles open-weight, et ce qui ne l'est pas — encore."
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
Soyons clairs : les modèles déployés ici sont des **7-8B paramètres**. Sur les tâches agentiques complexes, leur niveau est **très en-deçà** d'un Sonnet 4.6 ou d'un Opus 4.7 — ils hallucinent davantage, bouclent sur les erreurs là où un modèle _frontier_ passe sans effort.

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

Avant d'entrer dans le coeur de la bête, voici une vue d'ensemble:

{{< img src="architecture-vllm.png" width="1200" >}}

La stack s'organise en **trois couches**, qui suivent le chemin d'une requête :

1. **Accès** — l'**Envoy AI Gateway** en frontal, et le **vLLM Semantic Router** qui choisit dynamiquement le modèle quand c'est nécessaire.
2. **Inférence** — une _fleet_ de pods **vLLM** qui servent les modèles sur GPU.
3. **Stockage** — les poids des modèles mutualisés sur un volume partagé.

L'ensemble est piloté par **GitOps** (Flux réconcilie tout depuis [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref)) et exposé en privé via **Tailscale** — rien n'atterrit sur Internet public. Chaque couche est détaillée dans les sections qui suivent.

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

* **Tests unitaires** — un fichier [`main_test.k`](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/infrastructure/base/crossplane/configuration/kcl/inference-service/main_test.k) valide chaque comportement (`kcl test` tourne en CI sur chaque PR).
* **Packaging OCI versionné** — la composition est publiée comme image OCI (`oci://ghcr.io/smana/cloud-native-ref/crossplane-inference-service:0.6.0`), référencée par tag immuable.
* **Schéma de claim validé côté API server** — `kubectl apply` rejette les claims incohérentes (par exemple `minReplicas > maxReplicas`) **avant** que la composition ne se déclenche.

Versions, tests, schéma — pas juste "du YAML qu'on copie-colle".
{{% /notice %}}

---

## :brain: Couche 1 — Les modèles et leur stockage

On va maintenant remonter les trois couches du diagramme, en commençant par le socle. Deux décisions structurantes ici : **quel matériel est financièrement accessible** (et donc quels modèles peuvent tenir dedans), et **comment leurs poids sont stockés et chargés à la demande**.

### L'écosystème en mai 2026

Ces derniers mois, la qualité des modèles open-weight a progressé **considérablement**. Sur **SWE-bench Verified**[^swebench], les deux meilleurs modèles open-weight sont désormais **DeepSeek V4 Pro (Max)** (80.6%) et **Kimi K2.6** (80.2%) — ils se défendent plutôt bien face aux leaders propriétaires (Claude Opus 4.7 à 87.6%, GPT-5.5 à 88.7%). Côté plus léger, **Qwen2.5-Coder** et **Qwen3-Coder** (Alibaba) restent des références pour la génération de code.

[^swebench]: Verified est aujourd'hui considéré comme partiellement contaminé : OpenAI a [cessé de le reporter](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) au profit de [SWE-bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro_public), où Kimi K2.6 mène les open-weight à 58.6%.

Mais entre "pouvoir télécharger le modèle depuis HuggingFace" et "le faire tourner sur du GPU **financièrement accessible**", il y a un monde. Les meilleurs modèles open-weight pour le coding demandent des configurations **multi-H100** ou **multi-H200**, et même les modèles intermédiaires comme **Qwen3-Coder-30B-A3B-FP8** réclament au minimum un **L40S 48GB**.</br>

Le facteur limitant ici n'est pas la solution technique mais le **coût**. Pour cette démonstration, je me suis limité à des modèles modestes.

{{% notice info "Combien coûterait un DeepSeek V4 Pro ou un Kimi K2.6 ?" %}}
Pour fixer un ordre de grandeur : faire tourner un **DeepSeek V4 Pro** (1.6T paramètres, ~49B actifs en MoE) ou un **Kimi K2.6** (~1T total, ~32B actifs) en FP8 demande typiquement **8 à 16× H100 80GB**. En 24/7 sur AWS, le budget mensuel se situe entre **40 000 et 80 000 $**. Même en spot ou en _reserved_ sur un an, on reste largement au-dessus de **15 000 $/mois**.

À titre personnel, c'est évidemment hors de portée. La démo reste donc sur du L4 24GB et on choisit des modèles qui rentrent dedans.
{{% /notice %}}

### Stockage des poids : S3 Files

Charger ~15GB de poids depuis HuggingFace à chaque cold-start de pod, c'est lent et coûteux (bande passante egress). La solution retenue : [**Amazon S3 Files**](https://aws.amazon.com/blogs/aws/launching-s3-files-making-s3-buckets-accessible-as-file-systems/), une feature AWS **toute récente** (GA en **avril 2026**) qui expose un bucket S3 comme un **filesystem NFS conforme POSIX**, montable comme un volume Kubernetes partagé entre pods.

Les avantages dans notre cas :

* **Cold-start divisé** — un Job de préchargement télécharge les poids depuis HuggingFace une seule fois lors du premier déploiement ; chaque pod vLLM monte ensuite le volume directement, démarrage en quelques secondes
* **Multi-attach natif** — plusieurs pods (réplicas, modèles distincts) partagent le même bucket sans configuration particulière, contrairement à un PVC EBS
* **Cache intelligent** — préfetching automatique, latence ~1ms sur l'_active set_ (les poids fréquemment lus restent sur EFS sous le capot)
* **Tarification EFS** sur les données actives uniquement, et **tarif S3 standard** (~$0.023/GB/mois) sur le stockage long terme — bien moins cher qu'un PVC EBS provisionné en taille fixe
* **POSIX complet** — vLLM lit les poids comme depuis un disque local, sans adaptation

{{% notice note "Le tout premier remplissage reste lent" %}}
S3 Files n'est pas magique sur le _bootstrap_ initial : le Job de préchargement doit télécharger les ~15GB de poids depuis HuggingFace **une première fois** vers S3. Cette étape est limitée par la bande passante réseau de l'instance (~10 Gbps sur `g6.xlarge`) et prend typiquement plusieurs dizaines de secondes. Le gain s'amortit ensuite : tous les démarrages **suivants** (réplicas, redéploiements, autres modèles partageant le bucket) montent le volume en quelques secondes.
{{% /notice %}}

---

## :gear: Couche 2 — La couche d'inférence : vLLM sur GPU


Maintenant que nous avons choisi nos modèles, il faut un moyen efficace de les **servir efficacement**. Cette solution a un impact non négligeable sur la latence, le troughput et le coût.

### vLLM Production Stack

[**vLLM**](https://github.com/vllm-project/vllm) est un **moteur d'inférence open-source** spécialisé dans le serving de LLMs sur GPU, devenu un standard de fait. C'est le composant qui fait tourner les modèles dans cette stack, déployé via son [Helm chart Production Stack](https://github.com/vllm-project/production-stack). Les fonctionnalités qui comptent ici :

* **API OpenAI-compatible** native — tous les clients (OpenWebUI, Continue, OpenCode…) parlent à vLLM sans adaptation.
* **Support fp8 mature** au niveau hardware sur L4 / L40S / H100 / H200 — divise la VRAM par deux.
* **Parser Hermes pour le function-calling** — indispensable pour faire fonctionner les `tools[]` des clients agentiques.
* **Continuous batching** + **paged attention** — le couple qui démultiplie le throughput d'un GPU : typiquement **3 à 10× plus** qu'un serving naïf type HuggingFace Transformers ou Ollama default[^vllm-bench].

{{% notice info "_Continuous batching_ et _paged attention_, en deux phrases" %}}
**Continuous batching** : un serving naïf attend qu'un lot complet de requêtes arrive avant de le traiter (batching statique). vLLM, lui, **insère chaque nouvelle requête dans le batch GPU en cours** — le GPU ne dort jamais entre deux requêtes.

**Paged attention** : le _KV cache_ (l'état d'attention par token, qui domine la consommation VRAM en inférence) est découpé en **pages de taille fixe**, comme la mémoire virtuelle d'un OS. Conséquence : plusieurs séquences de longueurs très différentes partagent la même VRAM **sans fragmentation**, ce qui démultiplie le nombre de séquences concurrentes par GPU.
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

Un flag du YAML ci-dessus mérite un mot supplémentaire : **`enablePrefixCaching: true`**, crucial pour le FIM. Lors d'une session d'autocomplete, chaque requête envoyée par l'IDE contient le même **préfixe** (le code du fichier autour du curseur) ; seuls les derniers caractères changent à mesure que le développeur tape. Avec le _prefix caching_, vLLM **réutilise** le KV cache déjà calculé pour ce préfixe partagé d'une requête à l'autre, au lieu de tout recalculer — c'est ce qui permet de tenir la latence p95 sous les 200 ms en tab-complete intensif.

### Autoscaling avec KEDA

L'HPA standard de Kubernetes est limité à CPU/mémoire — des signaux qui ne reflètent **pas** la charge réelle d'un pod vLLM (le facteur limitant est la VRAM et le _batch_ interne du moteur, pas le CPU hôte). [**KEDA**](https://keda.sh) résout ce problème en permettant de scale sur **n'importe quel signal externe** : Prometheus, file d'attente, événement métier, etc.

#### Scaler sur des indicateurs pertinents

Pour chaque modèle, le `ScaledObject` KEDA s'appuie sur deux métriques Prometheus exposées **par vLLM** :

* `vllm:num_requests_running / vllm:num_requests_max` — la saturation du _batch_ interne (combien de requêtes vLLM traite en parallèle par rapport à son max).
* `vllm:gpu_cache_usage_perc` — la pression sur le KV cache, qui monte vite avec les contextes longs.

```yaml
# Extrait de la composition Crossplane
triggers:
  - type: prometheus
    metadata:
      query: max(vllm:num_requests_running) / max(vllm:num_requests_max)
      threshold: "0.7"   # 70% du batch occupé
  - type: prometheus
    metadata:
      query: avg(vllm:gpu_cache_usage_perc)
      threshold: "0.85"  # 85% du KV cache
```

L'objectif est d'**anticiper**: ces indicateurs montent *avant* qu'une file d'attente ne se forme côté utilisateur, ce qui permet à KEDA d'ajouter un réplica suffisamment tôt pour absorber la montée en charge **sans dégrader la latence des requêtes en cours**. C'est tout l'écart entre un autoscaler qui *réagit* (file qui grossit, latence qui explose) et un autoscaler qui *anticipe*.

#### Karpenter prend le relais sur les nœuds

KEDA scale les **réplicas vLLM**, mais ces réplicas ont besoin d'un GPU disponible pour être planifiés. C'est **Karpenter** qui s'occupe de la couche en dessous : quand un pod ne trouve pas de noeud GPU libre, Karpenter provisionne automatiquement une instance fournissant du GPU (cycle ~60 s) ; et dès qu'un noeud n'héberge plus de pod GPU, il est décommissionné.

---

## :twisted_rightwards_arrows: Couche 3 — La couche d'accès : Gateway + routage

Abordons la **porte d'entrée** de la plateforme : comment une requête arrive jusqu'au bon pod vLLM, et qui a le droit de l'envoyer.

### Envoy AI Gateway : la porte d'entrée

[**Envoy AI Gateway**](https://github.com/envoyproxy/ai-gateway) est un projet open-source bâti sur Envoy Gateway, dédié à la gestion du trafic vers les services de Generative AI. C'est lui qui agit ici comme **point d'entrée unique** de la plateforme. Ses fonctionnalités principales :

* **API OpenAI-compatible** — les clients adressent un endpoint unique (`https://llm.priv.cloud.ogenki.io/v1/...`) sans connaître les pods vLLM individuels.
* **Routage par modèle** — la Gateway extrait le `model` du body de la requête OpenAI (ou d'un header `x-ai-eg-model`) et dispatche vers le bon `AIServiceBackend` Kubernetes (détails dans la sous-section suivante).
* **Authentification par Bearer token** — un `SecurityPolicy` Envoy Gateway vérifie chaque requête contre une liste de tokens connus.
* **Tracking de coûts par tokens** — `llmRequestCosts` extrait input/output/total tokens depuis les réponses, base utile pour du _rate limiting_ par tenant ou par modèle (préparé mais non activé ici).
* **Multi-provider** — bien que la stack soit 100% self-hosted aujourd'hui, on pourrait demain pointer un client vers OpenAI / Anthropic / Bedrock à travers la même Gateway, sans toucher au code côté client.

Côté réseau, **Cilium NetworkPolicies** appliquent un _zero-trust_ strict entre pods, et l'exposition externe se fait uniquement via **Tailscale** (`tag:k8s` ACL) — aucune route publique.

### Deux modes de routage complémentaires

Un seul modèle ne fait pas tout : **coder pour le code**, **reasoner pour les maths**, **guard pour la safety**. La porte d'entrée combine **deux mécanismes complémentaires**, mobilisés selon le besoin :

* **Routage explicite** — le client cible directement un modèle (`model: xplane-qwen-coder`) via le header `x-ai-eg-model` ou le body de la requête. **[Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway)** dispatche nativement, latence négligeable. C'est ce que font Continue (autocomplete) et OpenCode quand ils savent quel modèle utiliser.
* **Routage sémantique** — le client demande le modèle virtuel **`MoM`** (_Mixture of Models_) et **[vLLM Semantic Router (Iris)](https://github.com/vllm-project/semantic-router)**, déployé en **sidecar HTTP classifier**, analyse le prompt pour choisir le bon modèle (coder, reasoner, guard). Coût : ~250-300ms de classification (le classifier mmBERT tourne sur CPU — aucune pression sur la VRAM des pods GPU), payé **uniquement** quand `MoM` est demandé.

C'est cette dualité qui fait la puissance de l'archi : un client qui sait ce qu'il veut paie zéro latence supplémentaire, un client générique (OpenWebUI) profite d'un routage intelligent automatique — sans jamais avoir à choisir entre les deux.

### Iris : le routage sémantique en bref

[**Iris (vLLM Semantic Router)**](https://github.com/vllm-project/semantic-router) est le projet open-source qui implémente la logique de **_Mixture of Models_** (`MoM`). C'est un service léger, déployé en sidecar à côté de l'AI Gateway : il intercepte les requêtes adressées au modèle virtuel `MoM`, analyse le prompt et choisit dynamiquement le modèle réel qui va y répondre.

Sous le capot, Iris s'appuie sur un classifier compact (~100M paramètres, dérivé de BERT, servi sur CPU — donc aucune pression sur la VRAM des GPUs) qui évalue le prompt selon **différents critères** : l'intention (code, raisonnement, multilingue…), la présence d'une éventuelle tentative de _jailbreak_, ou la détection de **données personnelles (PII)**. Selon le verdict, Iris route vers le modèle approprié — ou applique un _guardrail_ de sécurité dédié.

**L'intérêt** : les clients adressent un seul endpoint (`MoM`) et bénéficient d'un routage adapté à chaque prompt sans coder leur propre logique de sélection. Le coût (~250-300 ms de classification) est le seul _trade-off_, payé uniquement sur les requêtes adressées à `MoM`.

---

## :computer: Les clients

Une plateforme sans clients n'est qu'une démo de plomberie. Voyons les trois surfaces qui consomment notre stack.

### OpenWebUI : le chat web

[**OpenWebUI**](https://openwebui.com/) est probablement le standard de facto pour exposer une UI de chat type ChatGPT au-dessus d'une API OpenAI-compatible. Déployé à `chat.priv.cloud.ogenki.io` (toujours via Tailscale).

<video controls width="100%" preload="metadata" muted>
  <source src="openwebui-screencast.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la lecture vidéo.
</video>

Dans le dropdown de modèles, on retrouve les `xplane-*` exposés par Iris (grâce à `include_config_models_in_list: true`) plus le modèle virtuel `MoM`. Quand on sélectionne `MoM`, c'est Iris qui décide en lisant le prompt. Quand on sélectionne un modèle précis, on bypass Iris et on économise les ~250ms du classifier.

Cas d'usage typiques : chat exploratoire, tests rapides de modèles, accès non-dev (pour quelqu'un qui n'utilise pas le terminal). Le response header `x-vsr-selected-model` permet de vérifier en direct ce qui a réellement servi la requête.

### Continue VSCode : l'IDE

[**Continue**](https://continue.dev/) est l'extension VSCode/JetBrains la plus mature pour brancher une API OpenAI-compatible sur un IDE. Configuration minimale dans `~/.continue/config.yaml` :

```yaml
models:
  - name: Qwen Coder (chat / agentic)
    provider: openai
    model: xplane-qwen-coder
    apiBase: https://llm.priv.cloud.ogenki.io/v1
    apiKey: ${env:OPENAI_API_KEY}  # pragma: allowlist secret
    roles:
      - chat
      - edit
      - apply
    defaultCompletionOptions:
      maxTokens: 2048
      temperature: 0.2

  - name: Qwen Coder FIM (autocomplete)
    provider: openai
    model: xplane-qwen-coder-fim
    apiBase: https://llm.priv.cloud.ogenki.io/v1
    apiKey: ${env:OPENAI_API_KEY}  # pragma: allowlist secret
    roles:
      - autocomplete
    template: qwen
    defaultCompletionOptions:
      maxTokens: 256
      temperature: 0.0
```

{{% notice tip "Le FIM (_Fill-In-the-Middle_) : pourquoi un modèle Base ?" %}}
Le FIM est ce qui fait la différence entre une **autocomplete réactive (<200ms)** et un truc lourdingue. Continue envoie un prompt structuré avec les tokens spéciaux `<|fim_prefix|>...<|fim_suffix|>...<|fim_middle|>`, et le modèle complète le trou — c'est très différent d'un chat classique.

**Le Base (et non l'Instruct) est obligatoire** : les versions Instruct sont fine-tunées sur des conversations, ce qui dilue les tokens FIM appris pendant le pré-entraînement. Utiliser un Instruct ici, c'est obtenir des complétions souvent off-topic ou bavardes.
{{% /notice %}}

{{% notice warning "Le piège de `template: qwen`" %}}
Sans `template: qwen`, Continue envoie les prompts FIM au format CodeLlama (`<PRE>... <SUF>... <MID>`) — totalement différent du format Qwen (`<|fim_prefix|>...<|fim_suffix|>...<|fim_middle|>`).

Conséquence : le modèle ne parse pas les tokens et l'autocomplete devient soit muet, soit pollué par des bouts du prompt. C'est l'erreur la plus commune quand on connecte Continue à un Qwen-Coder.
{{% /notice %}}

Avec ça, on a un autocomplete <200ms p95 (grâce au prefix cache et au pod toujours warm) et un chat qui fonctionne pour les tâches non-agentiques (génération de fonction, refactor d'un bloc, explication de code).

### OpenCode CLI : l'agent loop

[**OpenCode**](https://opencode.ai/) est ma surface principale dans cette stack — c'est le client qui se rapproche le plus de l'expérience Claude Code, en open source.

<video controls width="100%" preload="metadata" muted>
  <source src="opencode-screencast.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la lecture vidéo.
</video>

#### Pourquoi OpenCode et pas autre chose

La **vraie raison** : OpenCode publie un _compatibility shim_ explicite avec Claude Code. Les primitives sont quasi-isomorphes :

| Claude Code | OpenCode |
|---|---|
| `CLAUDE.md` | `AGENTS.md` (le legacy filename est aussi reconnu) |
| `~/.claude/skills/<name>/SKILL.md` | `skills/<name>/SKILL.md` (compat shim) |
| `~/.claude/agents/<name>.md` | `agent/<name>.md` (`mode: subagent`) |
| `~/.claude/commands/<name>.md` | `command/<name>.md` |
| `settings.json` MCP block | `opencode.json` `mcp` block (per-agent glob scoping) |

**Tout ce qu'on a construit côté Claude Code se porte directement.** C'est ça qui fait la différence par rapport à Aider, Crush ou Continue agent mode — on ne réécrit pas son workflow, on le branche sur un autre backend.

#### La config concrète

Voici l'`opencode.json` que j'utilise (extrait minimal) :

<details>
<summary><strong>Voir la config <code>opencode.json</code></strong></summary>

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "self-hosted": {
      "name": "Self-hosted (cloud-native-ref LLM platform)",
      "type": "openai",
      "options": {
        "baseURL": "https://llm.priv.cloud.ogenki.io/v1",
        "apiKey": "{env:OPENAI_API_KEY}"
      },
      "models": {
        "xplane-qwen-coder": {
          "name": "Qwen2.5-Coder-7B-Instruct (RECOMMENDED for coding)",
          "limit": { "context": 32768, "output": 4096 }
        },
        "xplane-qwen3-8b": {
          "name": "Qwen3-8B (reasoning, multilingual, planning)",
          "limit": { "context": 32768, "output": 4096 }
        }
      }
    }
  },
  "agent": {
    "build": { "model": "self-hosted/xplane-qwen-coder" },
    "plan":  { "model": "self-hosted/xplane-qwen3-8b" }
  }
}
```

</details>

Deux choses à noter :

* **Routing par-agent** : l'agent `build` utilise le coder dédié, l'agent `plan` utilise le modèle reasoning. C'est exactement le pattern qui fait que chaque sous-tâche utilise le bon outil.
* **`limit.context` et `limit.output`** explicites : sans eux, OpenCode peut envoyer des prompts qui dépassent la `maxModelLen` de vLLM, qui retourne alors une erreur cryptique du type _"maximum context length is X tokens, however you requested Y tokens"_. Mieux vaut prévenir.

{{% notice info "Ce qu'on garde, ce qu'on perd" %}}
**On garde** : la portabilité des skills/agents/commands/SDD/MCPs, l'expérience CLI, le workflow `/clear` `/compact`, la philosophie générale.

**On perd** : la qualité brute de Sonnet 4.6 / Opus 4.7 sur les tâches agentiques complexes (raisonnement multi-fichier, tool-use sur 5+ étapes). Le function-calling de Qwen2.5-Coder-7B fonctionne, mais il est fragile au-delà de quelques étapes — c'est connu et documenté dans le code de la plateforme.
{{% /notice %}}

---

## :bar_chart: Supervision et évaluation continue

Une stack qu'on déploie sans la mesurer, c'est un POC. Voyons comment on observe la nôtre.

### Le dashboard Grafana

Un dashboard dédié `LLM Platform` est committé dans le repo (`apps/base/ai/llm/grafana-dashboard.yaml`) et déployé via `Grafana Operator`. Il agrège les métriques scrapées par VictoriaMetrics depuis les endpoints `/metrics` de chaque pod vLLM.

{{< img src="grafana-dashboard-llm.png" alt="Dashboard Grafana — vue d'ensemble de la fleet LLM" width="1200" >}}

On y retrouve les indicateurs clés :

* **Fleet at a glance** : nombre de modèles actifs, requêtes en cours, taux de requêtes, utilisation moyenne du KV cache
* **Scale & queue** : nombre de replicas vLLM par modèle (les triggers KEDA en action), profondeur de la file d'attente
* **Throughput** : tokens/seconde par modèle (prompt vs génération) — utile pour comparer les modèles entre eux

Sur le screenshot ci-dessus, la stack est au repos : 4 modèles actifs (un replica chacun), 0 requête en cours, KV cache à 0.01%. C'est l'état "always warm" sans charge — exactement ce qu'on veut entre deux pics d'usage.

### SLOs et alerting

Le fichier `vmrule-llm-slo.yaml` définit les règles VictoriaMetrics qui matérialisent les SLOs :

* Latence p95 par modèle (alerte si dépassement seuil)
* Taux d'erreur (alerte si > 1% sur 5 min)
* Saturation GPU (warning à 80%, critical à 95%)

Une particularité à garder en tête quand on supervise une plateforme LLM : **le volume des métriques scale avec le débit de tokens, pas avec le nombre de requêtes**, et la cardinalité des labels FinOps (tenant, modèle, route, agent) explose plus vite que sur un service web classique. C'est précisément le retour qu'en fait [Samuel Desseaux à propos de VictoriaMetrics et de l'agentic AI](https://www.linkedin.com/posts/sdesseaux_ai-observability-sre-share-7458572389568671745-D_pU) : le déplacement de "service is down" vers "tenant X a dépassé son budget horaire" change la nature même des alertes. VictoriaMetrics absorbe ça nativement (multi-tenancy au stockage, agrégation in-flight, ingestion OTLP) — ce qui justifie de le poser dès le _day 1_ plutôt que de migrer plus tard.

Le tout s'intègre au stack alerting global de la plateforme — décrit dans l'[article observability/alerting](/fr/post/series/observability/alerts/).

### Évaluation continue avec Promptfoo

Un sujet qu'on aborde rarement dans les blogs LLM-self-hosted : **comment savoir si la qualité régresse** quand on bump un modèle ou tweak la cascade SR ?

[**Promptfoo**](https://www.promptfoo.dev/) permet de définir une suite d'évaluations déclarative (prompts d'entrée + assertions sur la sortie) et de la faire tourner sur n'importe quel provider OpenAI-compatible. Dans notre stack, c'est packagé comme un **CronJob nightly** (`tooling/base/promptfoo/`) :

* La suite d'eval est déclarée en `ConfigMap`
* Le CronJob exécute Promptfoo une fois par nuit contre les 4 modèles + le routage MoM
* Les résultats sont parsés et **poussés en métriques Prometheus** (donc visualisables dans Grafana)

Cas d'usage typiques :

* **Catching de régressions** sur un bump de modèle ou de version vLLM
* **Comparaison cascade vs modèle direct** : est-ce que le routage MoM dégrade la qualité par rapport à un appel direct ?
* **Stress-test du tool-calling séquentiel** : combien d'appels d'outils consécutifs Qwen2.5-Coder-7B peut-il enchaîner sans se planter ?

Soyons honnêtes : les résultats absolus restent **en deçà** d'un Sonnet 4.6 ou d'un Opus 4.7. L'eval continue ne sert pas à se rassurer dans l'absolu, mais à mesurer **l'évolution dans le temps** et à éviter les régressions silencieuses. *C'est ce qui me permet de quantifier ce que j'affirme dans la conclusion.*

---

## :balance_scale: Bilan honnête

Trois mois après les premiers déploiements, voici ce qui marche, ce qui ne marche pas, et ce que ça coûte.

### Ce qui marche bien

* **Souveraineté complète** : pas un octet ne quitte AWS / Tailscale. Pour les données confidentielles, c'est non-négociable.
* **Choix des modèles et de leurs rôles** : la composition Crossplane rend trivial le swap d'un modèle. C'est presque trop facile.
* **FIM tab-complete <200ms** : indistinguable d'un Copilot ou d'un Continue avec backend Anthropic — l'expérience est là.
* **Cascade SR fonctionnelle, observable, déboggable** : le header `x-vsr-selected-model` permet de comprendre n'importe quelle décision a posteriori.
* **Évaluation continue automatisée** : Promptfoo en CronJob, c'est un signal fort de maturité opérationnelle.

### Ce qui ne suffit pas (encore)

* **Qualité Qwen2.5-Coder-7B vs Sonnet 4.6** : il y a un **gap réel** sur les tâches agentiques complexes (raisonnement multi-fichier, tool-use sur 5+ étapes, refactoring large). Le 7B se plante régulièrement là où Sonnet rebondit.
* **L40S indispo en eu-west-3** : on est bloqués sur L4, donc bloqués sous Qwen3-Coder-30B-A3B-FP8. Le modèle qui rapprocherait du frontier est… inaccessible géographiquement.
* **Function-calling fragile au-delà de quelques étapes** : Qwen2.5-Coder + parser Hermes, ça marche pour 2-3 outils consécutifs, ça casse souvent au-delà.
* **Latence du classifier SR (~300ms)** sur chaque requête `MoM` : OK pour du chat, sensible en agent loop.
* **Sensibilité à la quantification plus aggressive** : descendre du fp8 à l'INT4 ferait gagner ~50% de VRAM, mais dégrade le raisonnement et la fiabilité du tool-calling structuré de façon non-linéaire — c'est une voie à n'envisager qu'avec un harnais d'évaluation solide en parallèle[^quant-tradeoff]. Le fp8 reste à mon sens le sweet spot 2026 pour de l'inférence interactive.
* **Overhead VRAM réel sous-estimé** : la VRAM "théorique" d'un modèle (poids × bytes/param) ignore l'overhead KV cache, activations et padding du framework. Compter **+10 à +20%** au-delà du chiffre marketing — un modèle annoncé "fits in 24GB" peut très bien saturer un L4 dès que le contexte grandit[^vram-overhead].

[^quant-tradeoff]: Voir [Self-Hosted LLMs in the Real World — KDnuggets](https://www.kdnuggets.com/self-hosted-llms-in-the-real-world-limits-workarounds-and-hard-lessons) sur les trade-offs INT4 vs FP8 et les pièges de la quantification aggressive.
[^vram-overhead]: Détaillé dans le [Self-Hosted LLM Leaderboard d'Onyx](https://onyx.app/self-hosted-llm-leaderboard) — une heuristique utile à garder en tête avant de dimensionner un NodePool Karpenter.

### Coûts réels

Aux prix spot AWS région Paris en mai 2026 :

| Poste | Coût |
|---|---|
| L4 spot (`g6.xlarge`) — baseline (FIM toujours warm + autres modèles scalés dynamiquement) | ~$220/mois |
| Pics de charge (scale-up des autres modèles sous trafic) | +$0.30 à $1.20/h selon usage |
| S3 Files (poids modèles, ~30GB) | <$1/mois |
| EBS, network, overhead K8s | ~$5/mois |
| **Steady-state typique** | **~$240-280/mois** |

À comparer avec un abonnement Claude Code Pro à $200/mois pour 1 utilisateur, ou les coûts API à l'usage (Sonnet 4.6 : $3/M input + $15/M output)… **la stack devient économiquement pertinente seulement** dans deux scénarios : (1) usage très soutenu par plusieurs développeurs simultanés, (2) la souveraineté est non-négociable et le coût n'est pas le critère premier.

### À qui ça parle aujourd'hui

* **Équipes avec contraintes de confidentialité / souveraineté fortes** (santé, défense, finance) — c'est *le* cas d'usage clair
* **Plateformes internes mutualisées** (R&D LLM, formation, MCP customs)
* **Curieux qui veulent comprendre l'inférence LLM "from the inside"** — c'est aussi mon cas

### À qui ça ne parle pas (encore)

* **Développeurs solo** qui veulent juste remplacer Claude Code pour économiser
* **Équipes qui exigent la qualité Sonnet/Opus sur les workflows agentiques complexes** — il faudra encore quelques itérations open-weight pour combler le gap

---

## :thought_balloon: Dernières remarques

À moins d'utiliser de puissants modèles qui nécessitent du matériel spécifique très cher, il est difficile d'obtenir aujourd'hui une expérience *ne serait-ce que satisfaisante* en self-hosted. Avec les modèles utilisés dans cette démo, on a parfois l'impression de revenir à la préhistoire 😅.

**Mais cette expérimentation démontre la faisabilité** et permet d'**anticiper les bouleversements à venir**.

Le 24 avril 2026, **DeepSeek a annoncé son modèle V4** : un MoE 284B (13B actifs) baptisé **V4 Flash**, et un **V4-Pro** à 1.6T paramètres (49B actifs). Et surtout, **deux annonces qui changent la donne** :

* Le modèle tourne sur **GPUs Huawei Ascend 950** (en plus des Nvidia), avec un _Day 0 adaptation_ confirmé par Huawei, Cambricon et Hygon — la dépendance Nvidia recule. Un détail qui n'est pas anodin : Jensen Huang lui-même [a qualifié de "horrible outcome"](https://thenextweb.com/news/nvidia-huang-deepseek-huawei-chips-horrible-outcome) la perspective de DeepSeek tournant sur Huawei.
* Côté tarifs, DeepSeek V4-Pro est **environ 7× moins cher** que Claude Opus 4.7 et **6× moins cher** que GPT-5.5 sur les coûts API standards ([source VentureBeat](https://venturebeat.com/technology/deepseek-v4-arrives-with-near-state-of-the-art-intelligence-at-1-6th-the-cost-of-opus-4-7-gpt-5-5)). Sur les usages cachés, le ratio passe à des ordres de grandeur encore plus extrêmes.

Couplez ça à la baisse continue du prix matériel (les RTX A4000 deviennent abordables, les futurs B100 vont bouleverser le segment data-center) et le calcul change : **dans 12 à 18 mois**, l'option self-hosted pourrait devenir économiquement compétitive même hors souveraineté pure.

D'ici là ? La fondation est posée, le pattern est en place, les lignes de bascule sont identifiées. Quand un Qwen3-Coder-30B-A3B tourne correctement sur un L4 quantifié AWQ-4bit (chemin documenté dans [`docs/llm-platform-future-paths.md`](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/docs/llm-platform-future-paths.md)), le swap sera une PR de quelques lignes. C'est l'**intérêt principal** de cette démo : se mettre en position d'**adopter rapidement** ce qui s'annonce, plutôt que d'avoir à tout (re)construire le jour où l'open-weight rattrape le frontier.

### Verdict

Self-hosted aujourd'hui : **alternative crédible** pour la souveraineté et la confidentialité, **fondation solide** pour absorber les modèles de demain, **pas un remplaçant universel** pour Claude Code sur les tâches agentiques complexes — pas encore.

Et en attendant que Qwen3-Coder-30B ou DeepSeek V4 distillé tournent sur nos L4, je continue à utiliser Claude Code au quotidien. Sans renier l'investissement self-hosted : c'est un *complément*, pas un *substitut*. Pour les données sensibles, les expérimentations, les tâches non-critiques — la stack open-weight tourne. Pour le reste, Sonnet et Opus restent imbattables.

L'avenir, lui, est résolument hybride.

**Petite mise en abyme pour finir** : l'ensemble de cette stack a été conçu et construit avec **Claude Code** 🙃.

---

## :bookmark: Références

### Repos
- [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — La plateforme complète
- [`docs/decisions/`](https://github.com/Smana/cloud-native-ref/tree/main/docs/decisions) — ADRs (vLLM Production Stack, S3 Files…)
- [`docs/llm-platform-future-paths.md`](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/docs/llm-platform-future-paths.md) — Chemins d'évolution

### Composants techniques
- [vLLM Production Stack](https://github.com/vllm-project/production-stack) — Inférence LLM en production
- [vLLM Semantic Router (Iris)](https://github.com/vllm-project/semantic-router) — Routage intelligent multi-modèles
- [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway) — Gateway API pour LLM
- [KEDA HTTP add-on](https://kedify.io/keda-http-add-on) — Scale-from-zero HTTP
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
