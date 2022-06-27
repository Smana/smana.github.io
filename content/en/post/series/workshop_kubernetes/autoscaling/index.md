---
title: 'Kubernetes workshop: Resources allocation and autoscaling'
description: 'Resources allocation in Kubernetes and run an autoscaling test'
summary: 'Learn how to assign resources to containers and to experiment Kubernetes autoscaling'
date: "2021-05-06"
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

## Resources allocation in Kubernetes

Resources allocation in Kubernetes is made using `requests` and `limits` in the container's definition.

* `requests`: What the container is guaranteed to get. These values are used when the scheduler takes a decision on where (what node) to place a given pod.
* `limits`: Are values that cannot be exceeded

:information_source: You can use `explain` to have a look to the documentation of resources.

```console
kubectl explain --recursive pod.spec.containers.resources.limits
KIND:     Pod
VERSION:  v1

FIELD:    limits <map[string]string>

DESCRIPTION:
     Limits describes the maximum amount of compute resources allowed. More
...
```

The wordpress we've created in the [previous lab](03_wordpress.md) doesn't have resources definition.
There are different ways to edit its current state (`kubectl edit`, `apply`, `patch` ...)

```console
kubectl edit deploy wordpress
```

replace `resources: {}` with this block

```yaml
...
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 1000m
            memory: 200Mi
...
```

The pods resources usage can be displayed using (this might take a few seconds)

```console
kubectl top pods
NAME                               CPU(cores)   MEMORY(bytes)
wordpress-694866c6b7-mqxdd         1m           171Mi
wordpress-mysql-6c597b98bd-4mbbd   1m           531Mi
```

Configure the autoscaling base on cpu usage. When a pod reaches 50% of its allocated cpu a new pod is created.

```console
kubectl autoscale deployment wordpress --cpu-percent=50 --min=1 --max=5
horizontalpodautoscaler.autoscaling/wordpress autoscaled
```

It takes up to 15 seconds (default configuration) to get the first values

```console
kubectl get hpa
NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   <unknown>/50%   1         5         0          10s

kubectl get hpa
NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   1%/50%    1         5         1          20s
```

Now we'll run an HTTP bench using [wrk](https://github.com/wg/wrk). Open a **new shell** and run

```console
kubectl run -ti --rm bench --image=jess/wrk -- /bin/sh -c 'wrk -t12 -c100 -d180s http://wordpress'
```

During the benchmark above (3 minutes duration) let's have a look to the hpa

```console
watch kubectl get hpa
Every 2.0s: kubectl get hpa
hostname: Tue Jun 22 11:13:08 2021

NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   1%/50%    1         5         1          8m28s
```

After a few seconds we'll see that the upscaling will be done automatically. Here the number of replicas will reach the maximum we defined (5 pods).

```console
Every 2.0s: kubectl get hpa
hostname: Tue Jun 22 11:14:13 2021

NAME        REFERENCE              TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   998%/50%   1         5         5          9m33s
```

That was a pretty simple configuration, basing the autoscaling on CPU usage for a webserver makes sense. You can also base the autoscaling on any [other metrics](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics) that are reported by your application.

:arrow_right: [Next: Troubleshooting](/post/series/workshop_kubernetes/troubleshoot/)
