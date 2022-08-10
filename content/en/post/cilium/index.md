+++
author = "Smaine Kahlouch"
title = "Exploring `Cilium` projects"
date = "2022-07-02"
summary = "CNI, Hubble, ServiceMesh, Tetragon"
# featureImage = "crossplane_k3d.png"
codeMaxLines = 20
usePageBundles = true
toc = true
tags = [
    "kubernetes",
    "infrastructure",
    "network",
    "security",
    "observability"
]
thumbnail= "cilium-thumbnail.png"
+++

##### Draft

##### Plan Cilium introduction

* What is ebpf
* Networking with Cilium using ebpf and kube-proxy replacement. performances tunning.
* IPAM
* Network policies, editor
* Introduction to hubble (overhead cpu)

```console
k3d cluster create cilium
...
INFO[0027] Cluster 'cilium' created successfully!
INFO[0027] You can now use it like this:
kubectl cluster-info
```

```console
export GITHUB_USER=<YOUR_ACCOUNT>
export GITHUB_TOKEN=ghp_<REDACTED> # your personal access token
export GITHUB_REPO=devflux
flux bootstrap github --owner="${GITHUB_USER}" --repository="${GITHUB_REPO}" --personal=false --branch=cilium  --path=clusters/k3d-cilium
...
✔ all components are healthy
```

asdf plugin-add cilium cli
asdf install cilium-cli 1.12.0
asdf global cilium 1.12.0

### Cilium installed as part of the GKE cluster creation (Dataplane v2)

Cilium is an option that can be activated as the default CNI on GKE. So if you just want to benefit from ebpf performances and Cilium advanced network policies this is the easiest way to use it on GKE, it is already **configured by Google Cloud** as part of its managed Kubernetes service. <https://cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2>
```yaml
metadata:
  name: dev-cluster
spec:
  forProvider:
...
    networkConfig:
      datapathProvider: "ADVANCED_DATAPATH"
```





Installation documentation: <https://docs.cilium.io/en/stable/gettingstarted/k8s-install-helm/>

```yaml
apiVersion: container.gcp.crossplane.io/v1beta1
kind: NodePool
metadata:
  name: main-np
spec:
  forProvider:
...
    config:
...
      taints:
        - effect: "NO_EXECUTE"
          key: "node.cilium.io/agent-not-ready"
          value: "true"
```

```console
kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./crossplane.json -o yaml --dry-run=client | kubectl neat | kubeseal --format yaml --namespace crossplane-system - > ~/sources/devflux/infrastructure/k3d-cilium/crossplane/configuration/sealedsecret.yaml
```

```console
kubectl get cluster dev-cluster --template={{.spec.forProvider.clusterIpv4Cidr}}
10.144.0.0/14
```

#---------


In a [previous article](/post/crossplane_k3d/), we've seen how to use **Crossplane** so that we can manage cloud resources the same way as our applications. :heart: Declarative approach!
There were several steps and command lines in order to get everything working and reach our target to provision a dev Kubernetes cluster.

Here we'll achieve exactly the same thing but we'll do that in the **GitOps** way.
According to the [OpenGitOps](https://opengitops.dev/) working group there are 4 GitOps principles:

* The desired state of our system must be expressed **declaratively**.
* This state must be stored in a **versioning** system.
* Changes are pulled and **applied automatically** in the target platform whenever the desired state changes.
* If, for any reason, the current state is modified, it will be **automatically reconciled** with the desired state.

There are several GitOps engine options. The most famous ones are [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Flux](https://fluxcd.io/). We won't compare them here. I chose Flux because I like its composable architecture with different controllers, each one handling a core Flux feature (GitOps toolkit).

![flux_components](flux_components.png)

Learn more about **GitOps toolkit** components [here](https://fluxcd.io/docs/components/).

## :bullseye: Our target

Here we want to declare our desired infrastructure components only by adding **git** changes.
By the end of this article you'll get a GKE cluster provisioned using a local Crossplane instance.
We'll discover [**Flux**](https://fluxcd.io/) basics and how to use it in order to build a complete GitOps CD workflow.

<br>

## :ballot_box_with_check: Requirements

### :inbox_tray: Install required tools

First of all we need to install a few tools using [asdf](/post/asdf/asdf/)

Create a local file
.tool-versions

```console
cd ~/sources/devflux/

cat > .tool-versions <<EOF
flux2 0.31.3
kubectl 1.24.3
kubeseal 0.18.1
kustomize 4.5.5
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
kubectl         1.24.3          /home/smana/sources/devflux/.tool-versions
kubeseal        0.18.1          /home/smana/sources/devflux/.tool-versions
kustomize       4.5.5           /home/smana/sources/devflux/.tool-versions
```
<br>

### :key: Create a Github personal access token

In this article the git repository is hosted in Github. In order to be able to use the `flux bootstrap` a personnal access token is required.

Please follow [this procedure](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

{{% notice warning Warning %}}
Store the Github token in a safe place for later use
{{% /notice %}}

<br>

### :technologist: Clone the devflux repository

All the files used for the upcoming steps can be retrieved from [this repository](https://github.com/Smana/devflux).
You should **clone** it, that will be easier to copy them into your own repository.

```console
git clone https://github.com/Smana/devflux.git
```

<br>

## :rocket: Bootstrap flux in the Crossplane cluster

As we will often be using the `flux` CLI you may want to configure the bash|zsh completion
```console
source <(flux completion bash)
```

{{% notice warning Warning %}}
Here we consider that you already have a **local k3d instance**. If not you may want to either go through the whole [previous article](/post/crossplane_k3d) or just run the [local cluster creation](/post/crossplane_k3d#-create-the-local-k3d-cluster-for-crossplanes-control-plane).

Ensure that you're working in the right context

```console
kubectl config current-context
k3d-crossplane
```

{{% /notice %}}

Run the `bootstrap` command that will basically deploy all Flux's components in the namespace _flux-system_. Here I'll create a repository named **devflux** using my personal Github account.

```console
export GITHUB_USER=<YOUR_ACCOUNT>
export GITHUB_TOKEN=ghp_<REDACTED> # your personal access token
export GITHUB_REPO=devflux

flux bootstrap github --owner="${GITHUB_USER}" --repository="${GITHUB_REPO}" --personal --path=clusters/k3d-crossplane
► cloning branch "main" from Git repository "https://github.com/<YOUR_ACCOUNT>/devflux.git"
...
✔ configured deploy key "flux-system-main-flux-system-./clusters/k3d-crossplane" for "https://github.com/<YOUR_ACCOUNT>/devflux"
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

Running the bootstap command actually **creates a github repository** if it doesn't exist yet.
**Clone** it now for our upcoming changes. You'll notice that the first commit has been made by Flux.

```console
git clone https://github.com/<YOUR_ACCOUNT>/devflux.git
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

## :open_file_folder: Flux repository structure

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
| **/apps** | our applications | Here we'll deploy a demo application "online-boutique" |
| **/infrastructure** | base infrastructure/network components | Crossplane as it will be used to provision cloud resources but we can also find CSI/CNI/EBS drivers... |
| **/observability** | All metrics/apm/logging tools | Prometheus of course, Opentelemetry ... |
| **/security** | Any component that enhance our security level | SealedSecrets (see below) |

{{% notice info Info %}}
For the upcoming steps please refer to the demo repository [here](https://github.com/Smana/devflux)
{{% /notice %}}

Let's use this structure and begin to deploy applications :rocket:.

<br>

## :closed_lock_with_key: SealedSecrets

There are plenty of alternatives when it comes to secrets management in Kubernetes.
In order to securely **store secrets in a git repository** the GitOps way we'll make use of [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets).
It uses a custom resource definition named `SealedSecrets` in order to encrypt the Kubernetes secret at the client side then the controller is in charge of decrypting and generating the expected secret in the cluster.

### :hammer_and_wrench: Deploy the controller using Helm

The first thing to do is to declare the `kustomization` that handles all the security tools.

<span style="color:green">clusters/k3d-crossplane/security.yaml</span>

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

{{% notice info Info %}}
A _Kustomization_ is a custom resource that comes with Flux. It basically points to a set of Kubernetes resources managed with [kustomize](https://kustomize.io/)
The above **security** kustomization points to a local directory where the kustomize resources are.

```yaml
...
spec:
  path: ./security/k3d-crossplane
...
```

{{% notice note Note %}}
This is worth noting that there are two types on kustomizations. That can be confusing when you start playing with Flux.

* One managed by flux's kustomize controller. Its API is `kustomization.kustomize.toolkit.fluxcd.io`
* The other `kustomization.kustomize.config.k8s.io` is for the kustomize overlay
{{% /notice %}}

The <span style="color:green">kustomization.yaml</span> file is always used for the kustomize overlay. Flux itself doesn't need this overlay in all cases, but if you want to use features of a Kustomize overlay you will occasionally need to create it in order to access them. It provides instructions to the Kustomize CLI.

{{% /notice %}}


We will deploy SealedSecrets using the [Helm chart](https://github.com/bitnami-labs/sealed-secrets#helm-chart). So we need to declare the source of this chart.
Using the kustomize overlay system, we'll first create the `base` files that will be inherited at the cluster level.

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

Then we'll define the `HelmRelease` which references the above source.
Put the values you want to apply to the Helm chart under `spec.values`

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

If you're starting your repository from scratch you'll need to generate the `kustomization.yaml` file (kustomize overlay).

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

Now we declare the sealed-secret kustomization at the cluster level.
Just for the example we'll overwrite a value at the cluster level using kustomize's overlay system.

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

<span style="color:green">security/k3d-crossplane/sealed-secrets/kustomization.yaml</span>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patches:
  - helmrelease.yaml
```

Pushing our changes is the only thing to do in order to get sealed-secrets deployed in the target cluster.

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

After a few seconds (1 minutes by default) a new kustomization will appear.

 ```console
flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/d36a33c    False           True    Applied revision: main/d36a33c
security        main/d36a33c    False           True    Applied revision: main/d36a33c
```

And all the resources that we declared in the flux repository should be available and READY.

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
<br>

### 🧪 A first test SealedSecret

Let's use the CLI `kubeseal` to test it out. We'll create a `SealedSecret` that will be decrypted by the sealed-secrets controller in the cluster and create the expected secret _foobar_

```console
kubectl create secret generic foobar -n default --dry-run=client -o yaml --from-literal=foo=bar \
| kubeseal --namespace default --format yaml | kubectl apply -f -
sealedsecret.bitnami.com/foobar created

kubectl get secret -n default foobar
NAME     TYPE     DATA   AGE
foobar   Opaque   1      3m13s

kubectl delete sealedsecrets.bitnami.com foobar
sealedsecret.bitnami.com "foobar" deleted
```

<br>

## :cloud: Deploy and configure Crossplane

### :key: Create the Google service account secret

The first thing we need to do in order to get Crossplane working is to create the GCP serviceaccount. The steps have been covered [here](/post/crossplane_k3d/#-generate-the-google-cloud-service-account) in the previous article.
We'll create a `SealedSecret` _gcp-creds_ that contains  the serviceaccount file <span style="color:green">crossplane.json</span>.

<span style="color:green">infrastructure/k3d-crossplane/crossplane/configuration/sealedsecrets.yaml</span>

```console
kubectl create secret generic gcp-creds --context k3d-crossplane -n crossplane-system --from-file=creds=./crossplane.json --dry-run=client -o yaml \
| kubeseal --format yaml --namespace crossplane-system - > infrastructure/k3d-crossplane/crossplane/configuration/sealedsecrets.yaml
```
<br>

### 🔄 Crossplane dependencies

Now we will deploy Crossplane with Flux. I won't put the manifests here you'll find all of them in [this repository](https://github.com/Smana/devflux).
However it's important to understand that, in order to deploy and configure Crossplane properly we need to do that in a **specific order**.
Indeed several CRD's (custom resource definitions) are required:

1. First of all we'll install the crossplane **controller**.
2. Then we'll **configure** the `provider` because the custom resource is now available thanks to the crossplane controller installation.
3. Finally a provider installation deploys several CRDs that can be used to configure the provider itself and **cloud resources**.

The dependencies between kustomizations can be controlled using the parameters `dependsOn`.
Looking at the file <span style="color:green">clusters/k3d-crossplane/infrastructure.yaml</span>, we can see for example that the kustomization _infrastructure-custom-resources_ depends on the kustomization _crossplane_provider_ which itself depends on _crossplane-configuration_....

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: crossplane-provider
spec:
...
  dependsOn:
    - name: crossplane-core
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: crossplane-configuration
spec:
...
  dependsOn:
    - name: crossplane-provider
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: infrastructure-custom-resources
spec:
...
  dependsOn:
    - name: crossplane-configuration
```

Commit and push the changes for the kustomisations to appear. Note that they'll be reconciled in the defined order.

```console
flux get kustomizations
NAME                            REVISION        SUSPENDED       READY   MESSAGE
infrastructure-custom-resources                 False           False   dependency 'flux-system/crossplane-configuration' is not ready
crossplane-configuration                        False           False   dependency 'flux-system/crossplane-provider' is not ready
security                        main/666f85a    False           True    Applied revision: main/666f85a
flux-system                     main/666f85a    False           True    Applied revision: main/666f85a
crossplane-core                 main/666f85a    False           True    Applied revision: main/666f85a
crossplane-provider             main/666f85a    False           True    Applied revision: main/666f85a
```

Then all Crossplane components will be deployed, we can have a look to the `HelmRelease` status for instance.

```console
kubectl describe helmrelease -n crossplane-system crossplane
...
Status:
  Conditions:
    Last Transition Time:          2022-07-15T19:12:04Z
    Message:                       Release reconciliation succeeded
    Reason:                        ReconciliationSucceeded
    Status:                        True
    Type:                          Ready
    Last Transition Time:          2022-07-15T19:12:04Z
    Message:                       Helm upgrade succeeded
    Reason:                        UpgradeSucceeded
    Status:                        True
    Type:                          Released
  Helm Chart:                      crossplane-system/crossplane-system-crossplane
  Last Applied Revision:           1.9.0
  Last Attempted Revision:         1.9.0
  Last Attempted Values Checksum:  056dc1c6029b3a644adc7d6a69a93620afd25b65
  Last Release Revision:           2
  Observed Generation:             1
Events:
  Type    Reason  Age   From             Message
  ----    ------  ----  ----             -------
  Normal  info    20m   helm-controller  HelmChart 'crossplane-system/crossplane-system-crossplane' is not ready
  Normal  info    20m   helm-controller  Helm upgrade has started
  Normal  info    19m   helm-controller  Helm upgrade succeeded
```

And our **GKE cluster** should also be created because we defined a bunch of crossplane custom resources in <span style="color:green">infrastructure/k3d-crossplane/custom-resources/crossplane</span>

```console
kubectl get cluster
NAME          READY   SYNCED   STATE     ENDPOINT        LOCATION         AGE
dev-cluster   True    True     RUNNING   34.x.x.190      europe-west9-a   22m
```

<br>

### :rocket: Bootstrap flux in the dev cluster

Our local Crossplane cluster is now ready and it created our dev cluster and we also want it to be managed with Flux.
So let's configure Flux for this dev cluster using the same `bootstrap` command.

Authenticate to the newly created cluster. The following command will automatically change your current context.

```console
gcloud container clusters get-credentials dev-cluster --zone europe-west9-a --project <your_project>
Fetching cluster endpoint and auth data.
kubeconfig entry generated for dev-cluster.

kubectl config current-context
gke_<your_project>_europe-west9-a_dev-cluster
```

Run the bootstrap command for the `dev-cluster`.

```console
export GITHUB_USER=Smana
export GITHUB_TOKEN=ghp_<REDACTED> # your personal access token
export GITHUB_REPO=devflux

flux bootstrap github --owner="${GITHUB_USER}" --repository="${GITHUB_REPO}" --personal --path=clusters/dev-cluster
► cloning branch "main" from Git repository "https://github.com/Smana/devflux.git"
...
✔ configured deploy key "flux-system-main-flux-system-./clusters/dev-cluster" for "https://github.com/Smana/devflux"
...
✔ all components are healthy
```

{{% notice note Note %}}
It's worth noting that each Kubernetes cluster generates its own sealing keys. That means that if you recreate the dev-cluster, you must **regenerate all the sealedsecrets**.
In our example we declared a secret in order to set the Grafana credentials. Here's the command you need to run in order to create a new version of the sealedsecret and don't forget to use the proper **context** :wink:.

```console
kubectl create secret generic kube-prometheus-stack-grafana \
--from-literal=admin-user=admin --from-literal=admin-password=<yourpassword> --namespace observability --dry-run=client -o yaml \
| kubeseal --namespace observability --format yaml > observability/dev-cluster/kube-prometheus-stack/sealedsecrets.yaml
```

{{% /notice %}}

After a few seconds we'll get the following kustomizations deployed.

```console
flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
apps            main/1380eaa    False           True    Applied revision: main/1380eaa
flux-system     main/1380eaa    False           True    Applied revision: main/1380eaa
observability   main/1380eaa    False           True    Applied revision: main/1380eaa
security        main/1380eaa    False           True    Applied revision: main/1380eaa
```
Here we configured the prometheus stack and deployed a demo microservices stack named "[online-boutique](https://github.com/GoogleCloudPlatform/microservices-demo)"
This demo application exposes the frontend through a service of type `LoadBalancer`.

```console
kubectl get svc -n demo frontend-external
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
frontend-external   LoadBalancer   10.140.174.201   34.155.121.2   80:31943/TCP   7m44s
```

Use the _EXTERNAL_IP_

![online_boutique](online-boutique.png)

## :detective: Troubleshooting

The cheatsheet in [Flux's documentation](https://fluxcd.io/docs/cheatsheets/troubleshooting) contains many ways for troubleshooting when something goes wrong.
Here I'll just give a sample of my favorite command lines.

Objects that aren't ready

```console
flux get all -A --status-selector ready=false
```

Checking the logs of a given kustomization
```console
flux logs --kind kustomization --name infrastructure-custom-resources
2022-07-15T19:38:52.996Z info Kustomization/infrastructure-custom-resources.flux-system - server-side apply completed
2022-07-15T19:38:53.016Z info Kustomization/infrastructure-custom-resources.flux-system - Reconciliation finished in 66.12266ms, next run in 4m0s
2022-07-15T19:11:34.697Z info Kustomization/infrastructure-custom-resources.flux-system - Discarding event, no alerts found for the involved object
```

Show how a given pod is managed by Flux.

```console
flux trace -n crossplane-system pod/crossplane-5dc8d888d7-g95qx

Object:         Pod/crossplane-5dc8d888d7-g95qx
Namespace:      crossplane-system
Status:         Managed by Flux
---
HelmRelease:    crossplane
Namespace:      crossplane-system
Revision:       1.9.0
Status:         Last reconciled at 2022-07-15 21:12:04 +0200 CEST
Message:        Release reconciliation succeeded
---
HelmChart:      crossplane-system-crossplane
Namespace:      crossplane-system
Chart:          crossplane
Version:        1.9.0
Revision:       1.9.0
Status:         Last reconciled at 2022-07-15 21:11:36 +0200 CEST
Message:        pulled 'crossplane' chart with version '1.9.0'
---
HelmRepository: crossplane
Namespace:      crossplane-system
URL:            https://charts.crossplane.io/stable
Revision:       362022f8c7ce215a0bb276887115cb5324b35a3169723900c84456adc3538a8d
Status:         Last reconciled at 2022-07-15 21:11:35 +0200 CEST
Message:        stored artifact for revision '362022f8c7ce215a0bb276887115cb5324b35a3169723900c84456adc3538a8d'
```

If you want to check what would be the changes before pushing your commit. In thi given example I just increased the cpu requests for the sealed-secrets controller.

```console
flux diff kustomization security  --path security/k3d-crossplane
✓  Kustomization diffing...
► HelmRelease/kube-system/sealed-secrets drifted

metadata.generation
  ± value change
    - 6
    + 7

spec.values.resources.requests.cpu
  ± value change
    - 100m
    + 120m

⚠️ identified at least one change, exiting with non-zero exit code
```
<br>

## :broom: Cleanup

**Don't forget** to delete the Cloud resources if you don't want to have a bad suprise :dollar:!
Just comment the file <span style="color:green">infrastructure/k3d-crossplane/custom-resources/crossplane/kustomization.yaml</span>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # - cluster.yaml
  - network.yaml
```

<br>

## :clap: Achievements

With our current setup everything is configured using the GitOps approach:

* We can manage infrastructure resources using Crossplane.
* Our secrets are securely stored in our git repository.
* We have a dev-cluster that we can enable or disable just but commenting a yaml file.
* Our demo application can be deployed from scratch in seconds.

## 💭 final thoughts

Flux is probably the tool I'm using the most on a daily basis. It's really amazing!

When you get familiar with its concepts and the command line it becomes really easy to use and troubleshoot. You can use either Helm when a chart is available or Kustomize.

However we faced a few issues:

* It's not straightforward to find an efficient structure depending on the company needs. Especially when you have several Kubernetes controllers that depend on other CRDs.
* The Helm controller doesn't maintain a state of the Kubernetes resources deployed by the Helm chart. That means that if you delete a resource which has been deployed through a Helm chart, it won't be reconciled (It will change soon. Being discussed [here](https://github.com/fluxcd/helm-controller/issues/186#issuecomment-932107655))
* Flux doesn't provide itself a web UI and switching between CLIs (kubectl, flux ...) can be annoying from a developer perspective. (I'm going to test [weave-gitops](https://github.com/weaveworks/weave-gitops) )

I've been using Flux in production for more than a year and we configured it with the image automation so that the only thing a developer has to do is to merge a pull request and the new version of the application is automatically deployed in the target cluster.

I should probably give another try to ArgoCD in order to be able to compare these precisely 🤔.