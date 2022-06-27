---
title: 'Helm workshop: Templating exercises'
description: 'Test your skills with a few templating challenges'
summary: 'Test your skills with a few templating challenges'
date: "2021-06-07"
aliases:
  - workshop-helm
author: 'Smana'
usePageBundles: true

thumbnail: 'https://cncf-branding.netlify.app/img/projects/helm/horizontal/black/helm-horizontal-black.png'

categories:
  - devxp
tags:
  - Helm
  - Kubernetes

series:
  - Workshop Helm
---

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

# Proposition of solutions

{{% notice warning try_first %}}
You should try to find a solution by your own to the above exercises before checking these solutions
{{% /notice %}}


## 1

The following command generates a secrets in the templates directory

```console
kubectl create secret generic --dry-run=client eu-secret --from-literal=secret='myEuropeanSecret' -o yaml | kubectl neat > templates/secret.yaml
```

Then we'll enclose the environment variable definition with a condition depending on the region in the deployment template.

```yaml
          {{- if eq .Values.global.region "eu-west-1" }}
           - name: eu-secret
             valueFrom:
               secretKeyRef:
                 name: eu-secret
                 key: secret
         {{- end }}
```

## 2

First of all we need to add a new value

```yaml
printIP:
  enabled: True
```

Then this command will generate a job yaml

```console
kubectl create job my-ip --dry-run=client --image=busybox -o yaml -- wget -qO - http://ipinfo.io/ip | kubectl neat > templates/job.yaml
```

If we enclose the whole yaml, it won't be created if the boolean is False.
With the hook annotation here, the job will be created before any other resource will be applied.
We defined a delete policy "hook-failed" in order to keep the job, otherwise it would have been deleted.

```yaml
{{- if .Values.printIP.enabled -}}
apiVersion: batch/v1
kind: Job
metadata:
 name: my-ip
 annotations:
   "helm.sh/hook": pre-install,pre-upgrade
   "helm.sh/hook-weight": "1"
   "helm.sh/hook-delete-policy": hook-failed
...
{{- end -}}
```

## 3

If we want to keep the deployment easy to read, we would prefer adding the code in the _helpers.tpl

```yaml
{{/*
Environment variables
*/}}
{{- define "web.envVars" -}}
{{- range $key, $value := .Values.envVars }}
- name: {{ $key }}
  value: {{ $value }}
{{- end }}
{{- end -}}
```

Then this new variable could be used in the deployment as follows

```yaml
         env:
          {{- include "web.envVars" . | nindent 12  }}
```

## 4

The common labels can be changed in the file templates/_helpers.tpl. Here's a proposal
This one is tricky, I needed to dig back into the charts available in the stable github repository.

```yaml
codename: {{ printf "%s %s %s" (.Release.Name | trunc 3) .Chart.Name (now | date "20060102") | snakecase }}
```

## 5

Here's an option to achieve the expected results.

```yaml
         env:
           - name: ETCD_HOSTS
             value: "{{ range $index, $e := until (.Values.etcd.count|int)  }}{{- if $index }},{{end}}etcd-{{ $index }}{{- end }}"
```