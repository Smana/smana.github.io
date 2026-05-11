+++
author = "Smaine Kahlouch"
title = "Self-Hosted LLM stack : poser les fondations d'une alternative à `Claude Code`"
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
Cet article fait suite à [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) (les fondamentaux des coding agents) et [Quelques mois avec Claude Code](/fr/post/series/agentic_ai/ai-coding-tips/) (les workflows découverts à l'usage). **Ici, on prend du recul** : et si on hébergeait notre propre stack LLM ? Que peut-on raisonnablement faire en self-hosted aujourd'hui ?
{{% /notice %}}

J'utilise désormais l'agentic coding de façon quotidienne, comme beaucoup d'entre nous. Je suis globalement satisfait de l'expérience avec Claude Code, mais je garde un œil attentif sur l'**écosystème open-weight**, qui est lui aussi en pleine effervescence : Qwen, Llama, Mistral, GLM, DeepSeek — et sur la bataille que se mènent les différents acteurs du secteur.

Je me suis alors lancé un petit défi : **comment pourrions-nous héberger nos propres LLMs dans notre infrastructure ?** Souveraineté des données, choix du modèle, indépendance vis-à-vis d'un fournisseur, et surtout — la curiosité technique de voir ce qui est aujourd'hui *vraiment* faisable avec les outils modernes à notre disposition.

{{% notice warning "Cet article est une **démonstration**" %}}
Soyons honnêtes : sur les tâches agentiques complexes, un modèle open-weight 7-8B joue dans une **autre catégorie** qu'un Sonnet 4.6 ou un Opus 4.7 — l'écart est considérable. Et les modèles qui *pourraient* s'en approcher (DeepSeek V4 Pro, Kimi K2.6) demandent un matériel hors de portée à titre personnel.

Cet article ne raconte pas une migration. Il **pose les fondations** d'une alternative à faire évoluer à mesure que l'écosystème rattrape son retard, et démontre ce qui est *déjà* possible aujourd'hui.
{{% /notice %}}

## :dart: Objectifs de cet article

* Comprendre les **briques d'une stack LLM self-hosted** moderne sur Kubernetes
* Découvrir les choix techniques retenus : modèles open-weight, **vLLM Production Stack**, routage intelligent, autoscaling sur signaux GPU
* Voir comment on consomme cette plateforme depuis trois clients : **OpenWebUI**, **Continue VSCode**, **OpenCode CLI**
* Évaluer honnêtement ce qui marche, ce qui ne marche pas, et ce que ça coûte vraiment

{{% notice tip "TL;DR" %}}
Une stack LLM self-hosted complète sur Kubernetes pour **~$250/mois** (spot L4), avec **autocomplete IDE <200ms**, routage intelligent multi-modèles (`MoM`), et observabilité de bout en bout. **Crédible** dès aujourd'hui pour les besoins de souveraineté et de confidentialité ; **pas encore** un remplaçant universel de Claude Code sur les tâches agentiques complexes — mais la fondation est posée pour absorber les modèles de demain en quelques lignes de YAML.
{{% /notice %}}

{{% notice tip "Le repo de référence" %}}
Toute la stack est déployée par GitOps depuis [**cloud-native-ref**](https://github.com/Smana/cloud-native-ref) — au-delà de la couche LLM, le repo illustre **un écosystème cloud-native complet** :

* **GitOps** — Flux, réconciliation continue
* **Platform engineering** — Crossplane (une claim YAML génère tout l'outillage Kubernetes)
* **Observabilité** — VictoriaMetrics, VictoriaLogs, Grafana
* **Sécurité & accès** — gestion des secrets + exposition privée via Tailscale

Les choix techniques sont justifiés dans les ADR ([`docs/decisions/`](https://github.com/Smana/cloud-native-ref/tree/main/docs/decisions)).
{{% /notice %}}

---

## :world_map: Vue d'ensemble de la stack

Voici la carte avant de plonger dans les détails. Trois couches : **modèles**, **plateforme** (inférence + routage + scaling), **clients**.

{{< img src="architecture-simple.png" alt="Stack LLM self-hosted — vue simplifiée : clients, Envoy AI Gateway, Iris semantic router, 4 modèles vLLM, plan de contrôle (Crossplane, KEDA, Karpenter, Flux), S3" width="1200" >}}

Tout est déployé par **GitOps** (Flux), exposé en privé via **Tailscale**, et chaque modèle est défini par une **claim Crossplane** unique (`InferenceService`) qui génère l'ensemble des ressources Kubernetes nécessaires. C'est cette abstraction qui permet de basculer un modèle en modifiant quelques champs YAML.

---

## :brain: Les modèles open-weight retenus

Avant de plonger dans la plomberie Kubernetes, le choix le plus structurant : **quels modèles déployer ?** Cette décision contraint tout le reste — taille de GPU, quantification, latence atteignable, et budget.

### L'écosystème en mai 2026

Ces derniers mois, la qualité des modèles open-weight a progressé **considérablement**. Sur **SWE-bench Verified**[^swebench], les deux meilleurs modèles open-weight sont désormais **DeepSeek V4 Pro (Max)** (80.6%) et **Kimi K2.6** (80.2%), à environ 7 points des modèles propriétaires de pointe (Claude Opus 4.7 à 87.6%, GPT-5.5 à 88.7%). Côté plus léger, **Qwen2.5-Coder** et **Qwen3-Coder** (Alibaba) restent des références pour la génération de code, et **Qwen3** brille en raisonnement multilingual.

[^swebench]: Verified est aujourd'hui considéré comme partiellement contaminé : OpenAI a [cessé de le reporter](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/) au profit de [SWE-bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro_public), où Kimi K2.6 mène les open-weight à 58.6%.

Mais entre "pouvoir lire le poids sur HuggingFace" et "le faire tourner sur du GPU **financièrement accessible**", il y a un monde. Les meilleurs modèles open-weight pour le coding demandent des configurations **multi-H100** ou **multi-H200**, et même les modèles intermédiaires comme **Qwen3-Coder-30B-A3B-FP8** réclament au minimum un **L40S 48GB** — quand notre _NodePool_ standard tourne sur des **L4 24GB**. Le facteur limitant ici n'est pas technique : c'est tout simplement le **coût**, que je ne peux pas supporter sur mes moyens personnels.

{{% notice info "Combien coûterait un DeepSeek V4 Pro ou un Kimi K2.6 ?" %}}
Pour fixer un ordre de grandeur : faire tourner un **DeepSeek V4 Pro** (1.6T paramètres, ~49B actifs en MoE) ou un **Kimi K2.6** (~1T total, ~32B actifs) en FP8 demande typiquement **8 à 16× H100 80GB** selon la quantification — l'équivalent d'une à deux instances AWS `p5.48xlarge` à **~55 $/heure on-demand** chacune (us-east-1, mai 2026), ou d'une `p5e.48xlarge` (8× H200 141GB) à un tarif similaire. Soit, en 24/7, un budget mensuel de l'ordre de **40 000 à 80 000 $**. Même en spot ou en _reserved_ sur un an, on reste largement au-dessus de **15 000 $/mois**.

À titre personnel, c'est évidemment hors de portée. La démo reste donc sur du L4 24GB et on choisit des modèles qui rentrent dedans — un compromis assumé, documenté comme [chemin d'évolution](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/docs/llm-platform-future-paths.md) dans le repo.
{{% /notice %}}

### Quatre modèles, quatre rôles

Chaque modèle a un rôle bien défini dans la stack. La quantification **fp8** est utilisée partout.

{{% notice info "C'est quoi la quantification ?" %}}
Un modèle de langage stocke ses **poids** sous forme de nombres décimaux. En précision native (**FP16** / **BF16**), chaque poids occupe **16 bits** (2 octets) — c'est précis, mais ça consomme beaucoup de VRAM.

La **quantification** consiste à représenter ces mêmes poids sur **moins de bits**. On arrondit les valeurs avec une précision numérique réduite, ce qui **divise la mémoire** et **accélère les calculs**. La contrepartie : les sorties du modèle (probabilités sur les tokens) **dérivent légèrement** de celles du modèle non quantifié — mesurable sur des benchmarks comme la perplexité, MMLU ou HumanEval.

| Format | Bits/param | Dérive vs FP16 | Usage typique |
|---|---|---|---|
| FP16 / BF16 | 16 | 0 (référence) | Précision native, entraînement |
| **FP8** | **8** | <1% (imperceptible) | Standard inférence moderne |
| FP4 / INT4 | 4 | 3-5% (visible) | Modèles ≥1T paramètres |

Le **FP8** divise la VRAM par 2 et est supporté nativement (au niveau hardware) par les GPU récents : **L4, L40S, H100, H200** — d'où son adoption généralisée en inférence.
{{% /notice %}}

| Modèle | Rôle | Taille | Pourquoi celui-là ? |
|---|---|---|---|
| **`xplane-qwen-coder`** | Code chat, agentic edits | Qwen2.5-Coder-7B-Instruct (fp8) | Native function-calling, top-tier 7B coding benchmarks, Apache 2.0 |
| **`xplane-qwen-coder-fim`** | Tab-complete IDE (FIM) | Qwen2.5-Coder-1.5B (fp8) | **Modèle Base** (les FIM tokens sont dilués dans la version Instruct), rapide, partage le tokenizer Qwen |
| **`xplane-qwen3-8b`** | Général, multilingual, raisonnement | Qwen3-8B (fp8) | `thinking_mode` togglable pour le raisonnement, 32k contexte, support multilingual fort |
| **`xplane-llamaguard3-1b`** | Pre-filter safety | Llama-Guard-3-1B (fp16) | Détection jailbreak / prompt injection, modèle dédié à cette tâche |

{{% notice tip "Le FIM (_Fill-In-the-Middle_) : pourquoi un modèle Base ?" %}}
Le FIM est ce qui fait la différence entre une **autocomplete réactive (<200ms)** et un truc lourdingue. Continue envoie un prompt structuré avec les tokens spéciaux `<|fim_prefix|>...<|fim_suffix|>...<|fim_middle|>`, et le modèle complète le trou — c'est très différent d'un chat classique.

**Le Base (et non l'Instruct) est obligatoire** : les versions Instruct sont fine-tunées sur des conversations, ce qui dilue les tokens FIM appris pendant le pré-entraînement. Utiliser un Instruct ici, c'est obtenir des complétions souvent off-topic ou bavardes.
{{% /notice %}}

Voici la claim Crossplane qui définit le coder principal — c'est l'**abstraction-clé** de la plateforme, on y reviendra :

<details>
<summary><strong>Voir la claim <code>InferenceService</code> complète</strong></summary>

```yaml
# apps/base/ai/llm/qwen-coder.yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: InferenceService
metadata:
  name: xplane-qwen-coder
  namespace: llm
spec:
  model:
    repository: Qwen/Qwen2.5-Coder-7B-Instruct
    revision: c03e6d358207e414f1eca0bb1891e29f1db0e242  # pragma: allowlist secret
    quantization: fp8
    contextWindow: 32768
    maxNumSeqs: 32
    toolCallParser: hermes
    preload:
      enabled: true
  gpu:
    count: 1
    minVRAM: 16Gi
  routing:
    tier: medium
    specialty: code
  scaling:
    minReplicas: 1
    maxReplicas: 2
  cache:
    kvOffload:
      enabled: true
      sizeGB: 16
    prefixCache:
      enabled: true
```

</details>

Quarante lignes de YAML qui génèrent : un HelmRelease vLLM, un ScaledObject KEDA, une AIGatewayRoute, des CiliumNetworkPolicy, et un Job de préchargement des poids. **Conséquence directe : pour basculer vers un autre modèle, il suffit d'adapter quelques champs du bloc `model:` (le `repository:`, sa `revision:`, parfois la `quantization:` ou le `toolCallParser:`), de commiter, et Flux reconfigure toute la stack.** Le jour où l'écosystème open-weight aura suffisamment progressé, la migration sera quasi-triviale.

À titre d'illustration, voici la PR-type d'une migration future (passer du Qwen2.5-Coder-7B actuel à un Qwen3-Coder-30B le jour où il rentre sur le matériel) :

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

---

## :twisted_rightwards_arrows: Le routage intelligent

Un seul modèle ne fait pas tout : **coder pour le code**, **reasoner pour les maths**, **guard pour la safety**. Il faut donc router intelligemment chaque requête vers le bon modèle.

### Deux modes de routage complémentaires

La porte d'entrée combine **deux mécanismes complémentaires**, mobilisés selon le besoin :

* **Routage explicite** — le client cible directement un modèle (`model: xplane-qwen-coder`) via le header `x-ai-eg-model` ou le body de la requête. **[Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway)** dispatche nativement, latence négligeable. C'est ce que font Continue (autocomplete) et OpenCode quand ils savent quel modèle utiliser.
* **Routage sémantique** — le client demande le modèle virtuel **`MoM`** (_Mixture of Models_) et **[vLLM Semantic Router (Iris)](https://github.com/vllm-project/semantic-router)**, déployé en **sidecar HTTP classifier**, analyse le prompt pour choisir le bon modèle (coder, reasoner, guard). Coût : ~250-300ms de classification (le classifier mmBERT tourne sur CPU — aucune pression sur la VRAM des pods GPU), payé **uniquement** quand `MoM` est demandé.

C'est cette dualité qui fait la puissance de l'archi : un client qui sait ce qu'il veut paie zéro latence supplémentaire, un client générique (OpenWebUI) profite d'un routage intelligent automatique — sans jamais avoir à choisir entre les deux.

### Comment Iris décide quoi router

**L'objectif** : exposer un endpoint unique (`MoM`) qui lit chaque prompt et choisit automatiquement le modèle le plus adapté — sans que le client ait à connaître l'inventaire des modèles ni leurs spécialités.

Iris analyse chaque prompt avec un **classifier mmBERT** (~100M paramètres, servi en local) entraîné sur les catégories **MMLU** (_Massive Multitask Language Understanding_), disponibles dans la collection HuggingFace [llm-semantic-router](https://huggingface.co/llm-semantic-router). Trois classifiers tournent en parallèle :

* **_intent classifier_** — détermine la catégorie du prompt (code, math, multilingual, general…)
* **_jailbreak detector_** — repère les tentatives de prompt injection
* **_PII detector_** — repère les données personnelles

Selon la catégorie détectée, le routage suit cette table de décision :

| Catégorie détectée | Modèle ciblé | Particularité |
|---|---|---|
| `code` | `xplane-qwen-coder` | Function-calling Hermes activé |
| `reasoning` (math, physics) | `xplane-qwen3-8b` | `thinking_mode: true` |
| `multilingual` | `xplane-qwen3-8b` | Pas de thinking_mode |
| `general` (fallback) | `xplane-qwen3-8b` | Pas de thinking_mode |
| `code_with_reasoning` | `xplane-qwen-coder` | `use_reasoning: true` (signal-fusion) |
| **Jailbreak détecté** | `xplane-llamaguard3-1b` | Pre-filter, refus immédiat |

Côté config, Iris se résume à déclarer le modèle par défaut et les endpoints des modèles ciblés.

<details>
<summary><strong>Voir l'extrait de config Iris (HelmRelease)</strong></summary>

```yaml
# infrastructure/base/vllm-semantic-router/helmrelease.yaml
config:
  default_model: xplane-qwen3-8b
  router:
    auto_model_name: MoM
    include_config_models_in_list: true

  vllm_endpoints:
    - name: qwen3-8b
      address: xplane-qwen3-8b.llm.svc.cluster.local
      port: 8000
    - name: qwen-coder
      address: xplane-qwen-coder.llm.svc.cluster.local
      port: 8000
    - name: qwen-coder-fim
      address: xplane-qwen-coder-fim.llm.svc.cluster.local
      port: 8000
    - name: llamaguard3-1b
      address: xplane-llamaguard3-1b.llm.svc.cluster.local
      port: 8000
```

</details>

### Vérifier le routage depuis l'extérieur

Iris ajoute un header `x-vsr-selected-model` à la réponse — ce qui permet de **rejouer ou debugger n'importe quelle décision** :

```bash
curl -sS -X POST https://llm.priv.cloud.ogenki.io/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -D - \
  -d '{
    "model": "MoM",
    "messages": [{"role":"user","content":"refactor this Go function for clarity"}]
  }' | grep -i x-vsr-selected
# x-vsr-selected-model: xplane-qwen-coder
```

Code → coder. Math → reasoner. Multilingual → général. Jailbreak → guardrail. Le tout transparent pour le client.

---

## :gear: La plateforme d'inférence sous le capot

Modèles choisis, routage défini — reste à les **servir efficacement**. C'est ici que les choix techniques (moteur d'inférence, GPU, stockage) décident de la latence, du throughput et, _in fine_, du coût.

### vLLM Production Stack

Pour servir les modèles, on utilise [**vLLM Production Stack**](https://github.com/vllm-project/production-stack) — la déclinaison "production" de vLLM, qui apporte un Helm chart, un router de requêtes, et l'intégration KV cache offload (LMCache). Le choix vs KServe est documenté dans l'[ADR 0003](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/docs/decisions/0003-vllm-production-stack-over-kserve.md) : performances natives, support fp8 mature, parser Hermes pour le function-calling.

Au-delà de la stack opérationnelle, vLLM apporte surtout son **continuous batching** et son **paged attention** : sur un même GPU, un serving naïf (HuggingFace Transformers ou Ollama default) plafonne typiquement à **3 à 10× moins de throughput**[^vllm-bench]. Pour de l'inférence multi-utilisateurs, c'est la différence entre un POC et une plateforme.

[^vllm-bench]: Multiplicateur observé dans plusieurs _benchmarks_ et _roundups_ de fin 2025 — voir par exemple [Best Open Source LLMs You Can Self-Host](https://brightseotools.com/post/best-open-source-llms-you-can-self-host).

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

Deux options méritent une mention :

* **Prefix caching** : crucial pour le FIM. Chaque keystroke partage le même préfixe (le contenu du fichier jusqu'au curseur) — le préfixe cache transforme chaque keystroke suivant en lookup O(1) du KV cache déjà calculé.
* **KV offload (LMCache)** : permet d'étendre la mémoire effective du KV cache au-delà de la VRAM en utilisant la RAM hôte. C'est ce qui permet de tenir 32 séquences concurrentes en 32k contexte sur un L4 de 24Gi.

### GPU sur Kubernetes : Karpenter et NVIDIA L4

{{% notice info "L'inférence LLM est _memory-bound_, pas _compute-bound_" %}}
Pour l'inférence d'un LLM, **le facteur limitant n'est pas la compute mais la bande passante mémoire**. Un modèle 7B en fp8 doit lire ~7 Go de poids *par token généré* — un L4 (300 Go/s) plafonne donc en théorie autour de **~40 tokens/s** en single-stream, **indépendamment de ses TFLOPS**.

C'est ce qui explique pourquoi :

* Les configurations **multi-H100** (3.35 To/s chacun) restent dominantes pour les très gros modèles
* La **quantification FP8** — qui divise par deux le volume à transférer — a un impact disproportionné sur le throughput perçu[^bandwidth]
{{% /notice %}}

[^bandwidth]: Voir [Every AI Chip on Earth Is Starving for Data](https://sanjeevganjihal.substack.com/p/every-ai-chip-on-earth-is-starving) — analyse détaillée de la chute du ratio bytes/FLOP entre générations de GPU (H100 → B200 → Vera Rubin) et de ses conséquences sur l'inférence.

Le NodePool dédié provisionne des instances **`g6.xlarge` (NVIDIA L4 24GB)** en spot, avec une politique de _consolidation_ ajustée :

```yaml
# Extrait du NodePool gpu-l4
spec:
  consolidationPolicy: WhenEmpty
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-gpu-count
          values: ["1"]   # Single-GPU SKUs only
        - key: karpenter.k8s.aws/instance-family
          values: ["g6"]
      taints:
        - key: nvidia.com/gpu
          effect: NoSchedule
```

Le `consolidationPolicy: WhenEmpty` (et non le défaut `WhenEmptyOrUnderutilized`) protège le pod FIM toujours warm — un noeud qui héberge ce pod n'est jamais consolidé tant que le pod y est. Sans ça, Karpenter pourrait évincer le pod par optimisation… et casser la latence FIM <200ms à chaque coup.

### Stockage des poids modèles : S3 Files

Charger ~15GB de poids depuis HuggingFace à chaque cold-start de pod, c'est lent et coûteux (bande passante egress). La solution retenue : [**Amazon S3 Files**](https://aws.amazon.com/blogs/aws/launching-s3-files-making-s3-buckets-accessible-as-file-systems/), une feature AWS **toute récente** (GA en **avril 2026**) qui expose un bucket S3 comme un **filesystem NFS conforme POSIX**, montable comme un volume Kubernetes partagé entre pods.

Les avantages dans notre cas :

* **Cold-start divisé** — un Job de préchargement télécharge les poids depuis HuggingFace une seule fois lors du premier déploiement ; chaque pod vLLM monte ensuite le volume directement, démarrage en quelques secondes
* **Multi-attach natif** — plusieurs pods (réplicas, modèles distincts) partagent le même bucket sans configuration particulière, contrairement à un PVC EBS
* **Cache intelligent** — préfetching automatique, latence ~1ms sur l'_active set_ (les poids fréquemment lus restent sur EFS sous le capot)
* **Tarification EFS** sur les données actives uniquement, et **tarif S3 standard** (~$0.023/GB/mois) sur le stockage long terme — bien moins cher qu'un PVC EBS provisionné en taille fixe
* **POSIX complet** — vLLM lit les poids comme depuis un disque local, sans adaptation

L'[ADR 0004](https://github.com/Smana/cloud-native-ref/blob/wip/self-hosted-llm-platform-draft/docs/decisions/0004-amazon-s3-files-for-model-weights-storage.md) détaille les arbitrages face aux alternatives (EBS multi-attach, EFS classique, Mountpoint for S3 — ce dernier étant un produit distinct, plus ancien, avec un POSIX partiel).

{{% notice note "Le tout premier remplissage reste lent" %}}
S3 Files n'est pas magique sur le _bootstrap_ initial : le Job de préchargement doit télécharger les ~15GB de poids depuis HuggingFace **une première fois** vers S3. Cette étape est limitée par la bande passante réseau de l'instance (~10 Gbps sur `g6.xlarge`) et prend typiquement plusieurs dizaines de secondes. Le gain s'amortit ensuite : tous les démarrages **suivants** (réplicas, redéploiements, autres modèles partageant le bucket) montent le volume en quelques secondes.
{{% /notice %}}

### L'abstraction-clé : la composition Crossplane

C'est probablement le pattern le plus important de toute la plateforme. Chaque modèle est défini par une **claim** `InferenceService` (l'XR Crossplane) — une trentaine de lignes de YAML — qui génère **toutes** les ressources Kubernetes nécessaires :

| Ressource générée | Rôle |
|---|---|
| `HelmRelease` (vLLM) | Le pod vLLM lui-même |
| `ScaledObject` (KEDA) | Autoscaling sur signaux Prometheus |
| `AIGatewayRoute` | Routage Envoy AI Gateway |
| `CiliumNetworkPolicy` | Zero-trust egress/ingress |
| `Job` (preload) | Téléchargement des poids vers S3 |
| `VMServiceScrape` | Scraping métriques par VictoriaMetrics |

C'est ce pattern d'**API Platform** — une interface simple cachant une stack riche — qui rend la plateforme évolutive. Quand un dérivé de DeepSeek V4 (ou un modèle équivalent) sera disponible en quantification adaptée à L4, une PR de quelques lignes dans `apps/base/ai/llm/qwen-coder.yaml` (essentiellement `model.repository` et sa `revision:`) suffira à basculer.

### Authentification : Bearer token + AWS Secrets Manager

L'**Envoy Gateway `SecurityPolicy`** vérifie un Bearer token sur chaque requête. Les clés sont stockées dans **AWS Secrets Manager** (`platform/llm/api-keys`) sous forme d'un objet JSON keyé par identité (`openwebui_apikey`, `opencode_apikey`, `developer_apikey`…), et synchronisées dans Kubernetes via **External Secrets Operator**.

```bash
# Récupérer sa clé personnelle
aws secretsmanager get-secret-value \
  --secret-id platform/llm/api-keys \
  --query SecretString --output text | jq -r .openwebui_apikey

# Et l'exporter dans l'environnement (jamais en dur dans une config)
export OPENAI_API_KEY=$(aws secretsmanager get-secret-value \
  --secret-id platform/llm/api-keys \
  --query SecretString --output text | jq -r .openwebui_apikey)
```

Côté réseau, **Cilium NetworkPolicies** appliquent un **zero-trust** strict (chaque pod ne peut parler qu'aux services explicitement autorisés), et l'exposition externe se fait uniquement via **Tailscale** (`tag:k8s` ACL) — aucune route publique.

---

## :zap: Scaling : anticiper la saturation, ou disparaître quand inutile

C'est probablement la section la plus intéressante techniquement, parce qu'elle illustre un choix produit non-trivial.

### Le réflexe initial : scale-to-zero

Un GPU L4 spot, c'est ~$0.30/h. Toujours warm = $200/mois par modèle. Naïvement, on se dit : **scale-to-zero**, plus de pods quand personne ne les utilise, première requête déclenche le scale-up.

Techniquement, c'est faisable proprement avec le **[KEDA HTTP add-on](https://kedify.io/keda-http-add-on)** : un `HTTPScaledObject` par modèle, l'AI Gateway route vers un _interceptor_ qui met la requête en file d'attente le temps que le pod se lance. Le _cold-start budget_ pour un 7B est d'environ **180 secondes** : provisioning du nœud Karpenter (60s) + pull image (30s) + load des poids depuis S3 (30s) + compilation cudagraph (30s) + first token (30s).

180 secondes d'attente sur la première requête d'une session, c'est… **lourd** pour un usage interactif. Le scale-to-zero trahit alors sa promesse : oui, c'est gratuit au repos, mais l'UX devient instantanément pénible.

### Le pivot : autoscaling sur signaux anticipateurs

La vraie réponse, c'est de garder les modèles **toujours warm** (`min=1`) et de **scale-up sur signaux anticipateurs** — avant que la file d'attente ne se forme. Concrètement, le `ScaledObject` KEDA s'appuie sur **deux triggers Prometheus** scrapés depuis les métriques vLLM :

```yaml
# Extrait de la composition Crossplane
triggers:
  - type: prometheus
    metadata:
      query: |
        max(
          vllm:num_requests_running{served_model_name="xplane-qwen-coder"}
        ) /
        max(
          vllm:num_requests_max{served_model_name="xplane-qwen-coder"}
        )
      threshold: "0.7"   # 70% saturation queue interne
  - type: prometheus
    metadata:
      query: |
        avg(vllm:gpu_cache_usage_perc{served_model_name="xplane-qwen-coder"})
      threshold: "0.85"  # 85% pression KV cache
```

* `vllm:num_requests_running / vllm:num_requests_max` mesure la **saturation interne** : combien de séquences sont en cours par rapport au max configuré (`maxNumSeqs: 32`)
* `vllm:gpu_cache_usage_perc` mesure la **pression sur le KV cache** : si on s'approche de 85%, c'est que les contextes longs commencent à manger toute la VRAM

Ces deux triggers en signal-fusion permettent à KEDA de scale `1→2` ou `1→3` **avant** que la file d'attente ne se forme — la latence first-request reste prévisible. C'est plus subtil que "j'ai 100% de CPU, je scale", mais c'est exactement ce qu'il faut pour de l'inférence LLM.

### Quand le scale-to-zero garde du sens

L'approche always-warm n'est pas universelle. Pour des cas particuliers — modèles rarement appelés (le `xplane-llamaguard3-1b` n'est invoqué qu'en cas de jailbreak suspect), environnements dev/staging, modèles expérimentaux — la composition supporte aussi `minReplicas: 0` avec le KEDA HTTP add-on :

```yaml
# Cas d'un modèle peu utilisé : OK pour scale-to-zero
spec:
  scaling:
    minReplicas: 0    # Le pod disparait quand inactif
    maxReplicas: 1
  # ... la composition génère automatiquement un HTTPScaledObject
```

{{% notice tip "Ce qu'on retient" %}}
Le choix entre scale-to-zero et always-warm n'est pas un **choix technique** mais un **choix de produit** :

* **UX interactive** (chat, IDE) → les triggers Prometheus anticipateurs battent la « magie » du scale-from-zero
* **Workloads batch / sporadiques** → l'inverse est vrai

Avoir les deux modes dans la même composition, c'est ce qui fait la différence entre une plateforme et un POC.
{{% /notice %}}

### Le rôle de Karpenter

Une dernière subtilité : KEDA scale les **replicas vLLM**, Karpenter scale les **nœuds GPU**. Quand un nouveau pod doit être planifié et qu'il n'y a pas de noeud GPU disponible, Karpenter en provisionne un automatiquement (cycle ~60s). À l'inverse, quand le `consolidationPolicy: WhenEmpty` du NodePool détecte qu'un noeud n'héberge plus de pod GPU, il le décommissionne. La couche **Kubernetes reste élastique même quand vLLM ne l'est pas** — c'est cette double élasticité qui rend la facture finale supportable.

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
