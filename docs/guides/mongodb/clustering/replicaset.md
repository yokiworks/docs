---
title: MongoDB ReplicaSet Guide
menu:
  docs_{{ .version }}:
    identifier: mg-clustering-replicaset
    name: ReplicaSet Guide
    parent: mg-clustering-mongodb
    weight: 15
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# KubeDB - MongoDB ReplicaSet

This tutorial will show you how to use KubeDB to run a MongoDB ReplicaSet.

## Before You Begin

Before proceeding:

- Read [mongodb replication concept](/docs/guides/mongodb/clustering/replication_concept.md) to learn about MongoDB Replica Set clustering.

- You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial. Run the following command to prepare your cluster for this tutorial:

  ```console
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: The yaml files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Deploy MongoDB ReplicaSet

To deploy a MongoDB ReplicaSet, user have to specify `spec.replicaSet` option in `Mongodb` CRD.

The following is an example of a `Mongodb` object which creates MongoDB ReplicaSet of three members.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-replicaset
  namespace: demo
spec:
  version: "4.1"
  replicas: 3
  replicaSet:
    name: rs0
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/del-dorm/docs/examples/mongodb/clustering/demo-1.yaml
mongodb.kubedb.com/mgo-replicaset created
```

Here,

- `spec.replicaSet` represents the configuration for replicaset.
  - `name` denotes the name of mongodb replicaset.
  - `keyFileSecret` is deprecated now. Use `spec.certificateSecret` instead. For existing MongoDB instances, KubeDB operator will handle the migration by itself. `keyFileSecret` field will be removed in future.
- `spec.certificateSecret` (optional) is a secret name that contains keyfile (a random string)against `key.txt` key. Each mongod instances in the replica set and `shardTopology` uses the contents of the keyfile as the shared password for authenticating other members in the replicaset. Only mongod instances with the correct keyfile can join the replica set. _User can provide the `certificateSecret` by creating a secret with key `key.txt`. See [here](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/#create-a-keyfile) to create the string for `certificateSecret`._ If `certificateSecret` is not given, KubeDB operator will generate a `certificateSecret` itself.
- `spec.replicas` denotes the number of members in `rs0` mongodb replicaset.
- `spec.storage` specifies the StorageClass of PVC dynamically allocated to store data for this database. This storage spec will be passed to the StatefulSet created by KubeDB operator to run database pods. So, each members will have a pod of this storage configuration. You can specify any StorageClass available in your cluster with appropriate resource requests.

KubeDB operator watches for `MongoDB` objects using Kubernetes api. When a `MongoDB` object is created, KubeDB operator will create a new StatefulSet and a Service with the matching MongoDB object name. KubeDB operator will also create a governing service for StatefulSets with the name `<mongodb-name>-gvr`.

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS    AGE
mgo-replicaset   4.1       Running   3m13s
```

```console
$ kubedb describe mg -n demo mgo-replicaset
Name:               mgo-replicaset
Namespace:          demo
CreationTimestamp:  Wed, 29 Jan 2020 17:10:01 +0600
Labels:             <none>
Annotations:        <none>
Replicas:           3  total
Status:             Running
  StorageType:      Durable
Volume:
  StorageClass:  standard
  Capacity:      1Gi
  Access Modes:  RWO

StatefulSet:          
  Name:               mgo-replicaset
  CreationTimestamp:  Wed, 29 Jan 2020 17:10:03 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mgo-replicaset
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mgo-replicaset
  Annotations:        <none>
  Replicas:           824639242716 desired | 3 total
  Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed

Service:        
  Name:         mgo-replicaset
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-replicaset
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-replicaset
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.104.249.236
  Port:         db  27017/TCP
  TargetPort:   db/TCP
  Endpoints:    10.244.1.11:27017,10.244.2.5:27017,10.244.2.7:27017

Service:        
  Name:         mgo-replicaset-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-replicaset
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-replicaset
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.11:27017,10.244.2.5:27017,10.244.2.7:27017

Database Secret:
  Name:         mgo-replicaset-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-replicaset
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-replicaset
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  username:  4 bytes
  password:  16 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  Normal  Successful  3m    MongoDB operator  Successfully created stats service
  Normal  Successful  3m    MongoDB operator  Successfully created Service
  Normal  Successful  16s   MongoDB operator  Successfully created StatefulSet demo/mgo-replicaset
  Normal  Successful  16s   MongoDB operator  Successfully created MongoDB
  Normal  Successful  16s   MongoDB operator  Successfully created appbinding
  Normal  Successful  16s   MongoDB operator  Successfully patched stats service
  Normal  Successful  16s   MongoDB operator  Successfully patched StatefulSet demo/mgo-replicaset
  Normal  Successful  16s   MongoDB operator  Successfully patched MongoDB
```

```console
$ kubectl get statefulset -n demo
NAME             READY   AGE
mgo-replicaset   3/3     3m51s

$ kubectl get pvc -n demo
NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-mgo-replicaset-0   Bound    pvc-0d1b263d-9e04-4451-8cae-00bfd3713f66   1Gi        RWO            standard       4m
datadir-mgo-replicaset-1   Bound    pvc-46dfe664-d35c-4382-b312-547861192505   1Gi        RWO            standard       3m21s
datadir-mgo-replicaset-2   Bound    pvc-a2307675-fe38-48c1-bbe1-b8c56af7df8a   1Gi        RWO            standard       96s

$ kubectl get pv -n demo
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS   REASON   AGE
pvc-0d1b263d-9e04-4451-8cae-00bfd3713f66   1Gi        RWO            Delete           Bound    demo/datadir-mgo-replicaset-0   standard                4m9s
pvc-46dfe664-d35c-4382-b312-547861192505   1Gi        RWO            Delete           Bound    demo/datadir-mgo-replicaset-1   standard                3m23s
pvc-a2307675-fe38-48c1-bbe1-b8c56af7df8a   1Gi        RWO            Delete           Bound    demo/datadir-mgo-replicaset-2   standard                105s

$ kubectl get service -n demo
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
mgo-replicaset       ClusterIP   10.104.249.236   <none>        27017/TCP   4m21s
mgo-replicaset-gvr   ClusterIP   None             <none>        27017/TCP   4m21s
```

KubeDB operator sets the `status.phase` to `Running` once the database is successfully created. Run the following command to see the modified MongoDB object:

```yaml
$ kubedb get mg -n demo mgo-replicaset -o yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  creationTimestamp: "2020-01-29T11:10:01Z"
  finalizers:
  - kubedb.com
  generation: 3
  name: mgo-replicaset
  namespace: demo
  resourceVersion: "38302"
  selfLink: /apis/kubedb.com/v1alpha1/namespaces/demo/mongodbs/mgo-replicaset
  uid: b48140ff-f95e-46bd-8acb-15005b6677c6
spec:
  certificateSecret:
    secretName: mgo-replicaset-cert
  clusterAuthMode: keyFile
  databaseSecret:
    secretName: mgo-replicaset-auth
  podTemplate:
    controller: {}
    metadata: {}
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  kubedb.com/kind: MongoDB
                  kubedb.com/name: mgo-replicaset
              namespaces:
              - demo
              topologyKey: kubernetes.io/hostname
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  kubedb.com/kind: MongoDB
                  kubedb.com/name: mgo-replicaset
              namespaces:
              - demo
              topologyKey: failure-domain.beta.kubernetes.io/zone
            weight: 50
      livenessProbe:
        exec:
          command:
          - bash
          - -c
          - "if [[ $(mongo admin --host=localhost  --username=$MONGO_INITDB_ROOT_USERNAME
            --password=$MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase=admin
            --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then \n
            \         exit 0\n        fi\n        exit 1"
        failureThreshold: 3
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 5
      readinessProbe:
        exec:
          command:
          - bash
          - -c
          - "if [[ $(mongo admin --host=localhost  --username=$MONGO_INITDB_ROOT_USERNAME
            --password=$MONGO_INITDB_ROOT_PASSWORD --authenticationDatabase=admin
            --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then \n
            \         exit 0\n        fi\n        exit 1"
        failureThreshold: 3
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 1
      resources: {}
      serviceAccountName: mgo-replicaset
  replicaSet:
    name: rs0
  replicas: 3
  serviceTemplate:
    metadata: {}
    spec: {}
  sslMode: disabled
  storage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    storageClassName: standard
  storageType: Durable
  terminationPolicy: Delete
  updateStrategy:
    type: RollingUpdate
  version: "4.1"
status:
  observedGeneration: 3
  phase: Running
```

Please note that KubeDB operator has created a new Secret called `mgo-replicaset-auth` *(format: {mongodb-object-name}-auth)* for storing the password for `mongodb` superuser. This secret contains a `username` key which contains the *username* for MongoDB superuser and a `password` key which contains the *password* for MongoDB superuser.

If you want to use custom or existing secret please specify that when creating the MongoDB object using `spec.databaseSecret.secretName`. While creating this secret manually, make sure the secret contains these two keys containing data `username` and `password`. For more details, please see [here](/docs/concepts/databases/mongodb.md#specdatabasesecret).

## Redundancy and Data Availability

Now, you can connect to this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/). In this tutorial, we will insert document on primary member, and we will see if the data becomes available on secondary members.

At first, insert data inside primary member `rs0:PRIMARY`.

```console
$ kubectl get secrets -n demo mgo-replicaset-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mgo-replicaset-auth -o jsonpath='{.data.\password}' | base64 -d
2wT7CfFt_KDEdYww

$ kubectl exec -it mgo-replicaset-0 -n demo bash

mongodb@mgo-replicaset-0:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017/admin
MongoDB server version: 3.6.6
Welcome to the MongoDB shell.

rs0:PRIMARY> > rs.isMaster().primary
mgo-replicaset-0.mgo-replicaset-gvr.demo.svc.cluster.local:27017

rs0:PRIMARY> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB

rs0:PRIMARY> show users
{
	"_id" : "admin.root",
	"userId" : UUID("faa9d5bc-e713-4f7e-8d4b-3a90ea258faf"),
	"user" : "root",
	"db" : "admin",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}

rs0:PRIMARY> use newdb
switched to db newdb

rs0:PRIMARY> db.movie.insert({"name":"batman"});
WriteResult({ "nInserted" : 1 })

rs0:PRIMARY> db.movie.find().pretty()
{ "_id" : ObjectId("5b5efeea9d097ca0600694a3"), "name" : "batman" }

rs0:PRIMARY> exit
bye
```

Now, check the redundancy and data availability in secondary members.
We will exec in `mgo-replicaset-1`(which is secondary member right now) to check the data availability.

```console
$ kubectl exec -it mgo-replicaset-1 -n demo bash
mongodb@mgo-replicaset-1:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

rs0:SECONDARY> rs.slaveOk()
rs0:SECONDARY> > show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
newdb   0.000GB

rs0:SECONDARY> show users
{
	"_id" : "admin.root",
	"userId" : UUID("faa9d5bc-e713-4f7e-8d4b-3a90ea258faf"),
	"user" : "root",
	"db" : "admin",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}

rs0:SECONDARY> use newdb
switched to db newdb

rs0:SECONDARY> db.movie.find().pretty()
{ "_id" : ObjectId("5b5efeea9d097ca0600694a3"), "name" : "batman" }

rs0:SECONDARY> exit
bye

```

## Automatic Failover

To test automatic failover, we will force the primary member to restart. As the primary member (`pod`) becomes unavailable, the rest of the members will elect a primary member by election.

```console
$ kubectl get pods -n demo
NAME               READY     STATUS    RESTARTS   AGE
mgo-replicaset-0   1/1       Running   0          1h
mgo-replicaset-1   1/1       Running   0          1h
mgo-replicaset-2   1/1       Running   0          1h

$ kubectl delete pod -n demo mgo-replicaset-0
pod "mgo-replicaset-0" deleted

$ kubectl get pods -n demo
NAME               READY     STATUS        RESTARTS   AGE
mgo-replicaset-0   1/1       Terminating   0          1h
mgo-replicaset-1   1/1       Running       0          1h
mgo-replicaset-2   1/1       Running       0          1h

```

Now verify the automatic failover, Let's exec in `mgo-replicaset-1` pod,

```console
$ kubectl exec -it mgo-replicaset-1 -n demo bash
mongodb@mgo-replicaset-1:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017/admin
MongoDB server version: 3.6.6
Welcome to the MongoDB shell.

rs0:SECONDARY> rs.isMaster().primary
mgo-replicaset-2.mgo-replicaset-gvr.demo.svc.cluster.local:27017

# Also verify, data persistency
rs0:SECONDARY> rs.slaveOk()
rs0:SECONDARY> > show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
newdb   0.000GB

rs0:SECONDARY> show users
{
	"_id" : "admin.root",
	"userId" : UUID("faa9d5bc-e713-4f7e-8d4b-3a90ea258faf"),
	"user" : "root",
	"db" : "admin",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}

rs0:SECONDARY> use newdb
switched to db newdb

rs0:SECONDARY> db.movie.find().pretty()
{ "_id" : ObjectId("5b5efeea9d097ca0600694a3"), "name" : "batman" }
```

## Halt Database

User can halt the database by setting `spec.halted` to `true`. As the name suggests, this will halt the database by deleting all the kubernetes resources (including `sts`, `services`, rbac resources, etc) except `secrets` and `PVCs`. On the other hand, when users wishes to resume a halted database, they will need to set `spec.halted` to `false`.

Please Note that, if [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is set to `DoNotTerminate`, the halt database operation will fail with forbidden error message from admission controller. 

If [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is not set to `DoNotTerminate`, then kubedb will accept the user's halt request and also it will update the `spec.terminationPolicy` to `Halt`.

Set `spec.halted` to `true`.

```console
$ kubectl patch -n demo mg/mgo-replicaset -p '{"spec":{"halted":true}}' --type="merge"
mongodb.kubedb.com/mgo-replicaset patched
```

Database status will be set to `Halted` after successful deletion of kubernetes resources.

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS   AGE
mgo-replicaset   4.1       Halted   30m
```

```console
$ kubedb describe mg -n demo mgo-replicaset
Name:               mgo-replicaset
Namespace:          demo
CreationTimestamp:  Wed, 29 Jan 2020 17:10:01 +0600
Labels:             <none>
Annotations:        <none>
Replicas:           3  total
Status:             Halted
  StorageType:      Durable
Volume:
  StorageClass:  standard
  Capacity:      1Gi
  Access Modes:  RWO

Database Secret:
  Name:         mgo-replicaset-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-replicaset
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-replicaset
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  password:  16 bytes
  username:  4 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  Normal  Successful  18m   MongoDB operator  Successfully created stats service
  Normal  Successful  18m   MongoDB operator  Successfully created Service
  Normal  Successful  15m   MongoDB operator  Successfully created StatefulSet demo/mgo-replicaset
  Normal  Successful  15m   MongoDB operator  Successfully created MongoDB
  Normal  Successful  15m   MongoDB operator  Successfully created appbinding
  Normal  Successful  15m   MongoDB operator  Successfully patched stats service
  Normal  Successful  15m   MongoDB operator  Successfully patched StatefulSet demo/mgo-replicaset
  Normal  Successful  15m   MongoDB operator  Successfully patched MongoDB
```

## Resume Database

To resume the database, set `spec.halted` to `false`.

```console
$ kubectl patch -n demo mg/mgo-replicaset -p '{"spec":{"halted":false}}' --type="merge"
mongodb.kubedb.com/mgo-replicaset patched
```

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS    AGE
mgo-replicaset   4.1       Running   21m
```

Now, if you exec into the database, you can see that the datas are intact.

```console
$ kubectl exec -it mgo-replicaset-0 -n demo bash
root@mgo-replicaset-0:/# mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

rs0:PRIMARY> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
newdb   0.000GB

rs0:PRIMARY> show users
{
	"_id" : "admin.root",
	"userId" : UUID("faa9d5bc-e713-4f7e-8d4b-3a90ea258faf"),
	"user" : "root",
	"db" : "admin",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}

rs0:PRIMARY> use newdb
switched to db newdb

rs0:PRIMARY> db.movie.find().pretty()
{ "_id" : ObjectId("5e3169dc33a1f520c879584a"), "name" : "batman" }


> exit
bye
```

## WipeOut DormantDatabase

You can wipe out the database while deleting the object by setting `spec.terminationPolicy` to `WipeOut`. KubeDB operator will delete any relevant resources of this `MongoDB` database (i.e, PVCs, Secrets, Rbac resources, Statefulsets, stash backup data). It will also delete stash backup data stored in the Cloud Storage buckets.

```console
$ kubectl patch -n demo mg/mgo-replicaset -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
mongodb.kubedb.com/mgo-replicaset patched

$ kubectl delete -n demo mg/mgo-replicaset
mongodb.kubedb.com "mgo-replicaset" deleted
```

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/mgo-replicaset -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mgo-replicaset

kubectl delete ns demo
```

## Next Steps

- [Snapshot and Restore](/docs/guides/mongodb/snapshot/backup-and-restore.md) process of MongoDB databases using KubeDB.
- Take [Scheduled Snapshot](/docs/guides/mongodb/snapshot/scheduled-backup.md) of MongoDB databases using KubeDB.
- Initialize [MongoDB with Script](/docs/guides/mongodb/initialization/using-script.md).
- Initialize [MongoDB with Snapshot](/docs/guides/mongodb/initialization/using-snapshot.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/mongodb/monitoring/using-coreos-prometheus-operator.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/mongodb/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/mongodb/private-registry/using-private-registry.md) to deploy MongoDB with KubeDB.
- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- Detail concepts of [MongoDBVersion object](/docs/concepts/catalog/mongodb.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
