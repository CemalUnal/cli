---
title: Initialize Postgres using Script Source
menu:
  docs_0.8.0:
    identifier: pg-script-source-initialization
    name: Using Script
    parent: pg-initialization-postgres
    weight: 10
menu_name: docs_0.8.0
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Initialize PostgreSQL with Script

KubeDB supports PostgreSQL database initialization. This tutorial will show you how to use KubeDB to initialize a PostgreSQL database from script.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [minikube](https://github.com/kubernetes/minikube).

Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace "demo" created

$ kubectl get ns demo
NAME    STATUS  AGE
demo    Active  5s
```

> Note: Yaml files used in this tutorial are stored in [docs/examples/postgres](https://github.com/kubedb/cli/tree/master/docs/examples/postgres) folder in github repository [kubedb/cli](https://github.com/kubedb/cli).

## Create PostgreSQL with script source

PostgreSQL supports initialization with `.sh`, `.sql` and `.sql.gz` files. In this tutorial, we will use `data.sql` script from [postgres-init-scripts](https://github.com/kubedb/postgres-init-scripts.git) git repository to create a TABLE `dashboard` in `data` Schema.

As [gitRepo](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo) volume has been deprecated, we will use a ConfigMap as script source. You can use any Kubernetes supported [volume](https://kubernetes.io/docs/concepts/storage/volumes) as script source.

At first, we will create a ConfigMap from `data.sql` file. Then, we will provide this ConfigMap as script source in `init.scriptSource` of Postgres crd spec.

Following YAML describes a Postgres object that holds VolumeSource of initialization scripts.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: Postgres
metadata:
  name: script-postgres
  namespace: demo
spec:
  version: "10.2"
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 50Mi
  init:
    scriptSource:
      configMap:
        name: pg-init-script
```

Here,

- `init.scriptSource` specifies scripts used to initialize the database when it is being created.

VolumeSource provided in `init.scriptSource` will be mounted in Pod and will be executed while creating PostgreSQL.

Let's create a ConfigMap with initialization script,

```console
$ kubectl create configmap -n demo pg-init-script \
--from-literal=data.sql="$(curl -fsSL https://raw.githubusercontent.com/kubedb/postgres-init-scripts/master/data.sql)"
configmap/pg-init-script created
```

Now, create the Postgres crd which YAML we have shown above,

```console
$ kubectl create -f https://raw.githubusercontent.com/kubedb/cli/0.8.0/docs/examples/postgres/initialization/script-postgres.yaml 
postgres.kubedb.com/script-postgres created
```

Wait until Postgres goes in `Running` state. Verify that the database is in `Running` state using following command,

```console
 $ kubectl get pg -n demo script-postgres
NAME              VERSION   STATUS    AGE
script-postgres   10.2      Running   3m
```

You can use `kubedb describe` command to view which resources has been created by KubeDB for this Postgres object.

```console
$ kubedb describe pg -n demo script-postgres
Name:               script-postgres
Namespace:          demo
CreationTimestamp:  Tue, 04 Sep 2018 16:48:57 +0600
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"kubedb.com/v1alpha1","kind":"Postgres","metadata":{"annotations":{},"name":"script-postgres","namespace":"demo"},"spec":{"init":{"script...
Replicas:           1  total
Status:             Running
Init:
  scriptSource:
Volume:
    Type:       ConfigMap (a volume populated by a ConfigMap)
    Name:       pg-init-script
    Optional:   false
  StorageType:  Durable
Volume:
  StorageClass:  standard
  Capacity:      50Mi
  Access Modes:  RWO

StatefulSet:          
  Name:               script-postgres
  CreationTimestamp:  Tue, 04 Sep 2018 16:49:00 +0600
  Labels:               kubedb.com/kind=Postgres
                        kubedb.com/name=script-postgres
  Annotations:        <none>
  Replicas:           824638975264 desired | 1 total
  Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed

Service:        
  Name:         script-postgres
  Labels:         kubedb.com/kind=Postgres
                  kubedb.com/name=script-postgres
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.109.153.29
  Port:         api  5432/TCP
  TargetPort:   api/TCP
  Endpoints:    172.17.0.6:5432

Service:        
  Name:         script-postgres-replicas
  Labels:         kubedb.com/kind=Postgres
                  kubedb.com/name=script-postgres
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.100.56.76
  Port:         api  5432/TCP
  TargetPort:   api/TCP
  Endpoints:    172.17.0.6:5432

Database Secret:
  Name:         script-postgres-auth
  Labels:         kubedb.com/kind=Postgres
                  kubedb.com/name=script-postgres
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  POSTGRES_USER:      8 bytes
  POSTGRES_PASSWORD:  16 bytes

Topology:
  Type     Pod                StartTime                      Phase
  ----     ---                ---------                      -----
  primary  script-postgres-0  2018-09-04 16:49:04 +0600 +06  Running

No Snapshots.

Events:
  Type    Reason      Age   From               Message
  ----    ------      ----  ----               -------
  Normal  Successful  4m    Postgres operator  Successfully created Service
  Normal  Successful  4m    Postgres operator  Successfully created Service
  Normal  Successful  4m    Postgres operator  Successfully created StatefulSet
  Normal  Successful  4m    Postgres operator  Successfully created Postgres
  Normal  Successful  4m    Postgres operator  Successfully patched StatefulSet
  Normal  Successful  4m    Postgres operator  Successfully patched Postgres

```

Now let's connect to our Postgres `script-postgres`  using pgAdmin we have installed in [quickstart](/docs/guides/postgres/quickstart/quickstart.md#before-you-begin) tutorial.

Connection information:

- Host name/address: you can use any of these
  - Service: `script-postgres.demo`
  - Pod IP: (`$ kubectl get pods script-postgres-0 -n demo -o yaml | grep podIP`)
- Port: `5432`
- Maintenance database: `postgres`

Run following command to get Username,

 ```console
 $ kubectl get secrets -n demo script-postgres-auth -o jsonpath='{.data.\POSTGRES_USER}' | base64 -d
postgres
 ```

Run the following command to get Password,

```console
$ kubectl get secrets -n demo script-postgres-auth -o jsonpath='{.data.\POSTGRES_PASSWORD}' | base64 -d
NC1fEq0q5XqHazB8
```

In PostgreSQL, run following query to check `pg_catalog.pg_tables` to confirm initialization.

```console
select * from pg_catalog.pg_tables where schemaname = 'data';
```

 | schemaname | tablename | tableowner | hasindexes | hasrules | hastriggers | rowsecurity |
 | ---------- | --------- | ---------- | ---------- | -------- | ----------- | ----------- |
 | data       | dashboard | postgres   | true       | false    | false       | false       |

We can see TABLE `dashboard` in `data` Schema which is created for initialization.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
$ kubectl patch -n demo pg/script-postgres -p '{"spec":{"doNotPause":false}}' --type="merge"
$ kubectl delete -n demo pg/script-postgres

$ kubectl patch -n demo drmn/script-postgres -p '{"spec":{"wipeOut":true}}' --type="merge"
$ kubectl delete -n demo drmn/script-postgres

$ kubectl delete -n demo configmap/pg-init-script

$ kubectl delete ns demo
```

## Next Steps

- Learn about [taking instant backup](/docs/guides/postgres/snapshot/instant_backup.md) of PostgreSQL database using KubeDB Snapshot.
- Learn about initializing [PostgreSQL from KubeDB Snapshot](/docs/guides/postgres/initialization/snapshot_source.md).
- Want to setup PostgreSQL cluster? Check how to [configure Highly Available PostgreSQL Cluster](/docs/guides/postgres/clustering/ha_cluster.md)
- Wondering what features are coming next? Please visit [here](/docs/roadmap.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
