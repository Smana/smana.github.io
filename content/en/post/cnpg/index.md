+++
author = "Smaine Kahlouch"
title = '`CloudNativePG`: An easy way to run a PostgreSQL on Kubernetes'
date = "2022-10-28"
summary = "**CloudNativePG** is a Kubernetes operator that helps running and operationg PostgreSQL databases. Here we'll demonstrate how to create a server, perform backups and recoveries, monitoring and a few tips."
featureImage = "cnpg.png"
featured = true
codeMaxLines = 20
usePageBundles = true
toc = true
tags = [
    "data"
]
thumbnail= "cloudnativepg.png"
+++

While Kubernetes is now the de-facto platform for orchestrating stateless applications (Because containers don't store data they can be destroyed and recreated elsewhere), running applications that require to persist data in an ephemeral environment can be **quite challenging**. I'm convinced this is possible and we're seing several mature cloud-native database solutions (CockroachDB, TiDB, K8ssandra, Strimzi...) but there are a few **things to consider**:

* Maturity of the operator, declarative approach through CRDs.
* How to handle the storage (CSI, PVCs, local storage)
* Resiliency. What happens when something goes wrong?
* Backup / Recovery. How easy is it? Scheduled backups
* Replication / Scaling
* Connection Pooling
* Observability

I was looking for a solution to host a **PostgreSQL database**. This database is a requirement for a ticket reservation software named [Alf.io](https://alf.io/) that's being used for an upcoming event: [The Kubernetes Community Days France](https://community.cncf.io/events/details/cncf-kcd-france-presents-kubernetes-community-days-france-2023/). (By the way you're welcome to submit a talk 👐).

I considered several solutions but I was looking for a cloud agnostic and easy solution. I already played with several Kubernetes operators and I finally ended up looking at a new kid on the block: [**CloudNativePG**](https://cloudnative-pg.io/)

>CloudNativePG is the Kubernetes operator that covers the full lifecycle of a highly available PostgreSQL database cluster with a primary/standby architecture, using native streaming replication.

It has been submitted to the **CNCF** in order to join the _Sandbox_ projects.


## :bullseye: Our target

Here I'm gonna try to give an overview of the main CloudNativePG features.
Basically the idea is to create a PostgreSQL database on a GKE cluster.
Then we'll add a standby instance and we'll run a few resiliency tests.
We may want to see how it behaves in terms of performances and what are the observability tools available.
Finally we'll have a look to the backup/restore methods.


{{% notice info Info %}}
All the steps in this article will be made manually but you should consider using a GitOps engine.
Using Flux to manage everything has been covered in a [previous article](/post/deflux).

You may want to check a complete **example** that I built for my own use case [here](https://github.com/cncfparis/kcdfrance-gitops).
{{% /notice %}}


## :ballot_box_with_check: Requirements

### :inbox_tray: Install required tools

CloudNativePG comes with a handy `kubectl` plugin that gives insights of your PostgreSQL instance and allows to perform some operations.
It can be installed using [krew](https://krew.sigs.k8s.io/) as follows

```console
kubectl krew install cnpg
```

As this article is about running it in Google Cloud, you'll need to install the Google Cloud SDK using [this documentation](https://cloud.google.com/sdk/docs/install-sdk) and configure it in order to be able to create resources on your own GCP project.


### ☁️ Create the Google cloud requirements

There are  a few things to configure before creating our PostgreSQL instance: We'll create a bucket  which stores the backups and [WAL files](https://www.postgresql.org/docs/15/wal-intro.html). Then we need to give the permissions to our pods so that they'll be able to write into this newly created bucket.

Create the bucket using `gcloud` CLI

```console
gcloud storage buckets create --location=eu --default-storage-class=coldline gs://cnpg-ogenki
Creating gs://cnpg-ogenki/...

gcloud storage buckets describe gs://cnpg-ogenki
[...]
name: cnpg-ogenki
owner:
  entity: project-owners-xxxx0008
projectNumber: 'xxx00008'
rpo: DEFAULT
selfLink: https://www.googleapis.com/storage/v1/b/cnpg-ogenki
storageClass: STANDARD
timeCreated: '2022-10-15T19:27:54.364000+00:00'
updated: '2022-10-15T19:27:54.364000+00:00'
```

Give the permissions to the pods (PostgreSQL server) to write/read from the bucket using [**Workload Identity**](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).

{{% notice note Note %}}
The GKE cluster should be configured with _Workload Identity_ enabled.
Check your cluster configuration, you should get something with this command:

```console
gcloud container clusters describe <cluster_name> --format json --zone <zone> | jq .workloadIdentityConfig
{
  "workloadPool": "{{ gcp_project }}.svc.id.goog"
}
```
{{% /notice %}}


Create a Google Cloud service account

```console
gcloud iam service-accounts create cloudnative-pg --project=cloud-native-computing-paris
Created service account [cloudnative-pg].
```

Assign the `storage.admin` permission to the serviceaccount

```console
gcloud projects add-iam-policy-binding cloud-native-computing-paris \
--member "serviceAccount:cloudnative-pg@{{ gcp_project }}.iam.gserviceaccount.com" \
--role "roles/storage.admin"
[...]
- members:
  - serviceAccount:cloudnative-pg@{{ gcp_project }}.iam.gserviceaccount.com
  role: roles/storage.admin
etag: BwXrGA_VRd4=
version: 1
```

Allow the Kubernetes service account to impersonate the IAM service account. <br>
:information_source: ensure you use the proper format `serviceAccount:{{ gcp_project }}.svc.id.goog[{{ kubernetes_namespace }}/{{ kubernetes_serviceaccount }}]`

```console
gcloud iam service-accounts add-iam-policy-binding cloudnative-pg@{{ gcp_project }}.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser --member "serviceAccount:{{ gcp_project }}.svc.id.goog[demo/ogenki]"
Updated IAM policy for serviceAccount [cloudnative-pg@{{ gcp_project }}.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:cloud-native-computing-paris.svc.id.goog[demo/ogenki]
  role: roles/iam.workloadIdentityUser
etag: BwXrGBjt5kQ=
version: 1
```

We're ready to create the Kubernetes resources :muscle:

### :key: Create the users secrets

We need to create the users credentials that will be used during the bootstrap process (more info later on): The superuser and the newly created database owner.

```console
kubectl create secret generic cnpg-mydb-superuser --from-literal=username=postgres --from-literal=password=foobar --namespace demo
secret/cnpg-mydb-superuser created
```

```console
kubectl create secret generic cnpg-mydb-user --from-literal=username=smana --from-literal=password=barbaz --namespace demo
secret/cnpg-mydb-user created
```


## :hammer_and_wrench: Deploy the CloudNativePG operator using Helm

CloudNativePG is basically a Kubernetes operator which comes with some CRD's. Here we'll use the Helm chart.

```console
helm repo add cnpg https://cloudnative-pg.github.io/charts

helm upgrade --install cnpg --namespace cnpg-system \
--create-namespace charts/cloudnative-pg

kubectl get po -n cnpg-system
NAME                    READY   STATUS      RESTARTS   AGE
cnpg-74488f5849-8lhjr   1/1     Running     0          6h17m
```

## :rocket: Create a PostgreSQL server

<center><img src="single_instance.png" alt="single_instance" width="600" /></center>

Now we can create our first instance using the **custom resource** `Cluster`. In the following definition we want to start a PostgreSQL server, automatically create a database named `mydb`  and configure the credentials based on the [secrets created previously](#key-create-the-users-secrets).
For a full list of the available parameters please refer to the [API reference](https://cloudnative-pg.io/documentation/1.17/api_reference/).

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ogenki
  namespace: demo
spec:
  description: "PostgreSQL Demo Ogenki"
  imageName: ghcr.io/cloudnative-pg/postgresql:14.5
  instances: 1

  inheritedMetadata:
    annotations:
      iam.gke.io/gcp-service-account: cloudnative-pg@{{ gcp_project }}.iam.gserviceaccount.com

  bootstrap:
    initdb:
      database: mydb
      owner: smana
      secret:
        name: cnpg-mydb-user

  superuserSecret:
    name: cnpg-mydb-superuser

  storage:
    storageClass: standard
    size: 10Gi

  backup:
    barmanObjectStore:
      destinationPath: "gs://cnpg-ogenki"
      googleCredentials:
        gkeEnvironment: true
    retentionPolicy: "30d"

  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1Gi"
```

```console
kubectl create ns demo
namespace/demo created
```

```console
kubectl apply -f cluster.yaml
cluster.postgresql.cnpg.io/ogenki created
```

```console
kubectl cnpg status ogenki -n demo
Cluster Summary
Primary server is initializing
Name:              ogenki
Namespace:         demo
PostgreSQL Image:  ghcr.io/cloudnative-pg/postgresql:14.5
Primary instance:   (switching to ogenki-1)
Status:            Setting up primary Creating primary instance ogenki-1
Instances:         1
Ready instances:   0

Certificates Status
Certificate Name    Expiration Date                Days Left Until Expiration
----------------    ---------------                --------------------------
ogenki-ca           2023-01-13 20:02:40 +0000 UTC  90.00
ogenki-replication  2023-01-13 20:02:40 +0000 UTC  90.00
ogenki-server       2023-01-13 20:02:40 +0000 UTC  90.00

Continuous Backup status
First Point of Recoverability:  Not Available
No Primary instance found
Streaming Replication status
Not configured

Instances status
Name  Database Size  Current LSN  Replication role  Status  QoS  Manager Version  Node
----  -------------  -----------  ----------------  ------  ---  ---------------  ----
```

The first thing that runs under the hood is the **bootstrap** process. In our example we create a brand new database named `mydb` with an owner `smana` and credentials are retrieved from the secret we created previously.

```yaml
[...]
  bootstrap:
    initdb:
      database: mydb
      owner: smana
      secret:
        name: cnpg-mydb-user
[...]
```

```console
kubectl get po -n demo
NAME                    READY   STATUS      RESTARTS   AGE
ogenki-1                0/1     Running     0          55s
ogenki-1-initdb-q75cz   0/1     Completed   0          2m32s
```

After a few seconds the cluster becomes ready

```console
kubectl cnpg status ogenki -n demo
Cluster Summary
Name:               ogenki
Namespace:          demo
System ID:          7154833472216277012
PostgreSQL Image:   ghcr.io/cloudnative-pg/postgresql:14.5
Primary instance:   ogenki-1
Status:             Cluster in healthy state
Instances:          1
Ready instances:    1

[...]

Instances status
Name      Database Size  Current LSN  Replication role  Status  QoS        Manager Version  Node
----      -------------  -----------  ----------------  ------  ---        ---------------  ----
ogenki-1  33 MB          0/17079F8    Primary           OK      Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-xczh
```

## 🩹 Standby instance and resiliency

Add a standby instance by setting up the number of replicas to 2.

<center><img src="standby.png" alt="standby" width="600" /></center>

```console
kubectl edit cluster -n demo ogenki
cluster.postgresql.cnpg.io/ogenki edited
```

```console
kubectl cnpg status -n demo ogenki
Cluster Summary
Name:               ogenki
Namespace:          demo
System ID:          7155095145869606932
PostgreSQL Image:   ghcr.io/cloudnative-pg/postgresql:14.5
Primary instance:   ogenki-1
Status:             Creating a new replica Creating replica ogenki-2-join
Instances:          2
Ready instances:    1
Current Write LSN:  0/1707A30 (Timeline: 1 - WAL File: 000000010000000000000001)
```

```console
kubectl get po -n demo
NAME                  READY   STATUS    RESTARTS   AGE
ogenki-1              1/1     Running   0          3m16s
ogenki-2-join-xxrwx   0/1     Pending   0          82s
```

```console
kubectl cnpg status -n demo ogenki
Cluster Summary
Name:               ogenki
Namespace:          demo
System ID:          7155095145869606932
PostgreSQL Image:   ghcr.io/cloudnative-pg/postgresql:14.5
Primary instance:   ogenki-1
Status:             Cluster in healthy state
Instances:          2
Ready instances:    2
Current Write LSN:  0/3000060 (Timeline: 1 - WAL File: 000000010000000000000003)

[...]

Streaming Replication status
Name      Sent LSN   Write LSN  Flush LSN  Replay LSN  Write Lag  Flush Lag  Replay Lag  State      Sync State  Sync Priority
----      --------   ---------  ---------  ----------  ---------  ---------  ----------  -----      ----------  -------------
ogenki-2  0/3000060  0/3000060  0/3000060  0/3000060   00:00:00   00:00:00   00:00:00    streaming  async       0

Instances status
Name      Database Size  Current LSN  Replication role  Status  QoS        Manager Version  Node
----      -------------  -----------  ----------------  ------  ---        ---------------  ----
ogenki-1  33 MB          0/3000060    Primary           OK      Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-76k7
ogenki-2  33 MB          0/3000060    Standby (async)   OK      Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-xszc
```

<center><img src="promote.png" alt="promote" width="600" /></center>


```console
kubectl cnpg promote ogenki ogenki-2 -n demo
Node ogenki-2 in cluster ogenki will be promoted
```

```console
kubectl cnpg status -n demo ogenki
[...]

Streaming Replication status
Name      Sent LSN   Write LSN  Flush LSN  Replay LSN  Write Lag  Flush Lag  Replay Lag  State      Sync State  Sync Priority
----      --------   ---------  ---------  ----------  ---------  ---------  ----------  -----      ----------  -------------
ogenki-1  0/4004CA0  0/4004CA0  0/4004CA0  0/4004CA0   00:00:00   00:00:00   00:00:00    streaming  async       0

Instances status
Name      Database Size  Current LSN  Replication role  Status  QoS        Manager Version  Node
----      -------------  -----------  ----------------  ------  ---        ---------------  ----
ogenki-2  33 MB          0/4004CA0    Primary           OK      Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-xszc
ogenki-1  33 MB          0/4004CA0    Standby (async)   OK      Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-76k7
```

Now let's simulate a failover by deleting the primary pod

<center><img src="failover.png" alt="failover" width="600" /></center>

```console
Cluster Summary
Name:               ogenki
Namespace:          demo
System ID:          7155095145869606932
PostgreSQL Image:   ghcr.io/cloudnative-pg/postgresql:14.5
Primary instance:   ogenki-1
Status:             Failing over Failing over from ogenki-2 to ogenki-1
Instances:          2
Ready instances:    1
Current Write LSN:  0/4005D98 (Timeline: 3 - WAL File: 000000030000000000000004)

[...]
Instances status
Name      Database Size  Current LSN  Replication role  Status             QoS        Manager Version  Node
----      -------------  -----------  ----------------  ------             ---        ---------------  ----
ogenki-1  33 MB          0/40078D8    Primary           OK                 Burstable  1.17.0           gke-kcdfrance-main-np-0e87115b-76k7
ogenki-2  -              -            -                 pod not available  Burstable  -                gke-kcdfrance-main-np-0e87115b-xszc
```

After a few seconds, the cluster is healthy again

```console
kubectl get cluster -n demo
NAME     AGE   INSTANCES   READY   STATUS                     PRIMARY
ogenki   13m   2           2       Cluster in healthy state   ogenki-1
```

## 👁️ Monitoring


```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  labels:
    prometheus-instance: main
  name: cnpg-ogenki
  namespace: demo
spec:
  namespaceSelector:
    matchNames:
    - demo
  podMetricsEndpoints:
  - port: metrics
  selector:
    matchLabels:
      postgresql: ogenki
```

![observability](observability.png)


## :fire: Performances and benchmark

This is worth running a **performance test** in order to know the limits of your current server and keep a baseline for further improvements.

{{% notice note Note %}}
When it comes to performances there are many improvement areas we can work on. It mostly depends on the **target** we want to achieve. Indeed we don't want to waste time and money for performances we likely never need.

Here are the main things to look at:

* PostgreSQL tuning
* **Compute** resources (cpu and memory)
* **Disk** type IOPS, local storage ([local-volume-provisioner](https://docs.pingcap.com/tidb-in-kubernetes/stable/deploy-on-gcp-gke#use-local-storage)),
* dedicated disk for **WAL**
* Connection **pooling** [PGBouncer](https://cloudnative-pg.io/documentation/1.17/connection_pooling/#connection-pooling)
* Database optimization, analyzing the query plans using [**explain**](https://www.postgresql.org/docs/current/performance-tips.html)
{{% /notice %}}


```console
git clone git@github.com:EnterpriseDB/cnp-bench.git
cd cnp-bench

cat > hammerdb-benchmark/myvalues.yaml <<EOF
cnp:
  existingCluster: true
  existingSuperuserCredentials: cnpg-mydb-superuser
  existingHost: ogenki-rw
EOF

PG_NODE=$(kubectl get po -n demo -l postgresql=ogenki,role=primary -o jsonpath={.items[0].spec.nodeName})
kubectl label node ${PG_NODE} workload=postgresql
kubectl label node gke-kcdfrance-main-np-xxx-xxx workload=hammerdb

helm upgrade --install -n demo hammerdb-bench -f hammerdb-benchmark/myvalues.yaml  hammerdb-benchmark/

kubectl get po -n demo -l job-name=hammerdb-bench-hammerdb-benchmark
NAME                                      READY   STATUS     RESTARTS   AGE
hammerdb-bench-hammerdb-benchmark-lppl5   0/1     Init:1/3   0          8m5s

```

```console
kubectl logs -n demo hammerdb-bench-hammerdb-benchmark-9gthp -c hammerdb-pgschemabuild -f
[...]
Vuser 2:Loading Orders for D=9 W=4
Vuser 3:...1000
Vuser 2:...1000
Vuser 3:...2000
Vuser 2:...2000
Vuser 3:...3000
Vuser 3:Orders Done
Vuser 3:End:Sun Oct 16 07:38:59 UTC 2022
Vuser 3:FINISHED SUCCESS
Vuser 2:...3000
Vuser 2:Orders Done
Vuser 2:Loading Orders for D=10 W=4
Vuser 2:...1000
Vuser 2:...2000
Vuser 2:...3000
Vuser 2:Orders Done
Vuser 2:End:Sun Oct 16 07:39:00 UTC 2022
Vuser 2:FINISHED SUCCESS
```


## 💽 Backup and Restore


## :broom: Cleanup



## 💭 final thoughts

* Depending on the use case, the company constraints, the criticity of the application. Consider using Crossplane and composition to give an opinionated way of declaring managed databases in cloud providers. Requires more configuration.
* can be used localy or in a CI workflow