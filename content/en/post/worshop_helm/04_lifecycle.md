# Application lifecycle

Apply a change, anything. For example we will add a label `stage: dev`. Edit the file templates/_helpers.tpl

```yaml
{{- define "web.labels" -}}
stage: "dev"
...
```

Deploy a new revision with the same command we ran previously

```console
$ helm upgrade --install web web
```

Now we can have a look to the changes we’ve made so far to the release

```console
$ helm history web
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Mon Feb 15 15:11:09 2021        superseded      web-0.1.0       1.16.0          Install complete
...
4               Mon Feb 15 21:15:25 2021        superseded      web-0.1.0       1.16.0          Upgrade complete
5               Mon Feb 15 21:21:21 2021        deployed        web-0.1.0       1.16.0          Upgrade complete
```

We can then check what would be the changes if we rollback to the previous revision

```console
$ helm diff rollback web 4
default, web, Deployment (apps) has changed:
  # Source: web/templates/deployment.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    labels:
-     stage: "dev"
      helm.sh/chart: web-0.1.0
...
```

Now that we’re sure we can safely rollback to the previous revision

```console
$ helm rollback web 4
Rollback was a success! Happy Helming!
```