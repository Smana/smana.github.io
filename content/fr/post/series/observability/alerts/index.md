+++
author = "Smaine Kahlouch"
title = "`VictoriaMetrics` : Des alertes efficaces, de la théorie à la pratique 🛠️"
date = "2024-10-12"
summary = "Des `Core Web Vitals` aux `Golden Signals`, en passant par la configuration de notifications Slack, découvrez comment mettre en place des alertes efficaces avec VictoriaMetrics."
featured = true
codeMaxLines = 21
usePageBundles = true
toc = true
series = [
  "observability"
]
tags = [
    "observability",
    "monitoring",
    "alerting"
]
thumbnail= "thumbnail.png"
+++

{{% notice info "Mieux inspecter nos applications 👁️" %}}
Une fois que notre application est déployée, il est primordial de disposer d'indicateurs permettant d'identifier d'éventuels problèmes ainsi que suivre les évolutions de performance. Parmi ces éléments, les **métriques** et les **logs** jouent un rôle essentiel en fournissant des informations précieuses sur le fonctionnement de l'application. En complément, il est souvent utile de mettre en place un **tracing** détaillé pour suivre précisément toutes les actions réalisées par l'application.

Dans cette [série d'articles](http://localhost:1313/fr/series/observability/), nous allons explorer les différents aspects liés à la supervision applicative. L'objectif étant d'analyser en détail l'état de nos applications, afin d'améliorer leur **disponibilité** et leurs **performances**, tout en garantissant une expérience utilisateur optimale.
{{% /notice %}}

Lors d'un [précédent article](https://blog.ogenki.io/fr/post/series/observability/metrics/), nous avons vu comment collecter et visualiser des métriques. Celles-ci permettent d'analyser le comportement et les performances de nos applications. Il est tout aussi primordial de configurer des **alertes** afin d'être notifié en cas d'anomalies sur notre plateforme.

## 🎯 Objectifs

* 📊 Comprendre les approches standards pour définir des alertes efficaces : Les "**Core Web Vitals**" et les "**Golden Signals**"
* 🔍 Découvrir les langages **PromQL** et **MetricsQL** pour l'écriture de règles d'alerte
* ⚙️ Configurer des alertes de façon déclarative avec **VictoriaMetrics Operator**
* 📱 **Router ces alertes** vers différents canaux Slack

## 📋 Prérequis

La suite de cet article suppose que vous avez déjà :

* Une instance VictoriaMetrics fonctionnelle
* Un cluster Kubernetes configuré
* Un accès à un workspace Slack pour les notifications
* Les permissions nécessaires pour configurer les alertes

La mise en place d'alertes pertinentes est un élément crucial de toute stratégie d'observabilité. Cependant, définir des seuils appropriés et éviter la fatigue liée aux alertes nécessite une approche réfléchie et méthodique.


Nous allons voir dans cet article qu'il est très simple de positionner des seuils au delà desquels nous serions notifiés. Cependant faire en sorte que ces alertes soient **pertinentes** n'est pas toujours évident.

## 🔍 Qu'est ce qu'une bonne alerte?

<center><img src="chasseurs-chasseur.gif" width=300 alt=""></center>

Une alerte correctement configurée permet d'identifier et de résoudre les problèmes au sein de notre système de manière **proactive**, avant qu'ils ne s'aggravent. Des alertes efficaces doivent:

- Signaler des problèmes nécessitant une **intervention immédiate**.
- Etre déclenchées **au bon moment**: suffisamment tôt pour prévenir un impact sur les utilisateurs, sans toutefois être trop fréquentes au point de provoquer une fatigue liée aux alertes.
- Indiquer la **cause racine** ou la zone nécessitant une investigation. Pour ce faire il est recommandé d'effectuer une analyse permettant la priorisation d'indicateurs pertinents, avec un réel impact sur le métier ([SLIs](https://sre.google/sre-book/service-level-objectives/)).

Il est donc important de se focaliser sur un **nombre maitrisé** d'indicateurs à surveiller. Il existe pour cela des approches qui permettent de mettre en oeuvre une supervision efficace de nos systèmes.
Ici nous allons nous pencher sur 2 modèles d'alerte reconnus: Les **Core Web Vitals** et les **Golden Signals**.

### 🌐 Les "Core Web Vitals"

Les Core Web Vitals sont des métriques développées par Google pour évaluer l'**expérience utilisateur** sur les applications web. Ils mettent en évidence des indicateurs liés à la satisfaction des utilisateurs finaux et permettent de garantir que notre application offre de bonnes performances pour les utilisateurs réels. Ces métriques se concentrent sur trois aspects principaux :

<center><img src="core_web_vitals.png" width=800 alt="Core Web Vitals"></center>

- **Largest Contentful Paint** (LCP), *Temps de chargement de la page* : Le LCP mesure le temps nécessaire pour que le plus grand élément de contenu visible sur une page web (par exemple, une image, une vidéo ou un large bloc de texte) soit **entièrement rendu** dans la fenêtre d'affichage. Un bon LCP se situe en dessous de **2,5 secondes**.

- **Interaction to Next Paint** (INP), *Réactivité* : L'INP évalue la réactivité d'une page web en mesurant la **latence de toutes les interactions** utilisateur, telles que les clics, les taps et les entrées clavier, etc... Elle reflète le temps nécessaire pour qu'une page réagisse visuellement à une interaction, c'est-à-dire le délai avant que le navigateur affiche le prochain rendu après une action de l'utilisateur. un bon INP doit être inférieur à **200 millisecondes**

- **Cumulative Layout Shift** (CLS), *Stabilité visuelle* : Le CLS évalue la **stabilité visuelle** en quantifiant les décalages de mise en page inattendus sur une page, lorsque des éléments se déplacent pendant le chargement ou l'interaction. Un bon score CLS est inférieur ou égal à **0,1**.

La performance d'un site web est considérée satisfaisante si elle atteint les seuils décrits ci-dessus au **75ᵉ percentile**, favorisant ainsi une bonne expérience utilisateur et, par conséquent, une meilleure rétention et un meilleur référencement ([SEO](https://en.wikipedia.org/wiki/Search_engine_optimization)).

Cependant, l'ajout d'alertes spécifiques sur ces métriques doit être mûrement réfléchi. Contrairement aux indicateurs opérationnels classiques, tels que la disponibilité ou le taux d'erreurs, qui reflètent directement la stabilité du système, Les *Web Vitals* dépendent de **nombreux facteurs externes**, comme les conditions réseau des utilisateurs ou leurs appareils, rendant les seuils plus complexes à surveiller efficacement.

Pour éviter une surcharge d'alertes inutiles, ces alertes doivent uniquemement cibler des **dégradations significatives**. Par exemple, une augmentation soudaine du **CLS** (stabilité visuelle) ou une détérioration continue du **LCP** (temps de chargement) sur plusieurs jours peuvent indiquer des problèmes importants nécessitant une intervention.

Enfin, ces alertes nécessitent des outils adaptés, comme le *RUM (Real User Monitoring)* pour les données réelles ou le *Synthetic Monitoring* pour des tests simulés, qui requièrent une solution spécifique non abordée dans cet article.

### ✨ Les "Golden Signals"

<center><img src="golden_signals.png" width=300 alt="Golden Signals"></center>

Les _Golden Signals_ sont un ensemble de **quatre indicateurs clés**, largement utilisés dans le domaine de la supervision des systèmes et des applications, notamment avec des outils comme Prometheus. Ces signaux permettent de surveiller la santé et la performance des applications de manière efficace. Ils sont particulièrement appropriés dans le contexte d'une architecture distribuée:

* **La Latence** ⏳: Elle inclut à la fois le temps des requêtes réussies et le temps des requêtes échouées. La latence est cruciale car une augmentation du temps de réponse peut indiquer des problèmes de performance.

* **Le Trafic** 📶: Il peut être mesurée en termes de nombre de requêtes par seconde, de débit de données, ou d'autres métriques qui expriment la charge du système.

* **Les erreurs** ❌: Il s'agit du taux d'échec des requêtes ou des transactions. Cela peut inclure des erreurs d'application, des erreurs d'infrastructure ou toute situation où une requête ne s'est pas terminée correctement (par exemple, des réponses HTTP 5xx ou des requêtes rejetées).

* **La saturation** 📈: C'est une mesure de l'utilisation des ressources du système, comme le CPU, la mémoire ou la bande passante réseau. La saturation indique à quel point le système est proche de ses limites. Un système saturé peut entraîner des ralentissements ou des pannes.

Ces Golden Signals sont essentiels car ils permettent de **concentrer la surveillance sur les aspects critiques** qui peuvent rapidement affecter l'expérience utilisateur ou la performance globale du système. Avec Prometheus, ces signaux sont souvent surveillés via des métriques spécifiques pour déclencher des alertes lorsque certains seuils sont dépassés.

Vous l'aurez compris: Definir des alertes ça se réfléchit! Maintenant entrons dans le concret et voyons **comment définir des seuils à partir de nos métriques**.

{{% notice info "D'autres méthodes et indicateurs" %}}
J'ai évoqué ici 2 méthodologies qui, je trouve, sont un bon point de départ pour ajuster au mieux notre système d'alerting. Ceci-dit il en existe d'autres, chacune avec leurs spécificités. On peut ainsi citer [USE](https://www.brendangregg.com/usemethod.html) ou [RED](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/).

De même, au-delà des Core Web Vitals présentés plus haut, d'autres métriques web comme **[FCP](https://web.dev/articles/fcp)** (First Contentful Paint) ou **[TTFB](https://web.dev/articles/ttfb)** (Time To First Byte) peuvent s'avérer utiles selon vos besoins spécifiques.

Le choix des métriques à surveiller dépendra de votre contexte et de vos objectifs. L'essentiel est de garder à l'esprit qu'une bonne stratégie d'alerting repose sur un ensemble ciblé d'indicateurs pertinents 🎯
{{% /notice %}}

## 💻 Exprimer des requêtes avec PromQL/MetricsQL

Les métriques collectées avec Prometheus peuvent être requetées avec un langage spécifique appelé `PromQL` (Prometheus Query Language). Ce langage permet d'extraire des données de supervision, d'effectuer des **calculs**, d'**agréger** les résultats, d'appliquer des **filtres**, mais aussi de configurer des **alertes**.

(ℹ️ Se référer au [précédent article](https://blog.ogenki.io/fr/post/series/observability/metrics/#-quest-ce-quune-m%C3%A9trique) pour comprendre ce que l'on entend par métrique.)

PromQL est un langage puissant dont voici quelques exemples simples appliqués aux métriques exposées par un serveur web Nginx :

* Nombre total de requêtes traitées (`nginx_http_requests_total`) :
  ```promql
  nginx_http_requests_total
  ```

* Nombre moyen de requêtes par seconde sur une fenêtre de 5 minutes.
  ```promql
  rate(nginx_http_requests_total[5m])
  ```

* Nombre de requêtes HTTP par seconde retournant un code d'erreur 5xx sur les 5 dernières minutes.
  ```promql
  rate(nginx_http_requests_total{status=~"5.."}[5m])
  ```

* Nombre de requêtes par seconde, agrégé par pod et filtré sur le namespace "myns" sur les 5 dernières minutes.
  ```promql
  sum(rate(nginx_http_requests_total{namespace="myns"}[5m])) by (pod)
  ```

💡 Dans les exemples ci-dessus, nous avons mis en évidence deux _Golden Signals_ : le trafic 📶 et les erreurs ❌.

`MetricsQL` est le langage utilisé avec VictoriaMetrics. Il se veut compatible avec PromQL avec de légères différences qui permettent de faciliter l'écriture de requêtes complexes.</br>
Il apporte aussi de nouvelles fonctions dont voici quelques exemples:

* `histogram(q)`: Cette fonction calcule un histogramme pour chaque groupe de points ayant le même horodatage, ce qui est utile pour visualiser un grand nombre de séries temporelles (timeseries) via un heatmap. </br>
  Pour créer un histogramme des requêtes HTTP
  ```promql
  histogram(rate(vm_http_requests_total[5m]))
  ```

* `quantiles("phiLabel", phi1, ..., phiN, q)`: Utilisé pour extraire plusieurs quantiles (ou percentiles) d'une métrique donnée . </br>
  Pour calculer les 50e, 90e et 99e percentiles du taux de requêtes HTTP
  ```promql
  quantiles("percentile", 0.5, 0.9, 0.99, rate(vm_http_requests_total[5m]))
  ```

Afin de pouvoir tester ses requêtes, vous pouvez utiliser la démo fournie par VictoriaMetrics: https://play.victoriametrics.com

<center><img src="vmplay.png" width=700 alt=""></center>

## 🛠️ Configurer des alertes avec VictoriaMetrics Operator

VictoriaMetrics propose deux composants essentiels pour la gestion des alertes :
- **VMAlert** : responsable de l'évaluation des règles d'alerte
- **AlertManager** : gère le routage et la distribution des notifications

### VMAlert : Le moteur d'évaluation des règles

VMAlert est le composant qui évalue en continu les règles d'alerte définies. Il supporte deux types de règles :

1. **Recording Rules** 📊
   - Pré-calculent des expressions PromQL complexes
   - Créent de nouvelles métriques (time series)
   - Optimisent les performances des dashboards
   - Exemple de recording rule :
   ```yaml
   groups:
     - name: recording_rules
       rules:
         - record: job:http_requests_total:rate5m
           expr: sum(rate(http_requests_total[5m])) by (job)
   ```

2. **Alerting Rules** 🚨
   - Définissent les conditions de déclenchement des alertes
   - Supportent des annotations pour enrichir les notifications
   - Permettent la classification par labels
   - Exemple d'alerte sur la latence :
   ```yaml
   groups:
     - name: latency_alerts
       rules:
         - alert: HighLatency
           expr: http_request_duration_seconds{quantile="0.9"} > 1
           for: 5m
           labels:
             severity: warning
           annotations:
             summary: "Latence élevée détectée"
             description: "La latence P90 dépasse 1s depuis 5 minutes"
             runbook_url: "https://wiki.example.com/runbooks/high-latency"
   ```

{{% notice tip "Des exemples concrets" %}}
<table>
  <tr>
        <td>
          <img src="repo_gift.png" style="width:80%;">
        </td>
        <td style="vertical-align:middle; padding-left:10px;" width="70%">

Le reste de cet article est issu d'un ensemble de configurations que vous pouvez retrouver dans le repository <strong><a href="https://github.com/Smana/cloud-native-ref">Cloud Native Ref</a></strong>.</br>
Il y est fait usage de nombreux opérateurs et notamment celui pour [VictoriaMetrics](https://github.com/VictoriaMetrics/operator).

L'ambition de ce projet est de pouvoir <strong>démarrer rapidement une plateforme complète</strong> qui applique les bonnes pratiques en terme d'automatisation, de supervision, de sécurité etc. </br>
Les commentaires et contributions sont les bienvenues 🙏
        </td>
  </tr>
</table>
{{% /notice %}}


### 💡 Déclarer une `VMRule`

Nous avons vu précédemment que VictoriaMetrics fournit un opérateur Kubernetes qui permet de gérer les différents composants de manière déclarative. Parmi les ressources personnalisées (Custom Resources) disponibles, la `VMRule` permet de définir des règles d'alertes et d'enregistrement (recording rules).

Si vous avez déjà utilisé l'[opérateur Prometheus](https://github.com/prometheus-operator/prometheus-operator), vous retrouverez une syntaxe très similaire car l'opérateur VictoriaMetrics est compatible avec les custom resources de Prometheus. (Ce qui nous permet de faciliter la migration 😉).

Prenons un exemple concret avec une `VMRule` qui surveille l'état de santé des ressources Flux :

[flux/observability/vmrule.yaml](https://github.com/Smana/cloud-native-ref/blob/main/flux/observability/vmrule.yaml)

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  labels:
    prometheus-instance: main
  name: flux-system
  namespace: flux-system
spec:
  groups:
    - name: flux-system
      rules:
        - alert: FluxReconciliationFailure
          annotations:
            message: Flux resource has been unhealthy for more than 5m
            description: "{{ $labels.kind }} {{ $labels.exported_namespace }}/{{ $labels.name }} reconciliation has been failing for more than ten minutes."
            runbook_url: "https://fluxcd.io/flux/cheatsheets/troubleshooting/"
            dashboard: "https://grafana.priv.${domain_name}/dashboards"
          expr: max(gotk_reconcile_condition{status="False",type="Ready"}) by (exported_namespace, name, kind) + on(exported_namespace, name, kind) (max(gotk_reconcile_condition{status="Deleted"}) by (exported_namespace, name, kind)) * 2 == 1
          for: 10m
          labels:
            severity: warning
```

Il est recommandé de suivre quelques **bonnes pratiques** pour donner le maximum de contexte afin d'identifier rapidement la cause racine.

1. **Nommage et Organisation** 📝
   - Utiliser des noms descriptifs pour les règles, comme `FluxReconciliationFailure`
   - Grouper les règles par composant Flux (ex: `flux-system`, `flux-controllers`)
   - Documenter les conditions de réconciliation dans les annotations

2. **Seuils et Durées** ⏱️
   - Ajuster la durée d'évaluation de l'alerte `for: 10m` pour éviter les faux positifs
   - Adapter les seuils selon le type de ressource supervisées.
   - Considérer des durées différentes selon l'environnement (prod/staging)

3. **Labels et Routing** 🏷️
   - Ajouter des labels pour le routage selon le contexte. Mon exemple n'est pas très poussé car il s'agit d'une configuration de démo. Mais nous pourrions très bien ajouter un label `team` afin de router vers la bonne équipe, ou avoir une politique de routage différente selon l'environnement.
     ```yaml
     labels:
       severity: [critical|warning|info]
       team: [sre|dev|ops]
       environment: [prod|staging|dev]
     ```

4. **L'importance des annotations** 📚

  Les annotations permettent d'ajouter divers informations sur le contexte de l'alerte
  - Une **description** claire du problème de réconciliation
  - Le lien vers le **runbook** de troubleshooting Flux
  - Le lien vers le **dashboard Grafana** dédié

5. **Expression PromQL Optimisée** 🔍
   ```yaml
   expr: |
     max(gotk_reconcile_condition{status="False",type="Ready"}) by (exported_namespace, name, kind)
     + on(exported_namespace, name, kind)
     (max(gotk_reconcile_condition{status="Deleted"}) by (exported_namespace, name, kind)) * 2 == 1
   ```
   Cette expression :
   - Surveille les conditions de réconciliation en échec
   - Prend en compte les ressources supprimées
   - Agrège par namespace, nom et type de ressource

## 💬 Intégration avec Slack

L'intégration avec Slack permet de recevoir les alertes directement dans vos canaux de communication. Voici comment la configurer de manière efficace.

### Configuration de l'Application Slack

1. **Création de l'Application** 🔧
   - Rendez-vous sur [https://api.slack.com/apps](https://api.slack.com/apps)
   - Cliquez sur "Create New App"
   - Choisissez "From scratch"
   - Nommez votre application (ex: "AlertManager")
   - Sélectionnez votre workspace

2. **Configuration des Permissions** 🔑
   Dans "OAuth & Permissions", ajoutez les scopes suivants :
   - `chat:write` (Requis)
   - `chat:write.public` (Pour poster dans les canaux publics)
   - `channels:read` (Pour lister les canaux)
   - `groups:read` (Pour les groupes privés)

3. **Installation et Token** 🎟️
   - Installez l'application dans votre workspace
   - Copiez le "Bot User OAuth Token" (commence par `xoxb-`)
   - Stockez ce token de manière sécurisée dans Kubernetes :

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: alertmanager-slack-token
     namespace: monitoring
   type: Opaque
   stringData:
     token: xoxb-your-token-here
   ```

### Configuration d'AlertManager pour Slack

1. **Configuration de Base** ⚙️
   ```yaml
   alertmanager:
     config:
       global:
         slack_api_url: "https://slack.com/api/chat.postMessage"
         resolve_timeout: 5m

       route:
         group_by: ['alertname', 'job', 'severity']
         group_wait: 30s
         group_interval: 5m
         repeat_interval: 4h
         receiver: 'slack-notifications'

       receivers:
         - name: 'slack-notifications'
           slack_configs:
           - channel: '#alerts'
             send_resolved: true
             icon_emoji: ':bell:'
             title: '{{ template "slack.title" . }}'
             text: '{{ template "slack.text" . }}'
   ```

2. **Templates Personnalisés** 📝
   ```yaml
   templates:
     - name: slack.title
       template: |
         [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
     - name: slack.text
       template: |
         {{ range .Alerts }}
         *Alert:* {{ .Labels.alertname }}
         *Description:* {{ .Annotations.description }}
         *Severity:* {{ .Labels.severity }}
         *Started:* {{ .StartsAt | since }}
         {{ if .Annotations.runbook }}*Runbook:* {{ .Annotations.runbook }}{{ end }}
         {{ end }}
   ```

<center><img src="alert.png" width=650 alt="Slack alert example"></center>

### 🎨 Personnalisation Avancée des Notifications

1. **Boutons d'Action** 🔘
   ```yaml
   slack_configs:
     - actions:
         - type: button
           text: "Voir le Runbook 📚"
           url: "{{ .CommonAnnotations.runbook_url }}"
         - type: button
           text: "Voir le Dashboard 📊"
           url: "{{ .CommonAnnotations.dashboard_url }}"
         - type: button
           text: "Silence 🔕"
           url: "{{ template "__alert_silence_link" . }}"
   ```

2. **Routage Intelligent** 🔀
   ```yaml
   route:
     routes:
       - match:
           severity: critical
         receiver: 'slack-critical'
         continue: true
       - match_re:
           service: ^(frontend|backend)$
         receiver: 'slack-apps'
   ```

## 🤖 Fonctionnalités Avancées

### Intégration avec Grafana OnCall

[Grafana OnCall](https://grafana.com/products/oncall/) permet d'améliorer la gestion des astreintes et des escalades d'incidents. Voici comment l'intégrer :

1. **Configuration de l'Intégration** 🔌
   ```yaml
   receivers:
     - name: 'grafana-oncall'
       webhook_configs:
         - url: 'http://oncall:8080/api/v1/alert'
           send_resolved: true
   ```

2. **Définition des Rotations** 📅
   - Configurez les équipes et les rotations dans Grafana OnCall
   - Associez les alertes aux équipes via les labels

### AI Runbooks 📚

Les AI Runbooks permettent d'automatiser la résolution des incidents en utilisant l'intelligence artificielle :

1. **Intégration avec un LLM** 🧠
   ```yaml
   annotations:
     runbook_ai: |
       {
         "model": "gpt-4",
         "context": "Application Java Spring Boot",
         "previous_incidents": "link_to_similar_incidents",
         "suggested_actions": [
           "Vérifier les logs applicatifs",
           "Analyser l'utilisation mémoire",
           "Redémarrer le service si nécessaire"
         ]
       }
   ```

2. **Automatisation des Actions** 🤖
   ```yaml
   - alert: HighMemoryUsage
     expr: container_memory_usage_bytes > 2e9
     for: 5m
     annotations:
       runbook_ai_action: |
         1. Collecter les dumps mémoire
         2. Analyser avec AI Memory Analyzer
         3. Suggérer des optimisations
   ```

### Métriques sur les Alertes 📊

Surveillez la qualité de vos alertes avec des métriques dédiées :

```yaml
- record: alert_quality_metrics
  expr: |
    sum(rate(alertmanager_notifications_total[24h])) by (integration)
    /
    sum(rate(alertmanager_notifications_failed_total[24h])) by (integration)
```


## 🔧 Troubleshooting et Maintenance

### Diagnostic des Problèmes Courants

1. **Alertes Non Déclenchées** 🤔
   - Vérifiez l'état de VMAlert :
     ```bash
     kubectl get pods -n monitoring -l app=vmalert
     kubectl logs -n monitoring -l app=vmalert
     ```
   - Validez vos expressions PromQL sur VictoriaMetrics UI
   - Contrôlez les timestamps des dernières métriques reçues

2. **Notifications Non Reçues** 📫
   - Vérifiez l'état d'AlertManager :
     ```bash
     kubectl get pods -n monitoring -l app=alertmanager
     ```
   - Consultez les logs pour les erreurs de connexion :
     ```bash
     kubectl logs -n monitoring -l app=alertmanager | grep "error"
     ```
   - Testez la connectivité Slack avec une alerte de test

3. **Faux Positifs Fréquents** ⚠️
   - Ajustez les seuils et les durées (`for`)
   - Utilisez des expressions plus robustes :
     ```yaml
     - alert: HighErrorRate
       expr: |
         (
           sum(rate(http_requests_total{status=~"5.."}[5m]))
           /
           sum(rate(http_requests_total[5m])) > 0.05
         )
       for: 5m
     ```
   - Implémentez des conditions de préqualification

### Maintenance et Bonnes Pratiques

1. **Revue Périodique** 📊
   - Analysez les métriques d'alertes mensuellement
   - Identifiez les alertes les plus fréquentes
   - Ajustez les seuils selon les retours d'expérience

2. **Documentation** 📝
   - Maintenez un catalogue d'alertes à jour
   - Documentez les procédures de résolution
   - Partagez les retours d'expérience

3. **Tests et Validation** ✅
   ```yaml
   - alert: TestAlert
     expr: vector(1)
     labels:
       severity: info
       type: test
     annotations:
       summary: "Alerte de test"
       description: "Cette alerte permet de valider la chaîne de notification"
   ```

## 🎯 Conclusion

La mise en place d'alertes efficaces avec VictoriaMetrics nécessite une approche méthodique et réfléchie. Nous avons vu comment :

1. **Définir des Alertes Pertinentes** 📋
   - Utiliser les Core Web Vitals pour la performance utilisateur
   - S'appuyer sur les Golden Signals pour la santé système
   - Éviter la fatigue d'alertes avec des seuils appropriés

2. **Configurer l'Infrastructure** ⚙️
   - Mettre en place VMAlert pour l'évaluation des règles
   - Configurer AlertManager pour la gestion des notifications
   - Intégrer Slack pour la communication d'équipe

3. **Optimiser et Maintenir** 🔄
   - Utiliser les recording rules pour les calculs complexes
   - Implémenter des templates personnalisés
   - Maintenir une documentation à jour

### Prochaines Étapes Suggérées

Pour aller plus loin dans votre implémentation :

1. **Automatisation** 🤖
   - Déploiement des règles via GitOps
   - Intégration avec des outils d'IA pour l'analyse
   - Automatisation des tests d'alertes

2. **Monitoring Avancé** 📈
   - Implémentation de SLOs basés sur les alertes
   - Corrélation avec les logs et le tracing
   - Dashboards dédiés au suivi des alertes

3. **Organisation** 👥
   - Définition des processus d'escalade
   - Formation des équipes
   - Revues post-incident systématiques

La supervision proactive via des alertes bien configurées est un élément clé de toute stratégie d'observabilité. En suivant les bonnes pratiques et en utilisant les outils appropriés, vous pouvez construire un système d'alerting robuste et efficace qui vous permettra d'identifier et de résoudre les problèmes avant qu'ils n'impactent vos utilisateurs.

{{% notice tip "Pour aller plus loin 🚀" %}}
Découvrez comment intégrer ces alertes avec d'autres composants de votre stack d'observabilité dans les prochains articles de cette série, notamment la corrélation avec les logs et le tracing distribué.
{{% /notice %}}


## 🔖 References

* https://web.dev/articles/vitals
* https://medium.com/@romanhavronenko/victoriametrics-promql-compliance-d4318203f51e
* https://victoriametrics.com/blog/alerting-recording-rules-alertmanager/


* Grafana oncall
* AI runbooks


https://docs.victoriametrics.com/vmalert/
VMAlert, ruler evaluation des alertes en fonction de seuils.
Alertmanage notifications
