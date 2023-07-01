+++
author = "Smaine Kahlouch"
title = "Appliquer les principes de GitOps à l'infrastructure: Introduction à `tf-controller`"
date = "2023-06-01"
summary = "**Weave tf-controller** est un opérateur Kubernetes qui permet d'appliquer du code Terraform et apporte certaines fonctionnalités manquantes (réconciliation, détection de dérive ...). Explorons les principales fonctionnalités 🕵️"
featureImage = "tf-controller.png"
featured = true
codeMaxLines = 20
usePageBundles = true
toc = true
tags = [
    "data"
]
thumbnail= "weavetf.png"
+++

**Terraform** est probablement l'outil "Infrastructure As Code" le plus utilisé pour construire, modifier et versionner les changements d'infrastructure Cloud.
Il s'agit d'un projet Open Source développé par Hashicorp et qui utilise le langage [FCL](https://github.com/hashicorp/hcl) pour déclarer l'état souhaité de resources Cloud.
L'état des ressources créées est stocké dans un fichier d'état (terraform state).

On peut considérer que Terraform est un outil "semi-déclaratif" car il n'y a pas de fonctionnalité de **réconciliation automatique** intégrée. Il existe différentes approches pour répondre à cette problématique, mais en règle générale, une modification sera appliquée en utilisant `terraform apply`. Le code est bien décrit dans des fichiers de configuration HCL (déclaratif) mais l'exécution est faite de manière impérative.
De ce fait, il peut y avoir de la dérive entre l'état déclaré et le réel (par exemple, un collègue qui serait passé sur la console pour changer un paramètre 😉).

❓❓ Alors, comment m'assurer que ce qui est commit dans mon repo git est vraiment appliqué. Comment être alerté s'il y a un changement par rapport à l'état désiré et comment appliquer automatiquement ce qui est dans mon code (GitOps) ?

C'est la promesse de [**tf-controller**](https://github.com/weaveworks/tf-controller), un operateur Kubernetes opensource de Weaveworks, étroitement lié à Flux (un moteur GitOps de la même société). [**Flux**](https://fluxcd.io/) est l'une des solutions que je plébiscite, et je vous invite donc à lire un [précédent article](https://blog.ogenki.io/post/devflux/).

{{% notice info Info %}}
L'ensemble des étapes décrites ci-dessous sont faites avec ce [**repo Git**](https://github.com/Smana/demo-tf-controller)
{{% /notice %}}

## :bullseye: Notre objectif

En suivant les étapes de cet article nous visons les objectifs suivant:

* Déployer un cluster Kubernetes qui servira de "**Control plane**". Pour résumer il hébergera le controlleur Terraform qui nous permettra de déclarer tous les éléments d'infrastructure souhaités.
* Utiliser **Flux** comme moteur GitOps pour toutes les resources Kubernetes.

Concernant le controleur Terraform, nous allons voir:

* Quelle est le moyen de définir des **dépendances** entre modules
* Création de **plusieurs resources** AWS: Zone route53, Certificat ACM, réseau, cluster EKS.
* Les différentes options de **reconciliation** (automatique, nécessitant une confirmation)
* Comment sauvegarder et **restaurer** un fichier d'état (tfstate)

## 🛠️ Installer le controleur Terraform

### Le cluster "Control Plane"

Afin de pouvoir utiliser le controleur Kubernetes `tf-controller`, il nous faut d'abord un cluster Kubernetes 😆.
Nous allons donc créer un cluster **control plane** en utilisant la ligne de commande `terraform` et les bonnes pratiques EKS.

{{% notice warning Warning %}}
Il est primordial que ce cluster soit résiliant, sécurisé et supervisé car il sera responsable de la gestion de l'ensemble des resources AWS créées par la suite.
Il est aussi fortement conseillé d'avoir une politique de sauvegarde du cluster en question, en utilisant par exemple [Velero](https://velero.io/)
{{% /notice %}}

Sans entrer dans le détail, le cluster "control plane" a été créé un utilisant [ce code](https://github.com/Smana/demo-tf-controller/tree/main/terraform/controlplane). Celà-dit, il est important de noter que toutes les opérations de déploiement d'application se font en utilisant Flux.

{{% notice info Info %}}

En suivant les instructions du [README](https://github.com/Smana/demo-tf-controller/blob/main/terraform/controlplane/README.md), un cluster EKS sera créé mais pas uniquement! </br>
Il faut en effet donner les permissions au controlleur Terraform pour appliquer les changements d'infrastructure.
De plus, Flux doit être installé et configuré afin d'appliquer la configuration définie [ici](https://github.com/Smana/demo-tf-controller/tree/main/clusters/controlplane-0).

Au final on se retrouve donc avec plusieurs éléments installés et configurés:

* les addons quasi indispensables que sont `aws-loadbalancer-controller` et `external-dns`
* les roles [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) pour ces mêmes composants sont installés en utilisant `tf-controller`
* La stack de supervision Prometheus / Grafana.
* `sealed-secrets` pour pouvoir stocker les secrets Kubernetes dans le repo Git en toute sécurité.
* Afin de démontrer tout cela au bout de quelques minutes l'interface web pour Flux est accessible via l'URL `gitops-<cluster_name>.<domain_name`>

Vérifier toute de même que le cluster est accessible et que Flux fonctionne correctement

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

### Le chart Helm et Flux

Maintenant que notre cluster GitOps est opérationnel, nous pouvons constater que **l'ajout le contrôleur Terraform** consiste à utiliser le chart Helm comme spécifié dans la [documentation](https://weaveworks.github.io/tf-controller/getting_started/#installation).

Il faut tout d'abord déclarer la source:

<span style="color:green">infrastructure/base/tf-controller/source.yaml</span>
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: tf-controller
spec:
  interval: 30m
  url: https://weaveworks.github.io/tf-controller
```

Et définir la HelmRelease:

<span style="color:green">infrastructure/base/tf-controller/release.yaml</span>
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

```console
kubectl get po -n flux-system -l app.kubernetes.io/instance=tf-controller
NAME                             READY   STATUS    RESTARTS   AGE
tf-controller-7ffdc69b54-c2brg   1/1     Running   0          2m6s
```

Dans le repo de demo il y a déjà un certain nombre de ressources AWS déclarées. Par conséquent, au bout de quelques minutes, le cluster fraichement se charge de la création de celles-cis:
[![asciicast](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq.png)](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq?&speed=2)

{{% notice info Info %}}

Bien que la plupart des opérations peuvent être effectuées de façon déclarative ou en utilisant les outils en ligne de commande `kubectl` et `flux`, il existe aussi un autre outil qui permet d'interragir avec les ressources terraform: [tfctl](https://docs.gitops.weave.works/docs/terraform/tfctl/)

{{% /notice %}}

## 🚀 Appliquer un changement

Parmis les [bonnes pratiques](https://www.terraform-best-practices.com/) avec Terraform, il y a l'usage de **[modules](https://developer.hashicorp.com/terraform/language/modules)**.</br>
Un module est un ensemble de resources Terraform liées logigement afin d'obtenir une seule unité réutilisable. Cela permet d'abstraire la complexité, de prendre des entrées, effectuer des actions spécifiques et produire des sorties.

Il est possible de créer ses modules et de les mettre à disposition dans des `Sources` ou d'utiliser les nombreux modules partagés et maintenus par les communautés.</br>
Il suffit alors d'indiquer un certains nombre de `variables` afin de l'adapter au contexte.

Avec `tf-controller`, la première étape consiste donc à indiquer la `Source` du module. Ici nous allons configurer le socle réseau sur AWS avec le module [terraform-aws-vpc](https://github.com/terraform-aws-modules/terraform-aws-vpc).

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

Nous pouvons ensuite créer la resource `Terraform` qui en fait usage.</br>
Les principaux paramètres qui permettent de contrôler la façon dont sont appliquées les modifications sont `.spec.approvePlan` et `.spec.autoApprove`

### 🚨 Détection de la dérive

Définir `spec.approvePlan` avec une valeur à `disable` permet uniquement de notifier que l'état actuel des ressources a dérivé par rapport au code Terraform.
Cela permet notamment de choisir le moment et la manière dont l'application des changements sera faite.

### 🔧 Application manuelle


```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: vpc-dev
spec:
  interval: 8m
  path: .
  destroyResourcesOnDeletion: true # You wouldn't do that on a prod env ;)
  approvePlan: "plan-v5.0.0@sha1:26c38a66f1"
  sourceRef:
    kind: GitRepository
    name: terraform-aws-vpc
    namespace: flux-system
  storeReadablePlan: human
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

### 🤖 Application automatique


### 🔄 Entrées et sorties: dépendances entre modules


## 💾 Sauvegarder et restaurer un tfstate

Dans mon cas je souhaite ne pas avoir a recréer la zone et le certificat à chaque destruction du controlplane. Voici un exemple des étapes à mener pour que je puisse **restaurer** l'état de ces resources lorsque j'utilise cette demo.
(Il s'agit d'une procédure manuelle, on préferera une solution de sauvegarde du cluster pour du la production.)


```console
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

Plan generated: set approvePlan: "plan-main@sha1:345394fb4a82b9b258014332ddd556dde87f73ab" to approve this plan.
To set the field, you can also run:

  tfctl approve route53-cloud-hostedzone -f filename.yaml
```


Exporter le tfstate présent dans le secret Kubernetes:

```console
WORKSPACE="default"
STACK="route53-cloud-hostedzone"
BACKUPDIR="${HOME}/tf-controller-backup"

mkdir -p ${BACKUPDIR}

kubectl get secrets -n flux-system tfstate-${WORKSPACE}-${STACK} -o jsonpath='{.data.tfstate}' | \
base64 -d | gzip -d > ${BACKUPDIR}/${WORKSPACE}-${STACK}.tfstate
```

```bash
gzip ${BACKUPDIR}/${WORKSPACE}-${STACK}.tfstate

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

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
...
flux-system     route53-cloud-hostedzone        Unknown Plan generated: set approvePlan: "plan-main@sha1:345394fb4a82b9b258014332ddd556dde87f73ab" to approve this plan.        true            16 minutes
```

```console
tfctl replan acm-cloud --request-timeout 0
 Replan requested for flux-system/acm-cloud
Error: timed out waiting for the condition
```

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
flux-system     route53-cloud-hostedzone        True    Outputs written: main@sha1:d0934f979d832feb870a8741ec01a927e9ee6644                                                     false           19 minutes
```

## :detective: Troubleshooting

break glass
break-glass tfctl

## 🔍 Focus sur certaines fonctionnalités de Flux

Oui j'ai un peu menti sur l'agenda 😝. Il me semblait nécessaire de mettre en lumière 2 fonctionnalités que je n'avais pas exploité jusque là et qui sont fort utiles!

### Substition de variables

Lorsque Flux est initiliasé un certain nombre de `Kustomization` spécifique à ce cluster sont crées.
Il est possible d'y indiquer des **variables de substitution** qui pourront être utilisés dans l'ensemble des ressources déployées par cette `Kustomization`. **Cela permet d'éviter un maximum la déduplication de code**.

J'ai découvert l'efficacité de cette fonctionnalité très récemment. Je vais décrire ici la façon dont je l'utilise:

Le code terraform qui crée un cluster EKS, génère aussi une `ConfigMap` qui contient les **variables propres au cluster**.
On y retrouvera bien sûr le nom du cluster, mais aussi tous les paramètres qui varient entre les clusters et qui sont utilisés dans les manifests Kubernetes.

<span style="color:green">terraform/controlplane/flux.tf</span>

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

Comme spécifié précedemment, les variables de substition sont définies dans les `Kustomization`. Prenons un exemple concret.
Ci-dessous on définie la Kustomization qui déploie les rôles IRSA pour le cluster `controlplane-0`. </br>
On consomme ici la `ConfigMap` crée à la création du cluster EKS.

<span style="color:green">clusters/controlplane-0/infrastructure.yaml</span>

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

Et donc voici un exemple de resource Kubernetes qui en fait usage. **Cet unique manifest peut être utilisé par tous les clusters!**.

<span style="color:green">infrastructure/base/external-dns/helmrelease.yaml</span>

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

### Web UI (Weave GitOps)

Dans mon [précédent article sur Flux](https://blog.ogenki.io/post/devflux/), je mentionnais le fait que l'un des inconvénients (si l'on compare avec son principale concurrent: ArgoCD) est le manque d'une interface Web. Bien que je sois un adepte de la ligne de commande, c'est parfois bien utile d'avoir une vue synthétique et de pouvoir effectuer certaines opération en quelques clicks :computer_mouse:

C'est désormais possible avec [Weave Gitops](https://github.com/weaveworks/weave-gitops)! Bien entendu ce n'est pas comparable avec l'UI d'ArgoCD, mais l'essentiel est là: Mettre en pause la réconcilation, visualiser les manifests, les dépendances, les événements...

![Weave-Gitops](weave-gitops.gif)

Il existe aussi le [plugin VSCode](https://github.com/weaveworks/vscode-gitops-tools) comme alternative.

## 💭 Dernière remarques

{{% notice note Note %}}

ne pas donner les droits Administrator.
ne pas donner accès au cluster.
Volontairement pousser loin l'exercice mais il faut une approche prudente. Drift detection, resources simples et limité à un petit nombre, application automatique uniquement de resources qui n'ont pas un impact critique sur la prod.

{{% /notice %}}

Limitations module existant outputs maps, différents type d'inputs, informations sensibles affichées en clair dans le statut de la resource.
