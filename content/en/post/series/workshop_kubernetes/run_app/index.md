---
title: 'Run an application on Kubernetes'
description: 'Deploy an application on Kubernetes. Learn what is a pod or a deployment using the kubectl CLI'
summary: 'Run an application, kubectl commands to create a pod, a deployment'
date: "2021-05-02"
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

### Namespaces

Namespaces allow to logically distribute your applications, generally based on teams, projects or applications stacks.
Resources names are unique within a namespace. That means that you could have a service named `webserver` on 2 different namespaces.

They can be used to isolate applications using network policies, or to define quotas.

For this training we'll work on a namespace named `foo`

```console
kubectl create ns foo
namespace/foo created
```

And we'll make use of the plugin `ns` installed in the previous section to set the default namespace as follows

```console
kubectl ns
Context "k3d-workshop" modified.
Active namespace is "foo".
```

Check that your `kubectl` is properly configured, here you can see the cluter and the namespace:

```console
kubectl config get-contexts
CURRENT   NAME           CLUSTER        AUTHINFO             NAMESPACE
*         k3d-workshop   k3d-workshop   admin@k3d-workshop   foo
```

### Create your first pod

Creating resources in Kubernetes is often done by applying a yaml/json definition through the API.

First of all we need to clone this repository and change the current path to its root

```console
git clone https://github.com/Smana/workshop_kubernetes_2021.git

cd workshop_kubernetes_2021.git
```

Start by creating a pretty simple `pod`:

```console
kubectl apply -f content/resources/kubernetes_workshop/pod.yaml --namespace foo
pod/web created

kubectl get po
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          98s
```

We can get detailed information about the pod as follows

```console
kubectl describe po web
Name:         web
Namespace:    foo
Priority:     0
Node:         k3d-workshop-agent-0/172.20.0.3
Start Time:   Fri, 18 Jun 2021 17:05:46 +0200
Labels:       run=web
Annotations:  <none>
Status:       Running
IP:           10.42.1.6
...
```

Or even get a specific attribute, here is an example to get the pod's IP

```console
kubectl get po web --template={{.status.podIP}}
10.42.1.6
```

This is worth noting that a pod isn't controlled by a `replicaset-controller`. That means that when it is deleted, it is not restarted automatically.

```console
kubectl delete po web
pod "web" deleted

kubectl get po
No resources found in foo namespace.
```

### A pod with 2 containers

Now create a new pod using the manifest `content/resources/kubernetes_workshop/pod2containers.yaml`. Look at its content, we will be using a shared temporary directory and we'll mount its content on both containers. That way we can share data between 2 containers of a given pod.

```console
kubectl apply -f content/resources/kubernetes_workshop/pod2containers.yaml
pod/web created

kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
web    2/2     Running   0          36s
```

We can check that the logs are accessible on the 2 containers

```console
kubectl logs web -c logger --tail=6 -f
Mon Jun 28 21:06:20 2021
Mon Jun 28 21:06:21 2021
Mon Jun 28 21:06:22 2021
Mon Jun 28 21:06:23 2021
Mon Jun 28 21:06:24 2021

kubectl exec web -c web -- tail -n 5 /log/out.log
Mon Jun 28 21:07:19 2021
Mon Jun 28 21:07:20 2021
Mon Jun 28 21:07:21 2021
Mon Jun 28 21:07:22 2021
Mon Jun 28 21:07:23 2021
```

Delete the pod

```console
kubectl delete po web
pod "web" deleted
```

### Create a simple webserver deployment

A `deployment` is a resource that describes the desired state of an application. Kubernetes will ensure that its **current** status is aligned with the **desired** one.

Creating a simple deployment can be done using `kubectl`

```console
kubectl create deployment podinfo --image stefanprodan/podinfo
deployment.apps/podinfo created
```

After a few seconds the deployment will be up to date, meaning that the a pod is up and running.

```console
kubectl get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
podinfo   1/1     1            1           14s
```

### Replicas and scaling

A deployment creates a `replicaset` under the hood in order to ensure that the number of replicas (pods) matches the desired one.

```console
kubectl get replicasets
NAME                 DESIRED   CURRENT   READY   AGE
podinfo-7fbb45ccfc   1         1         1       36s
```

Creating a deployment without specifying the number of replicas will create a single replica. We can scale it on demand using

```console
kubectl scale deploy podinfo --replicas 6
deployment.apps/podinfo scaled

kubectl rollout status deployment podinfo
Waiting for deployment "podinfo" rollout to finish: 4 of 6 updated replicas are available...
Waiting for deployment "podinfo" rollout to finish: 5 of 6 updated replicas are available...
deployment "podinfo" successfully rolled out
```

The default Kubernetes scheduler will try to spread evenly the pods according to the available resources on worker nodes.

```console
kubectl get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP           NODE                    NOMINATED NODE   READINESS GATES
podinfo-7fbb45ccfc-dwxtx   1/1     Running   0          114s   10.42.1.8    k3d-workshop-agent-0    <none>           <none>
podinfo-7fbb45ccfc-p2djv   1/1     Running   0          34s    10.42.1.11   k3d-workshop-agent-0    <none>           <none>
podinfo-7fbb45ccfc-4fk9z   1/1     Running   0          34s    10.42.1.9    k3d-workshop-agent-0    <none>           <none>
podinfo-7fbb45ccfc-gqwz6   1/1     Running   0          34s    10.42.1.10   k3d-workshop-agent-0    <none>           <none>
podinfo-7fbb45ccfc-4qgvs   1/1     Running   0          34s    10.42.0.8    k3d-workshop-server-0   <none>           <none>
podinfo-7fbb45ccfc-r6dn5   1/1     Running   0          34s    10.42.0.9    k3d-workshop-server-0   <none>           <none>
```

The deployment controller will ensure to start new pods if the number of replicas doesn't match its configuration.

```console
kubectl delete po $(kubectl get po -l app=podinfo -o jsonpath='{.items[0].metadata.name}')
pod "podinfo-7fbb45ccfc-r6dn5" deleted

kubectl describe rs podinfo-7fbb45ccfc
Name:           podinfo-7fbb45ccfc
Namespace:      foo
Selector:       app=podinfo,pod-template-hash=7fbb45ccfc
Labels:         app=podinfo
                pod-template-hash=7fbb45ccfc
Annotations:    deployment.kubernetes.io/desired-replicas: 6
                deployment.kubernetes.io/max-replicas: 8
                deployment.kubernetes.io/revision: 5
                deployment.kubernetes.io/revision-history: 1,3
Controlled By:  Deployment/podinfo
Replicas:       6 current / 6 desired
Pods Status:    6 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
Events:
  Type    Reason            Age                From                   Message
  ----    ------            ----               ----                   -------
...
  Normal  SuccessfulCreate  16h (x3 over 16h)  replicaset-controller  (combined from similar events): Created pod: podinfo-7fbb45ccfc-pkt4r
  Normal  SuccessfulCreate  96s                replicaset-controller  Created pod: podinfo-7fbb45ccfc-bkm8n

kubectl get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
podinfo   6/6     6            6           18m
```

### Rolling update

Using a deployment allows to manage the application lifecycle. Changing its configuration will trigger a **rolling update**.

First of all we'll change the image tag of our deployment

```console
kubectl set image deployment podinfo podinfo=stefanprodan/podinfo:5.2.1
deployment.apps/podinfo image updated
```

During a rolling update a new `replicaset` is created in order to update the application in place without any downtime.
New pods (with the current deployment state) will be created in the new replicaset while they will be deleted progressively from the previous replicaset.

```console
kubectl get rs -o wide
NAME                 DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                       SELECTOR
podinfo-7fbb45ccfc   0         0         0       21m   podinfo      stefanprodan/podinfo         app=podinfo,pod-template-hash=7fbb45ccfc
podinfo-564b4ddd7c   6         6         6       30s   podinfo      stefanprodan/podinfo:5.2.1   app=podinfo,pod-template-hash=564b4ddd7c
```

Keeping the old replicaset makes very easy to rollback

```console
kubectl rollout undo deployment podinfo
deployment.apps/podinfo rolled back

kubectl get rs -o wide
NAME                 DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                       SELECTOR
podinfo-7fbb45ccfc   6         6         6       22m   podinfo      stefanprodan/podinfo         app=podinfo,pod-template-hash=7fbb45ccfc
podinfo-564b4ddd7c   0         0         0       77s   podinfo      stefanprodan/podinfo:5.2.1   app=podinfo,pod-template-hash=564b4ddd7c
```

## Expose a deployment

Now that we have a running web application we may want to access it.
There are several ways to expose an app, here we'll use the easiest way: Create a service and run a `port-forward`.

The following command will create a `service` which will be in charge of forwarding calls through the tcp port 9898

```console
kubectl expose deploy podinfo --port 9898
service/podinfo exposed
```

We can get more information on the service as follows

```console
kubectl get svc -o yaml podinfo
apiVersion: v1
kind: Service
metadata:
  labels:
    app: podinfo
  name: podinfo
  namespace: foo
spec:
  clusterIP: 10.43.47.17
  clusterIPs:
  - 10.43.47.17
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 9898
  selector:
    app: podinfo
```

A service uses the `selector` above to identify on which pod to forward the traffic and usually creates the `endpoints` accordingly.

```console
kubectl get po -l app=podinfo
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-7fbb45ccfc-bkm8n   1/1     Running   1          146m
podinfo-7fbb45ccfc-sbqht   1/1     Running   2          18h
...

kubectl get endpoints
NAME      ENDPOINTS                                                     AGE
podinfo   10.42.0.16:9898,10.42.0.17:9898,10.42.1.18:9898 + 3 more...   92s
```

The service we've created has an IP that's only accessible from within the cluster. Using the `port-forward` command we're able to forward the traffic from our local machine to the application (through the API server).
Note that you can target either a deployment, a service or a single pod

```console

kubectl port-forward svc/podinfo 9898 &
Forwarding from 127.0.0.1:9898 -> 9898
Forwarding from [::1]:9898 -> 9898

curl http://localhost:9898
Handling connection for 9898
{
  "hostname": "podinfo-7fbb45ccfc-sbqht",
  "version": "6.0.0",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.5",
  "num_goroutine": "6",
  "num_cpu": "16"
}
```

## Cleanup

In this section we created 2 resources: a deployment and a service.

```console
fg
kubectl port-forward svc/podinfo 9898
^C

kubectl delete svc,deploy podinfo
service "podinfo" deleted
deployment.apps "podinfo" deleted
```

:arrow_right: [Next: Deploy a Wordpress](/post/series/workshop_kubernetes/application_stack/)
