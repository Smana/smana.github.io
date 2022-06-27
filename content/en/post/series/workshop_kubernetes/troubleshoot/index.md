---
title: 'Kubernetes workshop: Troubleshooting'
description: 'Troubleshooting command lines'
summary: 'Tips and tricks when something goes wrong'
date: "2021-05-07"
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

## Events

The first source of information when something goes wrong is the event stream. Note that you may want to sort them by creation time

```console
kubectl get events -n foo --sort-by=.metadata.creationTimestamp
...
20m         Normal    Created             pod/web-85575f4476-5pbqv    Created container nginx
20m         Normal    Started             pod/web-85575f4476-5pbqv    Started container nginx
20m         Normal    SuccessfulDelete    replicaset/web-987f6cf9     Deleted pod: web-987f6cf9-mzsxd
20m         Normal    ScalingReplicaSet   deployment/web              Scaled down replica set web-987f6cf9 to 0
```

## Logs

Having a look to a pod's logs is just the matter of running

```console
kubectl logs -f --tail=7 -c mysql wordpress-mysql-6c597b98bd-4mbbd
2021-06-24 08:27:38 1 [Note]   - '::' resolves to '::';
2021-06-24 08:27:38 1 [Note] Server socket created on IP: '::'.
2021-06-24 08:27:38 1 [Warning] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.
2021-06-24 08:27:38 1 [Warning] 'proxies_priv' entry '@ root@wordpress-mysql-6c597b98bd-4mbbd' ignored in --skip-name-resolve mode.
2021-06-24 08:27:38 1 [Note] Event Scheduler: Loaded 0 events
2021-06-24 08:27:38 1 [Note] mysqld: ready for connections.
Version: '5.6.51'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```

Alternatively you can use a tool made to display logs from multiple pods: [stern](https://github.com/wercker/stern).
A better way to explore logs is to send them to a central location using a tool such as [Loki](https://github.com/grafana/loki) or the well know EFK stack.

## Health checks

Kubernetes self healing system is mostly based on `health checks`. There are different types of health checks (please have a look to the [official documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

We'll add a new plugin to `kubectl` which is really useful to export a resource while cleaning useless metadatas: `neat`

```console
kubectl krew install neat
Updated the local copy of plugin index.
Installing plugin: neat
Installed plugin: neat
...
```

Let's create a new deployment using the image `nginx`

```console
kubectl create deploy web --image=nginx --dry-run=client -o yaml | kubectl neat > /tmp/web.yaml
```

Edit its content and add an HTTP health check on port 80. The endpoint must return a code ranging between 200 and 400 and it has to be a **relevant test** that shows the actual availability of the service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web
    spec:
      containers:
      - image: nginx
        name: nginx
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 3
```

```console
kubectl apply -f /tmp/web.yaml
deployment.apps/web created

kubectl describe deploy web | grep Liveness:
    Liveness:     http-get http://:80/ delay=3s timeout=1s period=3s #success=1 #failure=3
```

The pod should be up without any error

```console
kubectl get po -l app=web
NAME                   READY   STATUS    RESTARTS   AGE
web-85575f4476-6qvd5   1/1     Running   0          92s
```

We're going to simulate a service being unavailable, just change the path being checked. Here we'll use another method to modify a resource by creating a `patch` and applying it.

Create a yaml `/tmp/patch.yaml` file

```console
cat > /tmp/patch.yaml <<EOF
spec:
  template:
    spec:
      containers:
      - name: nginx
        livenessProbe:
          httpGet:
            path: /foobar
EOF
```

And we're going to apply our change as follows

```console
kubectl patch deployment web --patch "$(cat /tmp/patch.yaml)" --record
deployment.apps/web patched

kubectl describe deployment web | grep Liveness:
    Liveness:     http-get http://:80/foobar delay=3s timeout=1s period=3s #success=1 #failure=3
```

Now our pod should start to fail, the number of restarts increases

```console
kubectl get po -l app=web
web-987f6cf9-n4rnb                 1/1     Running   4          83s
```

Until the pod enter in a `CrashLoopBackOff`, meaning that it constantly restarts.

```console
kubectl get po -l app=web
NAME                 READY   STATUS             RESTARTS   AGE
web-987f6cf9-n4rnb   0/1     CrashLoopBackOff   5          3m23s
```

Describing the pod will give you a hint on the reason it restarts

```console
kubectl describe po web-987f6cf9-n4rnb | tail -n 5
Normal   Created    4m7s (x3 over 4m30s)   kubelet            Created container nginx
Normal   Started    4m7s (x3 over 4m30s)   kubelet            Started container nginx
Warning  Unhealthy  3m56s (x9 over 4m26s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    3m56s (x3 over 4m20s)  kubelet            Container nginx failed liveness probe, will be restarted
Normal   Pulling    3m56s (x4 over 4m35s)  kubelet            Pulling image "nginx"
```

Rollback the latest change in order to return to a working state.
Note that we used the option `--record` when we applied the patch. That helps saving changes history.

```console
kubectl rollout history deployment web
deployment.apps/web
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl patch deployment web --patch=spec:
  template:
    spec:
      containers:
      - name: nginx
        livenessProbe:
          httpGet:
            path: /foobar --record=true

kubectl rollout undo deployment web
deployment.apps/web rolled back
```

## Cleanup

```console
kubectl delete deploy web
deployment.apps "web" deleted
```

## learnk8s documentation

There is a great documentation that contains all the steps that help debugging a deployment: [https://learnk8s.io/troubleshooting-deployments](https://learnk8s.io/troubleshooting-deployments)

:arrow_right: [Next: RBAC](/post/series/workshop_kubernetes/rbac/)
