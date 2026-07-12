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

Ces trois objets sont désormais des **ressources composées** : Crossplane les possède, exactement comme il possède déjà le `Deployment` ou le `ScaledObject`. Elles portent l'`ownerReference` de la **ressource composite** — l'objet interne que Crossplane crée pour honorer la claim, et dont dépendent aussi le `Deployment` et les autres ressources générées — donc **le garbage collector de Kubernetes les emporte avec elle**. Supprimer un modèle supprime sa route — non pas parce qu'on y a pensé, mais parce qu'il n'existe aucun chemin où la route survive à son propriétaire. Le *backend fantôme* de la section précédente devient structurellement impossible, et la dérive inverse aussi : il n'y a plus deux fichiers à garder synchronisés, il n'y en a plus qu'un.

<!-- TODO-e2e: schéma avant/après de la propriété du routage (tâche 8) -->

### Le latch de readiness

Générer la route depuis la claim déplace toutefois une question qui, auparavant, était réglée à la main : **quand** la route doit-elle exister ? Deux échecs symétriques guettent, et ils font mal de façons opposées.

* **Une route qui apparaît trop tôt.** La claim est appliquée, l'`AIGatewayRoute` est publiée dans la foulée — mais le premier réplica vLLM charge encore ses poids. La Gateway route alors vers un backend sans endpoint prêt, et le client prend un `404` ou un `503` en pleine face, sur un modèle que la plateforme prétend servir.
* **Une route qui disparaît trop vite.** Le réflexe inverse — retirer la route dès que le `Deployment` n'est plus disponible — transforme la moindre indisponibilité passagère (un rolling update, un pod évincé, un nœud GPU recyclé par Karpenter) en *flapping* : la route sort de la config Envoy, y revient trente secondes plus tard, et le client voit alterner « modèle inconnu » et « modèle disponible ».

Le compromis retenu est asymétrique, parce que les deux erreurs ne coûtent pas le même prix. À la création, la route est **retenue** tant que le `Deployment` n'a pas atteint la condition `Available=True` : pas de route tant qu'aucun réplica ne peut répondre. Une fois publiée, en revanche, elle **ne sera plus jamais retirée**. C'est ce que j'appelle le **latch de readiness** : un portail à l'entrée, puis un verrou qui reste enclenché. L'idée : une indisponibilité transitoire reste un problème de *santé du backend*, pas de *topologie de routage* — c'est aux retries et au health checking d'Envoy de l'absorber, pas à la route de disparaître puis de revenir.

Techniquement, le latch tient sur l'**`ocds`** (_observed composed resources_) : l'état des ressources déjà composées, tel que Crossplane l'observe au début de chaque réconciliation. La composition y cherche sa propre `AIGatewayRoute` ; si elle l'y trouve, c'est que le portail s'est déjà ouvert, et elle la rend de nouveau sans même regarder l'état du `Deployment`.

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

La section précédente a lâché « adapters LoRA » en une ligne. Comme tout ce qui suit repose dessus, il faut deux minutes pour dire précisément ce que c'est — et surtout ce que ça n'est **pas**.

Un adapter **LoRA** (_Low-Rank Adaptation_) n'est pas un modèle. C'est un **delta de poids de faible rang** : plutôt que de réentraîner les matrices du modèle de base, on apprend une paire de petites matrices dont le produit vient s'ajouter à celles-ci. Le « faible rang » est tout le levier — sur la plateforme, il est fixé à **64** — et il se lit directement sur la balance : quelques dizaines de Mo pour l'adapter, contre les ~15 Go de poids du modèle de base.

De cette différence de nature découle la seule propriété qui compte pour la suite : **vLLM charge le modèle de base une fois, puis applique les adapters par-dessus, dans le même processus, sur le même GPU** (`--enable-lora`, `--lora-modules`). Servir trois « modèles » — la base et ses deux fine-tunes — ne coûte donc pas trois GPU. C'est un pod, un GPU, et **trois noms auxquels il répond**.

Deux adapters sont déclarés sur `xplane-qwen-coder`, tous deux posés sur exactement le même modèle de base (`Qwen/Qwen2.5-Coder-7B-Instruct`) :

| `loraAdapters[].name` | Source HuggingFace | Spécialisation |
|---|---|---|
| `xplane-qwen-coder-sql-dpo` | `jk200201/qwen2.5-coder-7b-sql-dpo` | génération SQL |
| `xplane-qwen-coder-securecode` | `scthornton/qwen2.5-coder-7b-securecode` | code sécurisé |

Chaque entrée de `spec.loraAdapters` nomme l'adapter — c'est ce nom qu'un client mettra dans son champ `model` — et pointe un dépôt HuggingFace **piné au commit SHA**, exactement comme le modèle de base l'est déjà via `model.repository` / `model.revision`. Un fine-tune qui bouge sous les pieds du serving est une régression que personne ne voit venir.

{{% notice note "Ces adapters, je ne les ai pas entraînés" %}}
Autant l'assumer : les deux adapters ci-dessus viennent de HuggingFace, entraînés par d'autres. Je les ai retenus parce qu'ils se posent sur le modèle de base que la plateforme sert déjà — pas parce que j'ai évalué leur qualité.

**Fine-tuner soi-même est un autre métier** : constitution du dataset, boucle d'entraînement, évaluation. Ce qui suit traite d'une question d'infrastructure — *comment expose-t-on progressivement un fine-tune au trafic réel ?* — pas d'une question de data science. J'y reviens dans « Et après ? ».
{{% /notice %}}

## :hatching_chick: Canary : 10 % du trafic, zéro GPU

Le scénario : `xplane-qwen-coder-sql-dpo` prétend être meilleur que le modèle de base sur la génération SQL. Sur un benchmark, peut-être. Sur **le** trafic — celui des vrais utilisateurs, avec leurs prompts et leurs fichiers à eux — la seule façon de le savoir est de lui en envoyer un peu. Dix pour cent, pendant quelques jours.

Voilà ce que ça coûte dans la claim :

```diff
# apps/base/ai/llm/qwen-coder.yaml
 spec:
   gateway:
     enabled: true
+    canaries:
+      - adapter: xplane-qwen-coder-sql-dpo
+        weightPercent: 10
```

Deux lignes utiles. Flux réconcilie, et une requête sur dix adressée à `xplane-qwen-coder` est désormais servie par le fine-tune.

### Sous le capot : un split qui n'envoie rien ailleurs

La règle de l'`AIGatewayRoute` qui répond au nom du modèle de base ne porte plus un seul `backendRef`, mais **un par entrée** : le backend de base d'abord, puis un backend par canary. Le backend de base conserve `100 - sum(weightPercent)`, chaque canary prend son `weightPercent`, et chaque canary reçoit son propre `AIServiceBackend` (`<claim>-canary-<i>`) porteur d'un champ décisif — **`modelNameOverride`**, qui vaut le nom de l'adapter *verbatim*.

| `backendRef` | Poids | `modelNameOverride` | `Backend` visé |
|---|---|---|---|
| `AIServiceBackend` de base | **90** (`100 - 10`) | *(aucun)* | `xplane-qwen-coder` |
| `AIServiceBackend` de canary (`<claim>-canary-<i>`) | **10** | `xplane-qwen-coder-sql-dpo` | `xplane-qwen-coder` |

Relisez la dernière colonne. **Les deux `backendRef` pointent le même `Backend`** — le même `Service`, les mêmes pods, le même GPU. Rien n'est envoyé ailleurs, parce qu'il n'y a pas d'ailleurs. Tout ce que fait le canary, c'est **réétiqueter** 10 % des requêtes : `modelNameOverride` réécrit le nom du modèle avant que la requête n'atteigne vLLM, et vLLM — qui a déjà l'adapter en mémoire, puisqu'il l'a chargé au démarrage — la sert avec le fine-tune plutôt qu'avec les poids de base.

<!-- TODO-e2e: schéma du split canary (tâche 8) -->

C'est là que se joue le « zéro GPU » du titre. Un canary applicatif classique coûte de la capacité : on déploie une v2 à côté de la v1, et pendant toute la durée de l'expérience on paie deux jeux de réplicas. Transposé ici, ce serait un **second pod vLLM**, donc un second GPU, donc un nœud de plus provisionné par Karpenter — pour évaluer un delta de quelques dizaines de Mo. Le canary LoRA n'ajoute ni réplica, ni nœud, ni ligne de facture : le split est une décision de *routage*, prise en amont, sur une flotte qui n'a pas bougé.

<!-- TODO-e2e: split réellement observé via vllm:lora_requests_info (SC-002 : entre 2% et 25% sur ≥50 requêtes, tolérance binomiale) -->

### Le canary n'écrase pas le nommage explicite

Le split n'est pas le seul chemin vers un adapter, et c'est important : chaque entrée de `loraAdapters[]` reçoit **sa propre règle de routage** (`x-ai-eg-model: <name>` → backend de base, poids 100). Une requête qui nomme explicitement l'adapter est donc servie **à 100 % par cet adapter**, indépendamment du canary en cours.

Les deux comportements coexistent, et c'est voulu :

* **Le client qui demande `xplane-qwen-coder`** entre dans le split — 90 % base, 10 % fine-tune. C'est la population d'expérience : elle ne sait pas qu'elle en fait partie, et c'est précisément ce qui rend le signal intéressant.
* **Le client qui demande `xplane-qwen-coder-sql-dpo`** est **pinné** : il obtient l'adapter, toujours. C'est le mode d'usage explicite — celui de la partie 3 — et c'est aussi ce qui permet à une éval [Promptfoo](/fr/post/series/agentic_ai/llm-self-hosted-stack/) de comparer deux modèles sans dépendre d'un tirage au sort.

Autrement dit, le canary répond à « ce fine-tune tient-il face au trafic réel ? », le pin répond à « que vaut ce fine-tune, toutes choses égales par ailleurs ? ». Les deux questions sont légitimes, et aucune des deux ne doit désactiver l'autre.

### Ce que l'admission refuse

Un pourcentage de trafic est exactement le genre de champ où une faute de frappe se paie en production. Les garde-fous sont donc **à l'admission**, en CEL dans l'XRD : `kubectl apply` échoue, et la composition ne se déclenche même pas.

* Des `canaries` **sans `gateway.enabled: true`** — une pondération sans route à pondérer.
* Un `canaries[].adapter` **absent de `loraAdapters[].name`** — on ne route pas vers un adapter que le pod ne charge pas.
* **Deux entrées visant le même adapter** — deux poids pour une seule cible, sans résolution évidente.
* Un `weightPercent` **hors de 1–99**.
* Une **somme des `weightPercent` supérieure à 99**.

La dernière règle mérite qu'on s'y arrête, parce que le plafond « naturel » aurait été 100. À 100 %, le modèle de base ne reçoit plus rien : le canary absorbe tout le trafic. Ce n'est plus un canary, c'est un **remplacement déguisé en expérience** — et déclaré dans un champ dont le nom promet exactement le contraire. Un remplacement, ça se lit dans le champ qui le dit (`model.repository`, `model.revision`), ça ne se déduit pas d'un poids qui a discrètement atteint son maximum.

Le plafond à **99** garantit donc qu'il reste toujours du trafic sur le modèle de base. Donc toujours une référence à laquelle comparer le canary, et toujours un chemin de retour immédiat : ramener le poids à 1, ou retirer l'entrée.

{{% notice tip "Pourquoi un array dès le premier jour" %}}
`gateway.canaries` est un **array** (quatre entrées au maximum), alors que le besoin du jour tenait dans une seule. Un simple objet aurait suffi — et aurait été un piège : passer d'un objet à un array plus tard est un **breaking change** d'API. Nouvelle version du schéma, conversion, migration de toutes les claims existantes… pour un champ qui n'aura rien gagné entre-temps.

Ce n'est pas de la prescience, c'est de l'emprunt : la lecture de la plateforme **Modelplane** — sur laquelle je reviens en fin d'article — a rendu évident que comparer *plusieurs* adapters en même temps est un cas d'usage normal, pas un exotisme. Le coût de l'array au jour 1, c'est quelques lignes de KCL. Le coût de la conversion au jour 100, ça s'appelle une migration.
{{% /notice %}}

## :unlock: L'échappatoire `engineArgs`

Ce troisième manque est le plus gênant à admettre, parce qu'il ne vient pas d'un oubli : il vient d'un refus. Le schéma de la claim n'expose que des champs curés, et j'avais délibérément décidé qu'il n'y aurait rien d'autre.

Le problème, c'est le rythme de vLLM. Le moteur ajoute des flags à chaque release — une politique d'ordonnancement, un format de chargement, un backend d'attention — bien plus vite qu'une API de plateforme ne peut les modéliser un par un. Chaque flag absent du schéma se payait donc au prix fort : une PR sur la composition KCL, les tests unitaires, la republication de l'image OCI, le bump du tag dans la `Composition`, puis Flux. **Une release de plateforme pour un essai d'une après-midi.**

Sauf qu'une API sans échappatoire ne bloque personne. Elle se fait **contourner**. On fait un `kubectl edit deployment`, on ajoute son flag, on teste — et jusqu'à la prochaine réconciliation de Crossplane, ça marche très bien. Le coût réel n'est pas le contournement lui-même, c'est ce qu'il fait à l'abstraction : dès l'instant où l'état réel du cluster ne se lit plus dans la claim, **la claim ment**. Et une abstraction qui ment est pire qu'une abstraction absente, parce qu'elle continue de prétendre décrire le système.

Le réflexe symétrique — un champ fourre-tout, non contraint, où l'on colle ce qu'on veut — n'est pas une solution non plus. Il ne fait que déplacer la casse : au lieu de sortir de l'abstraction pour la trouer, on la troue de l'intérieur, avec sa bénédiction.

### Un champ, une contrainte de forme

`spec.engineArgs` est un array de chaînes de caractères, **16 entrées au maximum**, transmises **verbatim** aux arguments du serveur vLLM. Aucune traduction, aucun mapping, aucune opinion : ce que l'utilisateur écrit est exactement ce que le moteur reçoit.

```yaml
# exemple : deux flags que le schéma curé ne modélise pas
spec:
  model:
    repository: Qwen/Qwen2.5-Coder-7B-Instruct
    revision: c03e6d358207e414f1eca0bb1891e29f1db0e242
  engineArgs:
    - --scheduling-policy=priority
    - --max-num-batched-tokens=8192
```

Une seule contrainte de forme, et elle n'a rien de cosmétique : **un seul token par entrée**. Soit `--flag`, soit `--flag=value` — jamais `--flag value` éclaté sur deux lignes de la liste. La raison tient à ce qui vient après : les entrées sont **inspectées à l'admission**, et une inspection qui regarde chaque entrée isolément est aveugle à un flag **passé en fraude**, coupé en deux morceaux dont aucun, pris seul, ne ressemble à ce qu'on cherche. La forme normalisée n'est pas une préférence d'écriture, c'est la condition pour que la vérification qui suit soit une vérification.

Les `engineArgs` sont concaténés **après** tous les arguments produits par la composition (`_vllmArgs = _managedVllmArgs + _engineArgs`) : la position est fixe, donc le résultat est déterministe. Mais que l'on soit clair — **ce n'est pas cet ordre qui protège le contrat de service.** Un moteur qui lit ses arguments de gauche à droite donne, par construction, le dernier mot à celui qui parle en dernier. Ce qui protège le contrat, c'est que certains flags ne peuvent tout simplement **pas entrer** dans cette liste.

### Seize flags qui ne passeront pas

Le tri est simple à énoncer : **tout passe, sauf ce dont l'autoscaling et le routage dépendent.** Ces flags-là sont **réservés**, et l'XRD les refuse à l'admission — `kubectl apply` échoue, la composition ne se déclenche même pas, rien n'atteint le cluster.

Dix-sept règles CEL portent cette vérification : une règle de forme (toute entrée doit commencer par `--`), et **seize règles de flags réservés, une par flag**. Le découpage peut sembler inutilement bavard — une seule règle bouclant sur une liste aurait fait le même travail de rejet. Elle aurait aussi produit le même message pour les seize : *« `engineArgs` contient un flag réservé »*. Un message qui dit à l'utilisateur qu'il a perdu, sans lui dire quoi faire. Une règle par flag, c'est **un message par flag** — et chaque message nomme le champ curé à utiliser à la place :

| Flag réservé | Ce que répond l'admission |
|---|---|
| `--model` | `use spec.model.repository` |
| `--max-model-len` | `use spec.model.contextWindow` |
| `--max-num-seqs` | `use spec.model.maxNumSeqs (it is the KEDA scaling denominator)` |
| `--gpu-memory-utilization` | `composition-managed (fixed at 0.92)` |
| `--enable-lora` | `use spec.loraAdapters` |
| `--port` | `serving-contract flag; the vLLM port is fixed at 8000 (Service/probes/Backend depend on it)` |

*(Extrait — les seize flags sont listés dans le README de la composition.)*

Prenez `--max-num-seqs`. C'est la taille du batch interne de vLLM, et c'est aussi le **dénominateur** du trigger KEDA vu en partie 3 : le `ScaledObject` divise `vllm:num_requests_running` par cette valeur pour obtenir un taux de saturation. Laissez un utilisateur la redéfinir dans `engineArgs`, et vLLM tourne avec un batch que l'autoscaler ignore — celui-ci continue de calculer un ratio contre une taille de batch qui n'existe plus. Rien ne casse, rien n'alerte : le scaling est simplement **faux**. Même logique pour `--enable-lora` et `--lora-modules` : le routage canary réécrit le nom du modèle vers un adapter que le pod est **censé** avoir chargé au démarrage.

### Le détail qui m'a le plus appris : `--port` et `--host`

Ces deux-là sont réservés alors que **la composition ne les émet jamais**. Elle ne passe ni `--port`, ni `--host` à vLLM — elle laisse le moteur prendre ses valeurs par défaut : le port `8000`, et l'écoute sur toutes les interfaces.

Sauf que ces défauts ne sont pas des détails d'implémentation, ce sont **le contrat de serving**. Le `targetPort` du `Service` vise 8000. Les probes de liveness et de readiness interrogent 8000. Le FQDN du `Backend` rendu par la Gateway pointe 8000. Trois ressources générées, trois hypothèses silencieuses sur une valeur que personne n'a écrite nulle part. Un `--port=9000` dans `engineArgs`, et les trois pointent dans le vide.

C'est la leçon que je retiens le plus volontiers de cette section, et elle n'a rien à voir avec les LLM : **la denylist d'une échappatoire doit couvrir non seulement ce que l'abstraction écrit, mais ce qu'elle suppose.** Ce sont deux ensembles différents, et le second est celui qu'on oublie — parce qu'un défaut sur lequel on s'appuie sans le déclarer n'apparaît dans aucun diff.

### Une seule source d'application

Reste la question qui décide de tout : **où vit la denylist ?**

La réponse tenue ici : **à un seul endroit**, le CEL de l'XRD. La composition, elle, ne vérifie rien. Elle appende les `engineArgs` verbatim et fait confiance à l'admission — si le contenu est arrivé jusqu'à elle, c'est qu'il est passé.

La tentation de la « défense en profondeur » était pourtant là : refiltrer côté KCL, par acquit de conscience. Ç'aurait été recréer exactement le problème que la première moitié de cet article vient de supprimer. Une liste de seize flags dans l'XRD, une liste de seize flags dans la composition. **Deux fichiers à garder synchronisés, donc deux dérives possibles, et aucune des deux ne fait de bruit** — un flag ajouté d'un côté seulement, et l'on obtient soit un flag accepté à l'admission puis silencieusement ignoré, soit un flag rejeté que la composition aurait très bien passé. C'est la même forme de bug que `route.yaml`. Autre fichier, autre sujet, même piège.

Mais « un seul endroit » est une promesse qui doit **tenir dans le temps**, et elle est fragile : les seize flags réservés sont précisément ceux que la composition émet. Le jour où quelqu'un ajoute un argument géré au KCL sans l'ajouter à la denylist de l'XRD, le trou se rouvre — et personne ne le verra. Un test verrouille donc les deux ensembles l'un à l'autre : `test_engine_args_denylist_lockstep` lit les flags que la composition produit et vérifie que chacun figure bien dans la denylist.

{{% notice tip "Un test qui ne peut pas échouer ne prouve rien" %}}
Un test de cohérence entre deux listes a une propriété désagréable : il passe au vert le jour où on l'écrit, et il passerait tout aussi bien s'il ne testait rien du tout. Une faute de frappe dans le nom d'une variable, une assertion qui compare une liste vide à une liste vide — et vous obtenez le pire des artefacts : un test qui **rassure**.

D'où l'étape suivante, qui n'a coûté que quelques minutes : **casser volontairement la denylist** (retirer un flag), relancer la suite, et vérifier que `test_engine_args_denylist_lockstep` **échoue bien**. C'est ce qu'on appelle une vérification par **mutation** : on introduit la faute exprès pour s'assurer que le filet la rattrape. Tant que vous n'avez pas vu un test rouge, vous n'avez pas de test — vous avez une ligne dans un rapport de CI.
{{% /notice %}}

`engineArgs` ne rend donc pas la plateforme plus permissive : il rend sa permissivité **lisible**. Ce qui n'est pas listé passe, sans demander la permission à personne. Ce qui est listé est refusé à l'`apply`, avec le nom du champ à utiliser à la place. **Liberté là où c'est sûr, garde-fous là où ça ne l'est pas.**

Et c'est sans doute ce qui se transpose le mieux de tout cet article, GPU ou pas : toute API de plateforme finira par rencontrer un utilisateur qui a besoin de quelque chose qu'elle n'a pas prévu. Elle a le choix entre le lui donner **sous conditions**, ou le voir pris **sans conditions**. Il n'y a pas de troisième option — il y a seulement des équipes qui n'ont pas encore découvert laquelle des deux elles avaient choisie.

## :eyes: Le modèle dit ce qu'il sert

## :bar_chart: Mesurer le canary

## :compass: Endpoint Picker : au-delà du round-robin

## :telescope: Modelplane : évolution convergente

## :thought_balloon: Dernières remarques

## :bookmark: Références
