+++
author = "Smaine Kahlouch"
title = "Manage tools versions with `asdf`"
date = "2022-05-24"
summary = "install and change between different versions of our usual tools (kubectl, terraform...)"
tags = [
    "tooling",
    "local"
]
thumbnail= "images/console.png"
+++


In order to install binaries and to be able to switch from a version to another I like to use [**asdf**](https://asdf-vm.com/).

Let's install k3d

```console
asdf plugin-add k3d
```

Check the versions available

```console
asdf list-all k3d| tail -n 3
5.4.0-dev.3
5.4.0
5.4.1
```

We'll install the latest version

```console
asdf install k3d $(asdf latest k3d)
* Downloading k3d release 5.4.1...
k3d 5.4.1 installation was successful!
```

Finally we can switch from a version to another. We can set a `global` version that would be used on all directories.

```console
asdf global k3d 5.4.1
```

or use a `local` version depending on the current directory

```console
cd /tmp
asdf local k3d 5.4.1

asdf current k3d
k3d             5.4.1           /tmp/.tool-versions
```
