+++
author = "Smaine Kahlouch"
title = "My Kubernetes cluster in GKE with Crossplane"
date = "2022-05-25"
summary = "Use a local **k3d** cluster in order to manage infrastructure in Google Cloud, create a **GKE** cluster and Velero to backup the **Crossplane**"
featured = true
codeMaxLines = 20
usePageBundles = true
tags = [
    "kubernetes",
    "infrastructure"
]
thumbnail= "crossplane_k3d.jpg"
+++

The target of this documentation is to be able to manage infrastructure using **Crossplane**.
And we want to backup the resources created so that we will be able to recreate everything from scratch.

Here are the steps we'll follow in order to get my personal Kubernetes cluster:

- [:whale: Create the local k3d cluster for Crossplane's control plane](#whale-create-the-local-k3d-cluster-for-crossplanes-control-plane)
- [:cloud: Generate the Google Cloud service account](#cloud-generate-the-google-cloud-service-account)
- [:construction: Deploy and configure Crossplane](#construction-deploy-and-configure-crossplane)
- [:rocket: Create a GKE cluster](#rocket-create-a-gke-cluster)
- [:floppy_disk: Backup the local Kubernetes cluster using Velero](#floppy_disk-backup-the-local-kubernetes-cluster-using-velero)

### :whale: Create the local k3d cluster for Crossplane's control plane

[**k3d**](https://k3d.io) is a lightweight kubernetes cluster that leverages k3s that runs in our local laptop.
There are several deployment models for Crossplane, we could for instance deploy the control plane on a management cluster on Kubernetes or a control plane per Kubernetes cluster.<br>
Here I chose a simple method which is fine for a personal use case: A **local Kubernetes instance** in which I'll deploy Crossplane.

Let's install k3d using [asdf](asdf.md).

```console
asdf plugin-add k3d

asdf install k3d $(asdf latest k3d)
* Downloading k3d release 5.4.1...
k3d 5.4.1 installation was successful!
```

Create a single node Kubernetes cluster.

```console
k3d cluster create crossplane
...
INFO[0043] You can now use it like this:
kubectl cluster-info

k3d cluster list
crossplane   1/1       0/0      true
```

Check that the cluster is reachable using the `kubectl` CLI.

```console
kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:40643
CoreDNS is running at https://0.0.0.0:40643/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:40643/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

We only need a single node for our Crossplane use case.

```console
kubectl get nodes
NAME                      STATUS   ROLES                  AGE   VERSION
k3d-crossplane-server-0   Ready    control-plane,master   26h   v1.22.7+k3s1
```
<br>

### :cloud: Generate the Google Cloud service account

{{% notice warning "Warning" %}}
Store the downloaded `crossplane.json` credentials file in a safe place.
{{% /notice %}}

Create a service account
```console
GCP_PROJECT=<your_project>
gcloud iam service-accounts create crossplane --display-name "Crossplane" --project=${GCP_PROJECT}
Created service account [crossplane].
```

Assign the proper permissions to the service account.
* Kubernetes Engine Admin

```console
gcloud projects add-iam-policy-binding "${GCP_PROJECT}" --member=serviceAccount:"${SA_EMAIL}" --role=roles/container.admin
Updated IAM policy for project [<project>].
bindings:
- members:
  - serviceAccount:service-xxx@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
...
version: 1

```

Download the service account key (json format)

```console
SA_EMAIL=$(gcloud iam service-accounts list --filter="email ~ ^crossplane" --format='value(email)')

gcloud iam service-accounts keys create crossplane.json --iam-account ${SA_EMAIL}
created key [ea2eb9ce2939127xxxxxxxxxx] of type [json] as [crossplane.json] for [crossplane@<project>.iam.gserviceaccount.com]
```
<br>

### :construction: Deploy and configure Crossplane

Now that we have a credentials file for Google Cloud, we can deploy the [**Crossplane**](https://crossplane.io/) operator and configure the `provider-gcp` provider.

{{% notice info "Note" %}}
Most of the following steps are issued from the [official documentation](https://crossplane.io/docs/v1.8/getting-started/install-configure.html)
{{% /notice %}}


We'll first use Helm in order to install the **operator**
```console
helm repo add crossplane-master https://charts.crossplane.io/master/
"crossplane-master" has been added to your repositories

helm repo update
...Successfully got an update from the "crossplane-master" chart repository

helm install crossplane --namespace crossplane-system --create-namespace \
--version 1.18.1 crossplane-stable/crossplane

NAME: crossplane
LAST DEPLOYED: Mon Jun  6 22:00:02 2022
NAMESPACE: crossplane-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Release: crossplane
...
```

Check that the operator is running properly.

```console
kubectl get po -n crossplane-system
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-rbac-manager-54d96cd559-222hc   1/1     Running   0          3m37s
crossplane-688c575476-lgklq                1/1     Running   0          3m37s
```

Now we'll configure Crossplane so that it will be able to create and manage GCP resources. This is done by configuring the **provider** `provider-gcp` as follows.

<span style="color:green">provider.yaml</span>
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: crossplane-provider-gcp
spec:
  package: crossplane/provider-gcp:v0.21.0
```

```console
kubectl apply -f provider.yaml
provider.pkg.crossplane.io/crossplane-provider-gcp created

kubectl get providers
NAME                      INSTALLED   HEALTHY   PACKAGE                           AGE
crossplane-provider-gcp   True        True      crossplane/provider-gcp:v0.21.0   10s
```

Create the Kubernetes secret that holds the GCP credentials file created [above](#cloud-generate-the-google-cloud-service-account)

```console
kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./crossplane.json
secret/gcp-creds created
```

Then we need to create a resource named `ProviderConfig` and reference the newly created secret.

<span style="color:green">provider-config.yaml</span>
```yaml
apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: ${PROJECT_ID}
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds
```

```console
kubectl apply -f provider-config.yaml
providerconfig.gcp.crossplane.io/default created
```

[crd](https://doc.crds.dev/github.com/crossplane/provider-gcp)


```yaml
apiVersion: container.gcp.crossplane.io/v1beta2
kind: Cluster
metadata:
  name: example-cluster
spec:
  forProvider:
    initialClusterVersion: "1.23"
    location: europe-west1
    autoscaling:
      autoprovisioningNodePoolDefaults:
        serviceAccount: sa-test
    networkConfig:
      enableIntraNodeVisibility: true
    loggingService: logging.googleapis.com/kubernetes
    monitoringService: monitoring.googleapis.com/kubernetes
    addonsConfig:
      gcePersistentDiskCsiDriverConfig:
        enabled: true
    network: "default"
  writeConnectionSecretToRef:
    namespace: default
    name: gke-conn
---
apiVersion: container.gcp.crossplane.io/v1beta1
kind: NodePool
metadata:
  name: crossplane-np
spec:
  forProvider:
    autoscaling:
      autoprovisioned: false
      enabled: true
      maxNodeCount: 5
      minNodeCount: 3
    clusterRef:
      name: example-cluster
    config:
      serviceAccount: sa-test
      machineType: n1-standard-1
      sandboxConfig:
        type: gvisor
      diskSizeGb: 120
      diskType: pd-ssd
      imageType: cos_containerd
      labels:
        test-label: crossplane-created
      oauthScopes:
      - "https://www.googleapis.com/auth/devstorage.read_only"
      - "https://www.googleapis.com/auth/logging.write"
      - "https://www.googleapis.com/auth/monitoring"
      - "https://www.googleapis.com/auth/servicecontrol"
      - "https://www.googleapis.com/auth/service.management.readonly"
      - "https://www.googleapis.com/auth/trace.append"
    initialNodeCount: 3
    locations:
      - "europe-west1-b"
```

<br>

### :rocket: Create a GKE cluster

<br>

### :floppy_disk: Backup the local Kubernetes cluster using Velero