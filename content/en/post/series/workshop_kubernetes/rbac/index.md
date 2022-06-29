---
title: 'Kubernetes workshop: Manage permissions in Kubernetes'
description: 'Assign permissions to the Kubernetes API resources'
summary: 'Assign permissions to the Kubernetes API resources'
date: "2021-05-08"
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

[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) is the method used by Kubernetes to authorize access to API resources.

:information_source: When it makes sense you can use the [default roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) that are available in all Kubernetes installation instead of having to maintain custom ones.

For this lab what we want to achieve is to give permissions the following permissions to an application `myapp`:

* read the configmaps in the namespace `foo`
* List the pods in all the namespaces

It is a good practice to configure your pod to make use of a `serviceaccount`. A serviceaccount are used to identify applications and give them permissions if necessary.

Create a service account

```console
kubectl create serviceaccount myapp
serviceaccount/myapp created
```

When a service account is created, a `token` is automatically generated and stored in a secret.

```console
kubectl describe sa myapp | grep -i token
Mountable secrets:   myapp-token-bz2zq
Tokens:              myapp-token-bz2zq

kubectl get secret myapp-token-bz2zq --template={{.data.token}} | base64 -d
eyJhb...EYxhjI_ckZ74A
```

Using a tool to decode the JWT token you should see the following content

```console
kubectl get secret myapp-token-bz2zq --template={{.data.token}} | base64 -d | jwt decode -

Token header
------------
{
  "alg": "RS256",
  "kid": "IdsXYO6E93xozgJg-LY2oETTPEHBJjydTU4vF2wy-wg"
}

Token claims
------------
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "foo",
  "kubernetes.io/serviceaccount/secret.name": "myapp-token-bz2zq",
  "kubernetes.io/serviceaccount/service-account.name": "myapp",
  "kubernetes.io/serviceaccount/service-account.uid": "eb606bdc-b713-4b7c-8da8-c4f71075995e",
  "sub": "system:serviceaccount:foo:myapp"
}
```


We're going to create a deployment that will be configured to used this serviceaccount. In the yaml you'll notice that we defined the `serviceAccountName`.

```console
kubectl apply -f content/resources/kubernetes_workshop/rbac/deployment.yaml
deployment.apps/myapp created
```

As we didn't assigned any permissions to this serviceaccount, our application won't be able to call any of the API endpoints

```console
POD_NAME=$(kubectl get po -l app=myapp -o jsonpath='{.items[0].metadata.name}')

kubectl exec ${POD_NAME} -- kubectl auth can-i -n foo --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
                                                [/api]                                []               [get]
                                                [/apis/*]                             []               [get]
                                                [/apis]                               []               [get]
                                                [/healthz]                            []               [get]
                                                [/healthz]                            []               [get]
                                                [/livez]                              []               [get]
                                                [/livez]                              []               [get]
                                                [/openapi/*]                          []               [get]
                                                [/openapi]                            []               [get]
                                                [/openid/v1/jwks]                     []               [get]
                                                [/readyz]                             []               [get]
                                                [/readyz]                             []               [get]
                                                [/version/]                           []               [get]
                                                [/version/]                           []               [get]
                                                [/version]                            []               [get]
                                                [/version]                            []               [get]
```

In order to allow it to read configmaps in the namespace foo, we're going to create 2 resources:
* A **role** which will describe the permissions and which is bounded to a namespace
* A **rolebinding** to assign this role to our application (serviceaccount)

Create the role

```console
kubectl apply -f content/resources/kubernetes_workshop/rbac/role.yaml
role.rbac.authorization.k8s.io/read-configmaps created
```

And assign it to the serviceaccount we've created previously

```console
kubectl create rolebinding -n foo myapp-configmap --serviceaccount=foo:myapp --role=read-configmaps
rolebinding.rbac.authorization.k8s.io/myapp-configmap created
```

Note that in the above command the serviceaccount must be specified with the namespace as a prefix and separated by a semicolon.

You don't have to restart the pod to get the permissions enabled.

```console
kubectl exec ${POD_NAME} -- kubectl auth can-i get configmaps -n foo
yes

kubectl exec ${POD_NAME} -- kubectl get cm
NAME               DATA   AGE
kube-root-ca.crt   1      3d20h
helloworld         2      2d2h
```

This is possible thanks to the token mounted within the container

```console
kubectl exec -ti ${POD_NAME} -- bash -c 'curl -skH "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default/api/v1/namespaces/foo/configmaps'
{
  "kind": "ConfigMapList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "76334"
  },
  "items": [
    {
      "metadata": {
        "name": "kube-root-ca.crt",
        "namespace": "foo",
        "uid": "c352e4cd-3b88-4400-80a0-cbba318794e4",
...
```

Finally we want to list the pods in all the namespaces of our cluster.
we need:

* A **clusterrole** which will describe the permissions that are cluster wide.
* A **clusterrolebinding** to assign this clusterrole to our application (serviceaccount)

```console
$ kubectl apply -f content/resources/kubernetes_workshop/rbac/clusterrole.yaml
clusterrole.rbac.authorization.k8s.io/list-pods created
```

```console
kubectl create clusterrolebinding -n foo myapp-pods --serviceaccount=foo:myapp --clusterrole=list-pods
clusterrolebinding.rbac.authorization.k8s.io/myapp-pods created
```

Now lets have a look to the permissions our applications has in the namespace `foo`

```console
kubectl exec ${POD_NAME} -- kubectl auth can-i -n foo --list
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
pods                                            []                                    []               [get list]
configmaps                                      []                                    []               [get watch list]
                                                [/.well-known/openid-configuration]   []               [get]
                                                [/api/*]                              []               [get]
                                                [/api]                                []               [get]
                                                [/apis/*]                             []               [get]
...
```