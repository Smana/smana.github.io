# Environment and ecosystem

Helm’s configuration is stored in the environment variable `$HELM_CONFIG_HOME` , by default `$HOME/.config/helm`

All the environment variables are described in the documentation [there](https://helm.sh/docs/helm/helm/).

Here are a few tools (with a wide adoption) that will add capabilities to Helm.

## Plugins

There are several [plugins](https://helm.sh/docs/community/related/) in order to extend Helm’s features.

Some of them are really useful (**kubeval**, **diff**, **secrets**).

## Helmfile

Helmfile is a really useful tool that allows you to declare the state of the releases on your cluster.

It helps keeping a **central view** of the releases deployed on a given cluster.

It automatically configures repositories, pulls dependencies and it is very helpful to build CI/CD workflows.

Of course it uses Helm under the hood and a few modules/plugins such as secrets decryption, helm diff

These steps are very basic, you should have a look at the documentation for further details.

Install the **helmdiff** plugin (used by helmfile)

```console
$ helm plugin install https://github.com/databus23/helm-diff
```

We’ll make use of some examples provided by CloudPosse

```console
$ git clone git@github.com:cloudposse/helmfiles.git
cd helmfiles
```

Let’s say we want to install the kubernetes dashboard and the [reloader](https://github.com/stakater/Reloader) tool.

```console
$ cat > releases/kubernetes-dashboard/dev.yaml <<EOF
installed: True
banner: "Workshop cluster"
EOF
```

Now we’ll create our main **helmfile.yaml** that describes all the releases we want to install

```console
$ cat > helmfile.yaml <<EOF
helmfiles:
  - path: "releases/kubernetes-dashboard/helmfile.yaml"
    values:
      - releases/kubernetes-dashboard/dev.yaml
  - path: "releases/reloader/helmfile.yaml"
    values:
      - installed: True
EOF
```

Now we can see what changes will be applied.

```console
$ helmfile diff
Adding repo stable https://charts.helm.sh/stable
"stable" has been added to your repositories

Comparing release=kubernetes-dashboard, chart=stable/kubernetes-dashboard
********************

        Release was not present in Helm.  Diff will show entire contents as new.

…
```

The command helm sync will install the releases

```console
$ helmfile sync
Adding repo stable https://charts.helm.sh/stable
"stable" has been added to your repositories

Affected releases are:
  kubernetes-dashboard (stable/kubernetes-dashboard) UPDATED

Upgrading release=kubernetes-dashboard, chart=stable/kubernetes-dashboard
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
...
```

You can list all the releases managed by the local helmfile.

```console
$ helmfile list
NAME                    NAMESPACE       ENABLED LABELS
kubernetes-dashboard    kube-system     true    chart:kubernetes-dashboard,component:monitoring,namespace:kube-system,repo:stable,vendor:kubernetes
reloader                reloader        true    chart:stakater/reloader,component:reloader,namespace:reloader,repo:stakater,vendor:stakater
```

Delete all the releases

```console
$ helmfile delete
Listing releases matching ^reloader$
reloader        reloader        1               2021-02-16 10:10:35.378800455 +0100 CET deployed        reloader-v0.0.68        v0.0.68

Deleting reloader
release "reloader" uninstalled
```
