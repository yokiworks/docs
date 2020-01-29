---
title: MongoDB Quickstart
menu:
  docs_{{ .version }}:
    identifier: mg-quickstart-quickstart
    name: Overview
    parent: mg-quickstart-mongodb
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# MongoDB QuickStart

This tutorial will show you how to use KubeDB to run a MongoDB database.

<p align="center">
  <img alt="lifecycle"  src="/docs/images/mongodb/mgo-lifecycle.svg">
</p>

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) is required to run KubeDB. Check the available StorageClass in cluster.

  ```console
  $ kubectl get storageclasses
  NAME                 PROVISIONER                AGE
  standard (default)   k8s.io/minikube-hostpath   4h
  ```

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial. Run the following command to prepare your cluster for this tutorial:

  ```console
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: The yaml files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Find Available MongoDBVersion

When you have installed KubeDB, it has created `MongoDBVersion` crd for all supported MongoDB versions. Check 0

```console
$ kubectl get mongodbversions
NAME       VERSION   DB_IMAGE                DEPRECATED   AGE
3.4        3.4       kubedb/mongo:3.4        true         76m
3.4-v1     3.4       kubedb/mongo:3.4-v1     true         76m
3.4-v2     3.4       kubedb/mongo:3.4-v2     true         76m
3.4-v3     3.4       kubedb/mongo:3.4-v3                  76m
3.4-v4     3.4.22    kubedb/mongo:3.4-v4                  76m
3.4.17     3.4.17    kubedb/mongo:3.4.17                  76m
3.4.22     3.4.22    kubedb/mongo:3.4.22                  76m
3.6        3.6       kubedb/mongo:3.6        true         76m
3.6-v1     3.6       kubedb/mongo:3.6-v1     true         76m
3.6-v2     3.6       kubedb/mongo:3.6-v2     true         76m
3.6-v3     3.6       kubedb/mongo:3.6-v3                  76m
3.6-v4     3.6.13    kubedb/mongo:3.6-v4                  76m
3.6.13     3.6.13    kubedb/mongo:3.6.13                  76m
3.6.8      3.6.8     kubedb/mongo:3.6.8                   76m
4.0        4.0.5     kubedb/mongo:4.0        true         76m
4.0-v1     4.0.5     kubedb/mongo:4.0-v1                  76m
4.0-v2     4.0.11    kubedb/mongo:4.0-v2                  76m
4.0.11     4.0.11    kubedb/mongo:4.0.11                  76m
4.0.3      4.0.3     kubedb/mongo:4.0.3                   76m
4.0.5      4.0.5     kubedb/mongo:4.0.5      true         76m
4.0.5-v1   4.0.5     kubedb/mongo:4.0.5-v1                76m
4.0.5-v2   4.0.5     kubedb/mongo:4.0.5-v2                76m
4.1        4.1.13    kubedb/mongo:4.1                     76m
4.1.13     4.1.13    kubedb/mongo:4.1.13                  76m
4.1.4      4.1.4     kubedb/mongo:4.1.4                   76m
4.1.7      4.1.7     kubedb/mongo:4.1.7      true         76m
4.1.7-v1   4.1.7     kubedb/mongo:4.1.7-v1                76m
4.1.7-v2   4.1.7     kubedb/mongo:4.1.7-v2                76m
```

## Create a MongoDB database

KubeDB implements a `MongoDB` CRD to define the specification of a MongoDB database. Below is the `MongoDB` object created in this tutorial.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-quickstart
  namespace: demo
spec:
  version: "4.1"
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: DoNotTerminate
```

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/del-dorm/docs/examples/mongodb/quickstart/demo-1.yaml
mongodb.kubedb.com/mgo-quickstart created
```

Here,

- `spec.version` is name of the MongoDBVersion crd where the docker images are specified. In this tutorial, a MongoDB 3.4-v3 database is created.
- `spec.storageType` specifies the type of storage that will be used for MongoDB database. It can be `Durable` or `Ephemeral`. Default value of this field is `Durable`. If `Ephemeral` is used then KubeDB will create MongoDB database using `EmptyDir` volume. In this case, you don't have to specify `spec.storage` field. This is useful for testing purposes.
- `spec.storage` specifies PVC spec that will be dynamically allocated to store data for this database. This storage spec will be passed to the StatefulSet created by KubeDB operator to run database pods. You can specify any StorageClass available in your cluster with appropriate resource requests.
- `spec.terminationPolicy` gives flexibility whether to `nullify`(reject) the delete operation of `MongoDB` crd or which resources KubeDB should keep or delete when you delete `MongoDB` crd. If admission webhook is enabled, It prevents users from deleting the database as long as the `spec.terminationPolicy` is set to `DoNotTerminate`. Learn details of all `TerminationPolicy` [here]

> Note: spec.storage section is used to create PVC for database pod. It will create PVC with storage size specified instorage.resources.requests field. Don't specify limits here. PVC does not get resized automatically.

KubeDB operator watches for `MongoDB` objects using Kubernetes api. When a `MongoDB` object is created, KubeDB operator will create a new StatefulSet and a Service with the matching MongoDB object name. KubeDB operator will also create a governing service for StatefulSets with the name `<mongodb-name>-gvr`.

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS    AGE
mgo-quickstart   4.1       Running   5m
```

```console
$ kubedb describe mg -n demo mgo-quickstart
Name:               mgo-quickstart
Namespace:          demo
CreationTimestamp:  Wed, 29 Jan 2020 15:33:17 +0600
Labels:             <none>
Annotations:        <none>
Replicas:           1  total
Status:             Running
  StorageType:      Durable
Volume:
  StorageClass:  standard
  Capacity:      1Gi
  Access Modes:  RWO

StatefulSet:          
  Name:               mgo-quickstart
  CreationTimestamp:  Wed, 29 Jan 2020 15:33:17 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mgo-quickstart
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mgo-quickstart
  Annotations:        <none>
  Replicas:           824641430124 desired | 1 total
  Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed

Service:        
  Name:         mgo-quickstart
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-quickstart
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-quickstart
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.102.184.179
  Port:         db  27017/TCP
  TargetPort:   db/TCP
  Endpoints:    10.244.1.7:27017

Service:        
  Name:         mgo-quickstart-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-quickstart
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-quickstart
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.7:27017

Database Secret:
  Name:         mgo-quickstart-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-quickstart
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-quickstart
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  username:  4 bytes
  password:  16 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  Normal  Successful  1m    MongoDB operator  Successfully created stats service
  Normal  Successful  1m    MongoDB operator  Successfully created Service
  Normal  Successful  52s   MongoDB operator  Successfully created StatefulSet demo/mgo-quickstart
  Normal  Successful  52s   MongoDB operator  Successfully created MongoDB
  Normal  Successful  52s   MongoDB operator  Successfully created appbinding
  Normal  Successful  52s   MongoDB operator  Successfully patched stats service
  Normal  Successful  52s   MongoDB operator  Successfully patched StatefulSet demo/mgo-quickstart
  Normal  Successful  52s   MongoDB operator  Successfully patched MongoDB
```

```console
$ kubectl get statefulset -n demo
NAME             READY   AGE
mgo-quickstart   1/1     2m5s

$ kubectl get pvc -n demo
NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-mgo-quickstart-0   Bound    pvc-47858cca-f71f-45cf-aace-a968cebc9a15   1Gi        RWO            standard       2m28s

$ kubectl get pv -n demo
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS   REASON   AGE
pvc-47858cca-f71f-45cf-aace-a968cebc9a15   1Gi        RWO            Delete           Bound    demo/datadir-mgo-quickstart-0   standard                3m

$ kubectl get service -n demo
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
mgo-quickstart       ClusterIP   10.102.184.179   <none>        27017/TCP   3m12s
mgo-quickstart-gvr   ClusterIP   None             <none>        27017/TCP   3m12s
```

KubeDB operator sets the `status.phase` to `Running` once the database is successfully created. Run the following command to see the modified MongoDB object:

```yaml
$ kubedb get mg -n demo mgo-quickstart -o yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  creationTimestamp: "2020-01-29T09:33:17Z"
  finalizers:
  - kubedb.com
  generation: 2
  name: mgo-quickstart
  namespace: demo
  resourceVersion: "26392"
  selfLink: /apis/kubedb.com/v1alpha1/namespaces/demo/mongodbs/mgo-quickstart
  uid: 5e6c180b-7415-4eac-80fa-b4596adf8be6
spec:
  databaseSecret:
    secretName: mgo-quickstart-auth
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
                  kubedb.com/name: mgo-quickstart
              namespaces:
              - demo
              topologyKey: kubernetes.io/hostname
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  kubedb.com/kind: MongoDB
                  kubedb.com/name: mgo-quickstart
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
      serviceAccountName: mgo-quickstart
  replicas: 1
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
  terminationPolicy: DoNotTerminate
  updateStrategy:
    type: RollingUpdate
  version: "4.1"
status:
  observedGeneration: 2
  phase: Running
```

Please note that KubeDB operator has created a new Secret called `mgo-quickstart-auth` *(format: {mongodb-object-name}-auth)* for storing the password for `mongodb` superuser. This secret contains a `username` key which contains the *username* for MongoDB superuser and a `password` key which contains the *password* for MongoDB superuser.

If you want to use custom or existing secret please specify that when creating the MongoDB object using `spec.databaseSecret.secretName`. While creating this secret manually, make sure the secret contains these two keys containing data `user` and `password`. For more details, please see [here](/docs/concepts/databases/mongodb.md#specdatabasesecret).

Now, you can connect to this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/). In this tutorial, we are connecting to the MongoDB server from inside the pod.

```console
$ kubectl get secrets -n demo mgo-quickstart-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mgo-quickstart-auth -o jsonpath='{.data.\password}' | base64 -d
oFWDscelkRRN2YOA

$ kubectl exec -it mgo-quickstart-0 -n demo sh

> mongo admin

> db.auth("root","oFWDscelkRRN2YOA")
1

> show dbs
admin  0.000GB
local  0.000GB
mydb   0.000GB

> show users
{
	"_id" : "admin.root",
	"userId" : UUID("de33b869-1e9b-42e1-83c5-b159733736d4"),
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

> use newdb
switched to db newdb

> db.movie.insert({"name":"batman"});
WriteResult({ "nInserted" : 1 })

> db.movie.find().pretty()
{ "_id" : ObjectId("5a2e435d7ec14e7bda785f16"), "name" : "batman" }

> exit
bye
```

## DoNotTerminate Property

When `terminationPolicy` is `DoNotTerminate`, KubeDB takes advantage of `ValidationWebhook` feature in Kubernetes 1.9.0 or later clusters to implement `DoNotTerminate` feature. If admission webhook is enabled, It prevents users from deleting or halting (`spec.halted: true`) the database as long as the `spec.terminationPolicy` is set to `DoNotTerminate`. You can see this below:

```console
$ kubedb delete mg mgo-quickstart -n demo
Error from server (BadRequest): admission webhook "mongodb.validators.kubedb.com" denied the request: mongodb "mgo-quickstart" can't be paused. To delete, change spec.terminationPolicy
```

```console
$ kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"halted":true}}' --type="merge"
Error from server (Forbidden): admission webhook "mongodb.mutators.kubedb.com" denied the request: Can't halt, since termination policy is 'DoNotTerminate'
```

Now, run `kubedb edit mg mgo-quickstart -n demo` to set `spec.terminationPolicy` to `Halt` (which will allow to set `spec.halted: true`) or remove this field (which default to `Delete`). Then you will be able to delete/pause the database.

```console
kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"terminationPolicy":"Halt"}}' --type="merge"
```

Learn details of all `TerminationPolicy` [here](/docs/concepts/databases/mongodb.md#specterminationpolicy).

## Halt Database

User can halt the database by setting `spec.halted` to `true`. As the name suggests, this will halt the database by deleting all the kubernetes resources (including `sts`, `services`, rbac resources, etc) except `secrets` and `PVCs`. On the other hand, when users wishes to resume a halted database, they will need to set `spec.halted` to `false`.

Please Note that, if [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is set to `DoNotTerminate`, the halt database operation will fail with forbidden error message from admission controller. 

If [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is not set to `DoNotTerminate`, then kubedb will accept the user's halt request and also it will update the `spec.terminationPolicy` to `Halt`.

Change the `spce.terminationPolicy`:

```console
$ kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"terminationPolicy":"Halt"}}' --type="merge"
mongodb.kubedb.com/mgo-quickstart patched
```

Database status will be set to `Halted` after successful deletion of kubernetes resources.

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS   AGE
mgo-quickstart   4.1       Halted   30m
```

## Resume Database

To resume the database, set `spec.halted` to `false`.

```console
$ kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"halted":false}}' --type="merge"
mongodb.kubedb.com/mgo-quickstart patched
```

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS    AGE
mgo-quickstart   4.1       Running   34m
```

Now, if you exec into the database, you can see that the datas are intact.

```console
$ kubectl exec -it mgo-quickstart-0 -n demo sh
> mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

> db.auth("root","oFWDscelkRRN2YOA")
1

> show dbs
admin  0.000GB
local  0.000GB
mydb   0.000GB

> show users
{
	"_id" : "admin.root",
	"userId" : UUID("de33b869-1e9b-42e1-83c5-b159733736d4"),
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

> use newdb
switched to db newdb

> db.movie.find().pretty()
{ "_id" : ObjectId("5a2e435d7ec14e7bda785f16"), "name" : "batman" }

> exit
bye
```

## WipeOut DormantDatabase

You can wipe out the database while deleting the object by setting `spec.terminationPolicy` to `WipeOut`. KubeDB operator will delete any relevant resources of this `MongoDB` database (i.e, PVCs, Secrets, Rbac resources, Statefulsets, stash backup data). It will also delete stash backup data stored in the Cloud Storage buckets.

```console
$ kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
mongodb.kubedb.com/mgo-quickstart patched

$ kubectl delete -n demo mg/mgo-quickstart
mongodb.kubedb.com "mgo-quickstart" deleted
```

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/mgo-quickstart -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mgo-quickstart

kubectl delete ns demo
```

## Tips for Testing

If you are just testing some basic functionalities, you might want to avoid additional hassles due to some safety features that are great for production environment. You can follow these tips to avoid them.

1. **Use `storageType: Ephemeral`**. Databases are precious. You might not want to lose your data in your production environment if database pod fail. So, we recommend to use `spec.storageType: Durable` and provide storage spec in `spec.storage` section. For testing purpose, you can just use `spec.storageType: Ephemeral`. KubeDB will use [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) for storage. You will not require to provide `spec.storage` section.
2. **Use `terminationPolicy: WipeOut`**. It is nice to be able to resume database from previous one. So, we create `DormantDatabase` and preserve all your `PVCs`, `Secrets`, `Snapshots` etc. If you don't want to resume database, you can just use `spec.terminationPolicy: WipeOut`. It will not create `DormantDatabase` and it will delete everything created by KubeDB for a particular MongoDB crd when you delete the crd. For more details about termination policy, please visit [here](/docs/concepts/databases/mongodb.md#specterminationpolicy).

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
