+++
author = "Smaine Kahlouch"
title = "Applying GitOps Principles to Infrastructure: An overview of `tf-controller`"
date = "2023-06-01"
summary = "**Weave tf-controller** is a Kubernetes operator that allows the execution of Terraform code and provides a few missing features (reconciliation, drift detection, etc.). Let's have a deeper look at it 🔍"
featureImage = "tf-controller.png"
featured = true
codeMaxLines = 20
usePageBundles = true
toc = true
tags = [
    "infrastructure"
]
thumbnail= "thumbnail.png"
+++

**Terraform** is probably the most used "Infrastructure As Code" tool for building, modifying, and versioning Cloud infrastructure changes.
It is an Open Source project developed by Hashicorp that uses the [HCL](https://github.com/hashicorp/hcl) language to declare the desired state of Cloud resources.
The state of the created resources is stored in a file called terraform state.

Terraform can be considered a "semi-declarative" tool as there is no built-in **automatic reconciliation** feature. There are several solutions to address this issue, but generally speaking, a modification will be applied using `terraform apply`. The code is actually written using the HCL configuration files (declarative), but the execution is done imperatively.
As a result, there can be a drift between the declared and actual state (for example, a colleague who would have changed something directly into the console 😉).

❓❓ So, how can I ensure that what is committed using Git is really applied. How to be notified if there is a change compared to the desired state and how to automatically apply what is in my code (GitOps)?

This is the promise of [**tf-controller**](https://github.com/weaveworks/tf-controller), an Open Source Kubernetes operator from Weaveworks, tightly related to Flux (a GitOps engine from the same company). [**Flux**](https://fluxcd.io/) is one of the solutions I really appreciate, that's why I invite you to have a look on my [previous article](https://blog.ogenki.io/post/devflux/)


{{% notice info Info %}}
All the steps described in this article come from this [**Git repo**](https://github.com/Smana/demo-tf-controller)
{{% /notice %}}

## 🎯 Our target

By following the steps in this article, we aim to achieve the following things:

* Deploy a **Control plane** EKS cluster. Long story short, it will host the Terraform controller that will be in charge of managing all the desired infrastructure components.
* Use **Flux** as the GitOps engine for all Kubernetes resources.

Regarding the Terraform controller, we will see:

* How to define **dependencies** between modules
* Creation of **several AWS resources**: Route53 Zone, ACM Certificate, network, EKS cluster.
* The different **reconciliation** options (automatic, requiring confirmation)
* How to backup and **restore** a Terraform state.

## 🛠️ Install the tf-controller

### ☸ The control plane

In order to be able to use the Kubernetes controller `tf-controller`, we first need a Kubernetes cluster 😆.
So we are going to create a **control plane** cluster using the `terraform` command line and EKS best practices.

{{% notice warning Warning %}}
It is crucial that this cluster is resilient, secure, and supervised as it will be responsible for managing all the AWS resources created subsequently.
{{% /notice %}}

Without going into detail, the control plane cluster was created using [this code](https://github.com/Smana/demo-tf-controller/tree/main/terraform/controlplane). That said, it is important to note that all application deployment operations are done using Flux.

{{% notice info Info %}}

By following the instructions in the [README](https://github.com/Smana/demo-tf-controller/blob/main/terraform/controlplane/README.md), an EKS cluster will be created but not only! </br>
Indeed, it is required to give permissions to the Terraform controller so it will able to apply infrastructure changes.
Furthermore, Flux must be installed and configured to apply the configuration defined [here](https://github.com/Smana/demo-tf-controller/tree/main/clusters/controlplane-0).

We end up with several components installed and configured:

* The almost unavoidable addons: `aws-loadbalancer-controller` and `external-dns`
* [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) roles for these same components are installed using `tf-controller`
* Prometheus / Grafana monitoring stack.
* `external-secrets` to be able to retrieve sensitive data from AWS secretsmanager.

To demonstrate all this after a few minutes the web interface for Flux is accessible via the URL `gitops-<cluster_name>.<domain_name`>

Still you should check that Flux is working properly

```console
aws eks update-kubeconfig --name controlplane-0 --alias controlplane-0
Updated context controlplane-0 in /home/smana/.kube/config
```

```console
flux check
...
✔ all checks passed

flux get kustomizations
NAME                    REVISION                SUSPENDED       READY   MESSAGE
flux-config             main@sha1:e2cdaced      False           True    Applied revision: main@sha1:e2cdaced
flux-system             main@sha1:e2cdaced      False           True    Applied revision: main@sha1:e2cdaced
infrastructure          main@sha1:e2cdaced      False           True    Applied revision: main@sha1:e2cdaced
security                main@sha1:e2cdaced      False           True    Applied revision: main@sha1:e2cdaced
tf-controller           main@sha1:e2cdaced      False           True    Applied revision: main@sha1:e2cdaced
...
```

{{% /notice %}}

### 📦 The Helm chart and Flux

Now that the control plane cluster is available we can **add the Terraform controller** and this is just the matter of using the Helm chart as follows.

We must declare the its `Source` first:

[source.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/base/tf-controller/source.yaml)
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: tf-controller
spec:
  interval: 30m
  url: https://weaveworks.github.io/tf-controller
```

Then we need to define the `HelmRelease`:

[release.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/base/tf-controller/release.yaml)
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: tf-controller
spec:
  releaseName: tf-controller
  chart:
    spec:
      chart: tf-controller
      sourceRef:
        kind: HelmRepository
        name: tf-controller
        namespace: flux-system
      version: "0.12.0"
  interval: 10m0s
  install:
    remediation:
      retries: 3
  values:
    resources:
      limits:
        memory: 1Gi
      requests:
        cpu: 200m
        memory: 500Mi
    runner:
      serviceAccount:
        annotations:
          eks.amazonaws.com/role-arn: "arn:aws:iam::${aws_account_id}:role/tfcontroller_${cluster_name}"
```

When this change is actually written into Git, the HelmRelease will be deployed and the `tf-controller` started:

```console
kubectl get hr -n flux-system
NAME            AGE   READY   STATUS
tf-controller   67m   True    Release reconciliation succeeded

kubectl get po -n flux-system -l app.kubernetes.io/instance=tf-controller
NAME                             READY   STATUS    RESTARTS   AGE
tf-controller-7ffdc69b54-c2brg   1/1     Running   0          2m6s
```

In this demo, there are already a several AWS resources declared. Therefore, after a few minutes, the cluster takes care of creating these:
[![asciicast](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq.png)](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq?&speed=2)

{{% notice info Info %}}
Although the majority of operations are performed declaratively or via the CLIs `kubectl` and `flux`, another tool allows to manage Terraform resources: [tfctl](https://docs.gitops.weave.works/docs/terraform/tfctl/)
{{% /notice %}}

## 🚀 Apply a change

One of the Terraform's [best practices](https://www.terraform-best-practices.com/) is to use **[modules](https://developer.hashicorp.com/terraform/language/modules)**.</br>
A module is a set of logically linked Terraform resources bundled into a single reusable unit. They allow to abstract complexity, take inputs, perform specific actions, and produce outputs.

You can create your own modules and make them available as `Sources` or use the many modules shared and maintained by communities.</br>
You just need to specify a few `variables` in order to fit to your context.

With `tf-controller`, the first step is therefore to define the `Source` of the module. Here we are going to configure the AWS base networking components (vpc, subnets...) using the [terraform-aws-vpc](https://github.com/terraform-aws-modules/terraform-aws-vpc) module.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: terraform-aws-vpc
  namespace: flux-system
spec:
  interval: 30s
  ref:
    tag: v5.0.0
  url: https://github.com/terraform-aws-modules/terraform-aws-vpc
```

Then we can make use of this Source within a `Terraform` resource:

[vpc/dev.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/controlplane-0/terraform/custom-resources/vpc/dev.yaml)

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: vpc-dev
spec:
  interval: 8m
  path: .
  destroyResourcesOnDeletion: true # You wouldn't do that on a prod env ;)
  storeReadablePlan: human
  sourceRef:
    kind: GitRepository
    name: terraform-aws-vpc
    namespace: flux-system
  vars:
    - name: name
      value: vpc-dev
    - name: cidr
      value: "10.42.0.0/16"
    - name: azs
      value:
        - "eu-west-3a"
        - "eu-west-3b"
        - "eu-west-3c"
    - name: private_subnets
      value:
        - "10.42.0.0/19"
        - "10.42.32.0/19"
        - "10.42.64.0/19"
    - name: public_subnets
      value:
        - "10.42.96.0/24"
        - "10.42.97.0/24"
        - "10.42.98.0/24"
    - name: enable_nat_gateway
      value: true
    - name: single_nat_gateway
      value: true
    - name: private_subnet_tags
      value:
        "kubernetes.io/role/elb": 1
        "karpenter.sh/discovery": dev
    - name: public_subnet_tags
      value:
        "kubernetes.io/role/elb": 1
  writeOutputsToSecret:
    name: vpc-dev
```

In summary: the terraform code from the `terraform-aws-vpc` source is applied using the variables defined within `vars`.

There are then several parameters that influence the `tf-controller` behavior. The main parameters that control how modifications are applied are `.spec.approvePlan` and `.spec.autoApprove`

### 🚨 Drift detection

Setting `spec.approvePlan` to `disable` only notifies that the current state of resources has drifted from the Terraform code.
This allows you to choose when and how to apply the changes.

{{% notice note Note %}}
This is worth noting that there is a missing section on **notifications**: Drift, pending plans, reconciliation problems. I'm trying to identify possible methods (preferably with Prometheus) and update this article as soon as possible.
{{% /notice %}}

### 🔧 Manual execution

The given example above (`vpc-dev`) does not contain the `.spec.approvePlan` parameter and therefore inherits the default value which is `false`.
In other words, the actual execution of changes (`apply`) is not done automatically.

A `plan` is executed and will be waiting for validation:

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
...
flux-system     vpc-dev                         Unknown Plan generated: set approvePlan: "plan-v5.0.0-26c38a66f12e7c6c93b6a2ba127ad68981a48671" to approve this plan.      true            2 minutes
```

I also advise to configure the `storeReadablePlan` parameter to `human`. This allows you to easily visualize the pending modifications using `tfctl`:

```console
tfctl show plan vpc-dev

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_default_network_acl.this[0] will be created
  + resource "aws_default_network_acl" "this" {
      + arn                    = (known after apply)
      + default_network_acl_id = (known after apply)
      + id                     = (known after apply)
      + owner_id               = (known after apply)
      + tags                   = {
          + "Name" = "vpc-dev-default"
        }
      + tags_all               = {
          + "Name" = "vpc-dev-default"
        }
      + vpc_id                 = (known after apply)

      + egress {
          + action          = "allow"
          + from_port       = 0
          + ipv6_cidr_block = "::/0"
          + protocol        = "-1"
          + rule_no         = 101
          + to_port         = 0
        }
      + egress {
...
Plan generated: set approvePlan: "plan-v5.0.0@sha1:26c38a66f12e7c6c93b6a2ba127ad68981a48671" to approve this plan.
To set the field, you can also run:

  tfctl approve vpc-dev -f filename.yaml
```

After reviewing the above modifications, you just need to add the identifier of the `plan` to validate and push the change to git as follows:

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: vpc-dev
spec:
...
  approvePlan: plan-v5.0.0-26c38a66f1
...
```

After a few seconds, a `runner` will be launched and will apply the changes:

```console
kubectl logs -f -n flux-system vpc-dev-tf-runner
2023/07/01 15:33:36 Starting the runner... version  sha
...
aws_vpc.this[0]: Creating...
aws_vpc.this[0]: Still creating... [10s elapsed]
...
aws_route_table_association.private[1]: Creation complete after 0s [id=rtbassoc-01b7347a7e9960a13]
aws_nat_gateway.this[0]: Still creating... [10s elapsed]
```

As soon as the apply is finished the status of the Terraform resource becomes "READY"

```console
kubectl get tf -n flux-system vpc-dev
NAME      READY   STATUS                                                                  AGE
vpc-dev   True    Outputs written: v5.0.0@sha1:26c38a66f12e7c6c93b6a2ba127ad68981a48671   17m
```

### 🤖 Automatic reconciliation

We can also enable **automatic reconciliation**. To do this, set the `.spec.autoApprove` parameter to `true`.

All IRSA resources are configured in this way:

[external-secrets.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/controlplane-0/terraform/irsa/base/external-secrets.yaml)
```yaml
piVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: irsa-external-secrets
spec:
  approvePlan: auto
  destroyResourcesOnDeletion: true
  interval: 8m
  path: ./modules/iam-role-for-service-accounts-eks
  sourceRef:
    kind: GitRepository
    name: terraform-aws-iam
    namespace: flux-system
  vars:
    - name: role_name
      value: ${cluster_name}-external-secrets
    - name: attach_external_secrets_policy
      value: true
    - name: oidc_providers
      value:
        main:
          provider_arn: ${oidc_provider_arn}
          namespace_service_accounts: ["security:external-secrets"]
```

So if I make any change on the AWS console for example, it will be quickly **overwritten** by the one managed by `tf-controller`.

{{% notice info Info %}}
The deletion policy of components created by a Terraform resource is controlled by the setting `destroyResourcesOnDeletion`. By default anything created is not destroyed by the controller. If you want to destroy the resources when the `Terraform` object is deleted you must set this parameter to `true`.

Here we want to be able to delete IRSA roles because they're tightly linked to a given EKS cluster
{{% /notice %}}

### 🔄 Inputs/Outputs and modules dependencies

When using Terraform, we often need to share data from one module to another. This is done using the [**outputs**](https://developer.hashicorp.com/terraform/language/values/outputs) that are defined within modules. </br>
So we need a way to store them somewhere and import them into another module.

Let's take again the given example above (`vpc-dev`). We can see at the bottom of the YAML file, the following block:

```yaml
...
  writeOutputsToSecret:
    name: vpc-dev
```

When this resource is applied, we will get a message confirming that the outputs are available ("Outputs written"):

```console
kubectl get tf -n flux-system vpc-dev
NAME      READY   STATUS                                                                  AGE
vpc-dev   True    Outputs written: v5.0.0@sha1:26c38a66f12e7c6c93b6a2ba127ad68981a48671   17m
```

Indeed this module exports many information (126).

```console
kubectl get secrets -n flux-system vpc-dev
NAME      TYPE     DATA   AGE
vpc-dev   Opaque   126    15s

kubectl get secret -n flux-system vpc-dev --template='{{.data.vpc_id}}' | base64 -d
vpc-0c06a6d153b8cc4db
```

Some of these are then used to create a dev EKS cluster. Note that you don't have to read them all, you can cherry pick a few chosen outputs from the secret:

[vpc/dev.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/controlplane-0/terraform/custom-resources/vpc/dev.yaml)
```yaml
...
  varsFrom:
    - kind: Secret
      name: vpc-dev
      varsKeys:
        - vpc_id
        - private_subnets
...
```

## 💾 Backup and restore a tfstate

For my demos, I don't want to recreate the zone and the certificate each time the control plane is destroyed (The DNS propagation and certificate validation take time). Here is an example of the steps to take so that I can **restore** the state of these resources when I use this demo.

{{% notice note Note %}}
This is a manual procedure to demonstrate the behavior of `tf-controller` with respect to state files. By default, these `tfstates` are stored in `secrets`, but we would prefer to configure a GCS or S3 backend.
{{% /notice %}}

The initial creation of the demo environment allowed me to save the state files (tfstate) as follows.

```console
WORKSPACE="default"
STACK="route53-cloud-hostedzone"
BACKUPDIR="${HOME}/tf-controller-backup"

mkdir -p ${BACKUPDIR}

kubectl get secrets -n flux-system tfstate-${WORKSPACE}-${STACK} -o jsonpath='{.data.tfstate}' | \
base64 -d > ${BACKUPDIR}/${WORKSPACE}-${STACK}.tfstate.gz
```

When the cluster is created again, tf-controller tries to create the zone because the state file is empty.

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
...
flux-system     route53-cloud-hostedzone        Unknown Plan generated: set approvePlan: "plan-main@sha1:345394fb4a82b9b258014332ddd556dde87f73ab" to approve this plan.        true            16 minutes

tfctl show plan route53-cloud-hostedzone

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_route53_zone.this will be created
  + resource "aws_route53_zone" "this" {
      + arn                 = (known after apply)
      + comment             = "Experimentations for blog.ogenki.io"
      + force_destroy       = false
      + id                  = (known after apply)
      + name                = "cloud.ogenki.io"
      + name_servers        = (known after apply)
      + primary_name_server = (known after apply)
      + tags                = {
          + "Name" = "cloud.ogenki.io"
        }
      + tags_all            = {
          + "Name" = "cloud.ogenki.io"
        }
      + zone_id             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + domain_name = "cloud.ogenki.io"
  + nameservers = (known after apply)
  + zone_arn    = (known after apply)
  + zone_id     = (known after apply)

Plan generated: set approvePlan: "plan-main@345394fb4a82b9b258014332ddd556dde87f73ab" to approve this plan.
To set the field, you can also run:

  tfctl approve route53-cloud-hostedzone -f filename.yaml
```

So we need to restore the terraform state as it was when the cloud resources where initially created.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tfstate-${WORKSPACE}-${STACK}
  namespace: flux-system
  annotations:
    encoding: gzip
type: Opaque
data:
  tfstate: $(cat ${BACKUPDIR}/${WORKSPACE}-${STACK}.tfstate.gz | base64 -w 0)
EOF
```

You will also need to trigger a plan manually

```console
tfctl replan route53-cloud-hostedzone
 Replan requested for flux-system/route53-cloud-hostedzone
Error: timed out waiting for the condition
```

We can then check that the state file has been updated

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
flux-system     route53-cloud-hostedzone        True    Outputs written: main@sha1:d0934f979d832feb870a8741ec01a927e9ee6644                                                     false           19 minutes
```

## 🔍 Focus on Key Features of Flux

Well, I lied a bit about the agenda 😝. Indeed I want to highlight two features that I hadn't explored until now and that are very useful!

### Variable Substitution

When Flux is initialized, some cluster-specific `Kustomization` files are applied.
It is possible to specify **variable substitutions** within these files, so that they can be used in all resources deployed by this `Kustomization`. **This helps to minimize code duplication**.

I recently discovered the efficiency of this feature. Here is how I use it:

The Terraform code that creates an EKS cluster also generates a `ConfigMap` that contains **cluster-specific variables** such as the cluster name, as well as all the parameters that vary between clusters.

[flux.tf](https://github.com/Smana/demo-tf-controller/blob/main/terraform/controlplane/flux.tf#L36)

```hcl
resource "kubernetes_config_map" "flux_clusters_vars" {
  metadata {
    name      = "eks-${var.cluster_name}-vars"
    namespace = "flux-system"
  }

  data = {
    cluster_name      = var.cluster_name
    oidc_provider_arn = module.eks.oidc_provider_arn
    aws_account_id    = data.aws_caller_identity.this.account_id
    region            = var.region
    environment       = var.env
    vpc_id            = module.vpc.vpc_id
  }
  depends_on = [flux_bootstrap_git.this]
}
```

As mentioned previously, variable substitutions are defined in the `Kustomization` files. Let's take a concrete example.</br>
Below, we define the Kustomization that deploys all the resources controlled by the `tf-controller`. </br>
Here, we declare the `eks-controlplane-0-vars` ConfigMap that was generated during the EKS cluster creation.

[infrastructure.yaml](https://github.com/Smana/demo-tf-controller/blob/main/clusters/controlplane-0/infrastructure.yaml#L2)

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tf-custom-resources
  namespace: flux-system
spec:
  prune: true
  interval: 4m0s
  path: ./infrastructure/controlplane-0/terraform/custom-resources
  postBuild:
    substitute:
      domain_name: "cloud.ogenki.io"
    substituteFrom:
      - kind: ConfigMap
        name: eks-controlplane-0-vars
      - kind: Secret
        name: eks-controlplane-0-vars
        optional: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: tf-controller
```

Finally, below is an example of a Kubernetes resource that makes use of it. **This single manifest can be used by all clusters!**.

[external-dns/helmrelease.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/base/external-dns/helmrelease.yaml)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: external-dns
spec:
...
  values:
    global:
      imageRegistry: public.ecr.aws
    fullnameOverride: external-dns
    aws:
      region: ${region}
      zoneType: "public"
      batchChangeSize: 1000
    domainFilters: ["${domain_name}"]
    logFormat: json
    txtOwnerId: "${cluster_name}"
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::${aws_account_id}:role/${cluster_name}-external-dns"
```

This will reduce significantly the number of overlays that were used to patch with cluster-specific parameters.

### Web UI (Weave GitOps)

In my [previous article on Flux](https://blog.ogenki.io/post/devflux/), I mentioned one of its downsides (when compared to its main competitor, ArgoCD): the lack of a Web interface. While I am a command line guy, this is sometimes useful to have a consolidated view and the ability to perform some operations with just a few clicks :computer_mouse:

This is now possible with [Weave GitOps](https://github.com/weaveworks/weave-gitops)! Of course, it is not comparable to ArgoCD's UI, but the essentials are there: pausing reconciliation, visualizing manifests, dependencies, events...

![Weave-Gitops](weave-gitops.gif)

There is also the [VSCode plugin](https://github.com/weaveworks/vscode-gitops-tools) as an alternative.

## 💭 Final thoughts

One might say "yet another infrastructure management tool from Kubernetes". Well this is true but despite a few minor issues faced along the way, which I [shared](https://github.com/weaveworks/tf-controller/issues?q=author%3ASmana) on the project's Git repo, I really enjoyed the experience. `tf-controller` provides a concrete answer to a common question: how to manage our infrastructure like we manage our code?

I really like the GitOps approach applied to infrastructure, and I had actually written an [article on Crossplane](https://blog.ogenki.io/post/crossplane_k3d/).
`tf-controller` tackles the problem from a different angle: using Terraform directly. This means that we can leverage our existing knowledge and code. There's no need to learn a new way of declaring our resources.</br>
This is an important criterion to consider because migrating to a new tool when you already have an existing infrastructure represents a significant effort. However, I would also add that `tf-controller` is only targeted at Flux users, which restricts its target audience.

Currently, I'm using a combination of Terraform, Terragrunt and RunAtlantis. `tf-controller` could become a serious alternative: We've talked about the value of Kustomize combined with variable substitions to avoid code deduplication. The project's roadmap also aims to display plans in pull-requests.
Another frequent need is to pass sensitive information to modules. Using a `Terraform` resource, we can inject variables from Kubernetes secrets. This makes it possible to use common secrets management tools, such as `external-secrets`, `sealed-secrets` ...

So, I encourage you to try `tf-controller` yourself, and perhaps even contribute to it 🙂

{{% notice note Note %}}

* The demo I used create quite a few resources, some of which are quite critical (like the network). So, keep in mind that this is just for the demo! I suggest taking a gradual approach if you plan to implement it: start by using drift detection, then create simple resources.
* I also took some shortcuts in terms of security that should be avoided, such as giving admin rights to the controller.
{{% /notice %}}