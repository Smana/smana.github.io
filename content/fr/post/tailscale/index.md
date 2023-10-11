+++
author = "Smaine Kahlouch"
title = "Sécuriser le Cloud avec `Tailscale` : Mise en œuvre d'un VPN simplifiée"
date = "2023-10-07"
summary = "Tailscale est une solution de **VPN** qui permet connecter des appareils ou serveurs de manière sécurisée. Comment en bénéficier pour accéder à une infrastruture Cloud?"
featured = true
codeMaxLines = 21
usePageBundles = true
toc = true
tags = [
    "security",
    "network"
]
thumbnail= "thumbnail.png"
+++

Lorsqu'on parle de sécurisation de l'accès aux ressources Cloud, l'une des règles d'or est d'éviter les expositions directes à Internet. La question qui se pose alors pour les Devs/Ops est : comment, par exemple, accéder à une base de données, un cluster Kubernetes ou un serveur via SSH sans compromettre la sécurité? </br>

Les **réseaux privés virtuels (VPN)** offrent une réponse en établissant un lien sécurisé entre différents éléments d'un réseau, indépendamment de leur localisation géographique. De nombreuses solutions existent, allant de modèles en SaaS aux solutions que l'on peut héberger soi-même, utilisant divers protocoles et étant soit open source, soit propriétaires.

Parmi ces options, je souhaitais vous parler de [**Tailscale**](https://tailscale.com/). Cette solution utilise le protocole `WireGuard`, réputé pour sa simplicité et sa performance.  Avec Tailscale, il est possible de connecter des appareils ou serveurs de manière sécurisée, comme s'ils étaient sur un même réseau local, bien qu'ils soient répartis à travers le monde.

## :bullseye: Nos objectifs

* Comprendre comment fonctionne `Tailscale`
* Mise en oeuvre d'une connexion sécurisée avec AWS en quelques minutes
* Interragir avec l'API d'un cluster EKS via un réseau privé
* Accéder à des services hébergés sur Kubernetes en utilisant le réseau privé

<center><img src="admin_console.png" width="400" /></center>

Pour le reste de cet article il faudra évidemment **créer un compte Tailscale**. A noter que l'authentification est déléguée à des fournisseurs d'identité tiers (ex: Okta, Onelogin, Google ...).

Lorsque le compte est crée, on a directement accès à la console de gestion ci-dessus. Elle permet notamment de lister les appareils connectés, de consulter les logs, de modifier la plupart des paramètres...

## 💡 Sous le capot

{{% notice info Terminologie %}}
**Mesh VPN**: Un _mesh VPN_ est un type de réseau VPN où chaque nœud (c'est-à-dire chaque appareil ou machine) est connecté à tous les autres nœuds du réseau, formant ainsi un maillage. À distinguer des configurations VPN traditionnelles qui sont conçues généralement "en étoile", où plusieurs clients se connectent à un serveur central.

**Zero trust**: Signifie que chaque machine, application ou utilisateur doit prouver son identité et son autorisation avant d'accéder à une ressource. On ne fait pas confiance simplement parce qu'une machine ou un utilisateur provient d'un réseau interne ou d'une certaine zone géographique. Chaque demande d'accès est traitée comme si elle venait d'une source non fiable, et elle doit être authentifiée et autorisée avant d'être accordée.

**Tailnet**: Dès la première utilisation de Tailscale, un _Tailnet_ est crée pour vous et correspond à votre propre réseau privé. Chaque appareil dans un tailnet reçoit une IP Tailscale unique, permettant une communication directe entre eux. Chacun de ces réseaux possède son propre nom ainsi qu'un label associé à une organisation.
{{% /notice %}}

<center><img src="mesh.png" width="500" /></center>

L'architecture de Tailscale est conçue de telle sorte que le _Control plane_ et le _Data plane_ sont clairement séparés:

* D'une part, il y a le **serveur de coordination**. Son rôle est d'échanger des métadonnées et des clés publiques entre tous les participants d'un Tailnet (La clé privée étant gardée en toute sécurité son nœud d'origine).

* D'autre part, les nœuds du Tailnet s'organisent en un **réseau maillé** (_Mesh_). Au lieu de passer par le serveur de coordination pour échanger des données, ces nœuds communiquent directement les uns avec les autres en mode **point à point**. Chaque nœud dispose d'une identité unique pour s'authentifier et rejoindre le Tailnet.

## :inbox_tray: Installation du client

La majorité des plateformes sont supportées et les procédures d'installation sont listées [ici](https://tailscale.com/kb/installation/).
En ce qui me concerne je suis sur Archlinux:

```console
sudo pacman -S tailscale
```

Il est possible de démarrer le service automatiquement au démarrage de la machine.
```console
sudo systemctl enable --now tailscaled
```

Pour enregistrer son ordinateur perso, lancer la commande suivante:
```console
sudo tailscale up --accept-routes

To authenticate, visit:

        https://login.tailscale.com/a/f50...
```

<center><img src="laptop_connected.png" width="850" /></center>


ℹ️ l'option `--accept-routes` est nécessaire sur Linux et permettra d'accepter les routes annoncées par les Subnet routers. On verra cela dans la suite de l'article

Vérifier que vous avez bien obtenu une IP du réseau Tailscale:
```console
tailscale ip -4
100.118.83.67

tailscale status
100.118.83.67   ogenki               smainklh@    linux   -
```

ℹ️ Pour les utilisateurs de Linux, vérifier que Tailscale fonctionne bien avec votre configuration DNS: Suivre [cette documentation](https://tailscale.com/kb/1188/linux-dns/).

{{% notice tip "Les sources" %}}
Toutes les étapes réalisées dans cet article proviennent de ce [**dépôt git**](https://github.com/Smana/demo-secured-eks)

Il va permettre de créer l'ensemble des composants qui ont pour objectif d'obtenir un cluster EKS de Lab et font suite à un précédent article sur [Cilium et Gateway API](https://blog.ogenki.io/fr/post/cilium-gateway-api/).

{{% /notice %}}

## ☁️ Accéder aux réseaux privés sur AWS

![Subnet router](subnet_router.png)

Afin de pouvoir accéder de manière sécurisée à l'ensemble des ressources disponibles sur AWS, il est possible de déployer un _**Subnet router**_.

Un _Subnet router_ est une instance Tailscale qui permet d'accéder à des sous-réseaux qui ne sont pas directement liés à Tailscale. Il fait office de **pont** entre le réseau privé virtuel de Tailscale (_Tailnet_) et d'autres réseaux locaux.

Nous pouvons alors **router des sous réseaux du Clouder à travers le VPN de Tailscale**.

⚠️ Pour ce faire, sur AWS, il faudra bien entendu configurer les _security groups_ correctement pour autoriser les Subnet routers.


### 🚀 Déployer un Subnet router

Entrons dans le vif du sujet et deployons un _Subnet router_ sur un réseau AWS!</br>
Tout est fait en utilisant le code **Terraform** présent dans le répertoire [terraform/network](https://github.com/Smana/demo-secured-eks/tree/main/terraform/network). Nous allons analyser la configuration spécifique à Tailscale qui est présente dans le fichier [tailscale.tf](https://github.com/Smana/demo-secured-eks/blob/main/terraform/network/tailscale.tf) avant de procéder au déploiement.

#### Le provider Terraform

Il est possible de configurer certains paramètres au travers de l'**API** Tailscale grâce au [provider Terraform](https://github.com/tailscale/terraform-provider-tailscale).
Pour cela il faut au préalable génerer une clé d'API 🔑 sur la console d'admin:

<center><img src="api_key.png" width="750" /></center>

Il faudra conserver cette clé dans un endroit sécurisé car elle est utilisé pour déployer le Subnet router

```hcl
provider "tailscale" {
  api_key = var.tailscale.api_key
  tailnet = var.tailscale.tailnet
}
```

<ins>**les ACL's**</ins>

Les ACL's permettent de définir qui est autorisé à communiquer avec qui (utilisateur ou appareil). À la création d'un compte, celle-cis sont très permissives et il n'y a aucune restriction (tout le monde peut parler avec tout le monde).

```hcl
resource "tailscale_acl" "this" {
  acl = jsonencode({
    acls = [
      {
        action = "accept"
        src    = ["*"]
        dst    = ["*:*"]
      }
    ]
...
}
```
{{% notice note Note %}}
Pour mon environnement de Lab, j'ai conservé cette configuration par défault car je suis la seule personne à y accéder. De plus les seuls appareils connectés à mon Tailnet sont mon laptop et le Subnet router. En revanche dans un cadre d'entreprise, il faudra bien y réfléchir. Il serait possible de définir une politique basée sur des groupes d'utilisitateurs ou sur les tags des noeuds.

Consulter cette [doc](https://tailscale.com/kb/1018/acls/) pour plus d'info.
{{% /notice %}}

<ins>**Les noms de domaines (DNS)**</ins>

<https://tailscale.com/kb/1141/aws-rds/#step-3-add-aws-dns-for-your-tailnet>

Next DNS provider ..

```hcl
resource "tailscale_dns_nameservers" "this" {
  nameservers = [
    "2a07:a8c0::9d:3ccb",
    cidrhost(module.vpc.vpc_cidr_block, 2)
  ]
}

resource "tailscale_dns_search_paths" "this" {
  search_paths = [
    "${var.region}.compute.internal"
  ]
}
```

<ins>**La clé d'authentification ("auth key")**</ins>

```hcl
resource "tailscale_tailnet_key" "this" {
  reusable      = true
  ephemeral     = false
  preauthorized = true
}
```
autoApprovers

#### Le module Terraform

J'ai créé un module très simple qui permet de déployer un autoscaling group sur AWS et de configurer Tailscale pour qu'au démarrage d'une instance elle rejoigne le Tailnet et que le routage vers les ressources privées soit immédiatement possible.
Ce module est publié dans le registry Terraform [ici](https://registry.terraform.io/modules/Smana/tailscale-subnet-router/aws/latest).

```hcl
module "tailscale_subnet_router" {
  source  = "Smana/tailscale-subnet-router/aws"
  version = "1.0.3"

  region = var.region
  env    = var.env

  name     = var.tailscale.subnet_router_name
  auth_key = tailscale_tailnet_key.this.key

  vpc_id                = module.vpc.vpc_id
  subnet_ids            = module.vpc.private_subnets
  advertise_routes      = [module.vpc.vpc_cidr_block]
...
}
```

### 💻 Une autre façon de faire du SSH

```console
ss -tulnp sport = :22
Netid                State                 Recv-Q                Send-Q                                 Local Address:Port                                 Peer Address:Port                Process
tcp                  LISTEN                0                     4096                                               *:22                                              *:*
```

```console
ssh root@ip-10-0-45-95
Rejected. Source key was revoked.
root@ip-10-0-45-95: Permission denied (tailscale).

sudo tailscale up --force-reauth --accept-routes

ssh root@ip-10-0-45-95
# Authentication checked with Tailscale SSH.
# Time since last authentication: 4s
Welcome to Ubuntu 23.04 (GNU/Linux 6.2.0-1013-aws x86_64)
...
root@ip-10-0-45-95:~#


ssh ubuntu@ip-10-0-45-95
# Tailscale SSH requires an additional check.
# To authenticate, visit: https://login.tailscale.com/a/31e3dd2d681c
```

### 🗄️ Accéder à une base de données RDS

Depuis le poste de travail

```console
nc -zv demo-tailscale.cymnaynfchjt.eu-west-3.rds.amazonaws.com 5432
demo-tailscale.cymnaynfchjt.eu-west-3.rds.amazonaws.com [10.0.52.80] 5432 (postgresql) open

psql -h demo-tailscale.cymnaynfchjt.eu-west-3.rds.amazonaws.com -U postgres
Password for user postgres:
psql (15.4, server 15.3)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, compression: off)
Type "help" for help.

postgres=>
```

### ☸ Plusieurs options sur Kubernetes

* Subnet router
* Proxy
* Sidecar
* Operator



## 👐 Open source et tarifs

Pricing raisonnable avec une vrai politique OpenSource et je suis pour payer une solution lorsqu'elle le mérite et que cela reste raisonnable. Encourager ces sociétés qui ont une sensibilité Open Source

Le [client Tailscale](https://github.com/tailscale/tailscale) est Open source sous license `BSD 3-Clause`

Self hosted <https://github.com/juanfont/headscale> non lié à la société Tailscale.

## 💭 Dernières remarques

Cloudflare, interface web, latencie. Ça fait longtemps.

Ce qui distingue Tailscale des autres VPN, c'est sa capacité à configurer des connexions de manière quasi instantanée sans nécessiter de configuration complexe.

## 🔖 References

<https://tailscale.com/blog/how-tailscale-works/#the-control-plane-key-exchange-and-coordination>
<https://tailscale.com/blog/2019-09-10-zero-trust/>
