
# Build your first chart

## Create a simple webserver chart (nginx)

In order to get familiar with a typical chart we will create a simple webserver chart.

```console
$ helm create web
Creating web
```

The above command will create a chart directory named **web**

| web/        |                      |                                                                                                                    |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------------------ |
| charts      |                      | directory that contains the subcharts                                                                              |
| Chart.yaml  |                      | metadatas (author, version, description), dependencies and  more                                                   |
| templates   |                      | contains all the templates basically kubernetes resources in the form of templated yaml files. (go template)       |
|             | deployment.yaml      |                                                                                                                    |
|             | helpers.tpl          | helpers, functions that can be used from the templates.                                                            |
|             | hpa.yaml             |                                                                                                                    |
|             | ingress.yaml         |                                                                                                                    |
|             | NOTES.txt            | This file is used to print information after a release has been successfully installed.                            |
|             | serviceaccount.yaml  |                                                                                                                    |
|             | service.yaml         |                                                                                                                    |
|             | tests                | contains a job that will run a command to check the application after it has been installed.                       |
|             | test-connection.yaml |                                                                                                                    |
| values.yaml |                      | Maybe the most important file. We’ll play with the values to define how the kubernetes resources will be rendered. |

## Testing the chart

Here’s a combo if you want to check properly your chart before actually deploying it:

**template + lint + kubeval + test**

### Golang errors

When you add templating changes, you should run the command `helm template --debug <chart_dir>`

```console
$ helm template --debug web
install.go:173: [debug] Original chart version: ""
install.go:190: [debug] CHART PATH: /tmp/web

Error: parse error at (web/templates/_helpers.tpl:73): unexpected EOF
helm.go:81: [debug] parse error at (web/templates/_helpers.tpl:73): unexpected EOF
```

Read carefully if there are error messages. Always use the option **--debug** to see the template rendering.

### Chart linting

The command `helm lint <chart_dir>` verifies that the chart is well-formed.

```console
$ helm lint web/
==> Linting web/
[ERROR] Chart.yaml: apiVersion 'v3' is not valid. The value must be either "v1" or "v2"
[INFO] Chart.yaml: icon is recommended
[ERROR] Chart.yaml: chart type is not valid in apiVersion 'v3'. It is valid in apiVersion 'v2'

Error: 1 chart(s) linted, 1 chart(s) failed
```

### Validate Kubernetes resources

In order to validate that the rendered kubernetes objects are well-formed we’ll make use of a tool named [kubeval](https://kubeval.instrumenta.dev/).

This is even easier by using the Helm plugin.

Install the plugin:

```console
$ helm plugin install https://github.com/instrumenta/helm-kubeval
Installing helm-kubeval v0.13.0 ...
helm-kubeval 0.13.0 is installed.
```

Then check the chart as follows

```console
$ helm kubeval web
The file web/templates/serviceaccount.yaml contains a valid ServiceAccount
The file web/templates/secret.yaml contains a valid Secret
...
```

Now you can safely install the chart

```console
$ helm upgrade --install web web
Release "web" has been upgraded. Happy Helming!
NAME: web
LAST DEPLOYED: Mon Feb 15 18:22:23 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

### Check that the application works as expected

This is a good practice to add [tests](https://helm.sh/docs/topics/chart_tests/) under the directory template/tests.

Basically, this is achieved with a job that you can call when the release is already installed (just after)

It returns a code 0 if the command succeeds.

In the chart we’ve already generated there’s a job that checks the webserver availability.

Check that the release is already installed

```console
$ helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
web     default         1               2021-02-15 15:11:09.036602795 +0100 CET deployed        web-0.1.0       1.16.0
```

```console
$ helm test web
NAME: web
LAST DEPLOYED: Mon Feb 15 15:11:09 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     web-test-connection
Last Started:   Mon Feb 15 16:55:17 2021
Last Completed: Mon Feb 15 16:55:19 2021
Phase:          Succeeded
```

## Dependencies

Sometimes, the application requires another component to work (caching, database, persistence …).

This dependency system has to be used with caution because this is generally recommended to manage the applications lifecycles independently from each other.

Let’s say that our webserver need to store the information related to the sessions in a Redis server.

We’ll add a redis server to our web application by declaring the dependency in the file chart.yaml.

```yaml
dependencies:
  - name: redis
    version: "12.6.4"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

As you may have noticed, this dependency will be pulled only if the **condition** redis.enabled is True.

So we need to change our **values.yaml** accordingly:

```yaml
redis:
  enabled: True
  master:
    persistence:
      enabled: False
```

Check all the available values for this chart [here](https://artifacthub.io/packages/helm/bitnami/redis).

Whenever you add a dependency and you’re using a local chart (on your laptop), you must run the following command to **pull** it

```console
$ helm dep update web
Hang tight while we grab the latest from your chart repositories...
….
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading redis from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
```

The dependencies are stored in the directory **charts**.

After [testing](#testing-the-chart) your changes you can install the release with the command

`helm upgrade --install <release_name> <chart_dir>`

```console
$ helm upgrade --install web web
Release "web" has been upgraded. Happy Helming!
NAME: web
LAST DEPLOYED: Mon Feb 15 18:22:23 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
```

You can notice that your webserver has been successfully installed along with a HA Redis cluster

```console
$ kubectl get po
NAME                   READY   STATUS      RESTARTS   AGE
web-74bf5c6c66-fjsmb   1/1     Running     0          3h14m
web-test-connection    0/1     Completed   0          90m
web-redis-master-0     1/1     Running     0          3m14s
web-redis-slave-0      1/1     Running     0          3m14s
web-redis-slave-1      1/1     Running     0          2m42s
```

## Hooks

Helm comes with a hook system that allows it to run jobs at given times of the lifecycle.

The description is crystal clear in [the documentation](https://helm.sh/docs/topics/charts_hooks/) and you’ll have the opportunity to add one later on during this workshop.

## Mastering the Golang template

The main challenge when you start using Helm is to learn all the tips and tricks of the **Golang template**

The official Helm [documentation](https://helm.sh/docs/chart_template_guide/) is very useful for that.

Your best friends when you write Helm templates are the **[Sprig functions](http://masterminds.github.io/sprig/)**, you should definitely add this to your bookmarks.

Furthermore, even if it has been deprecated, you should clone/fork the original [stable chart repository](https://github.com/helm/charts). Indeed it has a wide range of **examples**.

Note that most of the time, if you want to keep the kubernetes manifests readable, you would put most of the code in what we call helpers files. There’s often at least one named **_helpers.tpl.**

**Note**: Even if you can do pretty advanced things with this templating language, you shouldn’t overuse it in order to keep the kubernetes resources readable and the chart maintainable.
