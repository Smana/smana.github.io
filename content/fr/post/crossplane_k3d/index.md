+++
author = "Smaine Kahlouch"
title = "My Kubernetes cluster in GKE with Crossplane"
date = "2022-05-25"
summary = "Use a local **k3d** cluster in order to manage infrastructure in Google Cloud, create a **GKE** cluster and Velero to backup the **Crossplane**"
featured = true
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

{{% notice info "Info" %}}
In order to install binaries and to be able to switch from a version to another I usually use [**asdf**](https://asdf-vm.com/).

Let's install k3d

```console
asdf plugin-add k3d
```

Check the versions available

```console
asdf list-all k3d| tail -n 3
5.4.0-dev.3
5.4.0
5.4.1
```

We'll install the latest version

```console
asdf install k3d $(asdf latest k3d)
* Downloading k3d release 5.4.1...
k3d 5.4.1 installation was successful!
```

Finally we can switch from a version to another. We can set a `global` version that would be used on all directories.

```console
asdf global k3d 5.4.1
```

or use a `local` version depending on the current directory

```console
cd /tmp
asdf local k3d 5.4.1

asdf current k3d
k3d             5.4.1           /tmp/.tool-versions
```
{{% /notice %}}



```console
k3d cluster create crossplane
...
INFO[0043] You can now use it like this:
kubectl cluster-info
```

```console
kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:40643
CoreDNS is running at https://0.0.0.0:40643/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:40643/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

```console
k3d cluster list
crossplane   1/1       0/0      true
```

We only need a single node for our Crossplane use case:

```console
kubectl get nodes
NAME                      STATUS   ROLES                  AGE   VERSION
k3d-crossplane-server-0   Ready    control-plane,master   26h   v1.22.7+k3s1
```

### :cloud: Generate the Google Cloud service account

```console
GCP_PROJECT=<your_project>
gcloud iam service-accounts create crossplane --display-name "Crossplane" --project=${GCP_PROJECT}
Created service account [crossplane].
```

### :construction: Deploy and configure Crossplane

Deploy [**Crossplane**](https://crossplane.io/) and configure the `provider-gcp`

### :rocket: Create a GKE cluster


### :floppy_disk: Backup the local Kubernetes cluster using Velero