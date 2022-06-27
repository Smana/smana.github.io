---
title: 'Kubernetes workshop'
description: 'Introduction to Kubernetes basics through hands-on'
summary: 'Introduction to Kubernetes basics through hands-on'
date: "2021-05-01"
featured: true
aliases:
  - workshop-kubernetes
author: 'Smana'
usePageBundles: true

thumbnail: 'https://cncf-branding.netlify.app/img/projects/kubernetes/icon/black/kubernetes-icon-black.png'

categories:
  - containers

tags:
  - Kubernetes

series:
  - Workshop Kubernetes
---

This repository aims to quickly learn the basics of [Kubernetes](https://kubernetes.io/).

:warning: None of the examples given here are made for production.

## Requirements

* docker
* [k3d](https://k3d.io/) >5.x.x
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)
* (optional)[fzf](https://github.com/junegunn/fzf)

## Agenda

* **[Prepare your local Kubernetes environment](/post/series/workshop_kubernetes/local/)**
* **[Run an application on Kubernetes](/post/series/workshop_kubernetes/run_app/)**
* **[Deploy a Wordpress](/post/series/workshop_kubernetes/application_stack/)**
* **[Resources and autoscaling](/post/series/workshop_kubernetes/autoscaling/)**
* **[Troubleshooting](/post/series/workshop_kubernetes/troubleshoot/)**
* **[RBAC](/post/series/workshop_kubernetes/rbac/)**

## Cleanup

Pretty simple we’ll drop the whole k3d cluster

```console
$ k3d cluster delete workshop
INFO[0000] Deleting cluster 'workshop'
...
INFO[0008] Successfully deleted cluster workshop!
```

:arrow_right: You may want to continue with the [Helm workshop](/post/series/workshop_helm/intro/)
