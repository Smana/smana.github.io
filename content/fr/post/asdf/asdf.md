+++
author = "Smaine Kahlouch"
title = "Gérer les versions d'outils avec `asdf`"
date = "2022-05-24"
summary = "Installer et changer entre différentes versions de nos outils habituels (Kubectl, Terraform ...)"
tags = [
    "tooling",
    "local"
]
thumbnail= "images/console.png"
+++


Afin d'installer des binaires et de pouvoir passer d'une version à une autre, j'aime utiliser[**asdf**](https://asdf-vm.com/).

### :inbox_tray: Installation

L'installation recommandée consiste à utiliser Git comme suit

```console
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.10.0
```

Ensuite, selon votre coquille, voici les étapes restantes à suivre
```console
. $HOME/.asdf/asdf.sh
```

Et vous voudrez peut-être configurer l'achèvement du shell
```console
. $HOME/.asdf/completions/asdf.bash
```
<br>

### :rocket: Prenons un exemple

Liste tous les plugins disponibles et recherchez`k3d`

```console
asdf plugin-list-all | grep k3d
k3d                           https://github.com/spencergilbert/asdf-k3d.git
```

Installons K3D
```console
asdf plugin-add k3d
```

Vérifiez les versions disponibles
```console
asdf list-all k3d| tail -n 3
5.4.0-dev.3
5.4.0
5.4.1
```

Nous installerons la dernière version
```console
asdf install k3d latest
* Downloading k3d release 5.4.1...
k3d 5.4.1 installation was successful!
```

Enfin, nous pouvons passer d'une version à une autre.Nous pouvons définir une version "globale" qui serait utilisée sur tous les répertoires.
```console
asdf global k3d 5.4.1
```

ou utilisez une version `locale` en fonction du répertoire actuel
```console
cd /tmp
asdf local k3d 5.4.1

asdf current k3d
k3d             5.4.1           /tmp/.tool-versions
```
<br>

### :broom: Nettoyer

Désinstaller une version donnée

```console
asdf uninstall k3d 5.4.1
```

Retirer un plugin

```console
asdf plugin remove k3d
```
