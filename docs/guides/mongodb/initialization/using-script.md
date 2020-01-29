---
title: Initialize MongoDB using Script
menu:
  docs_{{ .version }}:
    identifier: mg-using-script-initialization
    name: Using Script
    parent: mg-initialization-mongodb
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Initialize MongoDB using Script

This tutorial will show you how to use KubeDB to initialize a MongoDB database with .js and/or .sh script.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```console
  $ kubectl create ns demo
  namespace/demo created
  ```

  In this tutorial we will use .js script stored in GitHub repository [kubedb/mongodb-init-scripts](https://github.com/kubedb/mongodb-init-scripts).

> Note: The yaml files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Prepare Initialization Scripts

MongoDB supports initialization with `.sh` and `.js` files. In this tutorial, we will use `init.js` script from [mongodb-init-scripts](https://github.com/kubedb/mongodb-init-scripts) git repository to insert data inside `kubedb` DB.

We will use a ConfigMap as script source. You can use any Kubernetes supported [volume](https://kubernetes.io/docs/concepts/storage/volumes) as script source.

At first, we will create a ConfigMap from `init.js` file. Then, we will provide this ConfigMap as script source in `init.scriptSource` of MongoDB crd spec.

Let's create a ConfigMap with initialization script,

```console
$ kubectl create configmap -n demo mg-init-script \
--from-literal=init.js="$(curl -fsSL https://github.com/kubedb/mongodb-init-scripts/raw/master/init.js)"
configmap/mg-init-script created
```

```console
$ kubectl get configmap -n demo mg-init-script -o yaml
apiVersion: v1
data:
  init.js: |-
    db = db.getSiblingDB('kubedb');
    db.people.insert({"firstname" : "kubernetes", "lastname" : "database" });
kind: ConfigMap
metadata:
  creationTimestamp: "2020-01-30T04:56:29Z"
  name: mg-init-script
  namespace: demo
  resourceVersion: "6555"
  selfLink: /api/v1/namespaces/demo/configmaps/mg-init-script
  uid: 2d7b34f6-06d7-4764-8fae-001df0cd2564
```

## Create a MongoDB database with Init-Script

Below is the `MongoDB` object created in this tutorial.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-init-script
  namespace: demo
spec:
  version: "4.1"
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  init:
    scriptSource:
      configMap:
        name: mg-init-script
```

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/del-dorm/docs/examples/mongodb/Initialization/demo-1.yaml
mongodb.kubedb.com/mgo-init-script created
```

Here,

- `spec.init.scriptSource` specifies a script source used to initialize the database before database server starts. The scripts will be executed alphabatically. In this tutorial, a sample .js script from the git repository `https://github.com/kubedb/mongodb-init-scripts.git` is used to create a test database. You can use other [volume sources](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes).  The \*.js and/or \*.sh sripts that are stored inside the root folder will be executed alphabatically. The scripts inside child folders will be skipped.

KubeDB operator watches for `MongoDB` objects using Kubernetes api. When a `MongoDB` object is created, KubeDB operator will create a new StatefulSet and a Service with the matching MongoDB object name. KubeDB operator will also create a governing service for StatefulSets with the name `<mongodb-crd-name>-gvr`, if one is not already present. No MongoDB specific RBAC roles are required for [RBAC enabled clusters](/docs/setup/install.md#using-yaml).

Wait for mongodb database's status to become `Running`.

```console
$ kubectl get mongodb -n demo
NAME              VERSION   STATUS    AGE
mgo-init-script   4.1       Running   38s
```


```console
$ kubedb describe mg -n demo mgo-init-script
Name:               mgo-init-script
Namespace:          demo
CreationTimestamp:  Thu, 30 Jan 2020 10:57:25 +0600
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
  Name:               mgo-init-script
  CreationTimestamp:  Thu, 30 Jan 2020 10:57:25 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mgo-init-script
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mgo-init-script
  Annotations:        <none>
  Replicas:           824641295708 desired | 1 total
  Pods Status:        1 Running / 0 Waiting / 0 Succeeded / 0 Failed

Service:        
  Name:         mgo-init-script
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-init-script
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-init-script
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.96.107.235
  Port:         db  27017/TCP
  TargetPort:   db/TCP
  Endpoints:    10.244.2.10:27017

Service:        
  Name:         mgo-init-script-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-init-script
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-init-script
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.2.10:27017

Database Secret:
  Name:         mgo-init-script-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mgo-init-script
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mgo-init-script
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  password:  16 bytes
  username:  4 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  Normal  Successful  48s   MongoDB operator  Successfully created stats service
  Normal  Successful  48s   MongoDB operator  Successfully created Service
  Normal  Successful  18s   MongoDB operator  Successfully created StatefulSet demo/mgo-init-script
  Normal  Successful  18s   MongoDB operator  Successfully created MongoDB
  Normal  Successful  18s   MongoDB operator  Successfully created appbinding
  Normal  Successful  18s   MongoDB operator  Successfully patched stats service
  Normal  Successful  18s   MongoDB operator  Successfully patched StatefulSet demo/mgo-init-script
  Normal  Successful  18s   MongoDB operator  Successfully patched MongoDB
```

```console
$ kubectl get statefulset -n demo
NAME              READY   AGE
mgo-init-script   1/1     30s

$ kubectl get pvc -n demo
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-mgo-init-script-0   Bound    pvc-52490b84-5327-4590-b5d8-cd067639cc17   1Gi        RWO            standard       79s

$ kubectl get pv -n demo
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE
pvc-52490b84-5327-4590-b5d8-cd067639cc17   1Gi        RWO            Delete           Bound    demo/datadir-mgo-init-script-0   standard                84s

$ kubectl get service -n demo
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
mgo-init-script       ClusterIP   10.96.107.235   <none>        27017/TCP   95s
mgo-init-script-gvr   ClusterIP   None            <none>        27017/TCP   95s
```

KubeDB operator sets the `status.phase` to `Running` once the database is successfully created. Run the following command to see the modified MongoDB object:

```yaml
$ kubedb get mg -n demo mgo-init-script -o yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  creationTimestamp: "2020-01-30T04:57:25Z"
  finalizers:
  - kubedb.com
  generation: 2
  name: mgo-init-script
  namespace: demo
  resourceVersion: "6789"
  selfLink: /apis/kubedb.com/v1alpha1/namespaces/demo/mongodbs/mgo-init-script
  uid: a0d98ac7-c979-4ca7-9bc0-0770de163ea1
spec:
  databaseSecret:
    secretName: mgo-init-script-auth
  init:
    scriptSource:
      configMap:
        name: mg-init-script
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
                  kubedb.com/name: mgo-init-script
              namespaces:
              - demo
              topologyKey: kubernetes.io/hostname
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  kubedb.com/kind: MongoDB
                  kubedb.com/name: mgo-init-script
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
      serviceAccountName: mgo-init-script
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
  terminationPolicy: Delete
  updateStrategy:
    type: RollingUpdate
  version: "4.1"
status:
  observedGeneration: 2
  phase: Running
```

Please note that KubeDB operator has created a new Secret called `mgo-init-script-auth` *(format: {mongodb-object-name}-auth)* for storing the password for MongoDB superuser. This secret contains a `username` key which contains the *username* for MongoDB superuser and a `password` key which contains the *password* for MongoDB superuser.
If you want to use an existing secret please specify that when creating the MongoDB object using `spec.databaseSecret.secretName`. While creating this secret manually, make sure the secret contains these two keys containing data `username` and `password`.

```console
$ kubectl get secrets -n demo mgo-init-script-auth -o yaml
apiVersion: v1
data:
  password: ZlR6NnNjM00xWHE3WjFIbA==
  username: cm9vdA==
kind: Secret
metadata:
  creationTimestamp: "2020-01-30T04:57:25Z"
  labels:
    app.kubernetes.io/component: database
    app.kubernetes.io/instance: mgo-init-script
    app.kubernetes.io/managed-by: kubedb.com
    app.kubernetes.io/name: mongodb
    app.kubernetes.io/version: "4.1"
    kubedb.com/kind: MongoDB
    kubedb.com/name: mgo-init-script
  name: mgo-init-script-auth
  namespace: demo
  resourceVersion: "6678"
  selfLink: /api/v1/namespaces/demo/secrets/mgo-init-script-auth
  uid: 397f2c64-aaf6-46ab-8aeb-7933c7419291
type: Opaque
```

Now, you can connect to this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/). In this tutorial, we are connecting to the MongoDB server from inside the pod.

```console
$ kubectl get secrets -n demo mgo-init-script-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mgo-init-script-auth -o jsonpath='{.data.\password}' | base64 -d
fTz6sc3M1Xq7Z1Hl

$ kubectl exec -it mgo-init-script-0 -n demo bash

root@mgo-init-script-0:/# mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

> show dbs
admin   0.000GB
config  0.000GB
kubedb  0.000GB
local   0.000GB

> use kubedb
switched to db kubedb

> db.people.find()
{ "_id" : ObjectId("5ba9d667981f02e927b6788e"), "firstname" : "kubernetes", "lastname" : "database" }

> exit
bye
```

As you can see here, the initial script has successfully created a database named `kubedb` and inserted data into that database successfully.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/mgo-init-script -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mgo-init-script

kubectl delete ns demo
```

## Next Steps

- Monitor your MongoDB database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/mongodb/monitoring/using-coreos-prometheus-operator.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/mongodb/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/mongodb/private-registry/using-private-registry.md) to deploy MongoDB with KubeDB.
- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- Detail concepts of [MongoDBVersion object](/docs/concepts/catalog/mongodb.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
