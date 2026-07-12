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

Le second fichier n'existe plus. Un champ dans la claim, et la route naît, vit et meurt avec le modèle :

```diff
# apps/base/ai/llm/qwen-coder.yaml
 spec:
   model:
     repository: Qwen/Qwen2.5-Coder-7B-Instruct
     revision: c03e6d358207e414f1eca0bb1891e29f1db0e242
     quantization: fp8
     contextWindow: 32768
     toolCallParser: hermes
   gpu:
     count: 1
+  gateway:
+    enabled: true
   scaling:
     minReplicas: 1
     maxReplicas: 2
```

Avec `gateway.enabled: true`, la composition ne rend plus seulement le pod : elle rend aussi les trois ressources qui décrivent **comment on l'atteint**, une série complète par claim.

| Ressource générée | Ce qu'elle déclare |
|---|---|
| **`Backend`** | l'adresse réelle du modèle : le FQDN du `Service` de la claim, port `8000` |
| **`AIServiceBackend`** | le *dialecte* que parle ce backend : le schéma OpenAI |
| **`AIGatewayRoute`** | la règle de dispatch — quel `model:` atterrit sur quel backend — rattachée à la Gateway (`parentRef: ai-gateway/envoy-ai-gateway-system`) |

Ces trois objets sont désormais des **ressources composées** : Crossplane les possède, exactement comme il possède déjà le `Deployment` ou le `ScaledObject`. Elles portent l'`ownerReference` de la claim, donc **le garbage collector de Kubernetes les emporte avec elle**. Supprimer un modèle supprime sa route — non pas parce qu'on y a pensé, mais parce qu'il n'existe aucun chemin où la route survive à son propriétaire. Le *backend fantôme* de la section précédente devient structurellement impossible, et la dérive inverse aussi : il n'y a plus deux fichiers à garder synchronisés, il n'y en a plus qu'un.

<!-- TODO-e2e: schéma avant/après de la propriété du routage (tâche 8) -->

### Le latch de readiness

Générer la route depuis la claim déplace toutefois une question qui, auparavant, était réglée à la main : **quand** la route doit-elle exister ? Deux échecs symétriques guettent, et ils font mal de façons opposées.

* **Une route qui apparaît trop tôt.** La claim est appliquée, l'`AIGatewayRoute` est publiée dans la foulée — mais le premier réplica vLLM charge encore ses poids. La Gateway route alors vers un backend sans endpoint prêt, et le client prend un `404` ou un `503` en pleine face, sur un modèle que la plateforme prétend servir.
* **Une route qui disparaît trop vite.** Le réflexe inverse — retirer la route dès que le `Deployment` n'est plus disponible — transforme la moindre indisponibilité passagère (un rolling update, un pod évincé, un nœud GPU recyclé par Karpenter) en *flapping* : la route sort de la config Envoy, y revient trente secondes plus tard, et le client voit alterner « modèle inconnu » et « modèle disponible ».

Le compromis retenu est asymétrique, parce que les deux erreurs ne coûtent pas le même prix. À la création, la route est **retenue** tant que le `Deployment` n'a pas atteint la condition `Available=True` : pas de route tant qu'aucun réplica ne peut répondre. Une fois publiée, en revanche, elle **ne sera plus jamais retirée**. C'est ce que j'appelle le **latch de readiness** : un portail à l'entrée, puis un verrou qui reste enclenché. Une indisponibilité transitoire reste un problème de *santé du backend* — les retries et le health checking d'Envoy sont faits pour ça — pas un problème de *topologie de routage*.

Techniquement, le latch tient sur l'**`ocds`** (_observed composed resource state_) : l'état des ressources déjà composées, tel que Crossplane l'observe au début de chaque réconciliation. La composition y cherche sa propre `AIGatewayRoute` ; si elle l'y trouve, c'est que le portail s'est déjà ouvert, et elle la rend de nouveau sans même regarder l'état du `Deployment`.

{{% notice tip "Le détail qui aurait fini en incident" %}}
La composition **écrit** la route sous une clé de ressource, et la **relit** sous cette même clé au tour suivant. Deux chaînes de caractères, à deux endroits du code : si elles divergent — un renommage d'un côté seulement — la relecture ne trouve jamais rien, le latch ne s'enclenche jamais, et la route est retirée à la première indisponibilité passagère venue. Un bug invisible en test, qui ne se manifeste qu'en production, au pire moment.

D'où une contrainte volontairement bête : les clés de lecture et d'écriture dérivent toutes deux d'une **constante unique** (`_gatewayRouteSuffix`). Elles ne peuvent pas diverger — il n'y a plus qu'une seule chaîne à renommer.
{{% /notice %}}

### Une migration, pas un big bang

Le basculement se fait modèle par modèle. `xplane-qwen-coder` passe en `gateway.enabled: true`, et **dans le même commit**, ses entrées sont retirées de `apps/base/ai/llm/ai-gateway-routes/route.yaml`. Un seul commit, parce qu'un instant où les deux sources déclarent la même route serait exactement le désordre qu'on cherche à supprimer. Les trois autres modèles de la plateforme suivront dans une PR ultérieure.

{{% notice note "La double comptabilité n'est pas encore morte" %}}
Soyons précis sur l'état réel : **un modèle sur quatre** est passé sous routage possédé par la composition. `route.yaml` existe toujours, il porte encore les trois autres, et il faut donc toujours y penser pour eux. Ce qui a changé, ce n'est pas que le fichier a disparu — c'est qu'il a cessé d'être *le seul* endroit possible, et qu'il a maintenant une date de péremption.
{{% /notice %}}

Reste que la route appartient désormais à la claim, et c'est là que ça devient intéressant : une route que l'on possède est une route que l'on peut **pondérer**.

## :dna: LoRA en deux minutes

## :hatching_chick: Canary : 10 % du trafic, zéro GPU

## :unlock: L'échappatoire `engineArgs`

## :eyes: Le modèle dit ce qu'il sert

## :bar_chart: Mesurer le canary

## :compass: Endpoint Picker : au-delà du round-robin

## :telescope: Modelplane : évolution convergente

## :thought_balloon: Dernières remarques

## :bookmark: Références
