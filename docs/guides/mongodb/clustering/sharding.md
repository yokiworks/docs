---
title: MongoDB Sharding Guide
menu:
  docs_{{ .version }}:
    identifier: mg-clustering-sharding
    name: Sharding Guide
    parent: mg-clustering-mongodb
    weight: 25
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# MongoDB Sharding

This tutorial will show you how to use KubeDB to run a sharded MongoDB cluster.

## Before You Begin

Before proceeding:

- Read [mongodb sharding concept](/docs/guides/mongodb/clustering/sharding_concept.md) to learn about MongoDB Sharding clustering.

- You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial. Run the following command to prepare your cluster for this tutorial:

  ```console
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: The yaml files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Deploy Sharded MongoDB Cluster

To deploy a MongoDB Sharding, user have to specify `spec.replicaSet` option in `Mongodb` CRD.

The following is an example of a `Mongodb` object which creates MongoDB Sharding of three members.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mongo-sh
  namespace: demo
spec:
  version: "4.1"
  shardTopology:
    configServer:
      replicas: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
    mongos:
      replicas: 2
      strategy:
        type: RollingUpdate
    shard:
      replicas: 3
      shards: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
```

```console
$ kubectl create -f https://github.com/kubedb/docs/raw/del-dorm/docs/examples/mongodb/clustering/mongo-sharding.yaml
mongodb.kubedb.com/mongo-sh created
```

Here,

- `spec.shardTopology` represents the topology configuration for sharding.
  - `shard` represents configuration for Shard component of mongodb.
    - `shards` represents number of shards for a mongodb deployment. Each shard is deployed as a [replicaset](/docs/guides/mongodb/clustering/replication_concept.md).
    - `replicas` represents number of replicas of each shard replicaset.
    - `prefix` represents the prefix of each shard node.
    - `configSource` is an optional field to provide custom configuration file for shards (i.e mongod.cnf). If specified, this file will be used as configuration file otherwise a default configuration file will be used.
    - `podTemplate` is an optional configuration for pods.
    - `storage` to specify pvc spec for each node of sharding. You can specify any StorageClass available in your cluster with appropriate resource requests.
  - `configServer` represents configuration for ConfigServer component of mongodb.
    - `replicas` represents number of replicas for configServer replicaset. Here, configServer is deployed as a replicaset of mongodb.
    - `prefix` represents the prefix of configServer nodes.
    - `configSource` is an optional field to provide custom configuration file for configSource (i.e mongod.cnf). If specified, this file will be used as configuration file otherwise a default configuration file will be used.
    - `podTemplate` is an optional configuration for pods.
    - `storage` to specify pvc spec for each node of configServer. You can specify any StorageClass available in your cluster with appropriate resource requests.
  - `mongos` represents configuration for Mongos component of mongodb. `Mongos` instances run as stateless components (deployment).
    - `replicas` represents number of replicas of `Mongos` instance. Here, Mongos is not deployed as replicaset.
    - `prefix` represents the prefix of mongos nodes.
    - `configSource` is an optional field to provide custom configuration file for mongos (i.e mongod.cnf). If specified, this file will be used as configuration file otherwise a default configuration file will be used.
    - `podTemplate` is an optional configuration for pods.
    - `strategy` is the deployment strategy to use to replace existing pods with new ones. This is optional. If not provided, kubernetes will use default deploymentStrategy, ie. `RollingUpdate`. See more about [Deployment Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).
- `spec.certificateSecret` (optional) is a secret name that contains keyfile(a random string) against `key.txt` key. Each mongod replica set instances in the topology uses the contents of the keyfile as the shared password for authenticating other members in the replicaset. Only mongod instances with the correct keyfile can join the replica set. _User can provide the `CertificateSecret` by creating a secret with key `key.txt`. See [here](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/#create-a-keyfile) to create the string for `CertificateSecret`._ If `CertificateSecret` is not given, KubeDB operator will generate a `CertificateSecret` itself.

KubeDB operator watches for `MongoDB` objects using Kubernetes api. When a `MongoDB` object is created, KubeDB operator will create a new StatefulSet and a Service with the matching MongoDB object name. KubeDB operator will also create governing services for StatefulSets with the name `<mongodb-name>-<node-type>-gvr`.

MongoDB `mongo-sh` state,

```console
$ kubectl get mg -n demo
NAME       VERSION   STATUS    AGE
mongo-sh   4.1       Running   6m9s
```

```console
$ kubedb describe mg -n demo mongo-sh
Name:               mongo-sh
Namespace:          demo
CreationTimestamp:  Wed, 29 Jan 2020 17:36:37 +0600
Labels:             <none>
Annotations:        <none>
Status:             Running
  StorageType:      Durable
No volumes.

StatefulSet:          
  Name:               mongo-sh-configsvr
  CreationTimestamp:  Wed, 29 Jan 2020 17:36:38 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mongo-sh
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mongo-sh
  Annotations:        <none>
  Replicas:           824639987012 desired | 3 total
  Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed

StatefulSet:          
  Name:               mongo-sh-shard0
  CreationTimestamp:  Wed, 29 Jan 2020 17:36:38 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mongo-sh
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mongo-sh
  Annotations:        <none>
  Replicas:           824639875108 desired | 3 total
  Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed

StatefulSet:          
  Name:               mongo-sh-shard1
  CreationTimestamp:  Wed, 29 Jan 2020 17:36:38 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mongo-sh
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mongo-sh
  Annotations:        <none>
  Replicas:           824639877300 desired | 3 total
  Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed

StatefulSet:          
  Name:               mongo-sh-shard2
  CreationTimestamp:  Wed, 29 Jan 2020 17:36:38 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mongo-sh
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mongo-sh
  Annotations:        <none>
  Replicas:           824639879764 desired | 3 total
  Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed

Deployment:           
  Name:               mongo-sh-mongos
  CreationTimestamp:  Wed, 29 Jan 2020 17:41:14 +0600
  Labels:               app.kubernetes.io/component=database
                        app.kubernetes.io/instance=mongo-sh
                        app.kubernetes.io/managed-by=kubedb.com
                        app.kubernetes.io/name=mongodb
                        app.kubernetes.io/version=4.1
                        kubedb.com/kind=MongoDB
                        kubedb.com/name=mongo-sh
  Annotations:          deployment.kubernetes.io/revision=1
  Replicas:           2 desired | 2 updated | 2 total | 2 available | 0 unavailable
  Pods Status:        2 Running / 0 Waiting / 0 Succeeded / 0 Failed

Service:        
  Name:         mongo-sh
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:  <none>
  Type:         ClusterIP
  IP:           10.105.242.203
  Port:         db  27017/TCP
  TargetPort:   db/TCP
  Endpoints:    10.244.1.31:27017,10.244.2.24:27017

Service:        
  Name:         mongo-sh-configsvr-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.21:27017,10.244.1.29:27017,10.244.2.18:27017

Service:        
  Name:         mongo-sh-shard0-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.22:27017,10.244.2.19:27017,10.244.2.22:27017

Service:        
  Name:         mongo-sh-shard1-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.25:27017,10.244.2.14:27017,10.244.2.23:27017

Service:        
  Name:         mongo-sh-shard2-gvr
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:    service.alpha.kubernetes.io/tolerate-unready-endpoints=true
  Type:         ClusterIP
  IP:           None
  Port:         db  27017/TCP
  TargetPort:   27017/TCP
  Endpoints:    10.244.1.26:27017,10.244.1.30:27017,10.244.2.15:27017

Database Secret:
  Name:         mongo-sh-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  password:  16 bytes
  username:  4 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  Normal  Successful  7m    MongoDB operator  Successfully created stats service
  Normal  Successful  7m    MongoDB operator  Successfully created stats service
  Normal  Successful  7m    MongoDB operator  Successfully created stats service
  Normal  Successful  7m    MongoDB operator  Successfully created stats service
  Normal  Successful  7m    MongoDB operator  Successfully created Service
  Normal  Successful  2m    MongoDB operator  Successfully created StatefulSet demo/mongo-sh-shard0
  Normal  Successful  2m    MongoDB operator  Successfully created StatefulSet demo/mongo-sh-shard1
  Normal  Successful  2m    MongoDB operator  Successfully created StatefulSet demo/mongo-sh-shard2
  Normal  Successful  2m    MongoDB operator  Successfully created StatefulSet demo/mongo-sh-configsvr
  Normal  Successful  2m    MongoDB operator  Successfully created appbinding
  Normal  Successful  2m    MongoDB operator  Successfully created MongoDB
  Normal  Successful  2m    MongoDB operator  Successfully created Deployment demo/mongo-sh-mongos
  Normal  Successful  2m    MongoDB operator  Successfully patched stats service
  Normal  Successful  2m    MongoDB operator  Successfully patched stats service
  Normal  Successful  2m    MongoDB operator  Successfully patched stats service
  Normal  Successful  2m    MongoDB operator  Successfully patched stats service
  Normal  Successful  2m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-shard0
  Normal  Successful  2m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-shard1
  Normal  Successful  2m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-shard2
  Normal  Successful  2m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-configsvr
  Normal  Successful  2m    MongoDB operator  Successfully patched Deployment demo/mongo-sh-mongos
  Normal  Successful  2m    MongoDB operator  Successfully patched MongoDB
```

`Sharding` and `ConfigServer` nodes are deployed as statefulset.

```console
$ kubectl get statefulset -n demo
NAME                 READY   AGE
mongo-sh-configsvr   3/3     8m44s
mongo-sh-shard0      3/3     8m44s
mongo-sh-shard1      3/3     8m44s
mongo-sh-shard2      3/3     8m44s
```

`Mongos` nodes are deployed as deployment.

```console
$ kubectl get deployments -n demo
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
mongo-sh-mongos   2/2     2            2           4m24s
```

All PVCs and PVs for MongoDB `mongo-sh`,

```console
$ kubectl get pvc -n demo
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-mongo-sh-configsvr-0   Bound    pvc-c317d4fb-12eb-4f59-94e5-a2d863272927   1Gi        RWO            standard       9m12s
datadir-mongo-sh-configsvr-1   Bound    pvc-45b60ac4-6d27-4f6d-b4b6-cdd20ceac074   1Gi        RWO            standard       8m6s
datadir-mongo-sh-configsvr-2   Bound    pvc-b170e9b6-e391-4001-b4b7-252a7420dd69   1Gi        RWO            standard       6m1s
datadir-mongo-sh-shard0-0      Bound    pvc-df98d0b6-263b-4bb4-9be9-549dcc2aa885   1Gi        RWO            standard       9m12s
datadir-mongo-sh-shard0-1      Bound    pvc-b91048d6-b5a7-4ae9-9b26-92403bb960c2   1Gi        RWO            standard       7m49s
datadir-mongo-sh-shard0-2      Bound    pvc-ce5a93d0-d0a6-4bbf-a2ee-db16de935fff   1Gi        RWO            standard       6m1s
datadir-mongo-sh-shard1-0      Bound    pvc-d0fdc856-6848-40f8-adcb-e60eee102124   1Gi        RWO            standard       9m12s
datadir-mongo-sh-shard1-1      Bound    pvc-507a6b1e-497a-453f-9353-c11ba7a0f805   1Gi        RWO            standard       7m34s
datadir-mongo-sh-shard1-2      Bound    pvc-4210d0aa-12de-4f9a-999b-da1893fd10e2   1Gi        RWO            standard       6m1s
datadir-mongo-sh-shard2-0      Bound    pvc-c44e5940-6622-49be-8355-1f9aeac651d2   1Gi        RWO            standard       9m12s
datadir-mongo-sh-shard2-1      Bound    pvc-63f0b4be-0e9b-41e9-ae2b-229dfe51477c   1Gi        RWO            standard       7m33s
datadir-mongo-sh-shard2-2      Bound    pvc-d60001d6-b3a5-4952-93a6-c37ac541a916   1Gi        RWO            standard       6m1s

$ kubectl get pv -n demo
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS   REASON   AGE
pvc-4210d0aa-12de-4f9a-999b-da1893fd10e2   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard1-2      standard                6m8s
pvc-45b60ac4-6d27-4f6d-b4b6-cdd20ceac074   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-configsvr-1   standard                7m55s
pvc-507a6b1e-497a-453f-9353-c11ba7a0f805   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard1-1      standard                7m42s
pvc-63f0b4be-0e9b-41e9-ae2b-229dfe51477c   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard2-1      standard                7m39s
pvc-b170e9b6-e391-4001-b4b7-252a7420dd69   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-configsvr-2   standard                6m9s
pvc-b91048d6-b5a7-4ae9-9b26-92403bb960c2   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard0-1      standard                7m53s
pvc-c317d4fb-12eb-4f59-94e5-a2d863272927   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-configsvr-0   standard                9m19s
pvc-c44e5940-6622-49be-8355-1f9aeac651d2   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard2-0      standard                9m18s
pvc-ce5a93d0-d0a6-4bbf-a2ee-db16de935fff   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard0-2      standard                6m9s
pvc-d0fdc856-6848-40f8-adcb-e60eee102124   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard1-0      standard                9m20s
pvc-d60001d6-b3a5-4952-93a6-c37ac541a916   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard2-2      standard                6m8s
pvc-df98d0b6-263b-4bb4-9be9-549dcc2aa885   1Gi        RWO            Delete           Bound    demo/datadir-mongo-sh-shard0-0      standard                9m19s
```

Services created for MongoDB `mongo-sh`

```console
$ kubectl get svc -n demo
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
mongo-sh                 ClusterIP   10.105.242.203   <none>        27017/TCP   9m40s
mongo-sh-configsvr-gvr   ClusterIP   None             <none>        27017/TCP   9m41s
mongo-sh-shard0-gvr      ClusterIP   None             <none>        27017/TCP   9m41s
mongo-sh-shard1-gvr      ClusterIP   None             <none>        27017/TCP   9m41s
mongo-sh-shard2-gvr      ClusterIP   None             <none>        27017/TCP   9m41s
```

KubeDB operator sets the `status.phase` to `Running` once the database is successfully created. It has also defaulted some field of crd object. Run the following command to see the modified MongoDB object:

```yaml
$ kubedb get mg -n demo mongo-sh -o yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  creationTimestamp: "2020-01-29T11:36:37Z"
  finalizers:
  - kubedb.com
  generation: 3
  name: mongo-sh
  namespace: demo
  resourceVersion: "42648"
  selfLink: /apis/kubedb.com/v1alpha1/namespaces/demo/mongodbs/mongo-sh
  uid: 5110faf5-945c-4917-b7f4-227014dbd05b
spec:
  certificateSecret:
    secretName: mongo-sh-cert
  clusterAuthMode: keyFile
  databaseSecret:
    secretName: mongo-sh-auth
  serviceTemplate:
    metadata: {}
    spec: {}
  shardTopology:
    configServer:
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
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.config: mongo-sh-configsvr
                  namespaces:
                  - demo
                  topologyKey: kubernetes.io/hostname
                weight: 100
              - podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      kubedb.com/kind: MongoDB
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.config: mongo-sh-configsvr
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
            failureThreshold: 3
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources: {}
          serviceAccountName: mongo-sh
      replicas: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
    mongos:
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
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.mongos: mongo-sh-mongos
                  namespaces:
                  - demo
                  topologyKey: kubernetes.io/hostname
                weight: 100
              - podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      kubedb.com/kind: MongoDB
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.mongos: mongo-sh-mongos
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
            failureThreshold: 3
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources: {}
          serviceAccountName: mongo-sh
      replicas: 2
      strategy:
        type: RollingUpdate
    shard:
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
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.shard: mongo-sh-shard${SHARD_INDEX}
                  namespaces:
                  - demo
                  topologyKey: kubernetes.io/hostname
                weight: 100
              - podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      kubedb.com/kind: MongoDB
                      kubedb.com/name: mongo-sh
                      mongodb.kubedb.com/node.shard: mongo-sh-shard${SHARD_INDEX}
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
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
                --quiet --eval \"db.adminCommand('ping').ok\" ) -eq \"1\" ]]; then
                \n          exit 0\n        fi\n        exit 1"
            failureThreshold: 3
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources: {}
          serviceAccountName: mongo-sh
      replicas: 3
      shards: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
  sslMode: disabled
  storageType: Durable
  terminationPolicy: Delete
  updateStrategy:
    type: RollingUpdate
  version: "4.1"
status:
  observedGeneration: 3
  phase: Running
```

Please note that KubeDB operator has created a new Secret called `mongo-sh-auth` _(format: {mongodb-object-name}-auth)_ for storing the password for `mongodb` superuser. This secret contains a `username` key which contains the _username_ for MongoDB superuser and a `password` key which contains the _password_ for MongoDB superuser.

If you want to use custom or existing secret please specify that when creating the MongoDB object using `spec.databaseSecret.secretName`. While creating this secret manually, make sure the secret contains these two keys containing data `username` and `password`. For more details, please see [here](/docs/concepts/databases/mongodb.md#specdatabasesecret).

## Connection Information

- Hostname/address: you can use any of these
  - Service: `mongo-sh.demo`
  - Pod IP: (`$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-mongos -o yaml | grep podIP`)
- Port: `27017`
- Username: Run following command to get _username_,

  ```console
  $ kubectl get secrets -n demo mongo-sh-auth -o jsonpath='{.data.\username}' | base64 -d
  root
  ```

- Password: Run the following command to get _password_,

  ```console
  $ kubectl get secrets -n demo mongo-sh-auth -o jsonpath='{.data.\password}' | base64 -d
  VayjRMWqns-yx0zu
  ```

Now, you can connect to this database through [mongo-shell](https://docs.mongodb.com/v3.6/mongo/).

## Sharded Data

In this tutorial, we will insert sharded and unsharded document, and we will see if the data actually sharded across cluster or not.

```console
$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-mongos
NAME                               READY   STATUS    RESTARTS   AGE
mongo-sh-mongos-65c7c9cccc-gwwbx   1/1     Running   0          13m
mongo-sh-mongos-65c7c9cccc-nslzh   1/1     Running   0          13m

$ kubectl exec -it mongo-sh-mongos-65c7c9cccc-gwwbx -n demo bash

mongodb@mongo-sh-mongos-65c7c9cccc-gwwbx:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD
MongoDB shell version v4.1.13
connecting to: mongodb://127.0.0.1:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("090b7364-74f4-4c68-a277-eb28451fa1e8") }
MongoDB server version: 4.1.13
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
Server has startup warnings: 
2020-01-29T11:41:39.350+0000 I  CONTROL  [main] 
2020-01-29T11:41:39.350+0000 I  CONTROL  [main] ** NOTE: This is a development version (4.1.13) of MongoDB.
2020-01-29T11:41:39.350+0000 I  CONTROL  [main] **       Not recommended for production.
2020-01-29T11:41:39.350+0000 I  CONTROL  [main] ** WARNING: You are running this process as the root user, which is not recommended.
2020-01-29T11:41:39.350+0000 I  CONTROL  [main] 

mongos>
```

To detect if the MongoDB instance that your client is connected to is mongos, use the isMaster command. When a client connects to a mongos, isMaster returns a document with a `msg` field that holds the string `isdbgrid`.

```console
mongos> rs.isMaster()
{
	"ismaster" : true,
	"msg" : "isdbgrid",
	"maxBsonObjectSize" : 16777216,
	"maxMessageSizeBytes" : 48000000,
	"maxWriteBatchSize" : 100000,
	"localTime" : ISODate("2020-01-29T11:55:55.469Z"),
	"logicalSessionTimeoutMinutes" : 30,
	"connectionId" : 190,
	"maxWireVersion" : 8,
	"minWireVersion" : 0,
	"ok" : 1,
	"operationTime" : Timestamp(1580298952, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1580298952, 1),
		"signature" : {
			"hash" : BinData(0,"ggZNIQohNiKKRx9HvhEEy9ynq+4="),
			"keyId" : NumberLong("6787327493494800404")
		}
	}
}
```

`mongo-sh` Shard status,

```console
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5e316e654a2cc3e885b7f985")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mongo-sh-shard0-0.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-1.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-2.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mongo-sh-shard1-0.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-1.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-2.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mongo-sh-shard2-0.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-1.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-2.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "4.1.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0) 
```

Shard collection `test.testcoll` and insert document. See [`sh.shardCollection(namespace, key, unique, options)`](https://docs.mongodb.com/manual/reference/method/sh.shardCollection/#sh.shardCollection) for details about `shardCollection` command.

```console
mongos> sh.enableSharding("test");
{
	"ok" : 1,
	"operationTime" : Timestamp(1580299004, 5),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1580299004, 5),
		"signature" : {
			"hash" : BinData(0,"FRclmF9/OkdmFK4eUqBaf3yMBxg="),
			"keyId" : NumberLong("6787327493494800404")
		}
	}
}

mongos> sh.shardCollection("test.testcoll", {"myfield": 1});
{
	"collectionsharded" : "test.testcoll",
	"collectionUUID" : UUID("d0d39751-2af9-4a76-9e51-8a08f311a934"),
	"ok" : 1,
	"operationTime" : Timestamp(1580299026, 15),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1580299026, 15),
		"signature" : {
			"hash" : BinData(0,"RRGRGitO2L9wVwAbfPPtxDFw4go="),
			"keyId" : NumberLong("6787327493494800404")
		}
	}
}

mongos> use test;
switched to db test

mongos> db.testcoll.insert({"myfield": "a", "otherfield": "b"});
WriteResult({ "nInserted" : 1 })

mongos> db.testcoll.insert({"myfield": "c", "otherfield": "d", "kube" : "db" });
WriteResult({ "nInserted" : 1 })

mongos> db.testcoll.find();
{ "_id" : ObjectId("5e3173297459578f9c747ca3"), "myfield" : "a", "otherfield" : "b" }
{ "_id" : ObjectId("5e3173337459578f9c747ca4"), "myfield" : "c", "otherfield" : "d", "kube" : "db" }
```

Run [`sh.status()`](https://docs.mongodb.com/manual/reference/method/sh.status/) to see whether the `test` database has sharding enabled, and the primary shard for the `test` database.

The Sharded Collection section `sh.status.databases.<collection>` provides information on the sharding details for sharded collection(s) (E.g. `test.testcoll`). For each sharded collection, the section displays the shard key, the number of chunks per shard(s), the distribution of documents across chunks, and the tag information, if any, for shard key range(s).

```console
mongos> sh.status();
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5e316e654a2cc3e885b7f985")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mongo-sh-shard0-0.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-1.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-2.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mongo-sh-shard1-0.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-1.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-2.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mongo-sh-shard2-0.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-1.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-2.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "4.1.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0) 
        {  "_id" : "test",  "primary" : "shard1",  "partitioned" : true,  "version" : {  "uuid" : UUID("5f4b734b-72e1-4c92-9e27-e4b32235092d"),  "lastMod" : 1 } }
                test.testcoll
                        shard key: { "myfield" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard1	1
                        { "myfield" : { "$minKey" : 1 } } -->> { "myfield" : { "$maxKey" : 1 } } on : shard1 Timestamp(1, 0) 
```

Now create another database where partiotioned is not applied and see how the data is stored.

```
mongos> use demo
switched to db demo

mongos> db.testcoll2.insert({"myfield": "ccc", "otherfield": "d", "kube" : "db" });
WriteResult({ "nInserted" : 1 })

mongos> db.testcoll2.insert({"myfield": "aaa", "otherfield": "d", "kube" : "db" });
WriteResult({ "nInserted" : 1 })


mongos> db.testcoll2.find()
{ "_id" : ObjectId("5e31736a7459578f9c747ca5"), "myfield" : "ccc", "otherfield" : "d", "kube" : "db" }
{ "_id" : ObjectId("5e3173717459578f9c747ca6"), "myfield" : "aaa", "otherfield" : "d", "kube" : "db" }
```

Now, eventually `sh.status()`

```
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5e316e654a2cc3e885b7f985")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mongo-sh-shard0-0.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-1.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-2.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mongo-sh-shard1-0.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-1.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-2.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mongo-sh-shard2-0.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-1.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-2.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "4.1.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0) 
        {  "_id" : "demo",  "primary" : "shard2",  "partitioned" : false,  "version" : {  "uuid" : UUID("879acd2d-5cd4-4600-a0d7-fa45c339ce01"),  "lastMod" : 1 } }
        {  "_id" : "test",  "primary" : "shard1",  "partitioned" : true,  "version" : {  "uuid" : UUID("5f4b734b-72e1-4c92-9e27-e4b32235092d"),  "lastMod" : 1 } }
                test.testcoll
                        shard key: { "myfield" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard1	1
                        { "myfield" : { "$minKey" : 1 } } -->> { "myfield" : { "$maxKey" : 1 } } on : shard1 Timestamp(1, 0) 
```

Here, `demo` database is not partitioned and all collections under `demo` database are stored in it's primary shard, which is `shard2`.

## Update number of ShardTopology Instances

User can increase or decrease the number of router/mongos `spec.shardTopology.mongos.replicas`.

At this moment, decreasing the number of shards and replicasets is not handled from KubeDB end. But, User can increase the number of instances if needed.

Here a table of allowed actions are given for mongodb `ShardTopology`,

|                                 Instances                                  | Increase | Decrease |
| :------------------------------------------------------------------------: | :------: | :------: |
|        # of replicas of Mongos `spec.shardTopology.mongos.replicas`        | &#10003; | &#10003; |
|               # of Shards `spec.shardTopology.shard.shards`                | &#10003; | &#10007; |
|     # of replicaset of each Shard `spec.shardTopology.shard.replicas`      | &#10003; | &#10007; |
| # of replicaset of ConfigServer `spec.shardTopology.configServer.replicas` | &#10003; | &#10007; |

Now edit MongoDB `mongo-sh` to increase `spec.shardTopology.shard.shards` to 4 and increase `spec.shardTopology.mongos` to 3.

```console
$ kubectl edit mg -n demo mongo-sh
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mongo-sh
  namespace: demo
  ...
spec:
  shardTopology:
    mongos:
      replicas: 3 # set 3
      ...
    shard:
      shards: 4 # set 4
      ...
  ...
```

Watch for pod changes,

```console
$ kubectl get po --all-namespaces -w
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
demo          mongo-sh-configsvr-0                    1/1     Running   0          8m12s
demo          mongo-sh-configsvr-1                    1/1     Running   0          7m41s
demo          mongo-sh-configsvr-2                    1/1     Running   0          7m17s
demo          mongo-sh-mongos-65c7c9cccc-gwwbx        1/1     Running   0          13m
demo          mongo-sh-mongos-65c7c9cccc-nslzh        1/1     Running   0          13m
demo          mongo-sh-shard0-0                       1/1     Running   0          7m4s
demo          mongo-sh-shard0-1                       1/1     Running   0          6m37s
demo          mongo-sh-shard0-2                       1/1     Running   0          6m20s
demo          mongo-sh-shard1-0                       1/1     Running   0          5m55s
demo          mongo-sh-shard1-1                       1/1     Running   0          5m29s
demo          mongo-sh-shard1-2                       1/1     Running   0          5m14s
demo          mongo-sh-shard2-0                       1/1     Running   0          4m59s
demo          mongo-sh-shard2-1                       1/1     Running   0          4m32s
demo          mongo-sh-shard2-2                       1/1     Running   0          4m11s
kube-system   coredns-fb8b8dccf-nzb5q                 1/1     Running   1          165m
kube-system   coredns-fb8b8dccf-tqldv                 1/1     Running   1          165m
kube-system   etcd-minikube                           1/1     Running   0          164m
kube-system   kube-addon-manager-minikube             1/1     Running   0          164m
kube-system   kube-apiserver-minikube                 1/1     Running   0          163m
kube-system   kube-controller-manager-minikube        1/1     Running   0          164m
kube-system   kube-proxy-qznbv                        1/1     Running   0          165m
kube-system   kube-scheduler-minikube                 1/1     Running   0          163m
kube-system   kubedb-operator-c8cd5c69-c5nkc          1/1     Running   0          44m
kube-system   kubernetes-dashboard-79dd6bfc48-l88rk   1/1     Running   4          165m
kube-system   storage-provisioner                     1/1     Running   0          164m
demo          mongo-sh-shard3-0                       0/1     Pending   0          0s
demo          mongo-sh-shard3-0                       0/1     Pending   0          0s
demo          mongo-sh-shard3-0                       0/1     Pending   0          9s
demo          mongo-sh-shard3-0                       0/1     Init:0/2   0          9s
demo          mongo-sh-shard3-0                       0/1     Init:1/2   0          10s
demo          mongo-sh-shard3-0                       0/1     Init:1/2   0          11s
demo          mongo-sh-shard3-0                       0/1     PodInitializing   0          30s
demo          mongo-sh-shard3-0                       0/1     Running           0          31s
demo          mongo-sh-shard3-0                       1/1     Running           0          41s
demo          mongo-sh-shard3-1                       0/1     Pending           0          0s
demo          mongo-sh-shard3-1                       0/1     Pending           0          0s
demo          mongo-sh-shard3-1                       0/1     Pending           0          5s
demo          mongo-sh-shard3-1                       0/1     Init:0/2          0          5s
demo          mongo-sh-shard3-1                       0/1     Init:1/2          0          6s
demo          mongo-sh-shard3-1                       0/1     Init:1/2          0          7s
demo          mongo-sh-shard3-1                       0/1     PodInitializing   0          15s
demo          mongo-sh-shard3-1                       0/1     Running           0          16s
demo          mongo-sh-shard3-1                       1/1     Running           0          23s
demo          mongo-sh-shard3-2                       0/1     Pending           0          0s
demo          mongo-sh-shard3-2                       0/1     Pending           0          0s
demo          mongo-sh-shard3-2                       0/1     Pending           0          2s
demo          mongo-sh-shard3-2                       0/1     Init:0/2          0          2s
demo          mongo-sh-shard3-2                       0/1     Init:1/2          0          4s
demo          mongo-sh-shard3-2                       0/1     Init:1/2          0          5s
demo          mongo-sh-shard3-2                       0/1     PodInitializing   0          14s
demo          mongo-sh-shard3-2                       0/1     Running           0          15s
demo          mongo-sh-shard3-2                       1/1     Running           0          22s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Pending             0          0s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Pending             0          0s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Init:0/2            0          0s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Init:1/2            0          1s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Init:1/2            0          2s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     PodInitializing     0          8s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   0/1     Running             0          9s
demo          mongo-sh-mongos-54b94d59c-dq5nx                   1/1     Running             0          19s
demo          mongo-sh-mongos-65c7c9cccc-nslzh                  1/1     Terminating         0          21m
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Pending             0          0s
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Pending             0          0s
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Init:0/2            0          0s
demo          mongo-sh-mongos-65c7c9cccc-nslzh                  0/1     Terminating         0          21m
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Init:1/2            0          2s
demo          mongo-sh-mongos-65c7c9cccc-nslzh                  0/1     Terminating         0          21m
demo          mongo-sh-mongos-65c7c9cccc-nslzh                  0/1     Terminating         0          21m
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Init:1/2            0          3s
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     PodInitializing     0          9s
demo          mongo-sh-mongos-54b94d59c-sbckp                   0/1     Running             0          10s
demo          mongo-sh-mongos-54b94d59c-sbckp                   1/1     Running             0          21s
demo          mongo-sh-mongos-65c7c9cccc-gwwbx                  1/1     Terminating         0          21m
demo          mongo-sh-mongos-65c7c9cccc-gwwbx                  0/1     Terminating         0          21m
demo          mongo-sh-mongos-65c7c9cccc-gwwbx                  0/1     Terminating         0          21m
demo          mongo-sh-mongos-65c7c9cccc-gwwbx                  0/1     Terminating         0          21m
```

You can see that an extra statefulset `mongo-sh-shard3` is created as 4th shard and one extra mongos instance also came up.

Notice that, all new mongos instances came up replacing old instances because of some changes in `shard` config. This update strategy follows `spec.shardTopology.mongos.strategy`, which is optional. If not provided, kubernetes will use default deploymentStrategy, ie. `RollingUpdate`. See more about [Deployment Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy).

```console
$ kubectl get deploy -n demo -w
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
mongo-sh-mongos   2/2     2            2           24m
mongo-sh-mongos   2/2     2            2           24m
mongo-sh-mongos   2/2     2            2           24m
mongo-sh-mongos   2/2     0            2           24m
mongo-sh-mongos   2/2     1            2           24m
mongo-sh-mongos   3/2     1            3           24m
mongo-sh-mongos   2/2     1            2           24m
mongo-sh-mongos   2/2     2            2           24m
mongo-sh-mongos   3/2     2            3           25m
mongo-sh-mongos   2/2     2            2           25m
```

Now check `sh.status()` in `mongos`,

```console
$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-mongos
NAME                              READY   STATUS    RESTARTS   AGE
mongo-sh-mongos-54b94d59c-dbzcj   1/1     Running   0          64s
mongo-sh-mongos-54b94d59c-qx468   1/1     Running   0          48s

$ kubectl exec -it mongo-sh-mongos-54b94d59c-dbzcj -n demo bash

mongodb@mongo-sh-mongos-54b94d59c-dbzcj:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5e316e654a2cc3e885b7f985")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mongo-sh-shard0-0.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-1.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-2.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mongo-sh-shard1-0.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-1.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-2.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mongo-sh-shard2-0.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-1.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-2.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard3",  "host" : "shard3/mongo-sh-shard3-0.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017,mongo-sh-shard3-1.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017,mongo-sh-shard3-2.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "4.1.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0) 
        {  "_id" : "demo",  "primary" : "shard2",  "partitioned" : false,  "version" : {  "uuid" : UUID("879acd2d-5cd4-4600-a0d7-fa45c339ce01"),  "lastMod" : 1 } }
        {  "_id" : "test",  "primary" : "shard1",  "partitioned" : true,  "version" : {  "uuid" : UUID("5f4b734b-72e1-4c92-9e27-e4b32235092d"),  "lastMod" : 1 } }
                test.testcoll
                        shard key: { "myfield" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard1	1
                        { "myfield" : { "$minKey" : 1 } } -->> { "myfield" : { "$maxKey" : 1 } } on : shard1 Timestamp(1, 0) 
```

## Halt Database

User can halt the database by setting `spec.halted` to `true`. As the name suggests, this will halt the database by deleting all the kubernetes resources (including `sts`, `services`, rbac resources, etc) except `secrets` and `PVCs`. On the other hand, when users wishes to resume a halted database, they will need to set `spec.halted` to `false`.

Please Note that, if [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is set to `DoNotTerminate`, the halt database operation will fail with forbidden error message from admission controller. 

If [spec.terminationPolicy](/docs/concepts/databases/mongodb.md#specterminationpolicy) is not set to `DoNotTerminate`, then kubedb will accept the user's halt request and also it will update the `spec.terminationPolicy` to `Halt`.

Set `spec.halted` to `true`.

```console
$ kubectl patch -n demo mg/mongo-sh -p '{"spec":{"halted":true}}' --type="merge"
mongodb.kubedb.com/mongo-sh patched
```

Database status will be set to `Halted` after successful deletion of kubernetes resources.

```console
$ kubectl get mongodb -n demo
NAME       VERSION   STATUS   AGE
mongo-sh   4.1       Halted   33m
```

```console
$ kubedb describe mg -n demo mongo-sh
Name:               mongo-sh
Namespace:          demo
CreationTimestamp:  Wed, 29 Jan 2020 17:36:37 +0600
Labels:             <none>
Annotations:        <none>
Status:             Halted
  StorageType:      Durable
No volumes.

Database Secret:
  Name:         mongo-sh-auth
  Labels:         app.kubernetes.io/component=database
                  app.kubernetes.io/instance=mongo-sh
                  app.kubernetes.io/managed-by=kubedb.com
                  app.kubernetes.io/name=mongodb
                  app.kubernetes.io/version=4.1
                  kubedb.com/kind=MongoDB
                  kubedb.com/name=mongo-sh
  Annotations:  <none>
  
Type:  Opaque
  
Data
====
  password:  16 bytes
  username:  4 bytes

Events:
  Type    Reason      Age   From              Message
  ----    ------      ----  ----              -------
  ...
  Normal  Successful  4m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-configsvr
  Normal  Successful  4m    MongoDB operator  Successfully patched Deployment demo/mongo-sh-mongos
  Normal  Successful  4m    MongoDB operator  Successfully patched MongoDB
  Normal  Successful  4m    MongoDB operator  Successfully patched StatefulSet demo/mongo-sh-shard0
```

## Resume Database

To resume the database, set `spec.halted` to `false`.

```console
$ kubectl patch -n demo mg/mongo-sh -p '{"spec":{"halted":false}}' --type="merge"
mongodb.kubedb.com/mongo-sh patched
```

```console
$ kubectl get mongodb -n demo
NAME             VERSION   STATUS    AGE
mongo-sh   4.1       Running   21m
```

Now, If you again exec into `pod` and look for previous data, you will see that, all the data persists.

```console
$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-mongos
NAME                              READY   STATUS    RESTARTS   AGE
mongo-sh-mongos-54b94d59c-qswl2   1/1     Running   0          22s
mongo-sh-mongos-54b94d59c-th4c6   1/1     Running   0          22s

$ kubectl exec -it mongo-sh-mongos-54b94d59c-qswl2 -n demo bash

mongodb@mongo-sh-mongos-54b94d59c-qswl2:/$ mongo admin --username=$MONGO_INITDB_ROOT_USERNAME --password=$MONGO_INITDB_ROOT_PASSWORD

mongos> use test;
switched to db test

mongos> db.testcoll.find();
{ "_id" : ObjectId("5e3173297459578f9c747ca3"), "myfield" : "a", "otherfield" : "b" }
{ "_id" : ObjectId("5e3173337459578f9c747ca4"), "myfield" : "c", "otherfield" : "d", "kube" : "db" }

mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5e316e654a2cc3e885b7f985")
  }
  shards:
        {  "_id" : "shard0",  "host" : "shard0/mongo-sh-shard0-0.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-1.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017,mongo-sh-shard0-2.mongo-sh-shard0-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard1",  "host" : "shard1/mongo-sh-shard1-0.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-1.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017,mongo-sh-shard1-2.mongo-sh-shard1-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/mongo-sh-shard2-0.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-1.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017,mongo-sh-shard2-2.mongo-sh-shard2-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard3",  "host" : "shard3/mongo-sh-shard3-0.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017,mongo-sh-shard3-1.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017,mongo-sh-shard3-2.mongo-sh-shard3-gvr.demo.svc.cluster.local:27017",  "state" : 1 }
  active mongoses:
        "4.1.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  3
        Last reported error:  Could not find host matching read preference { mode: "primary" } for set shard0
        Time of Reported error:  Wed Jan 29 2020 12:14:55 GMT+0000 (UTC)
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard0	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(1, 0) 
        {  "_id" : "demo",  "primary" : "shard2",  "partitioned" : false,  "version" : {  "uuid" : UUID("879acd2d-5cd4-4600-a0d7-fa45c339ce01"),  "lastMod" : 1 } }
        {  "_id" : "test",  "primary" : "shard1",  "partitioned" : true,  "version" : {  "uuid" : UUID("5f4b734b-72e1-4c92-9e27-e4b32235092d"),  "lastMod" : 1 } }
                test.testcoll
                        shard key: { "myfield" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shard1	1
                        { "myfield" : { "$minKey" : 1 } } -->> { "myfield" : { "$maxKey" : 1 } } on : shard1 Timestamp(1, 0) 

> exit
bye
```

## WipeOut DormantDatabase

You can wipe out the database while deleting the object by setting `spec.terminationPolicy` to `WipeOut`. KubeDB operator will delete any relevant resources of this `MongoDB` database (i.e, PVCs, Secrets, Rbac resources, Statefulsets, stash backup data). It will also delete stash backup data stored in the Cloud Storage buckets.

```console
$ kubectl patch -n demo mg/mongo-sh -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
mongodb.kubedb.com/mongo-sh patched

$ kubectl delete -n demo mg/mongo-sh
mongodb.kubedb.com "mongo-sh" deleted
```

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/mongo-sh -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mongo-sh

kubectl delete ns demo
```

## Next Steps

- Initialize [MongoDB with Script](/docs/guides/mongodb/initialization/using-script.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/mongodb/monitoring/using-coreos-prometheus-operator.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/mongodb/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/mongodb/private-registry/using-private-registry.md) to deploy MongoDB with KubeDB.
- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- Detail concepts of [MongoDBVersion object](/docs/concepts/catalog/mongodb.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
