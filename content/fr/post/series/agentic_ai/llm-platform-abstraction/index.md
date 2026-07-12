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

{{< img src="routing-ownership.png" alt="Propriété du routage : avant, deux sources de vérité ; après, une seule" width="1200" >}}

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

{{< img src="canary-split.png" alt="Split canary : un pod, un GPU, le modèle de base et deux adapters LoRA" width="1200" >}}

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

Ce troisième manque ne vient pas d'un oubli : il vient d'un refus. Le schéma de la claim n'expose que des champs curés, et j'avais délibérément décidé qu'il n'y aurait rien d'autre. Le problème, c'est le rythme de vLLM : le moteur ajoute des flags à chaque release, bien plus vite qu'une API de plateforme ne peut les modéliser un par un. Chaque flag absent du schéma se payait donc au prix fort : une PR sur la composition KCL, les tests unitaires, la republication de l'image OCI, le bump du tag dans la `Composition`, puis Flux. **Une release de plateforme pour un essai d'une après-midi.**

Sauf qu'une API sans échappatoire ne bloque personne : elle se fait **contourner**. Un `kubectl edit deployment`, le flag ajouté à la main, et jusqu'à la prochaine réconciliation de Crossplane ça marche très bien. Le coût réel n'est pas le contournement, c'est ce qu'il fait à l'abstraction : dès l'instant où l'état réel du cluster ne se lit plus dans la claim, **la claim ment**. Et une abstraction qui ment est pire qu'une abstraction absente, parce qu'elle continue de prétendre décrire le système.

Le réflexe symétrique — un champ fourre-tout, non contraint — ne résout rien non plus : on troue alors l'abstraction de l'intérieur, avec sa bénédiction.

### Un champ, une contrainte de forme

`spec.engineArgs` est un array de chaînes, **16 entrées au maximum**, transmises **verbatim** aux arguments du serveur vLLM. Aucune traduction, aucun mapping, aucune opinion : ce que l'utilisateur écrit est exactement ce que le moteur reçoit.

```yaml
# exemple : trois flags que le schéma curé ne modélise pas
spec:
  model:
    repository: Qwen/Qwen2.5-Coder-7B-Instruct
    revision: c03e6d358207e414f1eca0bb1891e29f1db0e242
  engineArgs:
    - --enforce-eager
    - --kv-cache-dtype=fp8
    - --rope-scaling={"rope_type":"yarn","factor":2.0}
```

Une seule contrainte de forme, et elle n'a rien de cosmétique : **un seul token par entrée**. Soit `--flag`, soit `--flag=value` — jamais `--flag value` éclaté sur deux lignes. Les entrées sont en effet **inspectées à l'admission**, une par une, et une inspection qui les regarde isolément serait aveugle à un flag réservé **passé en fraude**, coupé en deux morceaux dont aucun ne ressemble, pris seul, à ce qu'on cherche.

Les `engineArgs` sont concaténés **après** les arguments produits par la composition (`_vllmArgs = _managedVllmArgs + _engineArgs`) : la position est fixe, le résultat déterministe. Mais **ce n'est pas cet ordre qui protège le contrat de service** : un moteur qui lit ses arguments de gauche à droite donne le dernier mot à celui qui parle en dernier. Ce qui protège le contrat, c'est que certains flags ne peuvent tout simplement **pas entrer** dans cette liste.

### Seize flags qui ne passeront pas

Le tri est simple à énoncer : **tout passe, sauf ce dont l'autoscaling et le routage dépendent.** Ces flags-là sont **réservés**, et l'XRD les refuse à l'admission — `kubectl apply` échoue, rien n'atteint le cluster.

Dix-sept règles CEL portent cette vérification : une règle de forme (toute entrée commence par `--`), et **seize règles de flags réservés, une par flag**. Une règle unique bouclant sur une liste aurait rejeté tout aussi bien, mais elle aurait produit le même message pour les seize — un message qui dit à l'utilisateur qu'il a perdu, sans lui dire quoi faire. Une règle par flag, c'est **un message par flag**, et chaque message nomme le champ curé à utiliser à la place :

| Flag réservé | Ce que répond l'admission |
|---|---|
| `--model` | `use spec.model.repository` |
| `--max-model-len` | `use spec.model.contextWindow` |
| `--max-num-seqs` | `use spec.model.maxNumSeqs (it is the KEDA scaling denominator)` |
| `--gpu-memory-utilization` | `composition-managed (fixed at 0.92)` |
| `--enable-lora` | `use spec.loraAdapters` |
| `--port` | `serving-contract flag; the vLLM port is fixed at 8000 (Service/probes/Backend depend on it)` |

*(Extrait — les seize flags sont listés dans le README de la composition.)*

Prenez `--max-num-seqs` : c'est le **dénominateur** du trigger KEDA vu en partie 3. Laissez un utilisateur le redéfinir, et l'autoscaler continue de calculer son taux de saturation contre une taille de batch qui n'existe plus. Rien ne casse, rien n'alerte : le scaling est simplement **faux**.

Deux flags méritent un arrêt particulier, parce que **la composition ne les émet jamais** : `--port` et `--host`. Elle laisse le moteur prendre ses défauts — le port `8000`, l'écoute sur toutes les interfaces — et pourtant elle les réserve. Ces défauts ne sont pas des détails d'implémentation, ce sont **le contrat de serving** : le `targetPort` du `Service` vise 8000, les probes interrogent 8000, le FQDN du `Backend` rendu par la Gateway pointe 8000. Un `--port=9000` dans `engineArgs`, et les trois pointent dans le vide.

La leçon dépasse les LLM : **la denylist d'une échappatoire doit couvrir non seulement ce que l'abstraction écrit, mais ce qu'elle suppose.** Le second ensemble est celui qu'on oublie, parce qu'il n'apparaît dans aucun diff.

### Une seule source d'application

Reste la question qui décide de tout : **où vit la denylist ?** À **un seul endroit**, le CEL de l'XRD. La composition, elle, ne vérifie rien : elle appende les `engineArgs` verbatim et fait confiance à l'admission.

La tentation de la « défense en profondeur » était pourtant là : refiltrer côté KCL, par acquit de conscience. Ç'aurait été recréer exactement le problème que la première moitié de cet article vient de supprimer — une liste de seize flags dans l'XRD, une autre dans la composition. **Deux fichiers à garder synchronisés, donc deux dérives possibles, et aucune ne fait de bruit.** Même forme de bug que `route.yaml` : autre fichier, même piège.

Mais « un seul endroit » est une promesse fragile : les seize flags réservés sont précisément ceux que la composition émet. Le jour où quelqu'un ajoute un argument géré au KCL sans l'ajouter à la denylist de l'XRD, le trou se rouvre — et personne ne le verra. Un test verrouille donc les deux ensembles l'un à l'autre : `test_engine_args_denylist_lockstep` lit les flags que la composition produit et vérifie que chacun figure bien dans la denylist.

{{% notice tip "Un test qui ne peut pas échouer ne prouve rien" %}}
Un test de cohérence entre deux listes a une propriété désagréable : il passe au vert le jour où on l'écrit, et il passerait tout aussi bien s'il ne testait rien du tout. Une faute de frappe dans un nom de variable, une assertion qui compare une liste vide à une liste vide — et vous obtenez le pire des artefacts : un test qui **rassure**.

D'où l'étape suivante : **casser volontairement la denylist**, relancer la suite, et vérifier que `test_engine_args_denylist_lockstep` **échoue bien**. C'est une vérification par **mutation** : on introduit la faute exprès pour s'assurer que le filet la rattrape. Tant que vous n'avez pas vu un test rouge, vous n'avez pas de test — vous avez une ligne dans un rapport de CI.
{{% /notice %}}

`engineArgs` ne rend donc pas la plateforme plus permissive : il rend sa permissivité **lisible**. Ce qui n'est pas listé passe, sans demander la permission à personne. Ce qui est listé est refusé à l'`apply`, avec le nom du champ à utiliser à la place. **Liberté là où c'est sûr, garde-fous là où ça ne l'est pas.**

## :eyes: Le modèle dit ce qu'il sert

`kubectl get isvc` répondait `READY`, et c'était tout ce qu'il répondait. Une information de santé : un pod tourne, il accepte des requêtes. Pas **à quels noms** il les accepte. La question était triviale tant qu'un pod servait un modèle sous un nom. Elle ne l'est plus : le même pod répond maintenant à un modèle de base et à deux adapters, et une requête sur dix adressée au premier part discrètement vers l'un des seconds.

C'est le quatrième manque, comblé par un champ de statut. `status.servedModels` est une liste d'objets, republiée à chaque réconciliation :

| Champ | Ce qu'il porte |
|---|---|
| `name` | le nom que le client écrira dans son champ `model` |
| `kind` | `base` (les poids du modèle) ou `adapter` (un LoRA posé par-dessus) |
| `canaryWeightPercent` | la fraction de trafic captée, là où `gateway.canaries` en déclare une — un canary ne peut viser qu'un adapter, jamais le modèle de base |

Sur `xplane-qwen-coder` — un modèle de base, deux adapters, un canary à 10 % sur l'un d'eux — le champ porte trois entrées :

| `name` | `kind` | `canaryWeightPercent` |
|---|---|---|
| `xplane-qwen-coder` | `base` | — |
| `xplane-qwen-coder-sql-dpo` | `adapter` | `10` |
| `xplane-qwen-coder-securecode` | `adapter` | — |

<!-- TODO-e2e: confirmer sur la sortie réelle de status.servedModels que canaryWeightPercent est bien absent de l'entrée base (kind: base), et pas simplement égal à 100 - sum(weightPercent) -->

Et une colonne `SERVED MODELS` porte la même chose jusque dans un `kubectl get isvc`, sans `-o yaml` ni `jq`.

<!-- TODO-e2e: sortie réelle de kubectl get isvc -n llm avec la colonne SERVED MODELS -->

### C'est de la topologie, pas de la santé

Un détail conditionne toute la lecture de ce champ : `servedModels` est calculé **depuis le spec seul**. La composition ne consulte ni l'`ocds`, ni l'état du `Deployment`, ni celui de la route — elle dérive la liste des noms de ce que la claim déclare. Le champ est donc peuplé **dès la première réconciliation**, avant même que le pod n'existe.

C'est une projection de **topologie**, pas un signal de santé. Il répond à « à quels noms cette claim est-elle *censée* répondre ? », jamais à « répond-elle, là, maintenant ? ». Un pod en `CrashLoopBackOff` affiche les mêmes `servedModels` qu'un modèle en pleine forme — c'est le comportement voulu, pas une lacune. La santé, elle, a déjà ses réponses ailleurs : la condition `READY`, le latch de readiness, les métriques vLLM de la partie 3.

{{% notice note "Le piège : un JSONPath wildcard dans une colonne" %}}
La colonne `SERVED MODELS` a d'abord pointé le JSONPath qui s'imposait naturellement : `.status.servedModels[*].name` — *tous* les noms de la liste. Elle n'en aurait affiché **qu'un seul**.

Les colonnes de `kubectl get` ne sont pas rendues par le client, mais côté serveur, par le **convertisseur de table** de l'API server, qui applique le JSONPath des `additionalPrinterColumns` du CRD — et n'en retient que la **première correspondance**. Une cellule de tableau est un scalaire, pas une liste : le wildcard ne boucle pas, il sélectionne. La colonne aurait donc affiché `xplane-qwen-coder`, et les deux adapters n'auraient existé nulle part.

D'où un second champ, `status.servedModelsSummary` : un scalaire, les noms joints par des virgules, sur lequel la colonne pointe désormais.
{{% /notice %}}

Deux champs pour une seule vérité, après un article passé à traquer les doubles sources — l'ironie mérite d'être adressée. Mais `servedModelsSummary` n'est pas une seconde saisie : c'est une **fonction pure** de `servedModels`, calculée dans le même passage de la composition, et il n'existe aucun chemin par lequel les deux divergent. Le champ structuré reste l'API ; le scalaire n'est qu'une projection pour le CLI.

Et la leçon dépasse ce champ-là : **un statut que l'outil de tous les jours ne sait pas afficher n'est pas de la découvrabilité.** C'est de la donnée correcte, rangée dans un endroit que personne ne regarde.

## :bar_chart: Mesurer le canary

Tout ce qui précède produit un split : 90 % du trafic sur les poids de base, 10 % sur le fine-tune. Reste la seule question qui justifiait de le faire — **est-ce que ces 10 % se comportent mieux, ou moins bien, que les 90 % restants ?** Sans réponse, on n'a pas déployé un canary : on a détourné du trafic utilisateur vers autre chose en croisant les doigts. **Un canary qu'on ne peut pas mesurer n'est pas un canary, c'est un pari.**

La partie 3 a déjà présenté les métriques **`gen_ai.*`** de l'Envoy AI Gateway (v1.0) : le standard [OpenTelemetry Gen AI](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/), qui ne compte pas des requêtes HTTP mais le vocabulaire métier des LLM — tokens consommés, TTFT, latence par token de sortie. Il restait à les **brancher**.

Elles ne sont pas émises par le proxy Envoy lui-même, mais par l'**extproc** (_external processor_) : le sidecar auquel Envoy délègue le traitement spécifique aux LLM, seul sur le chemin de requête à voir à la fois le nom **demandé** et le nom **servi**. C'est là que le scrape se branche :

* **Cible** : le sidecar extproc, pas le conteneur Envoy — d'où un **`VMPodScrape`** (l'objet VictoriaMetrics qui cible des *pods*) et non un `VMServiceScrape`
* **Namespace** : `envoy-gateway-system`
* **Port** : `1064`, le port d'administration de l'extproc

Les séries atterrissent dans la VictoriaMetrics qui collecte déjà les métriques vLLM, et se lisent dans le même Grafana. Aucun nouvel outil.

### L'attribution canary vs base : une question ouverte

`modelNameOverride` réécrit le nom du modèle **verbatim** vers celui de l'adapter avant que la requête n'atteigne vLLM : une requête tirée dans les 10 % est servie sous le nom `xplane-qwen-coder-sql-dpo`, pas `xplane-qwen-coder`.

Reste une question que la PR #1559 laisse **explicitement ouverte**, et que je préfère nommer plutôt que deviner : les métriques `gen_ai.*`, qui portent le modèle en label, attribuent-elles ce trafic au nom **de l'adapter surchargé** — celui que vLLM a servi — ou au nom **du modèle de base** — celui que le client a demandé ? Seule une requête envoyée dans le canary, puis retrouvée (ou non) sous le nom de l'adapter, le dira.

Ce n'est pas un détail cosmétique. Si l'attribution suit le nom surchargé, le split de routage est **aussi**, gratuitement, un split de mesure. Sinon, canary et base se mélangent dans la même série, et l'attribution que le dashboard promet n'existe pas.

Le dashboard Grafana **`llm-gateway`** vise le premier scénario : les mêmes séries, découpées canary contre base. Voici ce qu'il donnerait à lire, si l'attribution se confirme :

* La **latence end-to-end** (`gen_ai.server.request.duration`) et le **TTFT** (`gen_ai.server.time_to_first_token`) : servir via un adapter LoRA coûte-t-il le même chemin que servir sur les poids de base ?
* La **latence par token de sortie** (`gen_ai.server.time_per_output_token`), qui dirait si le fine-tune génère au même rythme.
* La **consommation de tokens** (`gen_ai.client.token.usage`) : un fine-tune plus bavard que sa base, à qualité égale, est un fine-tune plus cher.
* Le **volume de requêtes** de chaque branche — de quoi vérifier que le split fait ce qu'il annonce.

<!-- TODO-e2e: trancher si gen_ai.* attribue le trafic canary au nom de l'adapter surchargé (modelNameOverride) ou au nom du modèle de base — réponse à documenter dans le README de la composition (SPEC-002, question ouverte PR #1559) -->
<!-- TODO-e2e: screenshot du dashboard Grafana llm-gateway avec l'attribution canary vs base -->

Ces courbes disent si le canary est plus lent, ou plus bavard, que la base. Elles ne disent pas s'il est **meilleur** : aucune métrique de gateway ne sait ce qu'est un bon SQL. La qualité reste le domaine de l'éval — [Promptfoo](/fr/post/series/agentic_ai/llm-self-hosted-stack/), vu en partie 3 — et c'est là que la règle de pin reprend tout son sens : une éval doit interroger l'adapter **de façon déterministe**, pas tomber dessus une fois sur dix. Promouvoir un fine-tune, c'est regarder les deux.

{{% notice note "Ce qui n'est pas livré : les traces" %}}
L'AI Gateway sait exporter des **traces OTLP**, et la cible naturelle serait **VictoriaTraces**. Sur cette plateforme, l'export est **volontairement désactivé** : je n'ai pas vérifié la compatibilité du chemin d'ingestion OTLP entre les deux, et tant que ce n'est pas fait, la case reste décochée.

Livrer un export vers un endpoint qu'on n'a pas testé, c'est livrer une ligne de configuration qui donne l'**illusion** d'une capacité : le jour où on voudrait savoir où part le temps entre la gateway, l'extproc et vLLM, on découvrirait qu'il n'y a rien au bout. Une case décochée est honnête ; une case cochée qui ne transporte rien ne l'est pas.
{{% /notice %}}

## :compass: Endpoint Picker : au-delà du round-robin

Tout ce qui précède décide **quel modèle** sert une requête. Reste **quel réplica** — tranché par défaut par le `Service` ClusterIP et son **round-robin** (chaque requête au pod suivant). Sur vLLM, c'est **hostile**, pour deux raisons qui attaquent ce que la [partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/) avait mis en place :

* **Il détruit la localité du prefix cache.** Le **prefix caching** réutilise le **KV cache** (les activations déjà calculées pour un préfixe donné), local à un pod. Éparpiller des requêtes qui partagent leur préfixe force chaque réplica à recalculer ce que son voisin a déjà en mémoire : du **prefill** (le calcul initial sur tout le prompt) redondant, payé deux fois, en GPU et en TTFT.
* **Il est aveugle à la charge.** Round-robin ne regarde ni la file d'attente d'un pod ni la pression sur son KV cache. Une requête peut atterrir sur un réplica saturé pendant qu'un autre tourne à vide.

### Un flag, un sous-système

L'**Endpoint Picker** (EPP), composant de la **Gateway API Inference Extension** (GAIE, v1.5.0 ; `InferencePool` GA depuis septembre 2025), décide **lequel** des réplicas sert chaque requête. Il score chaque pod sur les **mêmes** `/metrics` vLLM que la plateforme collecte déjà : file d'attente, KV cache, affinité de préfixe, adapter LoRA présent.

Côté claim, ça tient en un booléen :

```diff
 spec:
   gateway:
     enabled: true
+    endpointPicker:
+      enabled: true
```

Derrière ce booléen, la composition rend cinq ressources :

| Ressource générée | Ce qu'elle apporte |
|---|---|
| **`HelmRelease`** (chart `inferencepool`, v1.5.0) | l'installation de l'EPP |
| **`InferencePool`** | les réplicas à scorer, scopés à **sa propre claim**, jamais cross-modèle |
| **Deployment EPP** | le service de scoring lui-même |
| **`CiliumNetworkPolicy`** | default-deny du pod EPP |
| **`VMServiceScrape`** | l'EPP est aussi observé |

### KEDA et l'EPP ne se disputent rien

Le réflexe est d'y voir un doublon de l'autoscaler. Ce n'en est pas un : **ils opèrent à deux échelles de temps différentes.** KEDA décide **combien** de réplicas exister, en dizaines de secondes ; l'EPP décide **lequel** des réplicas existants sert la requête, en millisecondes. Les deux lisent les mêmes signaux vLLM, pour des décisions différentes.

Troisième trigger KEDA ajouté au passage : aux deux de la partie 3 s'ajoute **`vllm:num_requests_waiting`** (`scaling.queueLengthThreshold`, défaut **8**) — les requêtes acceptées mais pas encore traitées, le signal de pression **le plus précoce** des trois. Les triggers se combinent en **OR** : le premier à franchir son seuil déclenche le scale-up.

Sur le chiffre, restons honnête. Le projet **llm-d** rapporte jusqu'à **57× d'amélioration du TTFT** face au round-robin sous charge. C'est une mesure **externe**, faite par d'autres, sur leur charge et leur matériel — **je ne l'ai pas reproduite ici**, et elle décrit vraisemblablement un cas où le prefix cache a beaucoup à donner. Je la cite pour l'ordre de grandeur du levier, pas comme un résultat de cette plateforme.

{{% notice warning "EPP et canary : il faut choisir" %}}
Les deux fonctionnalités les plus visibles de cette PR sont **mutuellement exclusives**. Une claim qui active `gateway.endpointPicker` avec un `gateway.canaries` non vide est **rejetée à l'admission**.

La cause est structurelle : un `backendRef` pointant une `InferencePool` ne supporte **ni `weight`, ni `modelNameOverride`** — les deux mécanismes du canary — et une seule `InferencePool` est admise par règle. Rien à composer : ils parlent deux langages de `backendRef` incompatibles.

Il faut donc choisir, aujourd'hui. Un rejet net à l'`apply` vaut mieux qu'une combinaison silencieusement incomplète.
{{% /notice %}}

{{% notice tip "Au passage : le dernier maillon du cold-start" %}}
La partie 3 avait réglé le gros du démarrage à froid avec un **PVC S3 Files**, évitant de retélécharger ~15 Go depuis HuggingFace. Restait un maillon : la lecture de ces poids depuis le volume vers la VRAM.

Le **Run:ai Model Streamer** (`--load-format runai_streamer`) s'attaque à celui-là : selon la documentation du projet, il lit les poids de façon concurrente et les transfère vers le GPU au fil de la lecture, sans attendre qu'ils soient tous chargés en mémoire hôte. Embarqué dans l'image `vllm-openai` depuis la **v0.8.5**, il s'active par `model.streaming` sur le PVC déjà en place. Ses flags rejoignent la denylist `engineArgs`.

Deux capacités restent **descopées** : le chargement `s3://` direct et le résolveur LoRA, qui demandent **vLLM v0.9.0**. Combien de secondes cela fait-il gagner ? Pas encore mesuré — je le dirai quand je l'aurai fait.
{{% /notice %}}

<!-- TODO-e2e: cold-start mesuré avec et sans le streamer -->

## :telescope: Modelplane : évolution convergente

Le 23 juin 2026 — donc pendant que cette PR se construisait — les créateurs de Crossplane ont publié [**Modelplane**](https://github.com/modelplaneai/modelplane) : un plan de contrôle open-source pour l'inférence, qui traite les **modèles**, et non les clusters, comme l'objet que l'on gère. Leur API porte la même séparation que celle-ci défend : côté équipe plateforme, `InferenceCluster` et `InferenceClass` déclarent la capacité disponible ; côté équipe ML, `ModelDeployment` et `ModelService` déclarent le modèle à faire tourner et le service qui l'expose ; le tout est servi par une `InferenceGateway` commune aux deux.

Autant le dire franchement : c'est la thèse de cet article, écrite par d'autres. Le découpage entre ce que l'équipe plateforme possède et ce que l'équipe ML déclare — `InferenceCluster`/`InferenceClass` d'un côté, `ModelDeployment`/`ModelService` de l'autre — est exactement celui que cette plateforme applique entre la composition et la claim. Quand les gens qui ont construit Crossplane arrivent à la même forme, ce n'est pas une coïncidence : c'est que le problème, lui, a une forme.

Sur la chronologie, coupons court : leur billet est postérieur à la v0.6.0 de cette plateforme (9 mai 2026), et je n'en tire rien du tout. Ce n'est pas une course, personne n'a copié personne — c'est une **évolution convergente**, deux équipes qui butent sur le même mur et en sortent avec le même plan. Ce que leurs docs de design ont réellement servi ici, c'est autre chose : une **grille de revue comparative**. Des questions posées par d'autres, à confronter aux choix déjà faits.

### Eux vont large, ici on va profond

La différence n'est pas de qualité, elle est de **périmètre** — et c'est ce qui rend la comparaison lisible.

| | Modelplane | `InferenceService` (ici) |
|---|---|---|
| **Unité gérée** | la **flotte** : multi-cluster, multi-cloud, GPU hétérogènes | **un seul cluster** |
| **Placement** | un scheduler choisit où poser chaque réplica | Karpenter, dans le cluster |
| **Moteur d'inférence** | agnostique (vLLM, SGLang, topologies de parallélisme arbitraires) | **couplé à vLLM**, avec une échappatoire |
| **Maturité** | v0.1, API annoncée comme instable | v0.8.0, en GitOps sur un cluster réel |

Le problème qu'ils résolvent — placer un modèle sur le bon GPU, dans le bon cluster, chez le bon fournisseur — **je ne l'ai tout simplement pas**. Cette plateforme est une référence mono-cluster ; prétendre qu'elle adresse la flotte serait du bruit.

En échange, elle va plus profond sur le seul cluster qu'elle connaît. Trois choses y sont livrées ici : le **canary pondéré implémenté** — le trafic est effectivement splitté entre adapters LoRA, validé à l'admission, couvert par des tests, à coût GPU nul ; le **durcissement production comme défaut**, pas comme remarque de fin — zero-trust Cilium, PSS restricted, secrets, autoscaling sur des signaux de saturation précoces de vLLM ; et le **GitOps de bout en bout** — tout ce qui précède arrive par Flux depuis un commit git, la claim restant la seule chose qu'un utilisateur écrit.

Et surtout, leurs propres docs le disent : Modelplane **compose** les projets de niveau cluster plutôt que de les remplacer. Une stack de la forme de celle décrite ici est exactement ce qui vivrait **en dessous** de la leur. Ce ne sont pas deux réponses concurrentes, ce sont deux étages.

### Ce que leur revue a changé, ici

C'est le point le plus intéressant, et le plus honnête : leur design a **modifié le code de cette PR**, et pas à la marge. Ce qu'il apporte tient en une discipline — la **forward-compatibilité** : concevoir un champ non pour le besoin du jour, mais pour que le besoin du mois prochain n'impose pas de casser l'API.

* **Des arrays plutôt que des objets.** Le champ `gateway.canaries` de la section canary est un array **dès le premier jour**, alors qu'une seule entrée suffisait au besoin. Passer d'un objet à un array plus tard est un breaking change ; l'inverse est gratuit.
* **Aucun nouveau champ requis.** Toute claim v0.6.0 s'applique telle quelle en v0.8.0. `gateway`, `canaries`, `engineArgs`, `endpointPicker` : tout est optionnel, et chaque défaut préserve le comportement d'avant.
* **Des flags fournis par l'utilisateur plutôt qu'injectés.** L'échappatoire `engineArgs` existe parce que cette revue a rendu évident qu'aucune API de plateforme ne rattrapera jamais le rythme d'un moteur d'inférence. Le choix n'est pas *modéliser ou interdire*, c'est *encadrer ou se faire contourner*.

Reste le caveat qui garde la comparaison honnête, et il joue contre moi : **ils sont engine-agnostic, je ne le suis pas.** La composition décrite ici parle vLLM, produit des flags vLLM, scale sur des métriques vLLM. C'est un choix de périmètre, pas un oubli — et c'est précisément ce qui rend possibles les signaux d'autoscaling curés et la denylist des seize flags réservés. On ne peut pas savoir que `--max-num-seqs` est le dénominateur d'un trigger KEDA **et** être agnostique au moteur qui l'expose. Le couplage est le prix de la profondeur ; l'échappatoire est là pour qu'il ne devienne pas une prison.

---

## :thought_balloon: Dernières remarques

Cette PR ne rend pas la plateforme plus puissante — un pod vLLM sur un GPU faisait déjà tourner un modèle en partie 3. Elle rend l'abstraction **honnête** : ce que la claim déclare est enfin ce que le cluster fait. Et le chemin pour y arriver a produit quatre leçons qui n'ont, au fond, rien à voir avec les LLM.

* **Si deux fichiers décrivent la même chose, ils divergeront.** Ce n'est pas un problème de discipline, c'est un problème de temps : la dérive est une fonction du nombre de jours, pas du sérieux de l'équipe. La seule issue est structurelle — la composition doit **posséder les deux**, ou il n'y a plus qu'un fichier.
* **Une route qui apparaît trop tôt envoie du trafic dans le vide ; une route qui disparaît trop vite fait du flapping.** Les deux erreurs sont symétriques, elles ne coûtent pas le même prix. Portail à la création, latch ensuite.
* **Une API sans échappatoire finit contournée ; une échappatoire sans garde-fous casse les invariants.** Entre les deux : une **denylist explicite**, appliquée en **un seul** endroit. Deux endroits, et on vient de recréer le premier problème.
* **Si `kubectl get` ne dit pas ce que la ressource fait vraiment, l'abstraction ment.** Un statut correct rangé là où personne ne le lit n'est pas de la découvrabilité.

Ce que je retiens surtout, c'est le déplacement qui les relie toutes les quatre : **l'unité de déploiement n'est plus « un pod qui sert un modèle », c'est « un modèle, avec sa politique de trafic, sa stratégie de rollout et sa découvrabilité »**. Tant qu'une abstraction ne modélise que le pod, tout le reste — la route, le split, les noms servis — retombe sur un humain, dans un fichier à côté, à la main.

### Et après ?

Le chaînon vraiment manquant, c'est le premier de cette liste : **fine-tuner mon propre adapter**. Les deux LoRA de cet article viennent de HuggingFace, entraînés par d'autres ; le canary sait donc exposer progressivement un fine-tune au trafic réel, mais il n'a encore jamais servi à valider *le mien*. C'est là que la boucle se ferme — et c'est un autre métier, que je n'ai pas encore fait.

* **`s3://` en chargement direct et le résolveur LoRA** — les deux capacités descopées de cette PR, qui demandent vLLM v0.9.0. De quoi supprimer le dernier détour par le PVC.
* **Les traces OTLP vers VictoriaTraces** — case volontairement décochée aujourd'hui, à cocher le jour où la compatibilité du chemin d'ingestion sera vérifiée, pas avant.
* **Les trois modèles restants**, à basculer sous routage possédé par la composition — et, dans le même mouvement, la suppression définitive de `route.yaml`. C'est ce commit-là, et pas celui de cette PR, qui tuera pour de bon la double comptabilité.

{{% notice note "Où en est la validation, exactement" %}}
Autant être précis sur ce qui est **prouvé** au moment où j'écris ces lignes :

* `kcl test` : **44/44 PASS**, dont le lockstep XRD ↔ composition vérifié par mutation.
* `./scripts/validate-kcl-compositions.sh` : **exit 0**.
* `crossplane render` puis Polaris, contre le **tag réel** de la composition (`0.8.0-pr1559`) : les ressources rendues sont celles décrites ici, et elles passent le durcissement.

Et ce qui **attend encore le passage e2e sur le cluster** : le split de trafic réellement observé, l'attribution `gen_ai` du canary, la sortie de la colonne `SERVED MODELS`, et le gain de cold-start du streamer. Tant que ces mesures ne sont pas faites, elles ne sont pas dans cet article — ni en chiffres, ni en captures d'écran.
{{% /notice %}}

---

## :bookmark: Références

### Le code

- [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — la plateforme complète
- [PR #1559](https://github.com/Smana/cloud-native-ref/pull/1559) — la composition `InferenceService` v0.6.0 → **v0.8.0**
- [`docs/specs/`](https://github.com/Smana/cloud-native-ref/tree/main/docs/specs) — les specs de cette PR : **002** (routage possédé + canary), **003** (`engineArgs` et `servedModels`), **004** (InferencePool + Endpoint Picker), **005** (cold-start / Model Streamer), **006** (observabilité `gen_ai`)

### Composants techniques

- [Envoy AI Gateway](https://aigateway.envoyproxy.io/) — la porte d'entrée compatible OpenAI, et ses [métriques `gen_ai`](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/)
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) — `InferencePool` et Endpoint Picker
- [Run:ai Model Streamer](https://github.com/run-ai/runai-model-streamer) — chargement concurrent des poids vers la VRAM
- [KEDA](https://keda.sh/) — autoscaling sur métriques vLLM
- [Crossplane](https://www.crossplane.io/) — le moteur de composition
- [KCL](https://www.kcl-lang.io/) — le langage dans lequel la composition est écrite

### Évolution convergente

- [Modelplane](https://github.com/modelplaneai/modelplane) — le plan de contrôle d'inférence des créateurs de Crossplane, à l'échelle de la flotte

### Articles précédents de la série

- [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) — Partie 1
- [Quelques mois avec Claude Code : tips et workflows](/fr/post/series/agentic_ai/ai-coding-tips/) — Partie 2
- [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) — Partie 3
