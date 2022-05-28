# Template challenge

Here you’ll be able to practice in order to get familiar with some of the possibilities offered by a templating language.

(pro tip: Don't forget the [testing](03_build_chart.md##testing-the-chart))

These examples may seem useless but the purpose of this is just playing with templates.

## 1 - "Configuration depends on region"

Create a secret that contains a key 'secret' and a value "myEuropeanSecret".
Set an environment variable from this secret only if the value global.region is 'eu-west1'
So the first step is to add the values into the values.yaml file.

```yaml
global:
  region: eu-west-1
```

## 2 - "Create only if"

Create a job that prints the pod's IP on stdout based on a boolean value `printIP.enabled`.
Use the "busybox" image and the command `wget -qO - http://ipinfo.io/ip`.
The job should be created only if the value is True, before every other resource has been created (pre-install and pre-upgrade **hooks**)

## 3 - "Looping"

Given the following values, create a loop whether in the deployment or in the helpers.tpl file in order to add the environment variables.

```yaml
envVars:
  key1: value1
  key2: value2
  key3: value3
```

## 4 -  "Playing with strings and sprigs"

Add to the "common labels", a new label "codename" with a value composed with the release name, the chart name and the date in the form 20061225.
The release name must be at most 3 characters long.
The whole string has to be in snakecase.

(you should get something like `codename: rel_web_20210215`)

## 5 - We want to create a list of etcd hosts in the form of "etcd-0,etcd-1,etcd-2" based on a integer that defines the number of etcd hosts

```yaml
etcd:
  count: 5
```

This list has to be defined in an environment variable ETCD_HOSTS

You can find here [**possible solutions**](templating_proposals.md)