---
title: 'Helm workshop: Introduction'
description: 'Learn the basis of Helm through a series of workshops'
summary: |
  This is  a series of workshops dedicated to Helm, the famous Kubernetes package manager.
  We'll start by preparing your local environment
date: "2021-06-01"
aliases:
  - workshop-helm
author: 'Smana'
usePageBundles: true

thumbnail: 'https://cncf-branding.netlify.app/img/projects/helm/horizontal/black/helm-horizontal-black.png'

categories:
  - devxp
tags:
  - Helm
  - Kubernetes

series:
  - Workshop Helm
---

## Requirements

* docker
* [k3d](https://k3d.io/) >5.x.x
* [helm](https://helm.sh/docs/intro/install/) >3.x.x
* [helmfile](https://github.com/roboll/helmfile)

In order to have an easily provisioned temporary playground we’ll make use of **k3d** which is a lightweight local Kubernetes instance.

After installing the binary you should enable the completion (bash or zsh) as follows (do the same for both helm and k3d).

```console
$ source <(k3d completion bash)
```

Then create the sandbox cluster named “**helm-workshop**”

```console
$ k3d cluster create helm-workshop
INFO[0000] Created network 'k3d-helm-workshop'
INFO[0000] Created volume 'k3d-helm-workshop-images'
INFO[0001] Creating node 'k3d-helm-workshop-server-0'
INFO[0006] Creating LoadBalancer 'k3d-helm-workshop-serverlb'
INFO[0007] (Optional) Trying to get IP of the docker host and inject it into the cluster as 'host.k3d.internal' for easy access
INFO[0010] Successfully added host record to /etc/hosts in 2/2 nodes and to the CoreDNS ConfigMap
INFO[0010] Cluster 'helm-workshop' created successfully!
INFO[0010] You can now use it like this:
kubectl cluster-info
```

Note that your current configuration should be automatically switched to the newly created cluster.

```console
$ kubectl config current-context
k3d-helm-workshop
```

* **[Playing with third party charts](01_third_party.md)**
* **[Environment and ecosystem](02_ecosystem.md)**
* **[Build your first chart](03_build_chart.md)**
* **[Application lifecycle](04_lifecycle.md)**
* **[Templating challenge](05_templating_practice.md)**

### Other considerations

#### Hosting and versioning

Most of the time we would want to share the charts in order to be used on different systems or to pull the dependencies.

There are multiple options for that, here are the ones that are generally used.


*   Chartmuseum is the official solution. This is a pretty simple webserver that exposes a Rest API.
*   Harbor. Its main purpose is to store images (containers), but it offers many other features such as vulnerability scanning, images signing and integrates chartmuseum.
*   Artifactory can be used to stored Helm charts too
*   An [OCI](https://opencontainers.org/) store (container registry).

Pushing the charts into a central location requires to manage the versions of the charts. Any changes should trigger a version bump in the Chart.yaml file.

#### Secrets management

One sensitive topic that we didn’t talk about is how to handle secrets.

This is not directly related to Helm but this is a general issue on Kubernetes.

There are many options, some of them work great with Helm, some others require managing secrets apart from Helm releases.

In the [ArgoCD documentation](https://argoproj.github.io/argo-cd/operator-manual/secret-management/) they tried to reference all the options available.

#### Cleanup

Pretty simple we’ll drop the whole k3d cluster

```console
$ k3d cluster delete helm-workshop
```