+++
author = "Smaine Kahlouch"
title = "Self-hosted LLM stack : quand l'abstraction grandit"
date = "2026-07-12"
summary = "Deux mois après avoir posé les fondations, l'abstraction `InferenceService` a révélé quatre manques : deux sources de vérité, aucune stratégie de rollout, aucune échappatoire, aucune découvrabilité. Voici comment on les a comblés — canary LoRA à coût GPU nul compris."
featured = true
codeMaxLines = 30
usePageBundles = true
toc = true
draft = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "vllm", "crossplane", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — Partie 4" %}}
Cet article fait suite à [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (partie 3), qui décrit la plateforme sur laquelle tout ce qui suit s'appuie. **Ici, on la fait vivre** — et on répare ce qui a cassé.
{{% /notice %}}

Ajouter un modèle à la plateforme, c'est un fichier YAML et un `git push`. Le pod démarre, les poids se chargent, la ressource passe `READY`. Et le client, lui, peut très bien prendre un `404` — parce que la **route** qui expose ce modèle sur la Gateway ne vit pas dans ce fichier. Elle vit dans un autre fichier, à côté, écrit à la main.

Ce n'est pas un bug : c'est exactement le contrat que j'ai livré dans la [partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/), il y a deux mois. La fondation, elle, a tenu — la **claim** (le manifeste YAML par lequel un utilisateur déclare *ce qu'il veut*, à charge pour la plateforme de produire les ressources Kubernetes correspondantes) fait toujours le job. Mais **la faire vivre** au quotidien a fait remonter quatre manques qui ne sont pas des détails d'implémentation : ce sont des trous dans l'abstraction elle-même.

Pour que cet article se lise seul, voici ce que la partie 3 a livré — servir un modèle open-weight sur la plateforme tenait dans ces quelques lignes :

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
    revision: c03e6d358207e414f1eca0bb1891e29f1db0e242
    quantization: fp8
    contextWindow: 32768
    toolCallParser: hermes
  gpu:
    count: 1
  scaling:
    minReplicas: 1
    maxReplicas: 2
```

De cette claim, la **composition** Crossplane — le programme qui traduit la claim en ressources concrètes — tire le `Deployment` vLLM, le `Service`, le Job de préchargement des poids, les `CiliumNetworkPolicy`, le `ScaledObject` KEDA et les règles d'alerte. Une dizaine de ressources à partir d'une vingtaine de lignes. Ce qu'elle n'en tire pas, c'est **la manière dont on atteint le modèle**.

## :dart: Objectifs

* Comprendre pourquoi une abstraction de plateforme doit **posséder le routage** qu'elle génère — et ce qui casse quand elle ne le possède pas
* Voir un **canary LoRA** déplacer une fraction du trafic vers un fine-tune, **à coût GPU nul** : même pod, même GPU, aucun réplica supplémentaire
* Comprendre ce qu'est une **échappatoire encadrée** : laisser passer l'imprévu sans laisser trouer le contrat de service
* Rendre la topologie **découvrable** : un `kubectl get` doit dire à quels noms le modèle répond réellement
* En tirer **quatre leçons transposables** à n'importe quelle abstraction de plateforme — LLM ou pas

{{% notice tip "Le code" %}}
Tout ce qui suit est dans [**cloud-native-ref**](https://github.com/Smana/cloud-native-ref) (PR #1559). La composition `InferenceService` passe de la **v0.6.0** (celle de la partie 3, du 9 mai 2026) à la **v0.8.0**.
{{% /notice %}}

---

## :mag: Ce qui clochait

Deux mois d'usage, quatre manques. Aucun n'est un incident spectaculaire — ce sont des frictions qui reviennent, et qui ont toutes la même origine : l'abstraction modélisait **le pod**, pas **le service rendu**.

### 1. Deux sources de vérité

La claim déclare *quel* modèle tourne. Un second fichier, `apps/base/ai/llm/ai-gateway-routes/route.yaml`, déclare *comment* on l'atteint : c'est lui qui dit à l'**Envoy AI Gateway** (le point d'entrée unique de la plateforme, compatible API OpenAI) que `model: xplane-qwen-coder` doit atterrir sur tel backend. Et ce fichier était **écrit à la main**.

Autant l'assumer franchement : la partie 3 listait l'`AIGatewayRoute` parmi les ressources générées par la composition. C'était vrai de l'intention, pas du code.

| Où c'est déclaré | Quoi | Qui l'écrit |
|---|---|---|
| `apps/base/ai/llm/qwen-coder.yaml` (la claim) | *quel* modèle tourne | l'utilisateur, via l'abstraction |
| `apps/base/ai/llm/ai-gateway-routes/route.yaml` | *comment* on l'atteint | l'utilisateur, à la main |

Deux fichiers qu'il faut penser à garder synchronisés, donc deux dérives possibles, et aucune des deux ne fait de bruit :

* **Un modèle sans route** — le pod tourne, le GPU est facturé, et la Gateway répond `404` à qui le demande.
* **Une route qui survit à son modèle** — la claim est supprimée, l'entrée de routage reste : un *backend fantôme* vers un `Service` qui n'existe plus.

### 2. Aucune stratégie de rollout

La bascule d'un modèle est **binaire**. On change `repository` et `revision`, Flux réconcilie, et 100 % du trafic part sur les nouveaux poids. Aucun palier, aucune fraction, aucun retour arrière progressif.

C'est d'autant plus dommage que la composition savait déjà charger des **adapters LoRA** (_Low-Rank Adaptation_ : un fine-tune léger, quelques dizaines de Mo, chargé **par-dessus** les poids du modèle de base, dans le même pod vLLM). La partie 3 n'en a pas dit un mot — et pour cause : un adapter comme `xplane-qwen-coder-sql-dpo` ou `xplane-qwen-coder-securecode` n'était joignable que par un client qui le **nommait explicitement**. Impossible de déplacer 10 % du trafic du modèle de base vers son fine-tune pour le valider sur du trafic réel. Un fine-tune que personne ne nomme est un fine-tune que personne n'évalue.

### 3. Aucune échappatoire

Le schéma de la claim expose des champs **curés** : `quantization`, `contextWindow`, `toolCallParser`… Tout ce qui n'y figure pas est hors d'atteinte. Vouloir essayer un flag vLLM une après-midi coûtait donc : une PR sur la composition KCL, les tests unitaires, la republication de l'image OCI, le bump du tag dans la `Composition`, et enfin Flux. Une release de plateforme pour un essai.

Le réflexe inverse — un champ fourre-tout non contraint — est pire : il laisse l'utilisateur écraser `--served-model-name` ou `--port`, c'est-à-dire casser silencieusement le contrat sur lequel reposent le `Service`, les probes et la route. Faute de savoir encadrer, j'avais choisi d'interdire.

### 4. Aucune découvrabilité

`kubectl get isvc` dit qu'un modèle est prêt. Il ne dit pas **à quels noms il répond**. Pour répondre à une question aussi élémentaire que « qu'est-ce que ce pod sert, au juste ? », il fallait ouvrir le fichier de routage, le recouper avec les arguments vLLM du `Deployment`, et reconstituer la topologie à la main. Rétro-ingénierer sa propre plateforme pour savoir ce qu'elle expose : ce n'est plus une abstraction, c'est une devinette.

---

Chacun de ces manques est comblé dans une section de cet article :

| Le manque | Ce qui le comble |
|---|---|
| Deux sources de vérité | La composition **possède** le routage (`gateway.enabled`) |
| Aucune stratégie de rollout | Des **canaries LoRA** pondérés (`gateway.canaries`) |
| Aucune échappatoire | `spec.engineArgs`, encadré à l'admission |
| Aucune découvrabilité | `status.servedModels` et une colonne `SERVED MODELS` |

Mais ils ont tous la même racine, et c'est la thèse de cet article : **l'unité de déploiement n'est plus « un pod qui sert un modèle »**. Ce que l'on déploie, c'est un modèle de base, *ses adapters*, *les noms sous lesquels on les atteint*, et *la fraction de trafic qui va à chacun*. Tant que l'abstraction ne modélise que le pod, tout le reste retombe sur l'humain — dans un fichier à côté, à la main.

## :door: Le routage rejoint le modèle

## :dna: LoRA en deux minutes

## :hatching_chick: Canary : 10 % du trafic, zéro GPU

## :unlock: L'échappatoire `engineArgs`

## :eyes: Le modèle dit ce qu'il sert

## :bar_chart: Mesurer le canary

## :compass: Endpoint Picker : au-delà du round-robin

## :telescope: Modelplane : évolution convergente

## :thought_balloon: Dernières remarques

## :bookmark: Références
