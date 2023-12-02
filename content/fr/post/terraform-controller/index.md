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
    "infrastructure"
]
thumbnail= "thumbnail.png"
+++

**Terraform** est probablement l'outil "Infrastructure As Code" le plus utilisé pour construire, modifier et versionner les changements d'infrastructure Cloud.
Il s'agit d'un projet Open Source développé par Hashicorp et qui utilise le langage [HCL](https://github.com/hashicorp/hcl) pour déclarer l'état souhaité de ressources Cloud.
L'état des ressources créées est stocké dans un fichier d'état (terraform state).

On peut considérer que Terraform est un outil "semi-déclaratif" car il n'y a pas de fonctionnalité de **réconciliation automatique** intégrée. Il existe différentes approches pour répondre à cette problématique, mais en règle générale, une modification sera appliquée en utilisant `terraform apply`. Le code est bien décrit dans des fichiers de configuration HCL (déclaratif) mais l'exécution est faite de manière impérative.
De ce fait, il peut y avoir de la dérive entre l'état déclaré et le réel (par exemple, un collègue qui serait passé sur la console pour changer un paramètre 😉).

❓❓ Alors, comment m'assurer que ce qui est commit dans mon repo git est vraiment appliqué. Comment être alerté s'il y a un changement par rapport à l'état désiré et comment appliquer automatiquement ce qui est dans mon code (GitOps) ?

C'est la promesse de [**tf-controller**](https://github.com/weaveworks/tf-controller), un operateur Kubernetes Open Source de Weaveworks, étroitement lié à Flux (un moteur GitOps de la même société). [**Flux**](https://fluxcd.io/) est l'une des solutions que je plébiscite, et je vous invite donc à lire un [précédent article](https://blog.ogenki.io/post/devflux/).

{{% notice info Info %}}
L'ensemble des étapes décrites ci-dessous sont faites avec ce [**repo Git**](https://github.com/Smana/demo-tf-controller)
{{% /notice %}}

## 🎯 Notre objectif

En suivant les étapes de cet article nous visons les objectifs suivant:

* Déployer un cluster Kubernetes qui servira de "**Control plane**". Pour résumer il hébergera le controlleur Terraform qui nous permettra de déclarer tous les éléments d'infrastructure souhaités.
* Utiliser **Flux** comme moteur GitOps pour toutes les ressources Kubernetes.

Concernant le controleur Terraform, nous allons voir:

* Quelle est le moyen de définir des **dépendances** entre modules
* Création de **plusieurs ressources** AWS: Zone route53, Certificat ACM, réseau, cluster EKS.
* Les différentes options de **reconciliation** (automatique, nécessitant une confirmation)
* Comment sauvegarder et **restaurer** un fichier d'état (tfstate)

## 🛠️ Installer le controleur Terraform

### ☸ Le cluster "Control Plane"

Afin de pouvoir utiliser le controleur Kubernetes `tf-controller`, il nous faut d'abord un cluster Kubernetes 😆.
Nous allons donc créer un cluster **control plane** en utilisant la ligne de commande `terraform` et les bonnes pratiques EKS.

{{% notice warning Warning %}}
Il est primordial que ce cluster soit résiliant, sécurisé et supervisé car il sera responsable de la gestion de l'ensemble des ressources AWS créées par la suite.
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
* `external-secrets` pour pouvoir récupérer des éléments sensibles depuis AWS secretsmanager.
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

### 📦 Le chart Helm et Flux

Maintenant que notre cluster "controlplane" est opérationnel, **l'ajout le contrôleur Terraform** consiste à utiliser le chart Helm.

Il faut tout d'abord déclarer la source:

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

Et définir la HelmRelease:

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

Lorsque ce changement est écrit dans le repo Git, la HelmRelease sera déployée et le contrôlleur `tf-controller` démarera

```console
kubectl get hr -n flux-system
NAME            AGE   READY   STATUS
tf-controller   67m   True    Release reconciliation succeeded

kubectl get po -n flux-system -l app.kubernetes.io/instance=tf-controller
NAME                             READY   STATUS    RESTARTS   AGE
tf-controller-7ffdc69b54-c2brg   1/1     Running   0          2m6s
```

Dans le repo de demo il y a déjà un certain nombre de ressources AWS déclarées. Par conséquent, au bout de quelques minutes, le cluster se charge de la création de celles-cis:
[![asciicast](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq.png)](https://asciinema.org/a/guDIpkVdD51Cyog9P5NYnuWSq?&speed=2)

{{% notice info Info %}}
Bien que la majorité des tâches puisse être réalisée de manière déclarative ou via les utilitaires de ligne de commande tels que `kubectl` et `flux`, un autre outil existe qui offre la possibilité d'interagir avec les ressources terraform : [tfctl](https://docs.gitops.weave.works/docs/terraform/tfctl/)
{{% /notice %}}

## 🚀 Appliquer un changement

Parmis les [bonnes pratiques](https://www.terraform-best-practices.com/) avec Terraform, il y a l'usage de **[modules](https://developer.hashicorp.com/terraform/language/modules)**.</br>
Un module est un ensemble de ressources Terraform liées logigement afin d'obtenir une seule unité réutilisable. Cela permet d'abstraire la complexité, de prendre des entrées, effectuer des actions spécifiques et produire des sorties.

Il est possible de créer ses propres modules et de les mettre à disposition dans des `Sources` ou d'utiliser les nombreux modules partagés et maintenus par les communautés.</br>
Il suffit alors d'indiquer quelques `variables` afin de l'adapter au contexte.

Avec `tf-controller`, la première étape consiste donc à indiquer la `Source` du module. Ici nous allons configurer le socle réseau sur AWS (vpc, subnets...) avec le module [terraform-aws-vpc](https://github.com/terraform-aws-modules/terraform-aws-vpc).

[sources/terraform-aws-vpc.yaml](https://github.com/Smana/demo-tf-controller/blob/main/infrastructure/controlplane-0/terraform/custom-resources/sources/terraform-aws-vpc.yaml)

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

Nous pouvons ensuite créer la ressource `Terraform` qui en fait usage:

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

Si l'on devait résumer grossièrement: le code terraform provenant de la source `terraform-aws-vpc` est utilisé avec les variables `vars`.

Il y a ensuite plusieurs paramètres qui influent sur le fonctionnement de `tf-controller`. Les principaux paramètres qui permettent de contrôler la façon dont sont appliquées les modifications sont `.spec.approvePlan` et `.spec.autoApprove`

### 🚨 Détection de la dérive

Définir `spec.approvePlan` avec une valeur à `disable` permet uniquement de notifier que l'état actuel des ressources a dérivé par rapport au code Terraform.
Cela permet notamment de choisir le moment et la manière dont l'application des changements sera effectuée.

{{% notice note Note %}}
De mon point de vue il manque une section sur les **notifications**: La dérive, les plans en attentes, les problèmese de réconcilation. J'essaye d'identifier les méthodes possibles (de préférence avec Prometheus) et de mettre à jour cet article dès que possible.
{{% /notice %}}

### 🔧 Application manuelle

L'exemple donné précédemment (`vpc-dev`) ne contient pas le paramètre `.spec.approvePlan` et hérite donc de la valeur par défaut qui est `false`.
Par conséquent, l'application concrète des modifications (`apply`), n'est pas faite automatiquement.

Un `plan` est exécuté et sera en attente d'une validation:

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
...
flux-system     vpc-dev                         Unknown Plan generated: set approvePlan: "plan-v5.0.0-26c38a66f12e7c6c93b6a2ba127ad68981a48671" to approve this plan.      true            2 minutes
```

Je conseille d'ailleurs de configurer le paramètre `storeReadablePlan` à `human`. Cela permet de visualiser simplement les modifications en attente en utilisant `tfctl`:

```console
tfctl show plan vpc-dev

Terraform used the selected providers to generate the following execution
plan. ressource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_default_network_acl.this[0] will be created
  + ressource "aws_default_network_acl" "this" {
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

Après revue des modifications ci-dessus, il suffit donc d'ajouter l'identifiant du `plan` à valider et de pousser le changement sur git comme suit:

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

En quelques instants un `runner` sera lancé qui se chargera d'appliquer les changements:

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

La réconciliation éffectuée, la ressource passe à l'état `READY: True`

```console
kubectl get tf -n flux-system vpc-dev
NAME      READY   STATUS                                                                  AGE
vpc-dev   True    Outputs written: v5.0.0@sha1:26c38a66f12e7c6c93b6a2ba127ad68981a48671   17m
```

### 🤖 Application automatique

Nous pouvons aussi activer la **réconciliation** automatique. Pour ce faire il faut déclarer le paramètre `.spec.autoApprove` à `true`.

Toutes les ressources IRSA sont configurées de la sorte:

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

Donc si je fais le moindre changement sur la console AWS par exemple, celui-ci sera rapidement **écrasé** par celui géré par `tf-controller`.

{{% notice info Info %}}
La politique de suppression d'une ressource Terraform est définie par le paramètre `destroyResourcesOnDeletion`.
Par défaut elles sont conservées et il faut donc que ce paramètre ait pour valeur `true` afin de détruire les éléments crées lorsque l'objet Kubernetes est supprimé.

Ici nous voulons la possibilité de supprimer les rôles IRSA. Ils sont en effet étroitement liés aux clusters.
{{% /notice %}}

### 🔄 Entrées et sorties: dépendances entre modules

Lorsque qu'on utilise Terraform, on a souvent besoin de passer des données d'un module à l'autre. Généralement ce sont les [**outputs**](https://developer.hashicorp.com/terraform/language/values/outputs) du module qui exportent ces informations. Il faut donc un moyen de les importer dans un autre module.

Reprenons encore l'exemple donné ci-dessus (`vpc-dev`). Nous notons en bas du YAML la directive suivante:

```yaml
...
  writeOutputsToSecret:
    name: vpc-dev
```

Lorsque cette ressource est appliquée nous aurons un message qui confirme que les outputs sont disponibles ("Outputs written"):

```console
kubectl get tf -n flux-system vpc-dev
NAME      READY   STATUS                                                                  AGE
vpc-dev   True    Outputs written: v5.0.0@sha1:26c38a66f12e7c6c93b6a2ba127ad68981a48671   17m
```

En effet ce module exporte de nombreuses informations (126):

```console
kubectl get secrets -n flux-system vpc-dev
NAME      TYPE     DATA   AGE
vpc-dev   Opaque   126    15s

kubectl get secret -n flux-system vpc-dev --template='{{.data.vpc_id}}' | base64 -d
vpc-0c06a6d153b8cc4db
```

Certains de ces éléments d'informations sont ensuite utilisés pour créer un cluster EKS de dev:

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

## 💾 Sauvegarder et restaurer un tfstate

Dans mon cas je ne souhaite pas recréer la zone et le certificat à chaque destruction du controlplane. Voici un exemple des étapes à mener pour que je puisse **restaurer** l'état de ces ressources lorsque j'utilise cette demo.

{{% notice note Note %}}
Il s'agit là d'une procédure manuelle afin de démontrer le comportement de `tf-controller` par rapport aux fichiers d'état. Par défaut ces `tfstates` sont stockés dans des `secrets` mais on préferera configurer un backend GCS ou S3
{{% /notice %}}

La création initiale de l'environnement de démo m'a permis de sauvegarder les fichiers d'état (tfstate) de cette façon.

```console
WORKSPACE="default"
STACK="route53-cloud-hostedzone"
BACKUPDIR="${HOME}/tf-controller-backup"

mkdir -p ${BACKUPDIR}

kubectl get secrets -n flux-system tfstate-${WORKSPACE}-${STACK} -o jsonpath='{.data.tfstate}' | \
base64 -d | gzip -d > ${BACKUPDIR}/${WORKSPACE}-${STACK}.tfstate
```

Lorsque le cluster est créé à nouveau, tf-controller essaye de créer la zone car le fichier d'état est vide.

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
...
flux-system     route53-cloud-hostedzone        Unknown Plan generated: set approvePlan: "plan-main@sha1:345394fb4a82b9b258014332ddd556dde87f73ab" to approve this plan.        true            16 minutes

tfctl show plan route53-cloud-hostedzone

Terraform used the selected providers to generate the following execution
plan. resource actions are indicated with the following symbols:
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

La procédure de restauration consiste donc à créer le secret à nouveau:

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

Il faudra aussi relancer un plan de façon explicite pour mettre à jour l'état de la ressource en question

```console
tfctl replan route53-cloud-hostedzone
 Replan requested for flux-system/route53-cloud-hostedzone
Error: timed out waiting for the condition
```

Nous pouvons alors vérifier que le fichier d'état a bien été mis à jour

```console
tfctl get
NAMESPACE       NAME                            READY   MESSAGE                                                                                                                 PLAN PENDING    AGE
flux-system     route53-cloud-hostedzone        True    Outputs written: main@sha1:d0934f979d832feb870a8741ec01a927e9ee6644                                                     false           19 minutes
```

## 🔍 Focus sur certaines fonctionnalités de Flux

Oui j'ai un peu menti sur l'agenda 😝. Il me semblait nécessaire de mettre en lumière 2 fonctionnalités que je n'avais pas exploité jusque là et qui sont fort utiles!

### Substition de variables

Lorsque Flux est initiliasé un certain nombre de `Kustomization` spécifique à ce cluster sont créés.
Il est possible d'y indiquer des **variables de substitution** qui pourront être utilisées dans l'ensemble des ressources déployées par cette `Kustomization`. **Cela permet d'éviter un maximum la déduplication de code**.

J'ai découvert l'efficacité de cette fonctionnalité très récemment. Je vais décrire ici la façon dont je l'utilise:

Le code terraform qui crée un cluster EKS, génère aussi une `ConfigMap` qui contient les **variables propres au cluster**.
On y retrouvera, bien sûr, le nom du cluster, mais aussi tous les paramètres qui varient entre les clusters et qui sont utilisés dans les manifests Kubernetes.

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

Comme spécifié précedemment, les variables de substition sont définies dans les `Kustomization`. Prenons un exemple concret.
Ci-dessous on définie la Kustomization qui déploie toutes les ressources qui sont consommées par `tf-controller` </br>
On déclare ici la ConfigMap `eks-controlplane-0-vars` qui avait été généré à la création du cluster EKS.

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

Enfin voici un exemple de ressource Kubernetes qui en fait usage. **Cet unique manifest peut être utilisé par tous les clusters!**.

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

Cela élimine totalement les overlays qui consistaient à ajouter les paramètres spécifiques au cluster.

### Web UI (Weave GitOps)

Dans mon [précédent article sur Flux](https://blog.ogenki.io/post/devflux/), je mentionnais le fait que l'un des inconvénients (si l'on compare avec son principale concurrent: ArgoCD) est le manque d'une interface Web. Bien que je sois un adepte de la ligne de commande, c'est parfois bien utile d'avoir une vue synthétique et de pouvoir effectuer certaines opération en quelques clicks :computer_mouse:

C'est désormais possible avec [Weave Gitops](https://github.com/weaveworks/weave-gitops)! Bien entendu ce n'est pas comparable avec l'UI d'ArgoCD, mais l'essentiel est là: Mettre en pause la réconcilation, visualiser les manifests, les dépendances, les événements...

![Weave-Gitops](weave-gitops.gif)

Il existe aussi le [plugin VSCode](https://github.com/weaveworks/vscode-gitops-tools) comme alternative.

## 💭 Remarques

Et voilà, nous arrivons au bout de notre exploration de cet autre outil de gestion d'infrastructure sur Kubernetes. Malgré quelques petits soucis rencontrés en cours de route, que j'ai [partagé](https://github.com/weaveworks/tf-controller/issues?q=author%3ASmana) sur le repo Git du projet, l'expérience m'a beaucoup plu. `tf-controller` offre une réponse concrète à une question fréquente : comment gérer notre infra comme on gère notre code ?

J'aime beaucoup l'approche GitOps appliquée à l'infrastructure, j'avais d'ailleurs écrit un [article sur Crossplane](https://blog.ogenki.io/post/crossplane_k3d/).
`tf-controller` aborde la problématique sous un angle différent: utiliser du Terraform directement. Cela signifie qu'on peut utiliser nos connaissances actuelles et notre code existant. Pas besoin d'apprendre une nouvelle façon de déclarer nos ressources.</br>
C'est un critère à prendre en compte car migrer vers un nouvel outil lorsque l'on a un existant représente un éffort non négligeable. Cependant j'ajouterais aussi que `tf-controller` s'adresse aux utilisateurs de Flux uniquement et, de ce fait, restreint le publique cible.

Aujourd'hui, j'utilise une combinaison de Terraform, Terragrunt et RunAtlantis. `tf-controller` pourrait devenir une alternative viable: Nous avons en effet évoqué l'intérêt de Kustomize associé aux substitions de variables pour la factorisation de code. Dans la roadmap du projet il y a aussi l'objectif d'afficher les plans dans les pull-requests.
Autre problématique fréquente: la nécessité de passer des éléments sensibles aux modules. En utilisant une ressource `Terraform`, on peut injecter des variables depuis des secrets Kubernetes. Ce qui permet d'utiliser certains outils, tels que `external-secrets`, `sealed-secrets` ...

Je vous encourage donc à essayer `tf-controller` vous-même, et peut-être même d'y apporter votre contribution 🙂.

{{% notice note Note %}}

* La démo que j'ai faite ici utilise pas mal de ressources, dont certaines assez cruciales (comme le réseau). Donc, gardez en tête que c'est juste pour la démo ! Je suggère une approche progressive si vous envisagez de le mettre en ouvre: commencez par utiliser la détection de dérives, puis créez des ressources simples.
* J'ai aussi pris quelques raccourcis en terme de sécurité à éviter absolument, notamment le fait de donner les droits admin au contrôleur.
{{% /notice %}}
