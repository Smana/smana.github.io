+++
author = "Smaine Kahlouch"
title = "VictoriaMetrics: Configurer des alertes pertinentes"
date = "2024-10-12"
summary = "Alerter efficacement, VictoriaMetrics operator, slack"
featured = true
codeMaxLines = 21
usePageBundles = true
toc = true
series = [
  "observability"
]
tags = [
    "observability"
]
thumbnail= "thumbnail.png"
+++

{{% notice info "Mieux inspecter nos applications 👁️" %}}
Une fois que notre application est déployée, il est primordial de disposer d'indicateurs permettant d'identifier d'éventuels problèmes ainsi que suivre les évolutions de performance. Parmi ces éléments, les **métriques** et les **logs** jouent un rôle essentiel en fournissant des informations précieuses sur le fonctionnement de l'application. En complément, il est souvent utile de mettre en place un **tracing** détaillé pour suivre précisément toutes les actions réalisées par l'application.

Dans cette [série d'articles](http://localhost:1313/fr/series/observability/), nous allons explorer les différents aspects liés à la supervision applicative. L'objectif étant d'analyser en détail l'état de nos applications, afin d'améliorer leur **disponibilité** et leurs **performances**, tout en garantissant une expérience utilisateur optimale.
{{% /notice %}}

Lors d'un [précédent article](https://blog.ogenki.io/fr/post/series/observability/metrics/) nous avons vu comment collecter et visualiser des métriques. Celles-cis permettent d'analyser le comportement et les performances de nos applications, en revanche il est aussi primordial de configurer des **alertes** afin d'être notifié en cas d'anomalies sur notre plateforme.

## 🎯 Objectifs

* Comprendre les principales méthodologies standards pour un alerting efficace.
* Découvrir les langages PromQL et MetricsQL
* Configurer des alertes sur un cas d'usage d'example.
* Router ces alertes sur des canaux slackss

Nous allons voir dans cet article qu'il est très simple de positionner des seuils au delà desquels nous serions notifiés. Cependant faire en sorte que ces alertes soient pertinentes n'est pas toujours évident.

## 🔍 Qu'est ce qu'une bonne alerte?

<center><img src="chasseurs-chasseur.gif" width=300 alt=""></center>

La gestion des alertes consiste à configurer des notifications pour informer nos équipes des problèmes critiques au sein de notre système. L'objectif est d'identifier et de résoudre ces problèmes de manière **proactive**, avant qu'ils ne s'aggravent. Des alertes efficaces doivent être :

- Signaler des problèmes nécessitant une **intervention immédiate**.
- Etre déclenchées **au bon moment**: suffisamment tôt pour prévenir un impact sur les utilisateurs, sans toutefois être trop fréquentes au point de provoquer une fatigue liée aux alertes.
- Indiquer la **cause racine** ou la zone nécessitant une investigation. Il y a notamment un travail de priorisation d'indicateurs pertinents, avec un réel impact sur le métier ([SLIs](https://sre.google/sre-book/service-level-objectives/)).

Il est donc important de se focaliser sur un **nombre maitrisé** d'indicateurs à surveiller. Il existe pour cela des approches qui permettent de mettre en oeuvre une supervision efficace de nos systèmes.
Ici nous allons nous pencher sur 2 modèles d'alerte reconnus: Les **Core Web Vitals** et les **Golden Signals**.

### 🌐 Les "Core Web Vitals"

Les Core Web Vitals sont des métriques développées par Google pour évaluer l'**expérience utilisateur** sur les applications web. Ils mettent en évidence des indicateurs liés à la satisfaction des utilisateurs finaux et permettent de garantir que notre application offre de bonnes performances pour les utilisateurs réels. Ces métriques se concentrent sur trois aspects principaux :

<center><img src="core_web_vitals.png" width=800 alt="Core Web Vitals"></center>

- **Largest Contentful Paint** (LCP), *Temps de chargement de la page* : Le LCP mesure le temps nécessaire pour que le plus grand élément de contenu visible sur une page web (par exemple, une image, une vidéo ou un large bloc de texte) soit **entièrement rendu** dans la fenêtre d'affichage. Un bon LCP se situe en dessous de **2,5 secondes**.

- **Interaction to Next Paint** (INP), *Réactivité* : L'INP évalue la réactivité d'une page web en mesurant la **latence de toutes les interactions** utilisateur, telles que les clics, les taps et les entrées clavier, etc... Elle reflète le temps nécessaire pour qu'une page réagisse visuellement à une interaction, c'est-à-dire le délai avant que le navigateur affiche le prochain rendu après une action de l'utilisateur. un bon INP doit être inférieur à **200 millisecondes**

- **Cumulative Layout Shift** (CLS), *Stabilité visuelle* : Le CLS évalue la **stabilité visuelle** en quantifiant les décalages de mise en page inattendus sur une page, lorsque des éléments se déplacent pendant le chargement ou l'interaction. Un bon score CLS est inférieur ou égal à **0,1**.

{{% notice info "D'autres indicateurs" %}}
Il existe également d'autres indicateurs, que l'on ne détaillera pas ici, mais qui méritent d'être étudiés : **[FCP](https://web.dev/articles/fcp)** (First Contentful Paint), **[TTFB](https://web.dev/articles/ttfb)** (Time To First Byte), etc...
{{% /notice %}}

La performance d'un site web est considérée satisfaisante si elle atteint les seuils décrits ci-dessus au **75ᵉ percentile**, favorisant ainsi une bonne expérience utilisateur et, par conséquent, une meilleure rétention et un meilleur référencement ([SEO](https://en.wikipedia.org/wiki/Search_engine_optimization)).

Cependant, l'ajout d'alertes spécifiques sur ces métriques doit être mûrement réfléchi. Contrairement aux indicateurs opérationnels classiques, tels que la disponibilité ou le taux d'erreurs, qui reflètent directement la stabilité du système, Les *Web Vitals* dépendent de **nombreux facteurs externes**, comme les conditions réseau des utilisateurs ou leurs appareils, rendant les seuils plus complexes à surveiller efficacement.

Pour éviter une surcharge d'alertes inutiles, ces alertes doivent uniquemement cibler des **dégradations significatives**. Par exemple, une augmentation soudaine du **CLS** (stabilité visuelle) ou une détérioration continue du **LCP** (temps de chargement) sur plusieurs jours peuvent indiquer des problèmes importants nécessitant une intervention.

Enfin, ces alertes nécessitent des outils adaptés, comme le *RUM (Real User Monitoring)* pour les données réelles ou le *Synthetic Monitoring* pour des tests simulés, qui requièrent une solution spécifique non abordée dans cet article.

### ✨ Les "Golden Signals"

Les Golden Signals (ou signaux d'or) sont un ensemble de **quatre indicateurs clés**, largement utilisés dans le domaine de la supervision des systèmes et des applications, notamment avec des outils comme Prometheus. Ces signaux permettent de surveiller la santé et la performance des applications de manière efficace. Ils sont particulièrement appropriés dans le contexte d'une architecture distribuée:

* **La Latence** ⏳: Elle inclut à la fois le temps des requêtes réussies et le temps des requêtes échouées. La latence est cruciale car une augmentation du temps de réponse peut indiquer des problèmes de performance.

* **Le Trafic** 📶: Il peut être mesurée en termes de nombre de requêtes par seconde, de débit de données, ou d'autres métriques qui expriment la charge du système.

* **Les erreurs** ❌: Il s'agit du taux d'échec des requêtes ou des transactions. Cela peut inclure des erreurs d'application, des erreurs d'infrastructure ou toute situation où une requête ne s'est pas terminée correctement (par exemple, des réponses HTTP 5xx ou des requêtes rejetées).

* **La saturation** 📈: C'est une mesure de l'utilisation des ressources du système, comme le CPU, la mémoire ou la bande passante réseau. La saturation indique à quel point le système est proche de ses limites. Un système saturé peut entraîner des ralentissements ou des pannes.

Ces Golden Signals sont essentiels car ils permettent de **concentrer la surveillance sur les aspects critiques** qui peuvent rapidement affecter l'expérience utilisateur ou la performance globale du système. Avec Prometheus, ces signaux sont souvent surveillés via des métriques spécifiques pour déclencher des alertes lorsque certains seuils sont dépassés.



Alerting is not one-size-fits-all. By understanding and implementing the RED methodology, Golden Signals, and Web Vitals, you can ensure that your alerts are meaningful, actionable, and aligned with both system health and user satisfaction. Start by identifying your key metrics, set clear thresholds, and continually refine your approach for better results.

RED and Golden Signals have significant overlap, making both powerful frameworks for system monitoring. While RED excels in simplicity for microservices, Golden Signals' broader scope is better suited for large-scale distributed systems. Meanwhile, Web Vitals ensures a focus on user satisfaction. By integrating these patterns, you’ll build a balanced and effective alerting system that keeps your applications performant and your users happy.



* Grafana oncall
* AI runbooks


https://docs.victoriametrics.com/vmalert/
VMAlert, ruler evaluation des alertes en fonction de seuils.
Alertmanage notifications

## 💻 Exprimer des requêtes avec PromQL/MetricsQL

Prometheus, promQL basics. MetricsQL
recording rules vs ... rules


## 🛠️ Configure des alertes avec VictoriaMetrics Operator


## 💬 Envoyer des alertes sur slack


### Create a Slack app

https://api.slack.com/apps

* Add a new application "Alertmanager"
* Go to "Oauth & Permissions",
    * under "Scopes" add
    * `chat:write`
    * `chat:write.public` (Optional)
    * `channels:read` (Optional)
    * `groups:read` (Optional)

<center><video controls width="800" autoplay loop muted>
    <source src="slack-permissions.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video></center>

    * Generate a token under "OAuth tokens" and install it to your workspace
* Choose an Icon under "General"
* Test it with
```bash
curl -X POST -H 'Authorization: Bearer xoxb-...' \
-H 'Content-type: application/json' \
--data '{"channel":"#alerts","text":"This is a test message from my Slack app!"}' \
https://slack.com/api/chat.postMessage
```

You should get a message like

```json
{
  "ok": true,
  "channel": "C07HXJ4G59D",
  "ts": "1724571622.070789",
...
}
```

#### Configure Alertmanager

```yaml
    alertmanager:
      enabled: true

      spec:
        secrets:
          - "victoria-metrics-k8s-stack-alertmanager-slack-app"

      config:
        global:
          slack_api_url: "https://slack.com/api/chat.postMessage"
          http_config:
            authorization:
              credentials_file: /etc/vm/secrets/victoria-metrics-k8s-stack-alertmanager-slack-app/token

```



<center><img src="grafana-alerts.png" width=1100 alt=""></center>



```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  labels:
    app.kubernetes.io/name: hubble
  name: hubble
  namespace: kube-system
spec:
  endpoints:
    - honorLabels: true
      interval: 10s
      path: /metrics
      port: hubble-metrics
      relabelConfigs:
        - action: replace
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: node
      scheme: http
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      k8s-app: hubble
```


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


https://web.dev/articles/vitals
