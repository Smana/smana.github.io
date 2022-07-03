+++
author = "Smaine Kahlouch"
title = "`Gitops` my infrastructure!"
date = "2022-07-15"
summary = "Manage my platform, applications and infrastructure with **Flux**"
featureImage = "devflux.png"
featured = true
codeMaxLines = 20
usePageBundles = true
toc = true
tags = [
    "gitops",
    "devxp"
]
thumbnail= "flux.png"
+++

:construction_worker: <span style="color:red">**Work in progess for this section**</span>

In a [previous article](https://blog.ogenki.io/post/crossplane_k3d/), we've seen how to use **Crossplane** so that we can manage cloud resources in the same way as our applications. :heart: YAML!
There were several steps and command lines in order to get everything working and reach our target to provision a dev Kubernetes cluster.

Here we'll achieve exactly the same thing but we'll do that in the **GitOps** way.
According to the [OpenGitOps](https://opengitops.dev/) working group there are 4 GitOps principles:

* The desired state of our system must be expressed **declaratively**.
* This state must be stored in **versioning** system.
* Changes are pulled and **applied automatically** in the target platform whenever the desired state changes.
* If, for any reason, the current state is modified, it will be **automatically reconciled** with the desired state.

There are several GitOps engine options. The most famous ones are [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Flux](https://fluxcd.io/). We won't compare them here. I chose Flux because I like its composable architecture with different controllers, each one handling a core Flux feature.

![flux_components](flux_components.png)

### :bullseye: Our target

Here we want to declare our desired infrastructure components only by adding **git** changes.
By the end of this article you'll get a GKE cluster provisionned using a local Crossplane instance.
We'll discover [**Flux**](https://fluxcd.io/) basics and how to use it in order to build a GitOps CD workflow.

<br>

### :open_file_folder: Init Flux configuration git repository

First we're going to create our initial directory tree in order to have an efficient Flux configuration structure.

```console
mkdir -vp ~/sources/devflux/{infrastructure,security}/base

cd ~/sources/devflux/

git init .
Initialized empty Git repository in /home/smana/sources/devflux/.git/

gh repo create devflux --public --source .
✓ Created repository Smana/devflux on GitHub
✓ Added remote git@github.com:Smana/devflux.git

echo "# GitOps with Flux" > README.md

git add .
git commit -m "Initial commit"

git push origin main
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 892 bytes | 892.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:Smana/devflux.git
 * [new branch]      main -> main

```

<br>

### :ballot_box_with_check: Requirements

First of all we need to install a few tools using [asdf](https://blog.ogenki.io/post/asdf/asdf/)

Create a local file
.tool-versions

```console
cd ~/sources/devflux/

cat > .tool-versions <<EOF
github-cli 2.13.0
flux2 0.31.3
k3d 5.4.1
kubeseal 0.18.0
EOF
```

```console
for PLUGIN in $(cat .tool-versions | awk '{print $1}'); do asdf plugin-add $PLUGIN; done

asdf install
Downloading ... 100.0%
Copying Binary
...
```

Check that all the required tools are actually installed.

```console
asdf current
flux2           0.31.3          /home/smana/sources/devflux/.tool-versions
github-cli      2.13.0          /home/smana/sources/devflux/.tool-versions
k3d             5.4.1           /home/smana/sources/devflux/.tool-versions
kubeseal        0.18.0          /home/smana/sources/devflux/.tool-versions
```

<br>

### :gear: Bootstrap flux

```console
source <(flux completion bash)
```

```console
kubectl config current-context
k3d-crossplane


export GITHUB_TOKEN=ghp_xxxxxx
flux bootstrap github --owner=Smana --repository=devflux --personal --path=clusters/k3d-crossplane
► cloning branch "main" from Git repository "https://github.com/Smana/devflux.git"
...
✔ configured deploy key "flux-system-main-flux-system-./clusters/k3d-crossplane" for "https://github.com/Smana/devflux"
...
✔ all components are healthy


kubectl get po -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-5985c795f8-gs2pc           1/1     Running   0          86s
notification-controller-6b7d7485fc-lzlpg   1/1     Running   0          86s
kustomize-controller-6d4669f847-9x844      1/1     Running   0          86s
source-controller-5fb4888d8f-wgcqv         1/1     Running   0          86s

flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/33ebef1    False           True    Applied revision: main/33ebef1

git branch --set-upstream-to=origin/main main
branch 'main' set up to track 'origin/main'.

git pull
Updating e0cf2ec..f7ebe62
Fast-forward
 clusters/k3d-crossplane/flux-system/gotk-components.yaml | 5566 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 clusters/k3d-crossplane/flux-system/gotk-sync.yaml       |   27 +
 clusters/k3d-crossplane/flux-system/kustomization.yaml   |    5 +
 3 files changed, 5598 insertions(+)
 create mode 100644 clusters/k3d-crossplane/flux-system/gotk-components.yaml
 create mode 100644 clusters/k3d-crossplane/flux-system/gotk-sync.yaml
 create mode 100644 clusters/k3d-crossplane/flux-system/kustomization.yaml

```

### :closed_lock_with_key: SealedSecrets


```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: kube-system
spec:
  releaseName: sealed-secrets
  chart:
    spec:
      chart: sealed-secrets
      sourceRef:
        kind: HelmRepository
        name: sealed-secrets
        namespace: flux-system
      version: "2.2.0"
  interval: 10m0s
  install:
    remediation:
      retries: 3
  values:
```

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 30m
  url: https://bitnami-labs.github.io/sealed-secrets
```


### :hammer_and_wrench: Deploy and configure Crossplane