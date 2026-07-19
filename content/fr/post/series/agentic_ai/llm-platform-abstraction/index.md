+++
author = "Smaine Kahlouch"
title = "Le modèle, pas le pod"
date = "2026-07-12"
summary = "L'unité de déploiement n'est plus un pod qui sert un modèle : c'est une claim déclarative unique qui possède, réconcilie et garbage-collecte d'un seul objet le pod vLLM *et* sa route — fin de la seconde source de vérité. Sur cette base, une répartition pondérée d'adapters LoRA à coût GPU nul devient un simple champ."
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

Ce que deux mois d'usage ont montré, c'est que la **promesse** attachée à cette interface — produire *toutes* les ressources dont un modèle a besoin — s'arrêtait avant le routage. C'est là toute la thèse de cet article, et elle tient en un déplacement : **l'unité de déploiement n'est plus « un pod qui sert un modèle ».** C'est une **claim déclarative unique qui possède, réconcilie et garbage-collecte d'un seul objet le pod vLLM *et* son entrée de routage** — supprimant la seconde source de vérité (un modèle sans route, c'est un `404` silencieux ; une route orpheline, c'est un backend fantôme). Et une fois la route rejointe à la claim, la politique de trafic la moins chère de l'écosystème — une **répartition pondérée d'adapters LoRA à coût GPU nul** — n'est plus qu'un champ de cette unité.

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

* Comprendre pourquoi une abstraction de plateforme doit **posséder le routage** qu'elle génère — et ce que le **garbage collector** de Kubernetes garantit une fois que c'est le cas
* Voir une **répartition pondérée d'adapters LoRA** déplacer une fraction du trafic vers un fine-tune, **à coût GPU nul** : même pod, même GPU, aucun réplica en plus
* Voir comment une **configuration avancée** encadrée laisse passer les options que le schéma curé n'a pas prévues, sans trouer le contrat de service — et où ce garde-fou **fuit encore**
* Voir un **routage sémantique** décider du modèle à partir du prompt, et se composer avec la répartition au lieu de lui disputer la décision
* En tirer des **leçons transposables** à n'importe quelle abstraction de plateforme — LLM ou pas

{{% notice tip "Le code" %}}
Tout ce qui suit est dans [**cloud-native-ref**](https://github.com/Smana/cloud-native-ref) (PR #1559). La composition `InferenceService` passe de la **v0.6.0** (celle de la partie 3, du 9 mai 2026) à la **v0.8.0**.
{{% /notice %}}

## :world_map: Vue d'ensemble

Avant d'entrer dans le détail, une vue à vol d'oiseau. Tout ce que décrit cet article se lit sur trois plans, et un seul objet les traverse — la claim.

<!-- DIAGRAM-EN-ATTENTE: schéma global .drawio (construit après le texte) -->
{{< img src="architecture.png" alt="Vue d'ensemble de la plateforme" width="1200" >}}

**Le plan de requête.** Un client parle à l'**Envoy AI Gateway** (le point d'entrée unique de la plateforme, compatible API OpenAI) en écrivant un nom dans son champ `model`. La Gateway en dérive un en-tête de routage, une `AIGatewayRoute` choisit le backend, et la requête atterrit sur un pod **vLLM** qui sert le modèle de base et ses adapters LoRA — sur un seul GPU.

**Le plan de contrôle.** Rien de tout cela n'est écrit à la main. Une claim `InferenceService` déclare le modèle voulu ; la composition Crossplane (écrite en KCL) en tire une dizaine de ressources — le `Deployment` vLLM, le `Service`, les objets de routage, le `ScaledObject` KEDA, les `CiliumNetworkPolicy`, les règles d'alerte. Crossplane les **possède**, les **réconcilie** en continu, et les **garbage-collecte** ensemble le jour où la claim disparaît.

**Le plan d'observabilité.** Les `/metrics` de vLLM et les métriques `gen_ai` émises par la Gateway remontent dans la même VictoriaMetrics, se lisent dans le même Grafana. Aucun outil de plus.

Le cadre de tout l'article tient dans une phrase : **une claim = un objet possédé, réconcilié et ramassé d'un bloc.** Le reste n'est que ce qu'on met dans cet objet.

## :door: La claim possède sa route

Commençons par la lacune la plus structurante, parce qu'elle touche à la **complétude** de l'abstraction : sa promesse est de produire *toutes* les ressources d'un modèle, et le routage y échappait. Un second fichier, `apps/base/ai/llm/ai-gateway-routes/route.yaml`, déclarait *comment* on atteint le modèle — quel `model:` atterrit sur quel backend — et il était **écrit à la main**. Deux fichiers à garder synchronisés, donc deux dérives possibles, et aucune ne fait de bruit :

* **Un modèle sans route** — le pod tourne, le GPU est facturé, et la Gateway répond `404` à qui le demande.
* **Une route qui survit à son modèle** — la claim supprimée, l'entrée de routage reste : un *backend fantôme* vers un `Service` qui n'existe plus.

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
| **`AIGatewayRoute`** | la règle de dispatch — quel `model:` atterrit sur quel backend — rattachée à la Gateway |

Ces trois objets sont désormais des **ressources composées** : Crossplane les possède comme il possède déjà le `Deployment`. Elles portent l'`ownerReference` de la **ressource composite** — l'objet interne que Crossplane crée pour honorer la claim — donc **le garbage collector de Kubernetes les emporte avec elle**. Supprimer un modèle supprime sa route, non parce qu'on y a pensé, mais parce qu'il n'existe aucun chemin où la route survive à son propriétaire. Le *backend fantôme* devient structurellement impossible, et la dérive inverse aussi : plus deux fichiers à synchroniser, un seul.

### Le latch de readiness

Générer la route depuis la claim déplace une question qui, avant, se réglait à la main : **quand** la route doit-elle exister ? Deux échecs symétriques guettent, et ils font mal de façons opposées.

* **Trop tôt.** La route est publiée, mais le premier réplica vLLM charge encore ses poids : la Gateway route vers un backend sans endpoint prêt, et le client prend un `404` ou un `503` sur un modèle que la plateforme prétend servir.
* **Trop nerveux.** Retirer la route dès que le `Deployment` n'est plus disponible transforme la moindre indisponibilité passagère (rolling update, pod évincé, nœud GPU recyclé par Karpenter) en *flapping* : la route sort de la config Envoy, y revient trente secondes plus tard.

Le compromis est asymétrique, parce que les deux erreurs ne coûtent pas le même prix. À la création, la route est **retenue** tant que le `Deployment` n'a pas atteint la condition `Available=True`. Une fois publiée, en revanche, elle **ne sera plus jamais retirée** : c'est le **latch de readiness** — un portail à l'entrée, puis un verrou qui reste enclenché. Une indisponibilité transitoire reste un problème de *santé du backend*, pas de *topologie de routage* ; c'est aux retries et au health checking d'Envoy de l'absorber. Techniquement, le latch tient sur l'état des ressources déjà composées, tel que Crossplane l'observe au début de chaque réconciliation : la composition y cherche sa propre route ; si elle l'y trouve, c'est que le portail s'est déjà ouvert.

Sur le cluster, le portail se voit à l'œil nu. Une claim fraîche, `gateway.enabled: true`, et on regarde apparaître les ressources :

```
[10:23:46] Available=False  route=0  pod=0/1 ContainerCreating
[10:25:54] Available=False  route=0  pod=0/1 Running       <- Running, mais pas prêt : toujours pas de route
[10:27:19] Available=False  route=0  pod=0/1 Running
[10:27:41] Available=True   route=1  pod=1/1 Running       <- la route apparaît
```

Le pod est resté **`Running` mais `0/1` pendant environ 90 secondes** — il chargeait ses poids sur le GPU — et la route est restée retenue tout ce temps. Le point qui compte : **le latch s'accroche à la condition `Available` du `Deployment`, pas à la phase du pod.** Un pod qui tourne n'est pas un pod qui sert, et seule la première des deux le sait.

Les deux backends — le `Backend` et l'`AIServiceBackend` — sont eux créés **immédiatement** : un FQDN et un port ne « démarrent » pas. **Seule la route est retenue** ; le portail ne bloque pas la construction du chemin, il bloque sa publication. Et à la suppression, la possession se vérifie d'un coup : les **onze** ressources composées disparaissent, route et backends compris. Aucune n'a survécu à sa claim.

{{% notice tip "Le détail qui aurait fini en incident" %}}
La composition **écrit** la route sous une clé de ressource, et la **relit** sous cette même clé au tour suivant. Deux chaînes de caractères, à deux endroits du code : si elles divergent — un renommage d'un côté seulement — la relecture ne trouve jamais rien, le latch ne s'enclenche jamais, et la route est retirée à la première indisponibilité passagère venue. Un bug invisible en test, qui ne se manifeste qu'en production, au pire moment.

D'où une contrainte volontairement bête : les clés de lecture et d'écriture dérivent toutes deux d'une **constante unique** (`_gatewayRouteSuffix`). Elles ne peuvent pas diverger — il n'y a plus qu'une seule chaîne à renommer.
{{% /notice %}}

### Une migration, pas un big bang

Le basculement se fait modèle par modèle. `xplane-qwen-coder` passe en `gateway.enabled: true` et, **dans le même commit**, ses entrées quittent `route.yaml` — un instant où les deux sources déclarent la même route serait exactement le désordre qu'on cherche à supprimer. Soyons précis sur l'état réel : **un modèle sur quatre** est migré. `route.yaml` porte encore les trois autres, et il faut toujours y penser pour eux ; ce qui a changé, ce n'est pas qu'il a disparu, c'est qu'il a maintenant une **date de péremption**.

Reste que la route appartient désormais à la claim. Et une route que l'on possède est une route que l'on peut **pondérer**.

## :dna: LoRA en bref

Tout ce qui suit repose sur les **adapters LoRA**, alors posons-les — surtout **à quoi ça sert**.

Un adapter **LoRA** (_Low-Rank Adaptation_) n'est pas un modèle : c'est un **delta de poids de faible rang** ajouté à ceux du modèle de base. Plutôt que de réentraîner les grandes matrices, on apprend une paire de petites matrices dont le produit s'ajoute aux premières. Le « faible rang » — ici fixé à **64** — se lit directement sur la balance : un adapter pèse **quelques dizaines de Mo** (un ordre de grandeur, pas une mesure faite ici), contre les ~15 Go de poids du modèle de base non quantifié.

Ce que LoRA évite, c'est un **second modèle complet**. Spécialiser un modèle — le rendre meilleur sur le SQL, sur le code sécurisé — c'est le **fine-tuner** : un fine-tuning classique réécrit les poids, et ce qu'on obtient est donc un second jeu de ~15 Go, donc un second pod, un second GPU, une seconde ligne de facture — et l'addition se répète à chaque spécialisation. LoRA casse cette proportionnalité : le fine-tune **s'ajoute** aux poids de base au lieu de les remplacer, et comme c'est un delta, rien n'empêche d'en poser plusieurs sur la même base.

C'est vLLM qui encaisse cette propriété, et c'est là que tout se joue : **il charge le modèle de base une fois, puis applique les adapters par-dessus, dans le même processus, sur le même GPU** (`--enable-lora`, `--lora-modules`). Servir trois « modèles » — la base et deux fine-tunes — ne coûte donc pas trois GPU : c'est **un pod, un GPU, et trois noms auxquels il répond**. C'est ce qui rend possible le « zéro GPU » de la section suivante.

Le contre-exemple compte tout autant : **LoRA n'apprend pas à un modèle une capacité qui lui manque fondamentalement.** Quelques dizaines de Mo de delta infléchissent un comportement, elles ne créent pas une compétence absente. Le cas d'usage a une forme reconnaissable — *le modèle de base est presque bon, on veut l'infléchir, pas le remplacer* :

* **Un vocabulaire de domaine** — les termes, les schémas et les usages d'un métier qu'un généraliste connaît mal.
* **Un style de code maison** — les patterns et bibliothèques qu'une équipe utilise réellement.

Deux adapters sont déclarés sur `xplane-qwen-coder`, posés sur exactement le même modèle de base (`Qwen/Qwen2.5-Coder-7B-Instruct`) :

| `loraAdapters[].name` | Source HuggingFace | Spécialisation |
|---|---|---|
| `xplane-qwen-coder-sql-dpo` | `jk200201/qwen2.5-coder-7b-sql-dpo` | génération SQL |
| `xplane-qwen-coder-securecode` | `scthornton/qwen2.5-coder-7b-securecode` | code sécurisé |

Chaque entrée nomme l'adapter — c'est ce nom qu'un client mettra dans son champ `model` — et pointe un dépôt HuggingFace **piné au commit SHA**, comme le modèle de base l'est déjà via `model.revision`. Un fine-tune qui bouge sous les pieds du serving est une régression que personne ne voit venir.

{{% notice note "Ces adapters, je ne les ai pas entraînés" %}}
Autant l'assumer : les deux adapters ci-dessus viennent de HuggingFace, entraînés par d'autres. Je les ai retenus parce qu'ils se posent sur le modèle de base que la plateforme sert déjà — pas parce que j'ai évalué leur qualité.

**Fine-tuner soi-même est un autre métier** : constitution du dataset, boucle d'entraînement, évaluation. Ce qui suit traite d'une question d'infrastructure — *comment expose-t-on progressivement un fine-tune au trafic réel ?* — pas d'une question de data science. J'y reviens dans « Et après ? ».
{{% /notice %}}

## :balance_scale: Répartition pondérée : 10 % du trafic, zéro GPU

Le scénario : `xplane-qwen-coder-sql-dpo` prétend être meilleur que le modèle de base sur la génération SQL. Sur un benchmark, peut-être. Sur **le** trafic — celui des vrais utilisateurs, avec leurs prompts à eux — la seule façon de le savoir est de lui en envoyer un peu. Dix pour cent, quelques jours. Voilà ce que ça coûte dans la claim :

```diff
# apps/base/ai/llm/qwen-coder.yaml
 spec:
   gateway:
     enabled: true
+    canaries:
+      - adapter: xplane-qwen-coder-sql-dpo
+        weightPercent: 10
```

Deux lignes utiles. Flux réconcilie, et une requête sur dix adressée à `xplane-qwen-coder` part sur le fine-tune plutôt que sur les poids de base.

### Sous le capot : un split qui n'envoie rien ailleurs

La règle de l'`AIGatewayRoute` ne porte plus un seul `backendRef` mais **un par entrée** : le backend de base à `100 - sum(weightPercent)`, puis un backend par canary à son `weightPercent`. Chaque canary reçoit son propre `AIServiceBackend` porteur d'un champ décisif — **`modelNameOverride`**, qui vaut le nom de l'adapter *verbatim*.

| `backendRef` | Poids | `modelNameOverride` | `Backend` visé |
|---|---|---|---|
| `AIServiceBackend` de base | **90** (`100 - 10`) | *(aucun)* | `xplane-qwen-coder` |
| `AIServiceBackend` de canary | **10** | `xplane-qwen-coder-sql-dpo` | `xplane-qwen-coder` |

Relisez la dernière colonne. **Les deux `backendRef` pointent le même `Backend`** — le même `Service`, les mêmes pods, le même GPU. Rien n'est envoyé ailleurs, parce qu'il n'y a pas d'ailleurs. Le split ne fait que **réétiqueter** 10 % des requêtes : `modelNameOverride` réécrit le nom du modèle avant qu'elles n'atteignent vLLM, et vLLM — qui a déjà l'adapter en mémoire, chargé au démarrage — les sert avec le fine-tune.

{{< img src="canary-split.png" alt="Split canary : un pod, un GPU, le modèle de base et deux adapters LoRA" width="1200" >}}

C'est là que se joue le « zéro GPU » du titre. Un canary applicatif classique déploie une v2 à côté de la v1 et paie deux jeux de réplicas pendant toute l'expérience ; transposé ici, ce serait un **second pod vLLM**, donc un second GPU, donc un nœud de plus provisionné par Karpenter — pour évaluer un delta de quelques dizaines de Mo. La répartition LoRA n'ajoute ni réplica, ni nœud, ni ligne de facture : c'est une décision de *routage*, prise en amont, sur une flotte qui n'a pas bougé.

Le split n'est d'ailleurs pas le seul chemin vers un adapter : chaque `loraAdapters[]` reçoit **sa propre règle** de routage (nommer explicitement l'adapter → 100 % de cet adapter). Une requête qui demande `xplane-qwen-coder-sql-dpo` est donc **pinnée** sur lui, indépendamment de la répartition en cours — c'est le mode d'usage explicite de la partie 3, et ce qui permet à une éval de comparer deux modèles sans dépendre d'un tirage au sort.

### Une répartition figée, pas un canary progressif

Un mot d'honnêteté, parce qu'un lecteur averti le verra sinon. Le champ s'appelle `canaries`, mais ce n'est **pas un canary** au sens de Flagger ou d'Argo Rollouts. Pas de *bake window*, pas d'analyse automatique des métriques, pas de progression 10 % → 25 % → 50 %, pas de rollback déclenché par un seuil d'erreurs. C'est une **répartition pondérée figée par commit** : le poids est ce que le YAML dit, jusqu'au prochain `git push`. La promotion, la progression, l'abandon automatique restent manuels.

Le chaînon manquant a un nom, et les briques pour l'alimenter sont déjà là : les `VMRule` et les métriques `gen_ai` de la suite donnent exactement le signal qu'un contrôleur de rollout progressif — Flagger, Argo Rollouts — consommerait pour décider seul. Le brancher est un chantier ouvert, pas une case cochée.

### Le split, mesuré

Soixante requêtes vers `model: xplane-qwen-coder`, canary à 10 % armé. Puis douze requêtes nommant explicitement l'adapter. Ce que la gateway a compté :

| `gen_ai_original_model` (demandé) | `gen_ai_request_model` (servi) | n |
|---|---|---|
| `xplane-qwen-coder` | `xplane-qwen-coder` | **56** |
| `xplane-qwen-coder` | `xplane-qwen-coder-sql-dpo` | **5** |
| `xplane-qwen-coder-sql-dpo` | `xplane-qwen-coder-sql-dpo` | **12** |

**8,2 % de trafic capté pour un poids configuré à 10 %.** Sur n=61 et p=0,1, on attend 6,1 ± 2,3 : cinq est exactement là où il doit être. Et les douze requêtes explicites sont servies par l'adapter, **toutes les douze, sans une seule fuite vers la base** — le pin ne cède pas au tirage au sort. La deuxième ligne est celle qui compte : le client a demandé le modèle de base, et c'est le fine-tune qui a répondu, sans qu'il le sache et sans un seul GPU de plus.

La gateway n'est pas un témoin fiable d'elle-même : elle affirme avoir réétiqueté des requêtes, elle ne prouve pas que vLLM a joué le jeu. Le moteur tranche depuis ses propres métriques :

```
$ vllm:lora_requests_info
running_lora_adapters='(none)'                      max_lora=2
running_lora_adapters='xplane-qwen-coder-sql-dpo'   max_lora=2
```

**Deux plans indépendants, une seule décision de routage.** La gateway dit avoir envoyé 5 requêtes sur l'adapter ; vLLM confirme l'avoir chargé et exécuté. Le « zéro GPU » n'est plus une propriété de construction : c'est un pod, un GPU, et le moteur qui le dit.

### La mesure tombe toute seule dans les métriques

Le montage a une propriété qu'aucune ligne d'instrumentation n'a produite : la gateway émet **deux labels distincts**, `gen_ai_original_model` (ce que le **client** a demandé) et `gen_ai_request_model` (ce qui a été **servi** — donc le `modelNameOverride` du canary). **Le split de routage est donc gratuitement un split de mesure** : c'est ce même couple de labels qui produit le tableau à trois lignes ci-dessus, sorti d'une seule requête PromQL. Un montage qui tombe dans les métriques standard sans qu'on l'y aide n'est pas un hack — c'est le signe qu'il suit le grain du système.

Le dashboard `llm-gateway` découpe donc les mêmes séries — latence end-to-end, TTFT, tokens consommés — canary contre base, dans le Grafana qui collecte déjà les métriques vLLM. Aucun nouvel outil.

<!-- SCREENSHOT-EN-ATTENTE: dashboard Grafana llm-gateway, attribution canary vs base -->

Ces courbes disent si le canary est plus lent, ou plus bavard, que la base. Elles ne disent pas s'il est **meilleur** : aucune métrique de gateway ne sait ce qu'est un bon SQL. La qualité reste le domaine de l'éval — [Promptfoo](/fr/post/series/agentic_ai/llm-self-hosted-stack/), vu en partie 3 — et c'est là que le pin reprend tout son sens : une éval doit interroger l'adapter **de façon déterministe**, pas tomber dessus une fois sur dix.

### Ce que l'admission refuse

Un pourcentage de trafic est exactement le genre de champ où une faute de frappe se paie en production. Les garde-fous sont donc **à l'admission**, écrits en **CEL** (_Common Expression Language_ : le langage d'expressions que l'API server évalue lui-même au moment de l'`apply`) dans l'**XRD** (_Composite Resource Definition_ : le schéma de l'API que la composition expose à ses utilisateurs). `kubectl apply` échoue, et la composition ne se déclenche même pas :

* Des `canaries` **sans `gateway.enabled: true`** — une pondération sans route à pondérer.
* Un `canaries[].adapter` **absent de `loraAdapters[].name`** — on ne route pas vers un adapter que le pod ne charge pas.
* **Deux entrées visant le même adapter** — deux poids pour une seule cible.
* Un `weightPercent` **hors de 1–99**, ou une **somme des poids supérieure à 99**.

Le plafond à **99** n'est pas cosmétique : à 100 %, le modèle de base ne reçoit plus rien, et un canary qui absorbe tout le trafic est un **remplacement déguisé en expérience** — déclaré dans un champ dont le nom promet le contraire. Un remplacement se lit dans le champ qui le dit (`model.repository`), il ne se déduit pas d'un poids qui a discrètement atteint son maximum. Le plancher garantit donc toujours une référence à laquelle comparer, et un chemin de retour immédiat : ramener le poids à 1, ou retirer l'entrée.

{{% notice tip "Pourquoi un array dès le premier jour" %}}
`gateway.canaries` est un **array** (quatre entrées au maximum), alors que le besoin du jour tenait dans une seule. Un simple objet aurait suffi — et aurait été un piège : passer d'un objet à un array plus tard est un **breaking change** d'API. Nouvelle version du schéma, conversion, migration de toutes les claims existantes… pour un champ qui n'aura rien gagné entre-temps.

Ce n'est pas de la prescience, c'est de l'emprunt : la lecture de **Modelplane** — sur laquelle je reviens plus bas — a rendu évident que comparer *plusieurs* adapters en même temps est un cas d'usage normal. Le coût de l'array au jour 1, c'est quelques lignes de KCL ; le coût de la conversion au jour 100, ça s'appelle une migration.
{{% /notice %}}

{{% notice warning "Une règle CEL sur un champ non borné est une règle non bornée" %}}
Ces règles ont failli ne jamais tourner — non parce qu'elles étaient fausses, mais parce qu'elles étaient **trop chères**. L'API server **estime statiquement** le coût de chaque règle CEL avant d'accepter le schéma, sur le **pire cas que le schéma autorise**. La règle qui croise les deux tableaux —

```
canaries.all(c, loraAdapters.exists(a, a.name == c.adapter))
```

— est une boucle dans une boucle, sur des comparaisons de chaînes. Tant que `loraAdapters` n'a **ni `maxItems`, ni `maxLength` sur ses noms**, l'estimateur suppose des tableaux infinis de chaînes infinies : le coût part à l'astronomique, et l'API server **rejette le CRD entier**. Aucun modèle n'est plus déclarable.

L'asymétrie est frappante : dans le **même fichier**, `engineArgs` était borné, et ses **dix-neuf** règles passaient sans problème. **Dix-neuf règles bornées coûtent moins cher qu'une seule non bornée.** La correction n'est donc pas de simplifier la règle, c'est de **borner ce sur quoi elle porte** (`maxItems: 8`, `maxLength: 63`). Une borne n'est pas de la cosmétique de schéma : c'est ce qui rend une règle **calculable**.
{{% /notice %}}

## :gear: `engineArgs` : ouvrir sans trouer

Une abstraction curée ne peut pas tout prévoir. Le schéma de la claim n'expose que des champs modélisés un par un, et vLLM ajoute des options à chaque release, plus vite qu'une API de plateforme ne peut les exposer. Chaque flag absent du schéma se payait au prix fort : une PR sur la composition, les tests, la republication de l'image OCI, le bump du tag, puis Flux. **Une release de plateforme pour un essai d'une après-midi.**

Une API qui ne laisse aucune sortie ne bloque personne : elle se fait **contourner**. Un `kubectl edit deployment`, le flag ajouté à la main, et jusqu'à la prochaine réconciliation de Crossplane ça marche très bien. Le coût réel n'est pas le contournement, c'est ce qu'il fait à l'abstraction : dès l'instant où l'état réel du cluster ne se lit plus dans la claim, **la claim ment**. Et une abstraction qui ment est pire qu'une abstraction absente, parce qu'elle continue de prétendre décrire le système.

`spec.engineArgs` est la troisième voie : un array de chaînes, **16 entrées au maximum**, transmises **verbatim** au serveur vLLM. Il laisse passer les **options non curées** — celles que le schéma n'a pas anticipées — sauf les **dix-huit** dont l'autoscaling et le routage dépendent.

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

Ce « milieu » — des champs curés d'un côté, une trappe verbatim de l'autre, une denylist qui valide à l'admission ce qui passe par la trappe — est la vraie contribution de cette version. Aucun des huit projets que j'ai comparés (KServe, llm-d, production-stack, KubeAI, Modelplane…) ne l'occupe : tous exposent des arguments moteur **bruts, non validés**. L'API fermée d'un côté, le fourre-tout non contraint de l'autre — le milieu gouverné manquait.

### Dix-huit flags qui ne passeront pas

Le tri est simple à énoncer : **tout passe, sauf ce dont l'autoscaling et le routage dépendent.** Ces flags sont **réservés**, et l'XRD les refuse à l'admission. **Dix-neuf** règles CEL portent la vérification : une règle de forme (toute entrée commence par `--`), et **dix-huit règles de flags réservés, une par flag** — donc un message par flag, pas un verdict générique qui laisse l'utilisateur sans issue. Ce n'est pas une intention, c'est ce que l'API server répond, verbatim, sur un cluster :

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

Chaque message **nomme le champ curé à utiliser à la place** : toute la différence entre une API qui dit non et une API qui dit où aller. Prenez `--max-num-seqs`, le **dénominateur** du trigger KEDA vu en partie 3 : laissez un utilisateur le redéfinir, et l'autoscaler continue de calculer son taux de saturation contre une taille de batch qui n'existe plus. Rien ne casse, rien n'alerte — le scaling est simplement **faux**. Même logique pour `--port` : la composition ne l'émet jamais, mais elle le réserve, parce que le `8000` par défaut n'est pas un détail d'implémentation, c'est le **contrat de serving** (le `targetPort` du `Service`, les probes, le FQDN du `Backend` en dépendent tous).

Dix-huit **dans cette version**, et le nombre bouge : chaque nouveau champ curé réserve le flag qu'il pilote — les deux du Run:ai Model Streamer l'ont déjà rejointe. C'est ce qui rend la question suivante décisive : une liste qui bouge ne doit vivre qu'à **un seul endroit**.

### Une seule source d'application — et sa fuite

La denylist vit à **un seul endroit**, le CEL de l'XRD. La composition, elle, ne refiltre rien : refiltrer côté KCL, par « défense en profondeur », recréerait exactement le problème de `route.yaml` — deux listes à garder synchronisées, deux dérives possibles, et aucune qui fait de bruit. Un test verrouille donc les deux ensembles l'un à l'autre (`test_engine_args_denylist_lockstep`) : le jour où quelqu'un ajoute un argument géré au KCL sans l'ajouter à la denylist, le CI le voit.

Reste à être franc sur une limite que je n'ai vue qu'après coup, et elle est gênante : **la denylist fuit.** Elle matche des chaînes qui commencent par des **tirets** — `--max-num-seqs`. Mais vLLM s'appuie sur un `FlexibleArgumentParser` qui **normalise les underscores en tirets** : `--max_num_seqs=512` et `--max-num-seqs=512` sont, pour le moteur, le même flag. Or `--max_num_seqs` (underscore) ne ressemble à aucune des dix-huit règles : il **passe l'admission**. Et comme les `engineArgs` de l'utilisateur sont appendés **en dernier**, argparse — qui donne le dernier mot au dernier argument — le fait **gagner** sur la valeur gérée par la composition. Le dénominateur KEDA est redéfini, en silence, par un alias que la denylist ne connaît pas.

La leçon n'est donc pas « valider à l'admission plutôt qu'au rendu » — ça, c'était déjà fait, et c'est banal. Elle est plus dérangeante : **un point d'application unique n'est sûr que s'il est étanche.** Le nôtre matchait des chaînes de caractères contre un parseur qui accepte des alias — et un filtre qui ne connaît pas tous les noms de ce qu'il filtre laisse forcément passer quelque chose. Le correctif est court (borner sur la forme normalisée, ou inverser l'ordre d'append pour que le géré gagne), mais l'aveu compte plus que le patch : c'est lui qui dit à quel point « un seul endroit » est une promesse exigeante.

`engineArgs` ne rend donc pas la plateforme plus permissive : il rend sa permissivité **lisible**. Ce qui n'est pas listé passe, sans demander la permission à personne ; ce qui est listé est refusé à l'`apply`, avec le nom du champ à utiliser à la place — modulo la fuite ci-dessus, qui reste à colmater. **Liberté là où c'est sûr, garde-fous là où ça ne l'est pas.**

## :eyes: Le modèle dit ce qu'il sert

`kubectl get isvc` répondait `READY`, et rien de plus. Une information de santé — un pod tourne — mais pas **à quels noms** il répond. La question était triviale tant qu'un pod servait un modèle sous un nom ; elle ne l'est plus, maintenant que le même pod répond à un modèle de base et à deux adapters, et qu'une requête sur dix adressée au premier part discrètement vers l'un des seconds. Un champ de statut, `status.servedModels`, porte la tranche — et une colonne `SERVED MODELS` la remonte jusque dans un `kubectl get`, sans `-o yaml` ni `jq` :

```
$ kubectl get inferenceservice -n llm
NAME                    SERVED MODELS                                                              SYNCED   READY
xplane-llamaguard3-1b   xplane-llamaguard3-1b                                                      True     True
xplane-qwen-coder       xplane-qwen-coder,xplane-qwen-coder-sql-dpo,xplane-qwen-coder-securecode   True     True
xplane-qwen-coder-fim   xplane-qwen-coder-fim                                                      True     True
xplane-qwen3-8b         xplane-qwen3-8b                                                            True     True
```

**Un pod, un GPU, trois noms**, lisibles à l'endroit où on pose la question. La leçon dépasse ce champ : **un statut que l'outil de tous les jours ne sait pas afficher n'est pas de la découvrabilité** — c'est de la donnée correcte, rangée là où personne ne regarde.

## :brain: Un cran au-dessus : le prompt choisit le modèle

Une mise au point avant tout : cette capacité **ne vit pas dans la claim**. Elle vit dans la Gateway, sous la forme d'un `EnvoyPatchPolicy` posé à côté des routes — elle **prolonge** la plateforme, elle n'est pas un attribut de l'abstraction `InferenceService`. Je la range ici parce qu'elle ferme la boucle de tout ce qui précède.

Jusqu'ici, le client **sait ce qu'il veut** : il écrit `xplane-qwen-coder` dans son champ `model`, donc il connaît le catalogue, les noms et lequel est bon en code. Hypothèse raisonnable pour un service, mauvaise pour un humain devant un chat. L'idée du **Mixture of Models** (`model: MoM`) retourne la question : le client n'annonce pas un modèle, il **envoie son prompt**, et la plateforme décide.

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

Ce routeur tournait déjà en partie 3, en bonne santé, scruté, avec ses métriques — et **jamais appelé** : les règles de l'`AIGatewayRoute` matchent des noms de modèles concrets, `MoM` n'en est pas un, la Gateway répondait `404`.

### Où l'insérer : avant celui qui décide

Le brancher tient en une question plus subtile qu'elle n'en a l'air : **à quel moment du chemin de requête ?** L'Envoy AI Gateway a son propre **extproc** (_external processor_ : un service que le proxy appelle en cours de traitement, capable de modifier les en-têtes **comme le corps**), et c'est lui qui **dérive l'en-tête `x-ai-eg-model` depuis le `model` du corps** — l'en-tête sur lequel *toutes* les règles de routage matchent. Le routeur sémantique doit donc s'exécuter **avant** lui : il réécrit `body.model`, de `MoM` vers un nom concret, et l'extproc dérive ensuite l'en-tête depuis un corps **déjà réécrit**.

La conséquence est la propriété qui fait tout tenir : **aucune règle de routage nouvelle.** Les routes existantes — celles de la composition, du split, du pin explicite — matchent telles quelles, sans savoir que MoM existe. L'objet prévu pour ça, l'`EnvoyExtensionPolicy`, ne sait pas exprimer « avant » : il **ajoute** ses filtres *après* ceux de l'AI Gateway, trop tard, l'en-tête est déjà dérivé. Le montage passe donc par un **`EnvoyPatchPolicy`** — un patch xDS brut qui insère, à l'**index 0** de la chaîne de filtres HTTP, l'extproc du routeur sémantique. Un patch xDS est couplé à la forme interne de la configuration Envoy, pas à une API stable : c'est le prix de l'ordre, et c'est le montage que le projet amont documente lui-même.

### Deux couches qui se composent

Le montage a produit un résultat que rien n'avait explicitement câblé. Une requête `MoM`, classée `code`, réécrite en `xplane-qwen-coder`… **est tombée dans la répartition LoRA** et a été servie par `xplane-qwen-coder-sql-dpo`. Deux décisions, prises par deux composants qui ne se connaissent pas : le **routeur sémantique** décide *quel modèle* (à partir du sens du prompt) ; la **route pondérée** décide *quels poids* (à partir d'un tirage à 10 %). Elles s'empilent au lieu de se disputer le même point de décision — une propriété de la **position dans la chaîne**, pas une intention : l'un agit sur le corps, l'autre sur l'en-tête qui en est dérivé. Chacune ignore l'autre, et c'est exactement pour ça qu'elles tiennent ensemble.

{{% notice note "Ce que ça ne fait pas" %}}
La classification est un modèle, pas une vérité. Un prompt ambigu tombe où il tombe, et rien ici ne mesure la **qualité** de la classification — je n'ai vérifié que le **câblage** : que le routeur soit appelé, qu'il réécrive, et que la route matche derrière. Savoir si `math` est le bon verdict pour une intégrale, c'est le domaine du routeur ; savoir si le chemin de requête l'appelle, c'est celui de la plateforme. Cet article ne traite que du second.
{{% /notice %}}

Une dernière honnêteté, parce qu'elle borne la thèse : la liste des modèles que MoM connaît est **statique, éditée à la main** dans le HelmRelease du routeur. Une cinquième claim reste invisible de MoM jusqu'à cette édition manuelle — la politique de trafic la plus haute n'est donc **pas encore possédée par la claim**. Et le filtre vit à l'**index 0, sur 100 % du trafic** : un SPOF assumé, à réplica unique pour l'instant.

## :telescope: Modelplane : deux étages, pas deux camps

Le 23 juin 2026 — pendant que cette PR se construisait — les créateurs de Crossplane ont publié [**Modelplane**](https://github.com/modelplaneai/modelplane) : un plan de contrôle d'inférence open-source à l'échelle de la **flotte**. Quand les gens qui ont bâti Crossplane arrivent à la même forme d'API — une séparation nette entre l'équipe plateforme qui déclare la capacité et l'équipe ML qui déclare le modèle — ce n'est pas une coïncidence : c'est que le problème, lui, a une forme. Personne n'a copié personne (leur billet est postérieur à la v0.6.0 de cette plateforme, du 9 mai 2026) : c'est une **évolution convergente**.

Et surtout, il n'y a **rien à départager**, parce que les deux couches ne se disputent pas le même terrain :

| | **Au-dessus du cluster** — Modelplane | **Dans le cluster** — `InferenceService` |
|---|---|---|
| **La question traitée** | *où* poser ce modèle : quel cluster, quel fournisseur, quel GPU ? | *comment* ce cluster le sert : ses adapters, sa route, son trafic |
| **L'objet géré** | la flotte : multi-cluster, multi-cloud, GPU hétérogènes | un cluster : la claim et les ressources qu'elle produit |

Leurs propres docs le disent sans détour : Modelplane **compose** les projets de niveau cluster plutôt que de les remplacer. Une stack de la forme décrite ici est très exactement ce qui vivrait **en dessous** de la leur. Le problème qu'ils résolvent — placer un modèle sur le bon GPU, dans le bon cluster, chez le bon fournisseur — je ne l'ai tout simplement pas : cette plateforme est une référence **mono-cluster**. Reste un caveat qui joue contre moi : **ils sont engine-agnostic, je ne le suis pas** — la composition parle vLLM, produit des flags vLLM, scale sur des métriques vLLM. C'est le prix de la profondeur : on ne peut pas savoir que `--max-num-seqs` est le dénominateur d'un trigger KEDA **et** rester agnostique au moteur qui l'expose.

## :thought_balloon: Dernières remarques

Cette PR ne rend pas la plateforme plus **puissante** — un pod vLLM sur un GPU faisait déjà tourner un modèle en partie 3. Elle rend l'abstraction **complète** et **évolutive** : la claim produit désormais *toutes* les ressources du modèle, routage compris, et sait le faire bouger par paliers. Le chemin pour y arriver a produit des leçons qui n'ont, au fond, rien à voir avec les LLM.

* **Si deux fichiers décrivent la même chose, ils divergeront.** Pas un problème de discipline, un problème de temps : la dérive est fonction du nombre de jours, pas du sérieux de l'équipe. La seule issue est structurelle — la composition doit **posséder les deux**, ou il n'y a plus qu'un fichier. Et posséder une ressource, c'est aussi décider **quand** elle existe : portail à la création, latch ensuite.
* **Une répartition de trafic ne coûte un GPU de plus que si l'abstraction confond le modèle et le pod.** La politique appartient au **modèle**, pas au réplica : déclarée au bon niveau, déplacer 10 % du trafic vers un fine-tune devient une décision de routage, pas une seconde flotte à provisionner et à payer.
* **Un point d'application unique n'est sûr que s'il est étanche.** Une denylist à un seul endroit est le bon design ; mais la mienne matchait des chaînes contre un parseur qui accepte des alias, et un alias underscore est passé. « Un seul endroit » ne vaut que si ce qu'on y écrit couvre tous les noms de ce qu'on filtre.
* **Si `kubectl get` ne dit pas ce que la ressource fait, l'abstraction ment.** Un statut correct rangé là où personne ne le lit n'est pas de la découvrabilité.
* **Accélérer de 11,7 % une phase qui pèse 30 % rapporte 4 %.** Le Run:ai Model Streamer tient exactement sa promesse sur le chargement des poids — mais celui-ci ne pèse qu'un tiers d'un cold-start de ~180 s. Le vrai levier est ailleurs : l'init du moteur, que je n'ai pas encore attaqué.

Ce que je retiens surtout, c'est le déplacement qui les relie : **l'unité de déploiement n'est plus « un pod qui sert un modèle », c'est une claim qui possède, réconcilie et garbage-collecte sa tranche verticale d'un seul objet** — le pod, sa route, son scaling, ses policies, son observabilité. Supprimer la seconde source de vérité était le vrai gain ; la répartition LoRA à coût GPU nul n'en est que le premier bénéfice, parce qu'une route qu'on possède est une route qu'on peut pondérer.

Avec deux limites que cet article a lui-même posées, et que je garde visibles. `gateway.endpointPicker` et `gateway.canaries` **s'excluent à l'admission** : dans une même claim, on déclare la politique de trafic **ou** la stratégie de placement, pas encore les deux. Et la politique de trafic la plus haute — le routage sémantique — **n'est pas encore possédée par la claim**. L'unité de déploiement est la bonne ; les prochains axes sont déjà nommés.

## :construction: Et après ?

Des chantiers ouverts, pas des promesses.

* **Fine-tuner mon propre adapter.** Les deux LoRA de cet article viennent de HuggingFace, entraînés par d'autres : la répartition sait exposer progressivement un fine-tune au trafic réel, mais elle n'a encore jamais servi à valider *le mien*. C'est un autre métier — dataset, boucle d'entraînement, éval — que je n'ai pas encore fait.
* **Brancher un rollout progressif.** La répartition est figée par commit ; les `VMRule` et les métriques `gen_ai` en place sont exactement ce qu'un contrôleur type **Flagger** ou **Argo Rollouts** consommerait pour promouvoir ou abandonner un canary tout seul. C'est le chaînon qui transformerait la répartition pondérée en vrai canary.
* **L'init du moteur** — les ~124 s que le streamer ne touche pas, dont 33 de capture de graphes CUDA. C'est là qu'est l'essentiel du cold-start. Le chargement `s3://` direct (vLLM v0.9.0) attaque la partie poids ; le reste demande autre chose.
* **Activer l'Endpoint Picker.** Il est câblé et opt-in, mais **pas encore activé** : à un ou deux réplicas par modèle, son bénéfice — router vers le bon réplica selon la charge et l'affinité de préfixe — ne dépasse pas son coût. Le gain de distribution par préfixe reste **non mesuré**, faute d'un budget GPU pour saturer plusieurs réplicas d'un même modèle (le `NodePool` plafonne à 4 GPU, que la flotte de 4 modèles sature exactement).
* **Les traces OTLP vers VictoriaTraces** — case volontairement décochée aujourd'hui, à cocher le jour où la compatibilité du chemin d'ingestion sera vérifiée, pas avant.
* **Les trois modèles restants**, à basculer sous routage possédé par la composition, et dans le même mouvement la **suppression définitive de `route.yaml`** — le commit qui tuera pour de bon la double comptabilité.

## :bookmark: Références

### Le code

- [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — la plateforme complète
- [PR #1559](https://github.com/Smana/cloud-native-ref/pull/1559) — la composition `InferenceService` v0.6.0 → **v0.8.0**

### Composants techniques

- [Envoy AI Gateway](https://aigateway.envoyproxy.io/) — la porte d'entrée compatible OpenAI, et ses [métriques `gen_ai`](https://aigateway.envoyproxy.io/docs/capabilities/observability/metrics/)
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) — `InferencePool` et Endpoint Picker
- [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) — la classification de prompts derrière `model: MoM`
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
