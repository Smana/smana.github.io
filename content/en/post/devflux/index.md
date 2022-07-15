+++
author = "Smaine Kahlouch"
title = "`Gitops` my infrastructure!"
date = "2022-07-19"
summary = "Create and update cloud resources and applications through git with **Flux**"
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

In a [previous article](/post/crossplane_k3d/), we've seen how to use **Crossplane** so that we can manage cloud resources in the same way as our applications. :heart: YAML!
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

### :ballot_box_with_check: Requirements

#### :inbox_tray: Install required tools

First of all we need to install a few tools using [asdf](/post/asdf/asdf/)

Create a local file
.tool-versions

```console
cd ~/sources/devflux/

cat > .tool-versions <<EOF
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
k3d             5.4.1           /home/smana/sources/devflux/.tool-versions
kubeseal        0.18.0          /home/smana/sources/devflux/.tool-versions
```
<br>

#### :key: Create a Github personal access token

In this article the git repository is hosted in Github. In order to be able to use the `flux bootstrap` a personnal access token is required.

Please follow [this procedure](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).


<br>

### :rocket: Bootstrap flux

As we will often be using the `flux` CLI you may want to configure the bash|zsh completion
```console
source <(flux completion bash)
```

{{% notice warning Kubernetes context %}}
Here we consider that you already have a **local k3d instance**. If not you may want to either go through the whole [previous article](/post/crossplane_k3d) or just run the [local cluster creation](/post/crossplane_k3d#-create-the-local-k3d-cluster-for-crossplanes-control-plane).

Ensure that you're working in the right context

```console
kubectl config current-context
k3d-crossplane
```

{{% /notice %}}

Run the bootstrap command that will basically deploy all flux's components in the namespace flux-system. Here I'll create a repository named **devflux** using my personal Github account.

```console
export GITHUB_USER=Smana
export GITHUB_TOKEN=ghp_<REDACTED> # your personal access token
export GITHUB_REPO=devflux

flux bootstrap github --owner="${GITHUB_USER}" --repository="${GITHUB_REPO}" --personal --path=clusters/k3d-crossplane
► cloning branch "main" from Git repository "https://github.com/Smana/devflux.git"
...
✔ configured deploy key "flux-system-main-flux-system-./clusters/k3d-crossplane" for "https://github.com/Smana/devflux"
...
✔ all components are healthy
```

Check that all the pods are running properly and that the `kustomization` flux-system has been successfully reconciled.
```console
kubectl get po -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-5985c795f8-gs2pc           1/1     Running   0          86s
notification-controller-6b7d7485fc-lzlpg   1/1     Running   0          86s
kustomize-controller-6d4669f847-9x844      1/1     Running   0          86s
source-controller-5fb4888d8f-wgcqv         1/1     Running   0          86s

flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/33ebef1    False           True    Applied revision: main/33ebef1
```

{{% notice info Kustomization %}}
describe what is a kustomization and not confuse with the kustomize object
{{% /notice %}}

```console
git clone https://github.com/Smana/devflux.git
Cloning into 'devflux'...

cd devflux

git log -1
commit 2beb6aafea67f3386b50cbc706fb34575844040d (HEAD -> main, origin/main, origin/HEAD)
Author: Flux <>
Date:   Thu Jul 14 17:13:27 2022 +0200

    Add Flux sync manifests

ls clusters/k3d-crossplane/flux-system/
gotk-components.yaml  gotk-sync.yaml  kustomization.yaml

```
<br>

### :open_file_folder: Flux repository structure

There are [several options](https://fluxcd.io/docs/guides/repository-structure/) for organizing your resources in the Flux configuration repository. Here is a proposition for the sake of this article.

```console
tree -d -L 2
.
├── apps
│   ├── base
│   └── dev-cluster
├── clusters
│   ├── dev-cluster
│   └── k3d-crossplane
├── infrastructure
│   ├── base
│   ├── dev-cluster
│   └── k3d-crossplane
├── observability
│   ├── base
│   ├── dev-cluster
│   └── k3d-crossplane
└── security
    ├── base
    ├── dev-cluster
    └── k3d-crossplane
```


| Directory   | Description     | Example   |
| --------  | -------- | ------ |
| **/apps** | our applications | Here we'll deploy a demo application |
| **/infrastructure** | base infrastructure/network components | Crossplane as it will be used to provision cloud resources but we can also find CSI/CNI/EBS drivers... |
| **/observability** | All metrics/apm/logging tools | Prometheus of course, Opentelemetry ... |
| **/security** | Any component that enhance our security level | SealedSecrets (see below) |

Let's use this directory and begin to deploy applications.

{{% notice info resources %}}
All the files used for the upcoming steps are stored within this blog repository.
So you should clone and change the current directory:

```console
git clone https://github.com/Smana/smana.github.io.git

ls content/resources/devflux/
apps  clusters  infrastructure  observability  security
```

{{% /notice %}}

<br>

### :closed_lock_with_key: SealedSecrets

There are plenty of alternatives when it comes to secrets management in Kubernetes.
In order to securely store secrets in a github repository in the gitops way we'll make use of SealedSecrets.
It uses a custom resource definition named `SealedSecrets` in order to encrypt the Kubernetes secret at the client side then the controller is in charge of decrypting and generating the expected secret in the cluster.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: security
  namespace: flux-system
spec:
  prune: true
  interval: 4m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./security/k3d-crossplane
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v1beta1
      kind: HelmRelease
      name: sealed-secrets
      namespace: kube-system
```

We will deploy SealedSecrets using the [Helm chart](https://github.com/bitnami-labs/sealed-secrets#helm-chart). So we need to declare the source of this chart.

<span style="color:green">security/base/sealed-secrets/source.yaml</span>

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

<span style="color:green">security/base/sealed-secrets/helmrelease.yaml</span>

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
      version: "2.4.0"
  interval: 10m0s
  install:
    remediation:
      retries: 3
  values:
    fullnameOverride: sealed-secrets-controller
    resources:
      requests:
        cpu: 80m
        memory: 100Mi
```

```console
kustomize create --autodetect
```

<span style="color:green">security/base/sealed-secrets/kustomization.yaml</span>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- helmrelease.yaml
- source.yaml
```

```console
cd security/dev-cluster/sealed-secrets
kustomize create --resources ../../base/sealed-secrets
```

<span style="color:green">security/k3d-crossplane/sealed-secrets/kustomization.yaml</span>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - helmrelease.yaml
```

<span style="color:green">security/k3d-crossplane/sealed-secrets/helmrelease.yaml</span>

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: kube-system
spec:
  values:
    resources:
      requests:
        cpu: 100m
```


```console
git commit -m "security: deploy sealed-secrets in k3d-crossplane"
[security/sealed-secrets 283648e] security: deploy sealed-secrets in k3d-crossplane
 6 files changed, 66 insertions(+)
 create mode 100644 clusters/k3d-crossplane/security.yaml
 create mode 100644 security/base/sealed-secrets/helmrelease.yaml
 create mode 100644 security/base/sealed-secrets/kustomization.yaml
 create mode 100644 security/base/sealed-secrets/source.yaml
 create mode 100644 security/k3d-crossplane/sealed-secrets/helmrelease.yaml
 create mode 100644 security/k3d-crossplane/sealed-secrets/kustomization.yaml
 ```


 ```console
flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/d36a33c    False           True    Applied revision: main/d36a33c
security        main/d36a33c    False           True    Applied revision: main/d36a33c
```

```console
flux get sources helm
NAME            REVISION                                                                SUSPENDED       READY   MESSAGE
sealed-secrets  4c0aa1980e3ec9055dea70abd2b259aad1a2c235325ecf51a25a92a39ac4eeee        False           True    stored artifact for revision '4c0aa1980e3ec9055dea70abd2b259aad1a2c235325ecf51a25a92a39ac4eeee'
```

```console
flux get helmrelease -n kube-system
NAME            REVISION        SUSPENDED       READY   MESSAGE
sealed-secrets  2.2.0           False           True    Release reconciliation succeeded
```


```console
kubectl create secret generic test-secret --from-literal=foo=bar -o yaml --dry-run=client \


```

### :hammer_and_wrench: Deploy and configure Crossplane


### :detective: Troubleshooting

flux logs --kind


```console
flux trace -n flux-system pod/helm-controller-88f6889c6-4bd4w

Object:        Pod/helm-controller-88f6889c6-4bd4w
Namespace:     flux-system
Status:        Managed by Flux
---
Kustomization: flux-system
Namespace:     flux-system
Path:          ./clusters/k3d-crossplane
Revision:      main/2beb6aafea67f3386b50cbc706fb34575844040d
Status:        Last reconciled at 2022-07-14 17:33:30 +0200 CEST
Message:       Applied revision: main/2beb6aafea67f3386b50cbc706fb34575844040d
---
GitRepository: flux-system
Namespace:     flux-system
URL:           ssh://git@github.com/Smana/devflux
Branch:        main
Revision:      main/2beb6aafea67f3386b50cbc706fb34575844040d
Status:        Last reconciled at 2022-07-14 17:13:32 +0200 CEST
Message:       stored artifact for revision 'main/2beb6aafea67f3386b50cbc706fb34575844040d'
```


flux diff