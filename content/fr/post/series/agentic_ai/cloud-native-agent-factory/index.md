+++
author = "Smaine Kahlouch"
title = "`Maku` : confier des issues GitHub à des agents sur ton cluster Kubernetes… sans leur faire confiance"
date = "2026-11-04"
summary = "Tu labellises une issue GitHub, un agent la traite dans une sandbox gVisor sous sa propre identité, un merge gate décide. Anatomie de Maku, l'agent factory que je fais tourner sur `cloud-native-ref` : identité par run, egress fermé, rooms append-only, classes de merge — avec les chiffres et ce qui manque encore."
featured = true
draft = true
codeMaxLines = 30
usePageBundles = true
toc = true
series = ["Agentic AI"]
tags = ["ai", "kubernetes", "security", "platform-engineering"]
thumbnail = "thumbnail.png"
+++

{{% notice info "Série Agentic AI — la suite logique 🏭" %}}
Dans les épisodes précédents : [les fondamentaux des coding agents](/fr/post/series/agentic_ai/ai-coding-agent/), [mes workflows Claude Code au quotidien](/fr/post/series/agentic_ai/ai-coding-tips/) et [une stack LLM self-hosted](/fr/post/series/agentic_ai/llm-self-hosted-stack/). Jusqu'ici l'agent tournait **sur mon laptop, sous mes yeux**. Ici on change d'échelle : l'agent tourne **sur le cluster, sans moi devant l'écran**, et c'est précisément ça qui pose problème.
{{% /notice %}}

Depuis quelques mois, je ne code presque plus sans agent. Et comme beaucoup, j'ai fini par me poser la question qui vient naturellement : pourquoi est-ce que je reste assis devant lui ? Une issue bien rédigée le soir, une PR à relire au café le matin, voilà le workflow dont j'avais envie. C'est exactement la promesse des **software factories** dont tout le monde parle en 2026 : des agents qui prennent un backlog et rendent des pull requests.

Sauf qu'une fois l'agent hors de ma machine, tout ce qui ne me dérangeait pas en local devient un vrai problème. Avec quel **token Git** pousse-t-il ? Vers **quels domaines** peut-il sortir ? Qui a **approuvé** quoi, et où est-ce écrit ? Et si le texte d'une issue contient une injection de prompt, jusqu'où va-t-elle ? Le modèle, finalement, s'est révélé la partie la moins compliquée : c'est une ligne de configuration. Ce qui m'a pris du temps, c'est **l'identité, l'isolation, le trafic sortant et la preuve** de ce qui s'est passé.

Cet article décrit **Maku**, l'agent factory que je fais tourner sur [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref), ma plateforme de référence : ce qu'elle fait, comment chaque brique cloud-native y contribue, et ce que ça donne pour de vrai — chiffres, drills et ratés compris. C'est long, mais chaque section se lit indépendamment : si tu ne veux que la sécurité, saute directement à [l'identité par run](#-une-identité-par-run-zéro-secret-dans-la-sandbox).

## 🎯 Objectifs

* 🏭 Comprendre ce qu'on appelle une **software factory**, et où se situe celle-ci sur l'échelle de l'autonomie
* 🧩 Voir comment **Crossplane, Cilium, gVisor, agent-sandbox, agentgateway, octo-sts et Kueue** s'assemblent pour qu'un run d'agent soit **isolé, identifié, mesuré et stoppable**
* 🔐 Suivre une **identité de bout en bout** : un humain, un run, un jeton GitHub d'une heure, et aucun secret dans la sandbox
* 🚦 Décortiquer le **merge gate** : des classes de changement, du mode *shadow* au mode *live*, et un disjoncteur
* 🩺 Fermer la boucle avec [RunLore](/fr/post/series/observability/runlore/) : d'une alerte à une PR mergée et vérifiée
* 📊 Lire des **chiffres réels** (coûts, cache, latence) et une liste honnête de ce qui manque encore

## 🔥 Pourquoi se compliquer la vie ?

Le déclic, c'est un vendredi soir et une demi-douzaine d'issues de documentation en retard sur `cloud-native-ref` : un lien cassé, une version de chart à mettre à jour, un chemin de script à corriger. Rien de passionnant, et pourtant chacune me coûte le même rituel : ouvrir Claude Code, lui coller l'issue, vérifier, pousser, ouvrir la PR. La version naïve d'une automatisation est évidente et prend une heure : un `Job` Kubernetes qui lance l'agent avec un **PAT GitHub dans un `Secret`** et une clé d'API de modèle dans une variable d'environnement.

Et c'est précisément là que ça devient inconfortable. Ce `Job` :

* pousse avec **mon** identité, ou avec un token qui vaut la mienne, sans date de péremption ;
* peut joindre **n'importe quel** domaine, donc exfiltrer ce qu'il lit si une issue le lui demande gentiment ;
* laisse comme seule trace un log de pod, qui disparaît avec lui ;
* et surtout, rien ne l'empêche d'**approuver** ou de **merger** ce qu'il vient d'écrire.

Un agent qui tourne en autonomie, c'est du **code non relu qui s'exécute avec des droits**. En local, je suis le garde-fou. Sur un cluster, le garde-fou doit être dans l'architecture. C'est ce qui a transformé un script d'un soir en plateforme, et ce que je décris ici.

## 🏭 Software factory : de quoi parle-t-on ?

Le vocabulaire s'est stabilisé cet été. Addy Osmani le résume ainsi dans [Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/) : une **boucle** (*loop*) est un agent qui observe, agit et vérifie jusqu'à une condition d'arrêt ; un **harnais** (*harness*) est tout ce qui entoure cette boucle — sandbox, outils, permissions, définition du « terminé » ; une **factory**, ce sont de nombreuses boucles harnachées, alimentées par une file de travail et filtrées par un **gate de revue** avant la production. Sa phrase clé : *« Verification, not generation, is the real constraint on a factory. »*

Dan Shapiro, lui, a proposé [cinq niveaux](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/) calqués sur ceux de la conduite autonome. Au niveau 3 tu supervises des agents et tu **lis chaque diff** ; au niveau 4 tu écris des specs, tu relis des plans et tu reviens voir si les tests passent ; au niveau 5, la *dark factory*, **plus personne ne lit le code**. L'exemple le plus commenté est celui de [StrongDM](https://simonwillison.net/2026/Feb/7/software-factory/) : *« Code must not be reviewed by humans »*, des scénarios de test tenus hors du dépôt pour que les agents ne puissent pas les voir, et un coût que Simon Willison estime à ~1 000 $ par ingénieur et par jour.

Et puis il y a le contre-exemple. Dex Horthy (HumanLayer) raconte, dans sa conférence *Harness Engineering is not Enough*, une factory entièrement automatisée qui a tenu **environ quatre mois** avant une panne majeure, longue à diagnostiquer. Osmani en tire le concept de **dette de compréhension** : *« the widening gap between how much code exists and how much any human still understands. »* La panne n'est pas spectaculaire ; elle est *« quiet and late »*.

{{% notice note "Où se situe cette factory ? 🧭" %}}
Clairement au **niveau 4**. J'écris l'issue, je relis les plans quand l'agent en propose, et je merge. Les agents implémentent, relisent et testent. Deux **classes** de changement seulement — les corrections de liens dans la documentation, et le revert d'un merge de la factory — ont le droit de se merger seules, après une batterie de contrôles, et ce droit a déjà été **retiré une fois** par le disjoncteur. Pas de *dark factory* ici, et ce n'est pas l'objectif : l'objectif est de savoir **exactement** ce qui doit être vrai avant qu'une machine merge quoi que ce soit.
{{% /notice %}}

Ce qui m'a frappé en lisant tout ça, c'est à quel point le débat porte sur les modèles et les boucles, et si peu sur le **harnais côté plateforme** : où tourne le code, avec quels droits, et qui le constate. Un guide récent de Firecrawl le dit sans détour : *« An AI software factory is five stages with a gate at each one, and the coding agent is the cheapest piece of it. »* Le projet [Agent Baseline](https://agentbaseline.org) a même publié une architecture de référence, *The Secure Software Factory*, organisée autour de quatre frontières : la **sandbox**, le **réseau**, la **chaîne d'approvisionnement** et **l'identité**. C'est, à peu de choses près, le plan de cet article.

Un mot de vocabulaire avant d'entrer dans le concret, parce que je vais beaucoup l'utiliser : une **sandbox**, c'est l'environnement isolé dans lequel l'agent exécute du code ; l'**egress**, c'est le trafic réseau qui en *sort* ; un **run**, c'est une exécution d'un agent pour une tâche, du démarrage du pod à sa fin ; et le **merge gate**, c'est l'ensemble des conditions à remplir pour qu'une PR soit mergée, qu'un humain le fasse ou non.

## 🗺️ Vue d'ensemble : trois dépôts, trois couches

L'ensemble repose sur trois dépôts, et la séparation n'est pas cosmétique : elle dit qui possède quelle frontière de confiance.

| Couche | Dépôt | Ce qu'elle apporte |
|---|---|---|
| **Le cluster hôte** | [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) | GKE (`gcp-0`), Cilium en *default deny*, Crossplane, Gateway API, ZITADEL, OpenBao, la stack Victoria… et le dépôt sur lequel les agents travaillent |
| **La plateforme IA** | [`ai-native-ref`](https://github.com/Smana/ai-native-ref) | Les manifests Flux qui hébergent Maku : runtime gVisor, `agent-sandbox`, l'agent gateway, `octo-sts`, Kueue, les policies Kyverno et Cilium, les dashboards et alertes |
| **Le produit** | [`maku`](https://github.com/Smana/maku) | Le contrôleur de la factory, le broker de *rooms*, le sidecar *bridge*, la CLI `maku`, la Web UI et l'app Slack |

Un détail d'architecture que j'aime bien : la plateforme IA arrive dans le cluster hôte sous la forme d'un **artefact OCI signé** (`oci://ghcr.io/smana/ai-native-ref`), épinglé par digest, vérifié par cosign et **suspendu par défaut**. Le cluster hôte publie un *contrat* (`HOST-CONTRACT.md`) qui liste ce qu'il fournit — namespaces, Gateways, mounts OpenBao, pool de nœuds gVisor — et la plateforme IA se déploie par-dessus. Le jour où il faut tout refaire, un `git revert` du pin suffit.

{{% notice tip "Maku, la factory, la plateforme : qui est qui ? 🎭" %}}
**Maku** (幕, « le rideau » en japonais) est le produit : `maku-factory` (le reconciler des tâches), `maku-broker` (le journal des rooms), `maku-bridge` (le sidecar dans chaque pod de run), la CLI `maku` et la Web UI. Quand j'écris « la factory » dans la suite, je parle du système complet : Maku **plus** la plateforme IA qui l'héberge — sandbox, identité, egress, gateway. Cette séparation est voulue : Maku ne fait **pas** confiance à l'agent, et ne cherche pas à le confiner lui-même ; c'est le travail de la plateforme.
{{% /notice %}}

Voici une tâche de bout en bout, dans ses grandes lignes :

{{< img src="agent-factory-overview.png" alt="Vue d'ensemble : un mainteneur labellise une issue, Maku ouvre une room et démarre un run dans une sandbox gVisor ; chaque appel passe par l'agent gateway vers les modèles, les outils MCP et octo-sts ; le run pousse une branche et ouvre une PR que le mainteneur relit" width="1200" >}}
</br>

1. **Un mainteneur** pose le label `factory/ready` sur une issue. Le même label posé par quelqu'un d'autre ne déclenche rien : Maku revérifie que l'auteur a bien le droit `write` sur le dépôt.
2. **Maku** prend un instantané de l'issue, crée une `Task` et une `Room`, choisit un *tier* (modèle, budget en tokens et en dollars, durée max), puis crée un `AgentRun`.
3. **Crossplane** compose le run : un `ServiceAccount` sans aucun droit Kubernetes, une `CiliumNetworkPolicy` dédiée et une `Sandbox` qui devient un pod **gVisor** dans le namespace `agents`.
4. **Le harnais** — [OpenHands](https://github.com/OpenHands/OpenHands) — clone le dépôt, travaille, pousse une branche `agent/<task>` et ouvre une PR. Chaque appel au modèle, à un outil MCP ou à GitHub passe par l'**agent gateway**, sous l'identité du run.
5. **Un run reviewer** relit le diff dans la même room et pose son verdict. Puis le **merge gate** décide : merge automatique pour une classe `live`, « j'aurais mergé » pour une classe `shadow`, et sinon un humain merge.

Tout ce qui précède est **narré sur l'issue GitHub** par Maku, étape par étape. Voilà par exemple ce qu'il a écrit, tel quel, pendant la correction d'une règle d'alerte le 28 octobre :

```text
Agent factory task `drq5gh2i` started run `ralxknfk` (implementer) on branch `agent/drq5gh2i`.
- Budget: 1.5 M tokens, 45 minutes (tier standard)
- Watch: https://rooms.priv.gcp.ogenki.io/r/drq5gh2i (tailnet only)
- Stop: apply the label `factory/stop`

Run `ralxknfk` of task `drq5gh2i` opened #2638.

Agent factory task `drq5gh2i` waits for a maintainer's review: the policy needs a maintainer's approval.

Agent factory task `drq5gh2i` is done: the pull request was merged. Merged by @Smana.
Tokens used: 490 k.
```

Budget affiché, lien vers la room, bouton d'arrêt, identité du mergeur : l'issue **est** le compte rendu. Voyons maintenant ce qui se cache derrière chacune de ces lignes.

## 🔐 Une identité par run, zéro secret dans la sandbox

C'est le cœur du sujet, et ce qui a le plus structuré les choix. La question de sécurité d'une factory tient en une phrase : **qui agit, avec quel jeton, et jusqu'où ce jeton porte-t-il ?**

La réponse s'appuie sur une primitive Kubernetes que l'on sous-estime souvent : le **jeton de ServiceAccount projeté**. C'est un JWT signé par le cluster, émis pour une **audience** précise (le service qui a le droit de l'accepter) et avec une **durée de vie courte**, renouvelé par le kubelet. Autrement dit : une identité par pod, gratuite, vérifiable hors ligne par quiconque connaît l'issuer du cluster. Toute la chaîne qui suit consiste à ne distribuer **que** ce type de jeton dans la sandbox, et à échanger le reste à l'extérieur.

{{< img src="agent-run-identity.png" alt="Anatomie d'un run : la claim AgentRun est composée par Crossplane en ServiceAccount, CiliumNetworkPolicy et Sandbox ; dans le pod gVisor, le harnais ne détient aucun jeton, identity-proxy attache le jeton du run à chaque appel vers l'agent gateway, maku-bridge détient le jeton de room ; octo-sts échange le jeton contre un token GitHub d'une heure limité aux branches agent/**" width="1200" >}}
</br>

Un pod de run contient **trois conteneurs**, et la répartition des jetons entre eux n'est pas un détail :

* **`harness`** (OpenHands, UID 10001) : le seul qui exécute du code non relu. Il n'a **aucun** jeton de ServiceAccount monté (`automountServiceAccountToken: false`, imposé par Kyverno), aucun `Secret` Kubernetes, aucune clé de fournisseur de modèle. Son unique identifiant est le token GitHub d'installation, **en mémoire**, révoqué dans le `preStop`.
* **`identity-proxy`** (Envoy, sidecar natif) : il détient les **jetons projetés** du ServiceAccount du run et les attache à chaque appel sortant. Le harnais parle à `127.0.0.1`, le proxy parle au gateway.
* **`maku-bridge`** : il détient un troisième jeton, d'audience `room-broker`, valable **600 secondes**, et pousse chaque événement du harnais vers le broker de rooms. Le harnais ne voit jamais ce jeton : un harnais compromis ne peut donc **pas forger** un verdict ou une décision dans la room.

Ces jetons sont de simples `serviceAccountToken` projetés, avec une **audience par destination** :

```yaml
# Extrait d'une Sandbox (agents.x-k8s.io/v1beta1) composée pour un run implementer
spec:
  podTemplate:
    spec:
      serviceAccountName: xplane-run-ralxknfk
      automountServiceAccountToken: false
      runtimeClassName: gvisor
      restartPolicy: Never
      activeDeadlineSeconds: 2700          # le runMinutes du tier, ici 45 min
      volumes:
        - name: public
          projected:
            sources:
              - serviceAccountToken: {audience: agent-router.implementer.public, expirationSeconds: 600, path: token}
        - name: sts
          projected:
            sources:
              - serviceAccountToken: {audience: octo-sts/Smana/cloud-native-ref/implementer, expirationSeconds: 600, path: token}
```

De l'autre côté, l'**agent gateway** ([agentgateway](https://agentgateway.dev/), déployé comme `Gateway` Kubernetes) valide ce JWT **hors ligne** contre l'issuer du cluster, exige que le `sub` commence par `system:serviceaccount:agents:`, puis **écrase** l'en-tête `x-ar-agent` avec ce `sub`. Un run ne peut pas se faire passer pour un autre : l'identité vient du jeton vérifié, jamais d'un en-tête. C'est aussi le gateway qui **injecte la clé du fournisseur** (Z.ai, Anthropic) depuis un Secret du namespace `agent-system` — la clé n'entre jamais dans `agents`. Et c'est lui qui compte les tokens par run et par flotte : quand un budget est épuisé, la réponse est un `429` explicite, observé en conditions réelles le 18 octobre :

```json
{"error":{"type":"rate_limit_error","code":"agent_budget_exhausted",
 "message":"agent-router refused this call: a gateway token budget is spent (limit 5000000, resets in 12877 s). Not retryable."}}
```

### Et GitHub, dans tout ça ?

C'est là que ça devient intéressant. Le run ne possède **aucun** credential GitHub durable. Quand le harnais a besoin de pousser, `git-credential-agent` demande au proxy un échange sur [`octo-sts`](https://github.com/octo-sts/app), auto-hébergé : le jeton projeté (audience `octo-sts/Smana/cloud-native-ref/implementer`) est vérifié contre une **politique de confiance versionnée dans le dépôt cible**, et échangé contre un token d'installation GitHub App valable **au plus une heure**, pour **un dépôt** et **un rôle** :

```yaml
# .github/chainguard/agent-implementer.sts.yaml (dans cloud-native-ref)
issuer_pattern: 'https://container\.googleapis\.com/v1/projects/ogenki-435905/locations/europe-west4-a/clusters/gcp-0'
subject_pattern: 'system:serviceaccount:agents:xplane-run-[a-z2-7]{8}'
audience: octo-sts/Smana/cloud-native-ref/implementer
permissions:
  contents: write
  pull_requests: write
  issues: read
  checks: read
```

Un reviewer ou un triager obtient la même chose en **lecture seule**. Et côté GitHub, des *rulesets* limitent l'App des agents aux branches `agent/**` : impossible de pousser sur `main`, impossible de poser un tag. J'ai vérifié chaque refus en runbook : push sur sa propre branche ✅, push sur `main` → `GH013`, push d'un reviewer → `403`, échange pour un autre dépôt → `404 unable to find trust policy`.

{{% notice info "Quatre GitHub Apps, un seul mergeur 🔑" %}}
La séparation des pouvoirs passe par **quatre** GitHub Apps : `ogenki-agents` (les runs, confinée à `agent/**`), `ogenki-agent-factory` (la narration sur les issues), `ogenki-merge-gate` (le statut de [policy-bot](https://github.com/palantir/policy-bot)) et `ogenki-agent-merger`, **la seule** qui ait le droit de contourner le ruleset pour merger — et sa clé n'est montée que dans le contrôleur de Maku, jamais dans une sandbox. Un agent ne peut ni approuver ni merger sa propre PR, par construction.
{{% /notice %}}

Et l'humain ? Il n'apparaît que deux fois dans la chaîne, et toujours **hors de la sandbox** : au départ (le label, ou un appel à l'API de Maku authentifié par ZITADEL) et à l'arrivée (la revue, le merge). Dans les rooms, son identité est `human:<sub>` ; celle d'un run est `agent:<runId>` ; celle de la factory `system:factory`. Les dépenses sont **imputées au labelliseur**, pas à un compte technique anonyme.

## 🧱 Isolation et egress : la sandbox ne sort que par la porte

L'identité ne sert à rien si le code peut s'échapper de son pod ou parler à n'importe qui. Deux choix ici.

### gVisor, et ce qu'il coûte

Chaque pod du namespace `agents` tourne sous la `RuntimeClass` **gVisor**, sur un pool de nœuds spot dédié (`agents-gvisor`, GKE Sandbox côté GCP). gVisor intercale un noyau en espace utilisateur, le *Sentry*, entre le conteneur et le noyau de l'hôte : une évasion par appel système doit d'abord traverser le Sentry. J'ai retenu gVisor plutôt que des microVMs parce qu'il est disponible **nativement** sur les deux clouds que je vise et que la primitive [`agent-sandbox`](https://agent-sandbox.sigs.k8s.io/) (projet Kubernetes SIG, `Sandbox`/`SandboxTemplate`/`SandboxClaim`) le supporte directement.

Le prix est réel, et il faut le connaître avant de le payer :

* Le profil seccomp `RuntimeDefault` est **désactivé** sous gVisor : `runsc` ignore `errnoRet` ([gvisor#14688](https://github.com/google/gvisor/issues/14688)), ce qui transforme le `clone3 → ENOSYS` attendu par glibc ≥ 2.34 en `EPERM`, et plus aucun thread ne démarre. C'est le Sentry qui tient la frontière, pas seccomp.
* Les I/O fichiers sont environ **1,74×** plus lentes (mesuré sur un clone + build représentatif).
* Cilium a besoin de `socketLB.hostNamespaceOnly: true` : la pile réseau de gVisor n'appelle jamais `connect()` dans le noyau hôte, donc le socket-LB ne peut pas traduire une ClusterIP.
* Et un pod gVisor ne reçoit pas les événements inotify du kubelet : un jeton projeté **tourné** sur l'hôte n'est jamais vu par le pod. La parade est simple — le TTL du jeton vaut la durée max du run — mais il fallait le découvrir.

Trois policies Kyverno figent la forme de tout pod dans `agents` : `runtimeClassName == gvisor`, pas de jeton d'API monté, et créé par une `Sandbox` — rien d'autre.

### L'egress, refusé à la résolution de nom

Le namespace `agents` commence par une `CiliumNetworkPolicy` **default deny** complète, et chaque run reçoit la sienne, composée par Crossplane. Sa forme :

```yaml
egress:
  - toEndpoints:
      - matchLabels: {io.kubernetes.pod.namespace: kube-system, k8s-app: kube-dns}
    toPorts:
      - ports: [{port: "53", protocol: UDP}]
        rules:
          dns:
            - matchPattern: "*"           # le proxy DNS de Cilium observe chaque requête
  - toFQDNs:
      - matchName: github.com
      - matchName: api.github.com
      - matchName: objects.githubusercontent.com
    toPorts:
      - ports: [{port: "443", protocol: TCP}]
  # + le data plane de l'agent gateway (8080/8081/8082) et le broker (8443)
```

Un nom non autorisé **ne résout pas** : l'agent reçoit `Temporary failure in name resolution`, pas un timeout TCP, et Hubble journalise un `dns-request proxy DROPPED`. Des *profils* opt-in (`pypi`, `npm`, `golang`, `crates`) ajoutent les registres de paquets quand une tâche en a besoin. Et pour la classe de données `internal`, c'est plus strict encore : **aucun** egress GitHub direct ; `git` passe par le gateway, qui n'autorise que `git-upload-pack` (la lecture) sur les dépôts explicitement listés.

{{% notice warning "Pourquoi autant d'insistance sur l'egress ? ☠️" %}}
Parce que c'est le troisième terme de la [*lethal trifecta*](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) de Simon Willison : accès à des données privées + exposition à du contenu non fiable + **capacité à communiquer vers l'extérieur**. Un agent qui lit des issues GitHub et du code tiers a forcément les deux premiers. Anthropic a documenté [un cas réel](https://www.anthropic.com/research/how-we-contain-claude) d'exfiltration **par l'endpoint d'inférence autorisé**, avec une clé fournie par l'attaquant, et conclut : *« Design for containment at the environment layer first, then steer behavior at the model layer. »* Je ne demande pas au modèle de se méfier ; je coupe le canal.
{{% /notice %}}

Dernière brique : **Kueue** admet les sandboxes dans des `ClusterQueue` plafonnées (16 CPU / 64 Gi / 4 pods pour la file `factory`), sans emprunt possible. Une factory qui s'emballe ne peut pas remplir le cluster.

## 📓 La room : la boîte noire de chaque run

Quand l'agent travaille sans toi, la question suivante est : **qu'est-ce qu'il a fait, exactement ?** Un log de pod ne suffit pas : il disparaît avec le pod, n'importe qui peut y écrire n'importe quoi, et il mélange ce que l'agent *dit* avoir fait avec ce qu'il a *réellement* appelé. La réponse de Maku s'appelle une **room** : un journal partagé par tâche, à la fois boîte noire d'avion et salon de discussion. Pendant le run, tu regardes chaque étape, tu peux poster une consigne, et tu approuves ce que l'agent n'a pas le droit de faire seul. Après le run, la room reste, et c'est elle que tu rouvres quand une PR te surprend trois semaines plus tard.

{{< img src="maku-room.png" alt="Une room Maku pendant une revue : l'état « Waiting — a maintainer decides », les chips budget (1,9 M / 3 M tokens, 64 %), PR, issue, dernier verdict du reviewer, puis le fil des tool calls du triager" width="1100" caption="Une room pendant un drill : le budget consommé, le dernier verdict, et chaque tool call du triager, repliable." >}}
</br>

Ce qui fait qu'on peut s'y fier :

* **Append-only.** Le rôle PostgreSQL du broker n'a que `SELECT` et `INSERT` sur la table des événements ; un trigger interdit de dé-sceller ou de re-dater. La preuve tient en une ligne :

  ```sql
  SET ROLE rooms_broker; UPDATE events SET payload = '{}' WHERE room_id = '3kq7x2ma';
  -- ERROR:  permission denied for table events
  ```

* **Séquencé sans trou.** Chaque événement reçoit un `seq` monotone sous verrou de ligne ; une clé d'idempotence `(room, client, seq d'origine)` fait qu'un rejeu renvoie l'événement déjà stocké sans en consommer un nouveau. À la fermeture, un événement de **sceau** interdit tout ajout.
* **Caviardé d'abord.** Chaque chaîne JSON passe par gitleaks **avant** d'être stockée, loguée ou comptée.
* **Attribué.** L'`actor` de chaque événement est tamponné par le broker à partir du jeton vérifié, jamais déclaré par l'émetteur. Les événements d'un agent sont du *contenu non fiable attribué à cet agent*, et un bridge ne peut pousser que des événements d'origine harnais : un verdict, un changement de pilote ou une décision d'approbation ne peuvent venir que de Maku ou d'un humain.

Tu accèdes aux rooms par la Web UI (derrière ZITADEL), par la CLI `maku` dans ton terminal, ou depuis ton propre coding agent via un serveur MCP. La CLI, volontairement, **ne peut rien approuver** : regarder, poster, mettre en file, forker — c'est tout. Les approbations se prennent dans la Web UI, et la carte d'approbation montre l'**action brute**, pas la prose de l'agent. La règle *four-eyes* (quand la room l'exige) interdit en plus à celui qui a soufflé la consigne d'approuver son résultat.

{{< img src="maku-rooms.png" alt="La liste des rooms de Maku : celles qui attendent un humain d'abord, chacune avec son dépôt, sa ligne d'état, sa PR et sa dernière activité" width="1100" caption="La liste des rooms : ce qui attend un humain remonte en premier." >}}
</br>

Pour te faire une idée sans rien installer, une room réelle est rejouable dans la [démo publique](https://blog.ogenki.io/maku/).

## 🚦 Le merge gate : des classes, du shadow, un disjoncteur

Voilà le point où l'on décide de ce qu'on délègue vraiment. Mon parti pris : l'autonomie n'est pas une fonctionnalité globale qu'on active, c'est un **privilège par classe de changement**, gagné statistiquement et **révocable**. Une *classe*, c'est une famille de PR définie par ce qu'elle a le droit de toucher et la forme attendue de son diff : `docs-links` pour les liens de la documentation, `deps-patch` pour les mises à jour de dépendances en version patch, `alert-rules` pour les règles d'alerte, et ainsi de suite. Chaque classe a son propre état — `shadow`, `live` ou simple revue humaine — et ses propres plafonds.

### Deux gardiens

Le premier gardien est [**policy-bot**](https://github.com/palantir/policy-bot), qui lit `.policy.yml` **depuis `main`** (jamais depuis la PR) et publie un statut requis. Voici la règle qui autorise l'une des deux classes *live* aujourd'hui (l'autre est le revert, réservé à l'App merger) :

```yaml
# .policy.yml (cloud-native-ref), extrait
- name: "low-risk: docs-links"
  if:
    has_author_in:            {users: ["ogenki-agents[bot]"]}
    only_has_contributors_in: {users: ["ogenki-agents[bot]"]}
    from_branch:              {pattern: '^agent/[a-z2-7]{8}$'}
    only_changed_files:       {paths: ['^(docs|website/content)/.+\.md$']}
    no_changed_files:
      paths: ['^\.policy\.yml$', '^\.github/', '(^|/)(AGENTS|CLAUDE)\.md$', ...]   # les « gate paths »
    modified_lines: {total: "< 21"}
    has_workflow_result: {conclusions: [success], workflows: [.github/workflows/ci.yaml]}
```

Une PR d'agent qui touche à `.github/`, à la politique elle-même, aux pins Crossplane ou aux mounts OpenBao ne peut **jamais** relever d'une classe automatique : ces *gate paths* repartent vers un humain, liens corrects ou pas. Un script CI échoue si une règle oublie l'un d'eux.

Le second gardien est le gate de Maku, qui applique une liste **ordonnée** de contrôles au head de la PR. En version courte :

| Ordre | Ce qui doit être vrai | Sinon |
|---|---|---|
| 1 | Le scan de secrets n'est pas rouge | `secret_scan_red` → escalade, pas de run de correction |
| 2 | Les checks CI requis sont verts | rouge → un run de correction borné, puis `ci_red` |
| 3 | Le dépôt désigne **cette** factory (`.github/agent-factory.yaml`) | `not_owned` |
| 4 | policy-bot est vert et posté par le bon login | `policy_absent`, `gate_path`, `policy_pending` |
| 5 | L'auteur est l'App des agents, sans approbation humaine déjà posée | `not_agent_authored`, `maintainer_approved` |
| 6 | Le reviewer a approuvé **ce** head ; le trailer `Agent-Run` nomme un run de la tâche | `verdict_stale`, `head_unreported`, `foreign_trailer` |
| 7 | Les conditions de la classe : fichiers autorisés, taille du diff, forme (`links-only`, `semver-patch`), holdouts | `file_not_allowed`, `patch_too_large`, `class_mismatch`… |
| 8 | Le disjoncteur de la classe n'a pas sauté ; les plafonds quotidiens tiennent | `class_paused`, `auto_merge_cap` |
| 9 | Chaque push de la branche vient de l'App des agents | `human_commit_in_pr` |
| 10 | La classe est `live` | sinon `shadow_would_arm` : un commentaire, pas de merge |

Chaque « non » est un **code de raison** que Maku écrit sur la PR et expose en métrique (`agent_factory_merge_decisions_total{class, verdict, reason}`). Une PR d'agent porte aussi un pied de provenance (`Agent-Room`, `Agent-Run`, `Agent-Task`, `Agent-Model`), mais la doc de Maku est explicite : *« Trailers and footer are provenance hints, never authorisation. »* Ce qui lie un commit à la factory, c'est l'identité qui a poussé, pas ce qui est écrit dedans.

{{< img src="merge-gate.png" alt="Le merge gate : une PR d'agent traverse dans l'ordre le scan de secrets, la CI, policy-bot, le verdict du reviewer sur ce head, les conditions de la classe, le disjoncteur et les plafonds ; une classe live est mergée par l'App merger puis surveillée, une classe shadow reçoit un commentaire « would auto-merge », tout le reste attend un humain" width="1200" >}}
</br>

### Du shadow au live, et retour

Une classe commence en **shadow** : le gate décide exactement comme en live, écrit *« would auto-merge: docs-links, checks green, verdict approve »* sur la PR, puis un humain merge. Chaque décision shadow nourrit un **tally** par classe. Pour passer en live, la **borne supérieure de Clopper-Pearson à 95 %** du taux de mauvais merges, calculée sur les merges mûrs et concluants, doit passer sous 10 %. Concrètement : **29 merges propres**, 46 s'il y en a un mauvais, 61 s'il y en a deux. Pas de démo réussie trois fois qui devient une autonomie générale.

Et dans l'autre sens, un **disjoncteur** : un seul revert dans les dix derniers merges de la classe la renvoie en revue humaine. Ce n'est pas théorique. Le premier — et à ce jour le seul — merge entièrement autonome a eu lieu le **18 octobre à 15:20 UTC** : la PR #2520, quatre URLs de dashboards Grafana remplacées par leurs formes canoniques, mergée par `ogenki-agent-merger[bot]` après CI verte, policy-bot vert et verdict du reviewer. Trente minutes de surveillance de `main` plus tard, Maku écrivait *« merged, and main stayed green for 30 minutes »*. Le label `factory/revert` a ensuite servi à exercer la voie de secours, en conditions réelles : Maku a ouvert #2521, et écrit :

> Auto-merge of the class `docs-links` goes back to human review until its recent merges pass without a revert (R41).

La classe `docs-links` est **toujours** en revue humaine depuis, et c'est exactement le comportement voulu. Deux enseignements du drill : le revert lui-même ne s'est pas auto-mergé (une interaction avec l'auto-merge natif de GitHub, corrigée depuis), et un disjoncteur qui saute sur un revert *demandé* ne distingue pas un drill d'un incident. Le second point reste ouvert.

## 🩺 RunLore ferme la boucle : de l'alerte à la PR vérifiée

Dans [l'article précédent](/fr/post/series/observability/runlore/), RunLore investiguait les alertes et apprenait de chaque résolution, mais s'arrêtait à la proposition. Maku lui donne une suite : depuis la v0.17.0, RunLore est un **client API ordinaire** de Maku, sous son propre principal machine (`system:runlore`, via ZITADEL), avec un plafond de **2 $ par tâche** et **10 $ par jour**.

{{< img src="runlore-loop.png" alt="La boucle RunLore : une alerte déclenche une investigation RunLore, qui crée une tâche investigate ; un run triager en lecture seule propose un correctif dans sa room ; un mainteneur publie la proposition avec factory/publish, la relit, puis la démarre avec factory/ready ; la PR est relue par un agent, mergée par un humain ; après un soak de 30 minutes RunLore interroge Alertmanager et rend son verdict" width="1200" >}}
</br>

Le point de design qui compte, ce sont les **deux labels** :

* `factory/publish` : la proposition de l'agent reste **dans sa room** tant qu'un mainteneur ne l'a pas relue. L'issue créée par Maku ne contient qu'un *stub* (alerte, ressource, confiance) ; aucun finding interne ne devient public sans ce geste.
* `factory/ready` : sur l'issue publiée, un **second** label autorise la dépense d'un run d'implémentation. Entre les deux, un humain a lu ce qui va devenir public **et** ce qui va coûter.

Voilà ce que ça a donné le **28 octobre**, sur une vraie alerte (`PodinfoNoAvailableReplica`, qui se déclenchait en permanence à cause d'une faute de frappe — `podnfo` — dans un sélecteur de `VMRule`) :

| Heure (UTC) | Quoi |
|---|---|
| 17:29 | RunLore rend un verdict `action_suggested` à 0,92 de confiance et crée la tâche `hvavktlo` ; Maku ouvre l'issue #2630 (stub) |
| 17:33 → 17:36 | Un run **triager**, en lecture seule, propose le correctif dans sa room : 206 k tokens, ~3 minutes |
| 17:36 | Je pose `factory/publish` ; Maku publie la proposition comme issue #2631 |
| 18:14 | Je pose `factory/ready` ; run implementer `ralxknfk` (tier standard, 1,5 M tokens / 45 min) |
| 18:16 | PR #2638 ouverte |
| 18:25 | Verdict du reviewer : **approve**, avec sa propre vérification (`kustomize build`, `grep` du nom fautif) |
| 19:20 | Je merge (classe `alert-rules`, revue humaine obligatoire) ; 490 k tokens au total |
| 21:32 | Le moniteur post-merge rend son verdict : CI de `main` saine, zéro PR de suivi |

Le lendemain, même scénario sur `AppWizardUnavailable` (#2657 → #2658 → #2659), et cette fois le moniteur a pu écrire : *« RunLore: resolved, alert AppWizardUnavailable »*. Le **soak** de 30 minutes après le merge interroge Alertmanager ; le verdict `resolved` ou `persisting` est celui d'Alertmanager, la ré-investigation ne sert que de preuve. Un `persisting` ouvre un suivi et peut faire sauter le disjoncteur de la classe.

Côté RunLore, la configuration tient en quelques lignes :

```yaml
factory:
  url: http://agent-factory.agent-system.svc.cluster.local:8082
  token_url: https://auth.example.com/oauth/v2/token      # ZITADEL, client credentials
  client_id_env: FACTORY_CLIENT_ID
  client_secret_env: FACTORY_CLIENT_SECRET
  repository: Smana/cloud-native-ref
  max_cost_usd: 2
  webhook: {secret_env: FACTORY_WEBHOOK_KEY}               # Standard Webhooks, HMAC-SHA256
  soak: 30m
  verify:
    alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
```

Deux agents, deux dépôts, un seul humain aux trois points qui comptent : ce qui devient public, ce qui coûte, et ce qui est mergé.

## 📊 Ce que ça coûte, ce que ça donne

Assez d'architecture, place aux chiffres. Je tiens à ce que chacun porte sa date, parce que tout bouge vite : un mois plus tard, le modèle, les prix et les tiers auront probablement changé. Tout ce qui suit a été mesuré sur `gcp-0`, avec **GLM-5.3** (Z.ai) comme modèle par défaut des agents pour le dépôt public — le même choix que pour la démo RunLore, et pour la même raison : une qualité proche des modèles frontière à un coût par token bien inférieur, ce qui compte quand un agent boucle des centaines de fois.

| Mesure | Valeur | Contexte |
|---|---|---|
| PRs d'agents mergées sur `cloud-native-ref` | **35** entre le 27 septembre et le 29 octobre | Docs, scripts, règles d'alerte, un script de santé Flux avec son dashboard et sa doc |
| …dont mergées sans humain | **1** (#2520), puis revert et disjoncteur | Voir plus haut |
| Part des tokens d'entrée lus en cache | **95,1 %** | 27,4 M sur 28,8 M, fenêtre de 24 h au 18 octobre |
| Coût mesuré sur cette fenêtre | **10,40 $** contre 41,64 $ sans cache | Recalcul tarifaire, pas un second run |
| Un run implementer complet (PR #2508, 225 étapes) | **1,65 $** | 4,43 M tokens d'entrée dont 4,16 M en cache |
| Tier `standard` | 1,5 M tokens / 45 min / **2,55 $** par run, 5,10 $ par tâche | Plafond flotte : 42,50 $ par jour |
| Démarrage d'une sandbox, pool vide | **1,5 à 3,5 min** (205 s sur un nœud neuf) | gVisor + image du harnais |
| Démarrage avec un nœud gardé chaud | **p50 16 s, p90 17 s** sur 10 runs | Le nœud coûte 125,80 $/mois : **désactivé** par défaut |
| Cohorte « issues difficiles » (15 issues, 14–15 octobre) | factory + GLM-5.3 : **0/15** (9 $) · Claude Code Action + Claude : **13/15** (15,80 $) | Un dépôt, un run par issue |

Deux lectures. D'abord, **le cache est le premier levier de coût** : la boucle d'un agent relit sans cesse le même contexte, et 95 % de hits divisent la facture par quatre. Ensuite, et c'est plus inconfortable, **le modèle compte énormément sur les tâches difficiles**. GLM-5.3 fait très bien les corrections de docs et de scripts qui constituent l'essentiel du flux ; sur quinze issues volontairement dures, il n'en a livré aucune là où Claude en livrait treize. C'est pour ça que la v3 de Maku travaille à **adapter la tâche au modèle** (un *scoper* qui borne l'issue avant de la lancer, un *splitter* qui la découpe) plutôt qu'à chercher le meilleur modèle unique.

Tout est instrumenté : les séries `agent_factory_*` couvrent phases, délai label → PR, devenir des PRs par classe (auto-mergée, mergée par un humain, fermée, revertée), décisions du gate avec leur raison, budgets et coûts par run, par principal et par modèle ; les rooms exposent `rooms_events_appended_total`, les caviardages et la latence du fan-out. Les traces ne contiennent **que des identifiants, le tier, la phase et un code de raison** — jamais le texte d'une issue ni un prompt. Le transcript complet reste dans la room, sous contrôle d'accès.

## 🔬 Les alternatives

Maku n'arrive pas sur un terrain vide. Pour situer :

| Solution | Ce que c'est | Ce que Maku fait autrement |
|---|---|---|
| [**GitHub Copilot coding agent**](https://docs.github.com/copilot/concepts/agents/cloud-agent/about-cloud-agent) | L'agent cloud de GitHub, dans un sandbox ou GitHub Actions, firewall sortant par liste d'autorisation | Clé en main et intégré à GitHub ; mais la frontière de confiance est chez GitHub, et [des rapports](https://bex.co/blog/2026/07/28/copilot-coding-agent-self-hosted-runners-trust-boundary) décrivent un firewall à **désactiver entièrement** pour atteindre des services internes |
| [**Codex cloud**](https://learn.chatgpt.com/docs/cloud) | Un environnement par tâche chez OpenAI, accès réseau configurable par domaines | Même logique : tu n'opères rien, tu ne possèdes ni le journal ni la politique de merge |
| [**OpenHands**](https://github.com/OpenHands/OpenHands) | Le harnais que j'utilise, qui propose aussi son *resolver* GitHub (issue labellisée → PR) et un cloud | Le resolver fait l'intake et la PR ; l'identité par run, l'egress, le gate et la room sont à construire autour |
| [**Optio**](https://github.com/jonwiggins/optio) | Orchestrateur self-hosted : tickets → agents dans des pods Kubernetes → PR, reprise sur échec CI | Le plus proche dans l'esprit ; un pod par dépôt plutôt qu'un sandbox par run, et pas de gate statistique par classe |
| **StrongDM** | La *dark factory* : aucun humain n'écrit ni ne relit, scénarios hors dépôt, clones de SaaS tiers | L'autre bout du spectre, et un budget d'un autre ordre |

Les produits gagnent sur le **temps jusqu'à la première PR**. Ce que cet assemblage apporte, c'est la propriété de la **frontière de confiance** et de la **piste d'audit** : je peux changer une règle d'egress, relier n'importe quel verdict à un run, et garder le journal, sans dépendre d'un service externe. Ce n'est pas une victoire universelle ; c'est un arbitrage, et il demande un cluster et de l'exploitation.

## 💭 Dernières remarques

Soyons clairs : c'est un chantier d'un mois, sur un seul dépôt et un seul cloud, et le cluster de référence est démonté entre deux sessions pour des raisons de coût. Voilà ce que j'en retiens, et ce qui manque.

* **Le modèle est la partie bon marché.** Le remplacer tient dans un alias du gateway. Faire qu'un run agisse sous une identité courte, sans secret durable ni sortie réseau implicite, et laisse une preuve consultable : c'est ça qui a structuré la plateforme, et c'est transposable à n'importe quel harnais.

* **L'autonomie se gagne par classe, et se perd par classe.** Un correctif de liens et une mise à jour de dépendance ne présentent ni le même risque ni la même preuve ; ils ne méritent pas le même droit. Le gate en shadow, le tally et le disjoncteur transforment une opinion (« ça marche bien ») en décision (« 29 merges propres, borne sous 10 % »). Aujourd'hui, `dead-links`, `deps-patch` et `deps-minor` accumulent leurs décisions shadow ; `deps-minor` ne pourra d'ailleurs **jamais** passer live, par construction.

* **Ce qui n'est pas encore là.** Aucun classifieur de complexité : chaque tâche est au tier `standard`. Le *scoper*, le *splitter*, le *tester test-first* et le moniteur post-merge sont livrés mais derrière des interrupteurs par classe, allumés sur une session et éteints ensuite. Les budgets par personne comptent sans refuser. Les rooms n'émettent pas encore de spans OpenTelemetry. Et le token GitHub est lié au dépôt et au rôle, pas au run : n'importe quel run implementer peut pousser sur n'importe quelle branche `agent/**` — le gate le rattrape par le trailer et l'identité de push, mais ce n'est pas une isolation.

* **La dette de compréhension reste le vrai risque.** Les rooms gardent la trace de *comment* un changement a été produit, et le fait que ce soit moi qui écrive l'issue et qui merge garde la décision humaine. Mais quatre mois, c'est aussi l'horizon de la factory de Dex Horthy. Je verrai dans quelques mois si relire des PR d'agents sur un dépôt que je connais bien suffit à ne pas décrocher.

La suite est écrite : un **second dépôt** pour éprouver la portabilité (le verrou de propriété, les politiques de confiance, les classes), quelques semaines de shadow pour voir une classe repasser live **statistiquement**, et un vrai bake-off de modèles open-weight sur les tiers. Tout est ouvert — [`maku`](https://github.com/Smana/maku), [`ai-native-ref`](https://github.com/Smana/ai-native-ref), [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — et les [issues](https://github.com/Smana/maku/issues) sont le meilleur endroit pour me dire où ça casse chez toi.

## 🔖 Références

### Dépôts et démo

* [`maku`](https://github.com/Smana/maku) — le produit : factory, rooms, merge gate, CLI · [démo publique d'une room](https://blog.ogenki.io/maku/)
* [`ai-native-ref`](https://github.com/Smana/ai-native-ref) — la plateforme IA déployable · [documentation](https://airef.ogenki.io)
* [`cloud-native-ref`](https://github.com/Smana/cloud-native-ref) — le cluster hôte · [documentation](https://cnref.ogenki.io)
* [RunLore](https://runlore.io) · [l'article de la série observabilité](/fr/post/series/observability/runlore/)

### Composants

* [Kubernetes agent-sandbox](https://agent-sandbox.sigs.k8s.io/) · [gVisor](https://gvisor.dev/docs/) · [Kueue](https://kueue.sigs.k8s.io/)
* [agentgateway](https://agentgateway.dev/) · [octo-sts](https://github.com/octo-sts/app) · [policy-bot](https://github.com/palantir/policy-bot)
* [OpenHands](https://github.com/OpenHands/OpenHands) · [Cilium — politiques DNS et FQDN](https://docs.cilium.io/en/stable/security/policy/language/#dns-based)

### Software factories

* [Addy Osmani — Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/)
* [Dan Shapiro — The five levels, from spicy autocomplete to the software factory](https://www.danshapiro.com/blog/2026/01/the-five-levels-from-spicy-autocomplete-to-the-software-factory/)
* [Simon Willison — la software factory de StrongDM](https://simonwillison.net/2026/Feb/7/software-factory/)
* [Firecrawl — How to build an AI software factory](https://www.firecrawl.dev/blog/ai-software-factory)
* [Agent Baseline — The Secure Software Factory, architecture de référence](https://agentbaseline.org/software-factory.pdf)
* [awesome-software-factories](https://github.com/varun1505/awesome-software-factories) — le paysage, régulièrement mis à jour

### Sécurité des agents

* [Simon Willison — The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
* [Anthropic — How we contain Claude](https://www.anthropic.com/research/how-we-contain-claude) · [Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
* [Configurer l'accès Internet du Copilot coding agent](https://github.blog/changelog/2025-07-15-configure-internet-access-for-copilot-coding-agent)
