+++
author = "Smaine Kahlouch"
title = "Le modèle, pas le pod"
date = "2026-07-12"
summary = "Deux mois d'usage de l'abstraction `InferenceService` ont dégagé cinq axes d'amélioration : le routage possédé par la claim, un rollout par paliers, un routage sémantique où le prompt choisit le modèle, une configuration avancée encadrée et une topologie lisible. Comment une abstraction de plateforme gagne en maturité — canary LoRA à coût GPU nul et mesures sur cluster comprises."
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
Cet article fait suite à [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) (partie 3), qui décrit la plateforme sur laquelle tout ce qui suit s'appuie. **Ici, on la fait vivre** — et on la fait mûrir.
{{% /notice %}}

Ajouter un modèle à la plateforme, c'est un fichier YAML et un `git push`. Le pod démarre, les poids se chargent, la ressource passe `READY`. Et le client, lui, peut très bien prendre un `404` — parce que la **route** qui expose ce modèle sur la Gateway ne vit pas dans ce fichier. Elle vit dans un autre fichier, à côté, écrit à la main.

Ce n'est pas un bug : c'est exactement le contrat livré dans la [partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/), il y a deux mois. Et ce contrat tient. La **claim** (le manifeste YAML par lequel un utilisateur déclare *ce qu'il veut*, à charge pour la plateforme de produire les ressources Kubernetes correspondantes) reste la bonne interface, et la plateforme produit bien les ressources du modèle.

Ce que deux mois d'usage ont montré, c'est que la **promesse** attachée à cette interface — produire *toutes* les ressources dont un modèle a besoin — s'arrêtait avant le routage. Elle ne s'y arrête plus. Et l'analyse qui a mené là a dégagé quatre autres axes, ceux qui font passer une abstraction de « ça marche » à « ça évolue sans casser ».

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

* Comprendre pourquoi une abstraction de plateforme doit **posséder le routage** qu'elle génère — c'est la promesse de complétude qui en dépend
* Voir un **canary LoRA** déplacer une fraction du trafic vers un fine-tune, **à coût GPU nul** : même pod, même GPU, aucun réplica supplémentaire
* Voir un **routage sémantique** décider du modèle **à partir du prompt** — et se composer avec le canary au lieu de lui disputer la décision
* Voir comment une **configuration avancée** encadrée laisse passer les options que le schéma curé n'a pas prévues, sans trouer le contrat de service
* Rendre la topologie **lisible** : un `kubectl get` doit dire à quels noms le modèle répond réellement
* En tirer **cinq leçons transposables** à n'importe quelle abstraction de plateforme — LLM ou pas

{{% notice tip "Le code" %}}
Tout ce qui suit est dans [**cloud-native-ref**](https://github.com/Smana/cloud-native-ref) (PR #1559). La composition `InferenceService` passe de la **v0.6.0** (celle de la partie 3, du 9 mai 2026) à la **v0.8.0**.
{{% /notice %}}

---

## :arrow_upper_right: Cinq axes d'amélioration

Le contrat de la partie 3 n'a pas bougé : une claim d'un côté, la plateforme qui en tire les ressources du modèle de l'autre. C'est en le mettant au travail pendant deux mois qu'une analyse délibérée a dégagé cinq axes. Le premier est le plus structurant, parce qu'il porte sur la **complétude** de l'abstraction : sa promesse est de produire *toutes* les ressources dont un modèle a besoin, et le routage y échappait. Les autres portent sur son **évolutivité** : rien ne permettait de faire bouger un modèle par paliers, ni de laisser le prompt choisir sa destination, ni de sortir des champs curés, ni de lire ce qu'un pod servait réellement.

### 1. Le routage échappait à la claim

La claim déclare *quel* modèle tourne. Un second fichier, `apps/base/ai/llm/ai-gateway-routes/route.yaml`, déclare *comment* on l'atteint : c'est lui qui dit à l'**Envoy AI Gateway** (le point d'entrée unique de la plateforme, compatible API OpenAI) que `model: xplane-qwen-coder` doit atterrir sur tel backend. Et ce fichier était **écrit à la main**.

Une précision pour qui relit la partie 3 : elle listait l'`AIGatewayRoute` parmi les ressources générées par la composition. C'était vrai de l'intention, pas du code — et c'est exactement la lacune de couverture que cette version comble.

| Où c'est déclaré | Quoi | Qui l'écrit |
|---|---|---|
| `apps/base/ai/llm/qwen-coder.yaml` (la claim) | *quel* modèle tourne | l'utilisateur, via l'abstraction |
| `apps/base/ai/llm/ai-gateway-routes/route.yaml` | *comment* on l'atteint | l'utilisateur, à la main |

Deux fichiers qu'il faut penser à garder synchronisés, donc deux dérives possibles, et aucune des deux ne fait de bruit :

* **Un modèle sans route** — le pod tourne, le GPU est facturé, et la Gateway répond `404` à qui le demande.
* **Une route qui survit à son modèle** — la claim est supprimée, l'entrée de routage reste : un *backend fantôme* vers un `Service` qui n'existe plus.

### 2. Aucun moyen de faire évoluer un modèle par paliers

La bascule d'un modèle est **binaire**. On change `repository` et `revision`, Flux réconcilie, et 100 % du trafic part sur les nouveaux poids. Aucun palier, aucune fraction, aucun retour arrière progressif.

La matière première était pourtant déjà là : la composition savait charger des **adapters LoRA** (_Low-Rank Adaptation_). La partie 3 n'en a pas dit un mot — et pour cause : un adapter comme `xplane-qwen-coder-sql-dpo` ou `xplane-qwen-coder-securecode` n'était joignable que par un client qui le **nommait explicitement**. Impossible de déplacer 10 % du trafic du modèle de base vers son fine-tune pour le valider sur du trafic réel. Un fine-tune que personne ne nomme est un fine-tune que personne n'évalue.

### 3. Le client devait savoir quel modèle il voulait

La flotte compte quatre modèles, chacun bon à quelque chose de précis : du code, du raisonnement, de la modération. Pour en tirer quoi que ce soit, le client devait **choisir lui-même** — écrire `xplane-qwen-coder` dans son champ `model`, et donc connaître le catalogue, ses noms et ses spécialités.

Le **routeur sémantique** de la partie 3 — un service qui classe le prompt et décide seul quel modèle doit le traiter — était censé lever exactement ça. Il tournait, en bonne santé, depuis des semaines. Il n'a **jamais été appelé** — parce que rien, sur le chemin de requête, ne le branchait à la Gateway.

### 4. Aucun moyen de sortir des champs curés

Le schéma de la claim expose des champs **curés** : `quantization`, `contextWindow`, `toolCallParser`… Tout ce qui n'y figure pas est hors d'atteinte, sans qu'aucune sortie ne soit prévue pour l'imprévu.

### 5. La topologie servie n'était pas lisible

`kubectl get isvc` dit qu'un modèle est prêt. Il ne dit pas **à quels noms il répond**. Pour répondre à une question aussi élémentaire que « qu'est-ce que ce pod sert, au juste ? », il fallait ouvrir le fichier de routage, le recouper avec les arguments vLLM du `Deployment`, et reconstituer la topologie à la main. Une abstraction qu'il faut rétro-ingénierer pour savoir ce qu'elle expose s'arrête à mi-chemin.

---

Chacun de ces axes est traité dans une section de cet article :

| L'axe | Ce qui l'adresse |
|---|---|
| Le routage échappait à la claim | La composition **possède** le routage (`gateway.enabled`) |
| Aucune évolution par paliers | Des **canaries LoRA** pondérés (`gateway.canaries`) |
| Le client devait choisir son modèle | Un **routage sémantique** : `model: MoM`, et le prompt tranche |
| Aucune sortie des champs curés | `spec.engineArgs`, encadré à l'admission |
| Topologie servie illisible | `status.servedModels` et une colonne `SERVED MODELS` |

Mais ils ont tous la même racine, et c'est la thèse de cet article : **l'unité de déploiement n'est plus « un pod qui sert un modèle »**. Ce que l'on déploie, c'est un modèle de base, *ses adapters*, *les noms sous lesquels on les atteint*, et *la fraction de trafic qui va à chacun*. Tant que l'abstraction ne modélise que le pod, tout le reste retombe sur l'humain — dans un fichier à côté, à la main. Et une fois ce déplacement fait, un dernier tombe tout seul : **le client n'a même plus à savoir quel modèle il vise.**

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

Sur le cluster, le portail se voit à l'œil nu. Une claim fraîche, `gateway.enabled: true`, et on regarde apparaître les ressources :

```
[10:23:46] Available=False  route=0  pod=0/1 ContainerCreating
[10:25:54] Available=False  route=0  pod=0/1 Running       <- Running, mais pas prêt : toujours pas de route
[10:27:19] Available=False  route=0  pod=0/1 Running
[10:27:41] Available=True   route=1  pod=1/1 Running       <- la route apparaît
```

Le pod est resté **`Running` mais `0/1` pendant environ 90 secondes** — il chargeait ses poids sur le GPU — et la route est restée retenue tout ce temps. C'est le point précis qui compte : **le latch s'accroche à la condition `Available` du `Deployment`, pas à la phase du pod.** Un pod qui tourne n'est pas un pod qui sert, et seule la première des deux le sait.

Un détail de conception que le terrain a rendu visible : les deux backends décrits plus haut — le `Backend` et l'`AIServiceBackend` — sont créés **immédiatement**. Ils sont statiquement prêts — un FQDN et un port ne « démarrent » pas. **Seule la route est retenue.** Le portail ne bloque pas la construction du chemin, il bloque sa publication.

Pendant que la route est retenue, les `resourceRefs` de la claim montrent déjà les backends :

```
$ kubectl get inferenceservice xplane-scratch-latch -o jsonpath='{...resourceRefs}'
  AIServiceBackend/xplane-scratch-latch          <- créé
  Backend/xplane-scratch-latch-direct            <- créé
  Deployment/xplane-scratch-latch
  ...
                                                 <- aucune AIGatewayRoute
```

Et à la suppression, la promesse de possession se vérifie d'un coup : les **onze** ressources composées disparaissent, route et backends compris. Aucune n'a survécu à sa claim.

{{% notice tip "Le détail qui aurait fini en incident" %}}
La composition **écrit** la route sous une clé de ressource, et la **relit** sous cette même clé au tour suivant. Deux chaînes de caractères, à deux endroits du code : si elles divergent — un renommage d'un côté seulement — la relecture ne trouve jamais rien, le latch ne s'enclenche jamais, et la route est retirée à la première indisponibilité passagère venue. Un bug invisible en test, qui ne se manifeste qu'en production, au pire moment.

D'où une contrainte volontairement bête : les clés de lecture et d'écriture dérivent toutes deux d'une **constante unique** (`_gatewayRouteSuffix`). Elles ne peuvent pas diverger — il n'y a plus qu'une seule chaîne à renommer.
{{% /notice %}}

### Une migration, pas un big bang

Le basculement se fait modèle par modèle. `xplane-qwen-coder` passe en `gateway.enabled: true`, et **dans le même commit**, ses entrées sont retirées de `apps/base/ai/llm/ai-gateway-routes/route.yaml`. Un seul commit, parce qu'un instant où les deux sources déclarent la même route serait exactement le désordre qu'on cherche à supprimer. Les trois autres modèles de la plateforme suivront dans une PR ultérieure.

{{% notice note "La double comptabilité n'est pas encore morte" %}}
Soyons précis sur l'état réel : **un modèle sur quatre** est passé sous routage possédé par la composition. `route.yaml` existe toujours, il porte encore les trois autres, et il faut donc toujours y penser pour eux. Ce qui a changé, ce n'est pas que le fichier a disparu — c'est qu'il a cessé d'être *le seul* endroit possible, et qu'il a maintenant une date de péremption.
{{% /notice %}}

Reste que la route appartient désormais à la claim. Et une route que l'on possède est une route que l'on peut **pondérer**.

## :dna: LoRA en deux minutes

Plus haut, cet article a lâché « adapters LoRA » en une ligne. Tout ce qui suit repose dessus, alors autant prendre deux minutes — pour dire ce que c'est, mais surtout **à quoi ça sert**.

Un adapter **LoRA** (_Low-Rank Adaptation_) n'est pas un modèle. C'est un **delta de poids de faible rang** : plutôt que de réentraîner les matrices du modèle de base, on apprend une paire de petites matrices dont le produit vient s'ajouter à celles-ci. Le « faible rang » est tout le levier — sur la plateforme, il est fixé à **64** — et il se lit directement sur la balance : un adapter se compte en **dizaines de Mo** (un ordre de grandeur, pas une mesure faite ici), contre les ~15 Go de poids du modèle de base.

### Ce que ça évite : un second modèle complet

Ce que LoRA résout se comprend mieux par ce qu'il faudrait faire sans lui. Spécialiser un modèle — le rendre meilleur sur le SQL, sur le code sécurisé, sur le jargon maison — c'est le **fine-tuner** : reprendre l'entraînement sur un jeu de données ciblé. Un fine-tuning classique réécrit les poids, et ce qu'on obtient au bout est donc… **un second modèle complet**. Un second jeu de ~15 Go à stocker, à charger, à servir : un second pod, un second GPU, un second nœud sur la facture. Et l'addition se répète à chaque spécialisation — deux fine-tunes, trois GPU.

LoRA casse cette proportionnalité. Le fine-tune ne remplace pas les poids de base, il **s'ajoute** à eux : quelques dizaines de Mo posés par-dessus les ~15 Go que la plateforme sert déjà. Et comme c'est un delta, rien n'empêche d'en poser plusieurs, sur la même base, en même temps.

C'est vLLM qui encaisse cette propriété, et c'est là que tout se joue : **il charge le modèle de base une fois, puis applique les adapters par-dessus, dans le même processus, sur le même GPU** (`--enable-lora`, `--lora-modules`). Servir trois « modèles » — la base et ses deux fine-tunes — ne coûte donc pas trois GPU. C'est **un pod, un GPU, et trois noms auxquels il répond**. C'est ce qui rend possible le « zéro GPU » de la section suivante.

### Quand on sort un adapter — et quand on ne le fait pas

Le cas d'usage a une forme reconnaissable : **le modèle de base est presque bon, et on veut l'infléchir, pas le remplacer.** Les deux adapters de cette plateforme en sont l'illustration — l'un spécialisé en génération SQL, l'autre en code sécurisé — mais la famille est plus large :

* **Un vocabulaire de domaine** — les termes, les schémas et les usages d'un métier qu'un modèle généraliste connaît mal.
* **Un style de code maison** — les patterns, la structure de projet et les bibliothèques qu'une équipe utilise réellement.

Le contre-exemple compte tout autant : **LoRA n'apprend pas à un modèle une capacité qui lui manque fondamentalement.** Quelques dizaines de Mo de delta infléchissent un comportement, elles ne créent pas une compétence absente des poids de base. Un modèle qui ne sait pas raisonner sur du code ne l'apprendra pas par adapter — pour ça, on change de modèle de base, et c'est `model.repository` qui en parle, pas `loraAdapters`.

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

Deux lignes utiles. Flux réconcilie, et une requête sur dix adressée à `xplane-qwen-coder` est routée vers le fine-tune plutôt que vers les poids de base.

### Sous le capot : un split qui n'envoie rien ailleurs

La règle de l'`AIGatewayRoute` qui répond au nom du modèle de base ne porte plus un seul `backendRef`, mais **un par entrée** : le backend de base d'abord, puis un backend par canary. Le backend de base conserve `100 - sum(weightPercent)`, chaque canary prend son `weightPercent`, et chaque canary reçoit son propre `AIServiceBackend` (`<claim>-canary-<i>`) porteur d'un champ décisif — **`modelNameOverride`**, qui vaut le nom de l'adapter *verbatim*.

| `backendRef` | Poids | `modelNameOverride` | `Backend` visé |
|---|---|---|---|
| `AIServiceBackend` de base | **90** (`100 - 10`) | *(aucun)* | `xplane-qwen-coder` |
| `AIServiceBackend` de canary (`<claim>-canary-<i>`) | **10** | `xplane-qwen-coder-sql-dpo` | `xplane-qwen-coder` |

Relisez la dernière colonne. **Les deux `backendRef` pointent le même `Backend`** — le même `Service`, les mêmes pods, le même GPU. Rien n'est envoyé ailleurs, parce qu'il n'y a pas d'ailleurs. Tout ce que fait le canary, c'est **réétiqueter** 10 % des requêtes : `modelNameOverride` réécrit le nom du modèle avant que la requête n'atteigne vLLM, et vLLM — qui a déjà l'adapter en mémoire, puisqu'il l'a chargé au démarrage — la sert avec le fine-tune plutôt qu'avec les poids de base.

{{< img src="canary-split.png" alt="Split canary : un pod, un GPU, le modèle de base et deux adapters LoRA" width="1200" >}}

C'est là que se joue le « zéro GPU » du titre. Un canary applicatif classique coûte de la capacité : on déploie une v2 à côté de la v1, et pendant toute la durée de l'expérience on paie deux jeux de réplicas. Transposé ici, ce serait un **second pod vLLM**, donc un second GPU, donc un nœud de plus provisionné par Karpenter — pour évaluer un delta de poids qui se compte en dizaines de Mo. Le canary LoRA n'ajoute ni réplica, ni nœud, ni ligne de facture : le split est une décision de *routage*, prise en amont, sur une flotte qui n'a pas bougé.

### Le split, mesuré

Soixante requêtes vers `model: xplane-qwen-coder`, canary à 10 % armé. Puis douze requêtes nommant explicitement l'adapter. Ce que la gateway a compté :

| `gen_ai_original_model` (demandé) | `gen_ai_request_model` (servi) | n |
|---|---|---|
| `xplane-qwen-coder` | `xplane-qwen-coder` | **56** |
| `xplane-qwen-coder` | `xplane-qwen-coder-sql-dpo` | **5** |
| `xplane-qwen-coder-sql-dpo` | `xplane-qwen-coder-sql-dpo` | **12** |

**8,2 % de trafic capté pour un poids configuré à 10 %.** Sur n=61 et p=0,1, on attend 6,1 ± 2,3 : cinq est exactement là où il doit être. Et les douze requêtes explicites sont servies par l'adapter, **toutes les douze, sans une seule fuite vers la base** — le pin ne cède pas au tirage au sort.

La deuxième ligne est celle qui compte : le client a demandé le modèle de base, et c'est le fine-tune qui a répondu. Sans qu'il le sache, et sans qu'un seul GPU de plus soit allumé.

Mais la gateway n'est pas un témoin fiable d'elle-même : elle affirme avoir réétiqueté des requêtes, elle ne prouve pas que vLLM a joué le jeu. C'est le moteur qui tranche, et il le fait depuis ses propres métriques :

```
$ vllm:lora_requests_info
running_lora_adapters='(none)'                      max_lora=2
running_lora_adapters='xplane-qwen-coder-sql-dpo'   max_lora=2
```

**Deux plans indépendants, une seule décision de routage.** La gateway dit avoir envoyé 5 requêtes sur l'adapter ; vLLM confirme l'avoir chargé et exécuté. Le « zéro GPU » n'est plus une propriété de construction : c'est un pod, un GPU, et le moteur qui le dit.

### Le canary n'écrase pas le nommage explicite

Le split n'est pas le seul chemin vers un adapter, et c'est important : chaque entrée de `loraAdapters[]` reçoit **sa propre règle de routage** (`x-ai-eg-model: <name>` (l'en-tête sur lequel la Gateway route — voir plus bas) → backend de base, poids 100). Une requête qui nomme explicitement l'adapter est donc servie **à 100 % par cet adapter**, indépendamment du canary en cours.

Les deux comportements coexistent, et c'est voulu :

* **Le client qui demande `xplane-qwen-coder`** entre dans le split — 90 % base, 10 % fine-tune. C'est la population d'expérience : elle ne sait pas qu'elle en fait partie, et c'est précisément ce qui rend le signal intéressant.
* **Le client qui demande `xplane-qwen-coder-sql-dpo`** est **pinné** : il obtient l'adapter, toujours. C'est le mode d'usage explicite — celui de la partie 3 — et c'est aussi ce qui permet à une éval [Promptfoo](/fr/post/series/agentic_ai/llm-self-hosted-stack/) de comparer deux modèles sans dépendre d'un tirage au sort.

Autrement dit, le canary répond à « ce fine-tune tient-il face au trafic réel ? », le pin répond à « que vaut ce fine-tune, toutes choses égales par ailleurs ? ». Les deux questions sont légitimes, et aucune des deux ne doit désactiver l'autre.

### Ce que l'admission refuse

Un pourcentage de trafic est exactement le genre de champ où une faute de frappe se paie en production. Les garde-fous sont donc **à l'admission**, écrits en **CEL** (_Common Expression Language_ : le langage d'expressions que l'API server évalue lui-même, au moment de l'`apply`) dans l'**XRD** (_Composite Resource Definition_ : le schéma de l'API que la composition expose à ses utilisateurs). `kubectl apply` échoue, et la composition ne se déclenche même pas.

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

{{% notice warning "Une règle CEL sur un champ non borné est une règle non bornée" %}}
Ces règles ont failli ne jamais tourner — pas parce qu'elles étaient fausses, mais parce qu'elles étaient **trop chères**.

L'API server **estime statiquement** le coût de chaque règle CEL avant d'accepter le schéma, et il l'estime sur le **pire cas que le schéma autorise**. La règle qui vérifie que chaque canary vise un adapter déclaré croise deux tableaux :

```
canaries.all(c, loraAdapters.exists(a, a.name == c.adapter))
```

Une boucle dans une boucle, sur des comparaisons de chaînes. Tant que `loraAdapters` n'a **ni `maxItems`, ni `maxLength` sur ses noms**, l'estimateur suppose des tableaux infinis de chaînes infinies : le coût part à l'astronomique, et l'API server **rejette le CRD entier** — pas la règle, le CRD. Aucun modèle n'est plus déclarable.

L'asymétrie est frappante : dans le **même fichier**, `engineArgs` était borné (16 entrées, 256 caractères), et ses **dix-neuf** règles passaient sans problème. **Dix-neuf règles bornées coûtent moins cher qu'une seule non bornée.**

La correction n'est donc pas de simplifier la règle, c'est de **borner ce sur quoi elle porte** : `maxItems: 8` sur `loraAdapters`, `maxLength: 63` sur leur nom. Et une contrainte en cascade que le code documente : le `maxLength` de `canaries[].adapter` doit rester **aligné** sur celui de `loraAdapters[].name`, puisque l'estimateur facture la comparaison au plus grand des deux.

Une borne n'est pas de la cosmétique de schéma. C'est ce qui rend une règle **calculable**.
{{% /notice %}}

## :brain: Le prompt choisit le modèle

Jusqu'ici, le client **sait ce qu'il veut**. Il écrit `xplane-qwen-coder` dans son champ `model`, donc il connaît le catalogue, les noms, et lequel est bon en code. C'est une hypothèse raisonnable pour un service. C'en est une mauvaise pour un humain devant un chat.

La flotte compte quatre modèles spécialisés. L'idée du **Mixture of Models** (`model: MoM`) est de retourner la question : le client n'annonce pas un modèle, il **envoie son prompt**, et la plateforme décide.

```json
{"model": "MoM", "messages": [{"role": "user", "content": "integral of x squared"}]}
```

Le [routeur sémantique vLLM](https://github.com/vllm-project/semantic-router) classifie le prompt et le fait atterrir sur le modèle qui va bien :

| Le prompt | Classé | Servi par |
|---|---|---|
| *refactor this function and debug the traceback* | `code` | `xplane-qwen-coder` |
| *write a kubectl yaml manifest* | `code` | `xplane-qwen-coder` |
| *integral of x squared* | `math` | `xplane-qwen3-8b` |
| *capital of France* | `general` | `xplane-qwen3-8b` |

Ce routeur tournait déjà en partie 3. En bonne santé, scruté, avec ses métriques — et **jamais appelé**. Les règles de l'`AIGatewayRoute` matchent sur des noms de modèles concrets ; `MoM` n'en est pas un ; la Gateway répondait `404`. Le mécanisme était décrit dans un commentaire de `route.yaml`, et nulle part ailleurs.

### Où l'insérer : avant celui qui décide

Le brancher tient en une question, et elle est plus subtile qu'elle n'en a l'air : **à quel moment du chemin de requête ?**

L'Envoy AI Gateway a son propre **extproc** (_external processor_ : un service que le proxy appelle en cours de traitement, et qui peut modifier les en-têtes **comme le corps** de la requête) — et c'est lui qui **dérive l'en-tête `x-ai-eg-model` depuis le `model` du corps** de la requête. Cet en-tête est ce sur quoi *toutes* les règles de routage matchent. Le routeur sémantique doit donc s'exécuter **avant** lui : il réécrit `body.model`, de `MoM` vers un nom concret, et l'extproc de la Gateway dérive ensuite l'en-tête depuis un corps **déjà réécrit**.

La conséquence est la propriété qui fait tout tenir : **aucune règle de routage nouvelle.** Les routes existantes — celles que la composition génère, celles du canary, celles du pin explicite — matchent telles quelles, sans savoir que MoM existe.

Reste à obtenir cet « avant ». L'objet prévu pour ça, l'`EnvoyExtensionPolicy`, ne sait pas l'exprimer : il **ajoute** ses filtres *après* ceux de l'AI Gateway — trop tard, l'en-tête est déjà dérivé. Le montage passe donc par un **`EnvoyPatchPolicy`** : un patch xDS brut qui insère, à **l'index 0** de la chaîne de filtres HTTP, le propre extproc du routeur sémantique — lui aussi un filtre `ext_proc`, au même titre que celui de la Gateway.

```yaml
# apps/base/ai/llm/ai-gateway-routes/semantic-router-patch.yaml
kind: EnvoyPatchPolicy
spec:
  type: JSONPatch
  targetRef: {kind: Gateway, name: ai-gateway}
  jsonPatches:
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
      name: envoy-ai-gateway-system/ai-gateway/http
      operation:
        op: add
        path: /default_filter_chain/filters/0/typed_config/http_filters/0   # <- index 0
        value:
          name: semantic-router-extproc
          typedConfig:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_proc.v3.ExternalProcessor
            grpcService:
              envoyGrpc:
                authority: vllm-semantic-router.llm:50051
```

Un patch xDS brut est un objet qu'on préfère éviter : il est couplé à la forme interne de la configuration Envoy, pas à une API stable. C'est ici le prix de l'ordre — et c'est le montage que le projet amont documente lui-même.

### Deux couches qui se composent

Le montage a produit un résultat que rien n'avait explicitement câblé. Une requête `MoM`, classée `code`, réécrite en `xplane-qwen-coder`… **est tombée dans le canary LoRA** et a été servie par `xplane-qwen-coder-sql-dpo`.

Deux décisions, prises par deux composants qui ne se connaissent pas :

* le **routeur sémantique** décide *quel modèle* — à partir du sens du prompt ;
* la **route pondérée** décide *quels poids* — à partir d'un tirage à 10 %.

Elles s'empilent au lieu de se disputer le même point de décision, et c'est une propriété de la **position dans la chaîne**, pas une intention : le routeur agit sur le corps, la route agit sur l'en-tête qui en est dérivé. Chacune ignore l'autre, et c'est exactement pour ça qu'elles tiennent ensemble.

MoM résout d'ailleurs à travers un modèle **migré** (routage possédé par la composition) *et* un modèle **non migré** (encore dans `route.yaml`). Les deux mondes coexistent pendant la migration — ce qui est la moindre des choses pour une migration qu'on annonce progressive.

{{% notice note "Ce que ça ne fait pas" %}}
La classification est un modèle, pas une vérité. Un prompt ambigu tombe où il tombe, et rien ici ne mesure la **qualité** de la classification — je n'ai vérifié que le **câblage** : que le routeur soit appelé, qu'il réécrive, et que la route matche derrière. Savoir si `math` est le bon verdict pour une intégrale, c'est le domaine du routeur ; savoir si le chemin de requête l'appelle, c'est celui de la plateforme. Cet article ne traite que du second.
{{% /notice %}}

## :gear: Configuration avancée : ouvrir sans trouer

Une abstraction curée ne peut pas tout prévoir. Le schéma de la claim n'expose que des champs modélisés un par un, et vLLM ajoute des options à chaque release, plus vite qu'une API de plateforme ne peut les exposer. Chaque flag absent du schéma se payait donc au prix fort : une PR sur la composition KCL, les tests unitaires, la republication de l'image OCI, le bump du tag dans la `Composition`, puis Flux. **Une release de plateforme pour un essai d'une après-midi.**

Une API qui ne laisse aucune sortie ne bloque personne : elle se fait **contourner**. Un `kubectl edit deployment`, le flag ajouté à la main, et jusqu'à la prochaine réconciliation de Crossplane ça marche très bien. Le coût réel n'est pas le contournement, c'est ce qu'il fait à l'abstraction : dès l'instant où l'état réel du cluster ne se lit plus dans la claim, **la claim ment**. Et une abstraction qui ment est pire qu'une abstraction absente, parce qu'elle continue de prétendre décrire le système.

Le réflexe symétrique — un champ fourre-tout, non contraint — ne résout rien non plus : on troue alors l'abstraction de l'intérieur, avec sa bénédiction. `spec.engineArgs` est la troisième voie : il laisse passer les **options non curées** — celles que le schéma n'a pas anticipées — sauf les **dix-huit** dont l'autoscaling et le routage dépendent.

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

### Dix-huit flags qui ne passeront pas

Le tri est simple à énoncer : **tout passe, sauf ce dont l'autoscaling et le routage dépendent.** Ces flags-là sont **réservés**, et l'XRD les refuse à l'admission — `kubectl apply` échoue, rien n'atteint le cluster.

**Dix-neuf** règles CEL portent cette vérification : une règle de forme (toute entrée commence par `--`), et **dix-huit règles de flags réservés, une par flag**. Une règle unique bouclant sur une liste aurait rejeté tout aussi bien, mais elle aurait produit le même message pour les dix-huit — un message qui dit à l'utilisateur qu'il a perdu, sans lui dire quoi faire. Une règle par flag, c'est **un message par flag**.

Ce n'est pas une intention, c'est ce que l'API server répond. Verbatim, sur un cluster :

```
$ kubectl apply --dry-run=server -f claim-avec-max-num-seqs.yaml
The InferenceService "cel-reserved-maxnumseqs" is invalid: spec: Invalid value:
  --max-num-seqs is composition-managed; use spec.model.maxNumSeqs
  (it is the KEDA scaling denominator)

$ kubectl apply --dry-run=server -f claim-avec-flag-eclate.yaml
The InferenceService "cel-bare-value" is invalid: spec: Invalid value:
  each spec.engineArgs entry must be a single token starting with '--'
  (use --flag=value, not --flag value)

$ kubectl apply --dry-run=server -f claim-avec-load-format.yaml
The InferenceService "cel-reserved-loadformat" is invalid: spec: Invalid value:
  --load-format is composition-managed; use spec.model.streaming.enabled
```

Chaque message **nomme le champ curé à utiliser à la place**. C'est toute la différence entre une API qui dit non et une API qui dit où aller.

Dix-huit **dans cette version**, et le nombre n'a rien de sacré : la denylist grandit avec la plateforme. Chaque nouveau champ curé réserve le flag qu'il pilote — les deux du Run:ai Model Streamer, plus bas dans cet article, **l'ont déjà rejointe** : c'est précisément pour ça qu'ils sont dix-huit et non seize. C'est ce qui rend la question suivante décisive : une liste qui bouge est une liste qui ne doit vivre qu'à **un seul endroit**.

Prenez `--max-num-seqs` : c'est le **dénominateur** du trigger KEDA vu en partie 3. Laissez un utilisateur le redéfinir, et l'autoscaler continue de calculer son taux de saturation contre une taille de batch qui n'existe plus. Rien ne casse, rien n'alerte : le scaling est simplement **faux**.

Deux flags méritent un arrêt particulier, parce que **la composition ne les émet jamais** : `--port` et `--host`. Elle laisse le moteur prendre ses défauts — le port `8000`, l'écoute sur toutes les interfaces — et pourtant elle les réserve. Ces défauts ne sont pas des détails d'implémentation, ce sont **le contrat de serving** : le `targetPort` du `Service` vise 8000, les probes interrogent 8000, le FQDN du `Backend` rendu par la Gateway pointe 8000. Un `--port=9000` dans `engineArgs`, et les trois pointent dans le vide.

La leçon dépasse les LLM : **les garde-fous d'une configuration avancée doivent couvrir non seulement ce que l'abstraction écrit, mais ce qu'elle suppose.** Le second ensemble est celui qu'on oublie, parce qu'il n'apparaît dans aucun diff.

### Une seule source d'application

Reste la question qui décide de tout : **où vit la denylist ?** À **un seul endroit**, le CEL de l'XRD. La composition, elle, ne vérifie rien : elle appende les `engineArgs` verbatim et fait confiance à l'admission.

La tentation de la « défense en profondeur » était pourtant là : refiltrer côté KCL, par acquit de conscience. Ç'aurait été recréer exactement le problème que la première moitié de cet article vient de supprimer — une liste de flags réservés dans l'XRD, une autre dans la composition. **Deux fichiers à garder synchronisés, donc deux dérives possibles, et aucune ne fait de bruit.** Même forme de bug que `route.yaml` : autre fichier, même piège.

Mais « un seul endroit » est une promesse fragile, parce que la denylist épouse ce que la composition émet — et que ce que la composition émet grandit à chaque version. Le jour où quelqu'un ajoute un argument géré au KCL sans l'ajouter à la denylist de l'XRD, le trou se rouvre — et personne ne le verra. Un test verrouille donc les deux ensembles l'un à l'autre : `test_engine_args_denylist_lockstep` lit les flags que la composition produit et vérifie que chacun figure bien dans la denylist.

{{% notice tip "Un test qui ne peut pas échouer ne prouve rien" %}}
Un test de cohérence entre deux listes a une propriété désagréable : il passe au vert le jour où on l'écrit, et il passerait tout aussi bien s'il ne testait rien du tout. Une faute de frappe dans un nom de variable, une assertion qui compare une liste vide à une liste vide — et vous obtenez le pire des artefacts : un test qui **rassure**.

D'où l'étape suivante : **casser volontairement la denylist**, relancer la suite, et vérifier que `test_engine_args_denylist_lockstep` **échoue bien**. C'est une vérification par **mutation** : on introduit la faute exprès pour s'assurer que le filet la rattrape. Tant que vous n'avez pas vu un test rouge, vous n'avez pas de test — vous avez une ligne dans un rapport de CI.
{{% /notice %}}

`engineArgs` ne rend donc pas la plateforme plus permissive : il rend sa permissivité **lisible**. Ce qui n'est pas listé passe, sans demander la permission à personne. Ce qui est listé est refusé à l'`apply`, avec le nom du champ à utiliser à la place. **Liberté là où c'est sûr, garde-fous là où ça ne l'est pas.**

## :eyes: Le modèle dit ce qu'il sert

`kubectl get isvc` répondait `READY`, et c'était tout ce qu'il répondait. Une information de santé : un pod tourne, il accepte des requêtes. Pas **à quels noms** il les accepte. La question était triviale tant qu'un pod servait un modèle sous un nom. Elle ne l'est plus : le même pod répond maintenant à un modèle de base et à deux adapters, et une requête sur dix adressée au premier part discrètement vers l'un des seconds.

C'est le cinquième axe, adressé par un champ de statut. `status.servedModels` est une liste d'objets, republiée à chaque réconciliation :

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

Et une colonne `SERVED MODELS` porte la même chose jusque dans un `kubectl get isvc`, sans `-o yaml` ni `jq` :

```
$ kubectl get inferenceservice -n llm
NAME                    SERVED MODELS                                                              SYNCED   READY
xplane-llamaguard3-1b   xplane-llamaguard3-1b                                                      True     True
xplane-qwen-coder       xplane-qwen-coder,xplane-qwen-coder-sql-dpo,xplane-qwen-coder-securecode   True     True
xplane-qwen-coder-fim   xplane-qwen-coder-fim                                                      True     True
xplane-qwen3-8b         xplane-qwen3-8b                                                            True     True
```

La deuxième ligne est celle qui n'existait pas avant : **un pod, un GPU, trois noms.** La question « qu'est-ce que ce pod sert, au juste ? » a maintenant une réponse à l'endroit où on la pose.

### C'est de la topologie, pas de la santé

Un détail conditionne toute la lecture de ce champ : `servedModels` est calculé **depuis le spec seul**. La composition ne consulte ni l'`ocds`, ni l'état du `Deployment`, ni celui de la route — elle dérive la liste des noms de ce que la claim déclare. Le champ est donc peuplé **dès la première réconciliation**, avant même que le pod n'existe.

C'est une projection de **topologie**, pas un signal de santé. Il répond à « à quels noms cette claim est-elle *censée* répondre ? », jamais à « répond-elle, là, maintenant ? ». Un pod en `CrashLoopBackOff` affiche les mêmes `servedModels` qu'un modèle en pleine forme — c'est le comportement voulu, pas une lacune. La santé, elle, a déjà ses réponses ailleurs : la condition `READY`, le latch de readiness, les métriques vLLM de la partie 3.

{{% notice note "Le piège : un JSONPath wildcard dans une colonne" %}}
La colonne `SERVED MODELS` a d'abord pointé le JSONPath qui s'imposait naturellement : `.status.servedModels[*].name` — *tous* les noms de la liste. Elle n'en aurait affiché **qu'un seul**.

Les colonnes de `kubectl get` ne sont pas rendues par le client, mais côté serveur, par le **convertisseur de table** de l'API server, qui applique le JSONPath des `additionalPrinterColumns` du CRD — et n'en retient que la **première correspondance**. Une cellule de tableau est un scalaire, pas une liste : le wildcard ne boucle pas, il sélectionne. La colonne aurait donc affiché `xplane-qwen-coder`, et les deux adapters n'auraient existé nulle part.

D'où un second champ, `status.servedModelsSummary` : un scalaire, les noms joints par des virgules, sur lequel la colonne pointe désormais.
{{% /notice %}}

{{% notice tip "Publier un statut depuis KCL : une clé de configuration décide de tout" %}}
Une composition KCL publie son statut en émettant un item **desired-composite** — l'objet composite lui-même, avec son `status`, au milieu des ressources qu'elle rend.

Cet item n'est honoré que si la fonction est câblée en **`target: Default`**. Sous **`target: Resources`**, la fonction ne connaît pas ce cas : elle traite l'item comme n'importe quelle ressource à composer. Le statut n'est donc jamais publié — et, pire, il **fuit en ressource composée fantôme**, un objet sans `spec` que Crossplane essaie d'appliquer et n'y arrive jamais. La réconciliation entière se bloque, pour un champ de statut.

Le KCL était juste ; c'était le **câblage** qui ne l'était pas. Une ligne dans la `Composition`, et rien dans le code de la composition ne le laissait deviner.
{{% /notice %}}

Deux champs pour une seule vérité, après un article passé à traquer les doubles sources — l'ironie mérite d'être adressée. Mais `servedModelsSummary` n'est pas une seconde saisie : c'est une **fonction pure** de `servedModels`, calculée dans le même passage de la composition, et il n'existe aucun chemin par lequel les deux divergent. Le champ structuré reste l'API ; le scalaire n'est qu'une projection pour le CLI.

Et la leçon dépasse ce champ-là : **un statut que l'outil de tous les jours ne sait pas afficher n'est pas de la découvrabilité.** C'est de la donnée correcte, rangée dans un endroit que personne ne regarde.

## :bar_chart: Mesurer le canary

Tout ce qui précède produit un split : 90 % du trafic sur les poids de base, 10 % sur le fine-tune. Reste la seule question qui justifiait de le faire — **est-ce que ces 10 % se comportent mieux, ou moins bien, que les 90 % restants ?** Sans réponse, on n'a pas déployé un canary : on a détourné du trafic utilisateur vers autre chose en croisant les doigts. **Un canary qu'on ne peut pas mesurer n'est pas un canary, c'est un pari.**

La partie 3 a déjà présenté les métriques **`gen_ai.*`** de l'Envoy AI Gateway (v1.0) : le standard [OpenTelemetry Gen AI](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/), qui ne compte pas des requêtes HTTP mais le vocabulaire métier des LLM — tokens consommés, **TTFT** (_time to first token_ : le délai entre la requête et le premier token de la réponse), latence par token de sortie. Il restait à les **brancher**.

Elles ne sont pas émises par le proxy Envoy lui-même, mais par l'extproc de la gateway — celui-là même qui dérive `x-ai-eg-model` : le sidecar auquel Envoy délègue le traitement spécifique aux LLM, seul sur le chemin de requête à voir à la fois le nom **demandé** et le nom **servi**. C'est là que le scrape se branche :

* **Cible** : le sidecar extproc, pas le conteneur Envoy — d'où un **`VMPodScrape`** (l'objet VictoriaMetrics qui cible des *pods*) et non un `VMServiceScrape`
* **Namespace** : `envoy-gateway-system`
* **Port** : `1064`, le port d'administration de l'extproc

Le port `1064`, le nom du port du conteneur et la règle d'autorisation réseau ne venaient d'aucune documentation : ils avaient été **lus dans le code source** de l'AI Gateway. Sur le cluster, la cible remonte :

```
up=1  job=envoy-gateway-system/envoy-ai-gateway-genai
```

Les trois étaient bons.

Les séries atterrissent dans la VictoriaMetrics qui collecte déjà les métriques vLLM, et se lisent dans le même Grafana. Aucun nouvel outil.

### L'attribution canary vs base : la gateway répond

`modelNameOverride` réécrit le nom du modèle **verbatim** vers celui de l'adapter avant que la requête n'atteigne vLLM. Le tableau du split reposait sur une propriété qu'il faut maintenant expliciter : la gateway émet **deux labels distincts**.

| Label | Ce qu'il porte |
|---|---|
| `gen_ai_original_model` | ce que le **client** a demandé |
| `gen_ai_request_model` | ce qui a été **servi** — donc le `modelNameOverride` du canary |

Conséquence, et elle est gratuite : **le split de routage est aussi un split de mesure.** Découper une série par `gen_ai_request_model` sépare le canary de la base ; la croiser avec `gen_ai_original_model` distingue en plus la population d'expérience (qui a demandé la base et reçu le fine-tune) des clients qui ont explicitement pinné l'adapter. C'est le tableau à trois lignes de la section canary — et il sort d'une seule requête PromQL.

Le dashboard **`llm-gateway`** n'a donc plus à choisir : les mêmes séries, découpées canary contre base.

* La **latence end-to-end** (`gen_ai.server.request.duration`) et le **TTFT** (`gen_ai.server.time_to_first_token`) : servir via un adapter LoRA coûte-t-il le même chemin que servir sur les poids de base ?
* La **latence par token de sortie** (`gen_ai.server.time_per_output_token`), qui dit si le fine-tune génère au même rythme.
* La **consommation de tokens** (`gen_ai.client.token.usage`) : un fine-tune plus bavard que sa base, à qualité égale, est un fine-tune plus cher.
* Le **volume de requêtes** de chaque branche — de quoi vérifier que le split fait ce qu'il annonce.

{{% notice tip "Le nom de la métrique n'est pas celui qu'on croit" %}}
Un détail qui coûte cher à qui l'ignore : la série s'appelle `gen_ai_client_token_usage_sum`, **sans suffixe d'unité**. `gen_ai_client_token_usage_tokens_sum` n'existe pas — et c'est pourtant ce que le dashboard visait dans sa première version.

Un panneau qui interroge une série inexistante ne lève **aucune erreur**. Il affiche « No data », ce qu'un panneau affiche aussi quand tout va bien et qu'il ne se passe rien. Le dashboard serait resté vide pour toujours, et personne n'aurait su faire la différence.
{{% /notice %}}

<!-- SCREENSHOT-EN-ATTENTE: dashboard Grafana llm-gateway, attribution canary vs base (à capturer) -->

Ces courbes disent si le canary est plus lent, ou plus bavard, que la base. Elles ne disent pas s'il est **meilleur** : aucune métrique de gateway ne sait ce qu'est un bon SQL. La qualité reste le domaine de l'éval — [Promptfoo](/fr/post/series/agentic_ai/llm-self-hosted-stack/), vu en partie 3 — et c'est là que la règle de pin reprend tout son sens : une éval doit interroger l'adapter **de façon déterministe**, pas tomber dessus une fois sur dix. Promouvoir un fine-tune, c'est regarder les deux.

{{% notice note "Ce qui n'est pas livré : les traces" %}}
L'AI Gateway sait exporter des **traces OTLP**, et la cible naturelle serait **VictoriaTraces**. Sur cette plateforme, l'export est **volontairement désactivé** : je n'ai pas vérifié la compatibilité du chemin d'ingestion OTLP entre les deux, et tant que ce n'est pas fait, la case reste décochée.

Livrer un export vers un endpoint qu'on n'a pas testé, c'est livrer une ligne de configuration qui donne l'**illusion** d'une capacité : le jour où on voudrait savoir où part le temps entre la gateway, l'extproc et vLLM, on découvrirait qu'il n'y a rien au bout. Une case décochée est honnête ; une case cochée qui ne transporte rien ne l'est pas.
{{% /notice %}}

## :compass: Endpoint Picker : au-delà du round-robin

Le canary et le routage sémantique décident tous deux **quel modèle** sert une requête. Reste **quel réplica** — tranché par défaut par le `Service` ClusterIP et son **round-robin** (chaque requête au pod suivant). Sur vLLM, c'est **hostile**, pour deux raisons qui attaquent ce que la [partie 3](/fr/post/series/agentic_ai/llm-self-hosted-stack/) avait mis en place :

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

{{% notice warning "Envoy Gateway ne sait pas ce qu'est une InferencePool" %}}
Le piège de cette fonctionnalité n'est pas dans la composition, il est **un étage plus bas**.

Envoy Gateway v1.8.2 ne contient **pas un seul fichier** mentionnant « inference ». Le support de l'`InferencePool` n'est pas natif : il est **entièrement délégué** à l'**extension server** de l'AI Gateway (_extension server_ : le composant d'Envoy Gateway à qui est déléguée toute configuration qu'Envoy ne sait pas traiter nativement). Et il faut **lui dire de déléguer** :

```yaml
extensionManager:
  backendResources:
    - group: inference.networking.k8s.io
      kind: InferencePool
      version: v1
```

Sans cette clé, tout le plan de contrôle a l'air parfait — l'`InferencePool` est `Accepted=True`, l'`AIGatewayRoute` aussi — et les requêtes prennent un `500`, sans qu'Envoy ait même *tenté* un upstream. Le pool n'existe simplement pas dans sa configuration.
{{% /notice %}}

{{% notice tip "Le chart EPP ne passe pas PSS restricted, et n'expose aucune clé pour ça" %}}
Le chart `inferencepool` (v1.5.0) rend son `Deployment` EPP **sans aucun `securityContext`** — ni au niveau du pod, ni du conteneur — et n'expose **aucune valeur** pour en poser un. Sous **PSS restricted** (_Pod Security Standards_, le profil le plus strict de Kubernetes), le pod est refusé à l'admission, Helm attend, abandonne, réinstalle, en boucle.

Deux issues. Un **webhook mutant cluster-wide** (type Kyverno) corrige tous les pods du cluster — et met une dépendance de plus sur le chemin d'admission de *tout* le monde, pour un besoin qui concerne un chart. Ou un **`postRenderer` Flux**, qui patche la release **et rien d'autre** : scopé à la claim, ramassé par le GC avec elle, invisible pour le reste du cluster.

C'est le second qui a été retenu. Un correctif local pour un problème local — quitte à ce qu'il soit moins élégant qu'une politique globale.
{{% /notice %}}

### KEDA et l'EPP ne se disputent rien

Le réflexe est d'y voir un doublon de l'autoscaler. Ce n'en est pas un : **ils opèrent à deux échelles de temps différentes.** KEDA décide **combien** de réplicas exister, en dizaines de secondes ; l'EPP décide **lequel** des réplicas existants sert la requête, en millisecondes. Les deux lisent les mêmes signaux vLLM, pour des décisions différentes.

Troisième trigger KEDA ajouté au passage : aux deux de la partie 3 s'ajoute **`vllm:num_requests_waiting`** (`scaling.queueLengthThreshold`, défaut **8**) — les requêtes acceptées mais pas encore traitées, le signal de pression **le plus précoce** des trois. Les triggers se combinent en **OR** : le premier à franchir son seuil déclenche le scale-up.

Sur le chiffre, restons honnête. Le projet **llm-d** rapporte jusqu'à **57× d'amélioration du TTFT** face au round-robin sous charge. C'est une mesure **externe**, faite par d'autres, sur leur charge et leur matériel — **je ne l'ai pas reproduite ici**, et elle décrit vraisemblablement un cas où le prefix cache a beaucoup à donner. Je la cite pour l'ordre de grandeur du levier, pas comme un résultat de cette plateforme.

Et la mienne n'est pas mesurée non plus. L'EPP tourne sur le cluster, il score les réplicas sur les métriques vLLM qu'il scrape lui-même, et le chemin de requête est propre. Mais la **distribution par préfixe** — le gain que tout ceci vise — demande de saturer plusieurs réplicas d'un même modèle avec du trafic qui partage ses préfixes. Le `NodePool` GPU est plafonné à quatre GPU, et la flotte de quatre modèles le sature exactement. Le garde-fou fait son travail ; la mesure attend un budget GPU qu'elle n'a pas encore.

{{% notice warning "EPP et canary : il faut choisir" %}}
Les deux fonctionnalités les plus visibles de cette PR sont **mutuellement exclusives**. Une claim qui active `gateway.endpointPicker` avec un `gateway.canaries` non vide est **rejetée à l'admission**.

La cause est structurelle : un `backendRef` pointant une `InferencePool` ne supporte **ni `weight`, ni `modelNameOverride`** — les deux mécanismes du canary — et une seule `InferencePool` est admise par règle. Rien à composer : ils parlent deux langages de `backendRef` incompatibles.

Il faut donc choisir, aujourd'hui. Un rejet net à l'`apply` vaut mieux qu'une combinaison silencieusement incomplète.
{{% /notice %}}

## :stopwatch: Le streamer tient sa promesse — sur les 30 % qui ne comptent pas

{{% notice tip "Au passage : le dernier maillon du cold-start" %}}
La partie 3 avait réglé le gros du démarrage à froid avec un **PVC S3 Files**, évitant de retélécharger ~15 Go depuis HuggingFace. Restait un maillon : la lecture de ces poids depuis le volume vers la VRAM.

Le **Run:ai Model Streamer** (`--load-format runai_streamer`) s'attaque à celui-là : selon la documentation du projet, il lit les poids de façon concurrente et les transfère vers le GPU au fil de la lecture, sans attendre qu'ils soient tous chargés en mémoire hôte. Embarqué dans l'image `vllm-openai` depuis la **v0.8.5**, il s'active par `model.streaming` sur le PVC déjà en place. Ses flags ont rejoint la denylist `engineArgs` — la confirmation, déjà annoncée plus haut, que cette liste grandit avec la plateforme.

Deux capacités restent **descopées** : le chargement `s3://` direct et le résolveur LoRA, qui demandent **vLLM v0.9.0**. Combien de secondes cela fait-il gagner ? La mesure suit, juste en dessous.
{{% /notice %}}

Même modèle, mêmes **8,8 Gio** de poids une fois quantifiés en fp8 (les ~15 Go du dépôt HuggingFace), un seul changement : `--load-format runai_streamer`.

```
                              streaming OFF    streaming ON      delta
Chargement des poids             62,53 s    →    55,22 s      −7,3 s (−11,7 %)
Init du moteur (profiling,
  kv cache, warmup)             123,77 s    →   124,02 s      inchangé
  dont capture de graphes CUDA      33 s    →       33 s      inchangé
```

Le streamer fait **exactement ce qu'il annonce** : −11,7 % sur le chargement des poids, un gain réel, sans contrepartie.

Et il ne change presque rien. Parce que le chargement des poids ne pèse qu'environ **30 %** d'un démarrage à froid de ~180 s. Les ~68 % restants, c'est l'**initialisation du moteur** — profiling mémoire, allocation du KV cache, warmup, dont 33 secondes de seule capture de graphes CUDA — et le streamer n'y touche pas. Par conception : ce n'est pas son travail.

Gain sur le cold-start **total** : **~4 %**.

C'est une leçon d'optimisation avant d'être une leçon de LLM. **Accélérer de 11,7 % une phase qui pèse 30 % rapporte 4 %** — et on peut passer une soirée entière à mesurer ce 11,7 % avec bonheur sans jamais regarder les 68 % d'à côté. Le vrai levier de cette plateforme n'est pas le chargement des poids : c'est l'init du moteur, et je ne l'ai pas encore attaqué.

Cela tranche aussi la question que la spec posait elle-même : le gain sur un PVC local est **réel mais modeste**, parce qu'un montage NFS ne laisse pas le streamer faire les *range-GETs* concurrents qui le font briller face à un object store. **Le gain promis vit dans la phase 2 — le chargement `s3://` direct** — et pas ici.

## :telescope: Modelplane : deux étages, pas deux camps

Le 23 juin 2026 — donc pendant que cette PR se construisait — les créateurs de Crossplane ont publié [**Modelplane**](https://github.com/modelplaneai/modelplane) : un plan de contrôle open-source pour l'inférence, à l'échelle de la **flotte**. Le mérite leur revient d'avoir mis le bon objet au centre : ce qu'on gère, ce sont des **modèles**, pas des clusters. Leur API porte d'ailleurs la même séparation des rôles que celle défendue ici — côté équipe plateforme, `InferenceCluster` et `InferenceClass` déclarent la capacité disponible ; côté équipe ML, `ModelDeployment` et `ModelService` déclarent le modèle à faire tourner et le service qui l'expose ; le tout est servi par une `InferenceGateway` commune aux deux.

Quand les gens qui ont construit Crossplane arrivent à la même forme, ce n'est pas une coïncidence : c'est que le problème, lui, a une forme. Sur la chronologie, coupons court — leur billet est postérieur à la v0.6.0 de cette plateforme (9 mai 2026), et je n'en tire rien du tout. Personne n'a copié personne : c'est une **évolution convergente**, deux équipes qui butent sur le même mur et en sortent avec le même plan. Et surtout, il n'y a **rien à départager** — ce ne sont pas deux réponses à la même question.

### Deux étages du même bâtiment

Modelplane opère **au-dessus** du cluster : provisionner des clusters GPU chez plusieurs fournisseurs, placer un modèle sur du matériel hétérogène, mettre une seule gateway devant l'ensemble. Ce que décrit cet article opère **à l'intérieur** d'un cluster. Les deux couches s'empilent ; elles ne se disputent pas le même terrain.

| | **Au-dessus du cluster** — Modelplane | **Dans le cluster** — `InferenceService` |
|---|---|---|
| **La question traitée** | *où* poser ce modèle : quel cluster, quel fournisseur, quel GPU ? | *comment* ce cluster le sert : ses adapters, sa route, son trafic |
| **L'objet géré** | la flotte : multi-cluster, multi-cloud, GPU hétérogènes | un cluster : la claim et les ressources qu'elle produit |
| **Le placement** | un scheduler choisit où atterrit chaque réplica | Karpenter, à l'intérieur du cluster |

Leurs propres docs le disent sans détour : Modelplane **compose** les projets de niveau cluster plutôt que de les remplacer. Une stack de la forme de celle décrite ici est très exactement ce qui vivrait **en dessous** de la leur. Le problème qu'ils résolvent — placer un modèle sur le bon GPU, dans le bon cluster, chez le bon fournisseur — **je ne l'ai tout simplement pas** : cette plateforme est une référence mono-cluster, et prétendre qu'elle adresse la flotte serait du bruit.

La contribution de cet article est donc d'un autre ordre : **creuser un seul cluster.** Le **canary LoRA pondéré**, à coût GPU nul par construction, validé à l'admission et couvert par les tests de la composition (`kcl test` : 44/44). Les **signaux d'autoscaling curés**, pris sur les métriques de saturation les plus précoces de vLLM. Et le **durcissement comme plancher**, pas comme remarque de fin : zero-trust Cilium, PSS restricted, secrets, et un GitOps de bout en bout où la claim reste la seule chose qu'un utilisateur écrit.

Ce qui était une réserve à la rédaction ne l'est plus : le rendu est prouvé, **et le split aussi** — sur du trafic vivant, corroboré par le moteur. Reste une seule case non cochée, et je la garde visible : la distribution par préfixe de l'Endpoint Picker, faute de GPU pour la produire.

### Ce que leur revue a changé, ici

Ce que leurs docs de design ont réellement servi ici, c'est une **grille de revue** : des questions posées par d'autres, à confronter aux choix déjà faits. Et c'est le point le plus intéressant — cette lecture a **modifié le code de cette PR**, et pas à la marge. Ce qu'elle apporte tient en une discipline : la **forward-compatibilité**, c'est-à-dire concevoir un champ non pour le besoin du jour, mais pour que le besoin du mois prochain n'impose pas de casser l'API.

* **Des arrays plutôt que des objets** — déjà détaillé dans l'encart « Pourquoi un array dès le premier jour » de la section canary.
* **Aucun nouveau champ requis.** Toute claim v0.6.0 s'applique telle quelle en v0.8.0. `gateway`, `canaries`, `engineArgs`, `endpointPicker` : tout est optionnel, et chaque défaut préserve le comportement d'avant.
* **Des flags fournis par l'utilisateur plutôt qu'injectés.** Le champ `engineArgs` existe parce que cette revue a rendu évident qu'aucune API de plateforme ne rattrapera jamais le rythme d'un moteur d'inférence. Le choix n'est pas *modéliser ou interdire*, c'est *encadrer ou se faire contourner*.

Reste le caveat qui garde l'exercice honnête, et il joue contre moi : **ils sont engine-agnostic, je ne le suis pas.** Modelplane parle vLLM, SGLang, TensorRT-LLM, TGI ou llama.cpp, sur NVIDIA, AMD, TPU, Trainium ou Gaudi. La composition décrite ici parle vLLM, produit des flags vLLM, scale sur des métriques vLLM. C'est un choix de périmètre, pas un oubli — et c'est précisément ce qui rend possibles les signaux d'autoscaling curés et la denylist des flags réservés. On ne peut pas savoir que `--max-num-seqs` est le dénominateur d'un trigger KEDA **et** être agnostique au moteur qui l'expose. Le couplage est le prix de la profondeur ; la configuration avancée est là pour qu'il ne devienne pas une prison.

---

## :thought_balloon: Dernières remarques

Cette PR ne rend pas la plateforme plus puissante — un pod vLLM sur un GPU faisait déjà tourner un modèle en partie 3. Elle rend l'abstraction **complète** et **évolutive** : la claim produit désormais *toutes* les ressources du modèle, routage compris, et elle sait le faire bouger par paliers. Le chemin pour y arriver a produit cinq leçons qui n'ont, au fond, rien à voir avec les LLM.

* **Si deux fichiers décrivent la même chose, ils divergeront.** Ce n'est pas un problème de discipline, c'est un problème de temps : la dérive est une fonction du nombre de jours, pas du sérieux de l'équipe. La seule issue est structurelle — la composition doit **posséder les deux**, ou il n'y a plus qu'un fichier. Et posséder une ressource, c'est aussi décider **quand** elle existe : une route qui apparaît trop tôt envoie du trafic dans le vide, une route qui disparaît trop vite fait du flapping. Portail à la création, latch ensuite.
* **Un rollout progressif ne coûte un GPU de plus que si l'abstraction confond le modèle et le pod.** La politique de trafic appartient au **modèle**, pas au réplica : dès qu'on la déclare au bon niveau, déplacer 10 % du trafic vers un fine-tune devient une décision de routage — pas une deuxième flotte à provisionner et à payer.
* **Les couches de décision se composent si — et seulement si — elles décident à des endroits différents.** Le routeur sémantique tranche *quel modèle* en réécrivant le corps de la requête ; la route pondérée tranche *quels poids* en lisant l'en-tête qui en est dérivé. Aucune des deux ne connaît l'autre, et c'est précisément pour ça qu'elles tiennent ensemble. Deux composants qui décident au même point du chemin se disputent ; deux composants qui décident à deux points s'empilent. La position dans la chaîne *est* le contrat.
* **Une API qui ne laisse aucune sortie finit contournée ; une configuration avancée sans garde-fous casse les invariants.** Entre les deux : une **denylist explicite**, appliquée en **un seul** endroit. Deux endroits, et on vient de recréer le premier problème.
* **Si `kubectl get` ne dit pas ce que la ressource fait vraiment, l'abstraction ment.** Un statut correct rangé là où personne ne le lit n'est pas de la découvrabilité.

Ce que je retiens surtout, c'est le déplacement qui les relie toutes : **l'unité de déploiement n'est plus « un pod qui sert un modèle », c'est « un modèle, avec sa politique de trafic, sa stratégie de rollout et sa découvrabilité »**. Tant qu'une abstraction ne modélise que le pod, tout le reste — la route, le split, les noms servis — retombe sur un humain, dans un fichier à côté, à la main. Et quand elle les modélise enfin, une dernière marche apparaît, gratuite : le client n'a plus besoin de savoir ce qu'il demande. `model: MoM`, et la plateforme s'en charge.

Avec une limite que cet article a lui-même établie, et qui reste à lever : aujourd'hui, `gateway.endpointPicker` et `gateway.canaries` **s'excluent à l'admission**. Dans une même claim, on déclare donc la politique de trafic **ou** la stratégie de rollout — pas encore les deux. L'unité de déploiement est la bonne ; le prochain axe d'amélioration est déjà nommé.

{{% notice tip "Le zero-trust n'est utile que le jour où il refuse quelque chose" %}}
Une trouvaille du passage sur cluster, minuscule et satisfaisante. Au démarrage, vLLM poste des statistiques d'usage vers `stats.vllm.ai`. Personne ne l'avait demandé, et personne ne l'aurait vu.

La `CiliumNetworkPolicy` par défaut-deny de la partie 3 l'a **bloqué**, sans qu'on lui demande rien non plus :

```
$ hubble observe --namespace llm --verdict DROPPED
llm/xplane-qwen-coder-... <> 172.67.154.127:443 (world)  Policy denied  DROPPED

$ cilium fqdn cache list | grep 172.67.154.127
172.67.154.127 -> stats.vllm.ai
```

Le plancher de sécurité de la partie 3 a fait exactement son travail, sur une sortie réseau que je n'avais pas prévue. C'est la seule preuve qui compte pour une politique par défaut-deny : celle qui arrive quand on ne la regardait pas. (Posture plus propre encore : `VLLM_NO_USAGE_STATS=1`, pour qu'il n'essaie même pas.)
{{% /notice %}}

### Et après ?

Le chaînon vraiment manquant, c'est le premier de cette liste : **fine-tuner mon propre adapter**. Les deux LoRA de cet article viennent de HuggingFace, entraînés par d'autres ; le canary sait donc exposer progressivement un fine-tune au trafic réel, mais il n'a encore jamais servi à valider *le mien*. C'est là que la boucle se ferme — et c'est un autre métier, que je n'ai pas encore fait.

* **L'init du moteur** — les 124 secondes que le streamer ne touche pas, dont 33 de capture de graphes CUDA. C'est là qu'est le cold-start, et c'est le prochain chantier. Le chargement `s3://` direct (vLLM v0.9.0) attaque la partie poids ; le reste demande autre chose.
* **Les traces OTLP vers VictoriaTraces** — case volontairement décochée aujourd'hui, à cocher le jour où la compatibilité du chemin d'ingestion sera vérifiée, pas avant.
* **Les trois modèles restants**, à basculer sous routage possédé par la composition — et, dans le même mouvement, la suppression définitive de `route.yaml`. C'est ce commit-là, et pas celui de cette PR, qui tuera pour de bon la double comptabilité.

{{% notice note "Où en est la validation, exactement" %}}
Tout ce qui est chiffré dans cet article vient d'un **cluster EKS reconstruit de zéro** (Cilium, Flux, quatre modèles sur quatre GPU L4), pas d'un rendu local :

* le **split canary** — 8,2 % mesurés pour 10 % configurés, corroborés par `vllm:lora_requests_info` côté moteur ;
* l'**attribution `gen_ai`** — deux labels, `original` et `request`, donc un split de mesure gratuit ;
* le **latch de readiness** — route retenue 90 s pendant que le pod chargeait ses poids, publiée à `Available=True`, et les onze ressources composées ramassées à la suppression ;
* les **garde-fous d'admission** — les sept rejets CEL, message par message ;
* `status.servedModels` et sa colonne, sur les quatre modèles de la flotte ;
* le **cold-start** avec et sans le Run:ai streamer, sur le même modèle et les mêmes poids ;
* `model: MoM`, sur trois classes de prompts, quatre requêtes.

Et ce qui **n'est pas mesuré**, donc pas affirmé : la **distribution par préfixe** de l'Endpoint Picker. Elle demande plusieurs réplicas d'un même modèle sous charge partageant leurs préfixes, et le budget GPU de cette plateforme ne le permet pas encore. L'EPP tourne, il score, le chemin est propre — mais le gain qu'il promet, je ne l'ai pas vu de mes yeux, et je ne le mettrai pas en chiffres tant que ce sera le cas.
{{% /notice %}}

---

## :bookmark: Références

### Le code

- [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — la plateforme complète
- [PR #1559](https://github.com/Smana/cloud-native-ref/pull/1559) — la composition `InferenceService` v0.6.0 → **v0.8.0**
- [`docs/specs/`](https://github.com/Smana/cloud-native-ref/pull/1559/files) — les specs de cette PR : **002** (routage possédé + canary), **003** (`engineArgs` et `servedModels`), **004** (InferencePool + Endpoint Picker), **005** (cold-start / Model Streamer), **006** (observabilité `gen_ai`). Elles arrivent **avec la PR** — elles ne sont pas encore sur `main`.

### Composants techniques

- [Envoy AI Gateway](https://aigateway.envoyproxy.io/) — la porte d'entrée compatible OpenAI, et ses [métriques `gen_ai`](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/)
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) — `InferencePool` et Endpoint Picker
- [Run:ai Model Streamer](https://github.com/run-ai/runai-model-streamer) — chargement concurrent des poids vers la VRAM
- [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) — la classification de prompts derrière `model: MoM`
- [KEDA](https://keda.sh/) — autoscaling sur métriques vLLM
- [Crossplane](https://www.crossplane.io/) — le moteur de composition
- [KCL](https://www.kcl-lang.io/) — le langage dans lequel la composition est écrite

### Évolution convergente

- [Modelplane](https://github.com/modelplaneai/modelplane) — le plan de contrôle d'inférence des créateurs de Crossplane, à l'échelle de la flotte

### Articles précédents de la série

- [Agentic Coding : concepts et cas concrets](/fr/post/series/agentic_ai/ai-coding-agent/) — Partie 1
- [Quelques mois avec Claude Code : tips et workflows](/fr/post/series/agentic_ai/ai-coding-tips/) — Partie 2
- [Self-hosted LLM stack : poser les fondations](/fr/post/series/agentic_ai/llm-self-hosted-stack/) — Partie 3
