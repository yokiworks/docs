---
title: TLS/SSL (Transport Encryption)
menu:
  docs_{{ .version }}:
    identifier: mg-tls-encryption
    name: TLS/SSL (Transport Encryption)
    parent: mg-tls
    weight: 15
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Run MongoDB with TLS/SSL (Transport Encryption)

KubeDB supports providing TLS/SSL encryption (via, `sslMode` and `clusterAuthMode`) for MongoDB. This tutorial will show you how to use KubeDB to run a MongoDB database with TLS/SSL encryption.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `cert-manger operator` (v0.12.0 or later) to your cluster to manage your SSL/TLS certificates.

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```console
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: YAML files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

KubeDB uses following crd fields to enable SSL/TLS encryption in Mongodb.

- `spec:`
  - `sslMode`
  - `tls:`
    - `issuerRef`
    - `certificate`
  - `clusterAuthMode`

Read about the fields in details in [mongodb concept](/docs/concepts/databases/mongodb.md),

`sslMode`, and `tls` is applicable for all kind of mongodb (i.e., `standalone`, `replicaset` and `sharding`), while `clusterAuthMode` provides [ClusterAuthMode](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-clusterauthmode) for mongodb clusters (i.e., `replicaset` and `sharding`).

When, SSLMode is anything other than `disabled`, users must specify the `tls.issuerRef` field. Kubedb uses the `issuer` or `clusterIssuer` referenced in the `tls.issuerRef` field, and the certificate specs provided in `tls.certificate` to generate certificate secrets. These certificate secrets are then used to generate required certificates including `ca.crt`, `mongo.pem` and `client.pem`.

The subject of `client.pem` certificate is added as `root` user in `$external` mongodb database. So, user can use this client certificate for `MONGODB-X509` `authenticationMechanism`.

## Create Issuer/ ClusterIssuer

We are going to create an example `Issuer` that will be used throughout the duration of this tutorial to enable SSL/TLS in MongoDB. Alternatively, you can follow this [cert-manager tutorial](https://cert-manager.io/docs/configuration/ca/) to create your own `Issuer`.

- Start off by generating you ca certificates using openssl.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./ca.key -out ./ca.crt -subj "/CN=mongo/O=kubedb"
```

- Now create a ca-secret using the certificate files you have just generated.

```bash
kubectl create secret tls mongo-ca \
     --cert=ca.crt \
     --key=ca.key \
     --namespace=demo
```

Now, create an `Issuer` using the `ca-secret` you have just created. The `YAML` file looks like this:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: mongo-ca-issuer
  namespace: demo
spec:
  ca:
    secretName: mongo-ca
```

Apply the `YAML` file:

```bash
kubectl apply -f ./docs/examples/mongodb/tls-ssl-encryption/issuer.yaml
```


## TLS/SSL encryption in MongoDB Standalone

Below is the YAML for MongoDB Standalone. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `standalone` (which is `requireSSL`).

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-tls
  namespace: demo
spec:
  version: "4.1.13-v1"
  sslMode: requireSSL
  tls:
    issuerRef:
      name: mongo-ca-issuer
      kind: Issuer
      apiGroup: "cert-manager.io"
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

### Deploy MongoDB Standalone

```console
$ kubectl create -f ./docs/examples/mongodb/tls-ssl-encryption/tls-standalone.yaml
mongodb.kubedb.com/mgo-tls created
```

Now, wait until `mgo-tls created` has status `Running`. i.e,

```console
$ kubectl get mg -n demo
NAME      VERSION     STATUS     AGE
mgo-tls   4.1.13-v1   Running    14s
```

### Verify TLS/SSL in MongoDB Standalone

Now, connect to this database through [mongo-shell](https://docs.mongodb.com/v4.0/mongo/) and verify if `SSLMode` has been set up as intended (i.e, `requireSSL`).

```console
$ kubectl describe secret -n demo mgo-tls-client-cert
Name:         mgo-tls-client-cert
Namespace:    demo
Labels:       <none>
Annotations:  cert-manager.io/alt-names:
                localhost,mgo-tls,mgo-tls.demo.svc,mgo-tls-client-cert,mgo-tls-client-cert.mgo-tls-gvr-gvr,mgo-tls-client-cert.mgo-tls-gvr-gvr.demo.svc,mg...
              cert-manager.io/certificate-name: mgo-tls-client-cert
              cert-manager.io/common-name: root
              cert-manager.io/ip-sans: 127.0.0.1
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: mongo-ca-issuer
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
tls.key:  1679 bytes
ca.crt:   1241 bytes
tls.crt:  1537 bytes
```

```console
$ kubectl exec -it mgo-tls-0 -n demo bash
mongodb@mgo-tls-0: # you are into container

mongodb@mgo-tls-0:/$ ls /var/run/mongodb/tls
ca.crt  client.pem  mongo.pem

mongodb@mgo-tls-0:/$ openssl x509 -in /var/run/mongodb/tls/client.pem -inform PEM -subject -nameopt RFC2253 -noout
subject=CN=root,O=kubedb:client

# Now use CN=root,O=kubedb:client as root 
mongodb@mgo-tls-0:/$ mongo --ssl --sslCAFile /var/run/mongodb/tls/ca.crt --sslPEMKeyFile /var/run/mongodb/tls/client.pem admin --host localhost --authenticationMechanism MONGODB-X509 --authenticationDatabase='$external' -u "CN=root,O=kubedb:client"

MongoDB shell version v4.1.13
connecting to: mongodb://localhost:27017/admin?authMechanism=MONGODB-X509&authSource=%24external&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("291b3459-3b68-4f0b-b96b-a334a3990024") }
MongoDB server version: 4.1.13
Server has startup warnings: 

---

> #you are connected to mongo cli

> db.adminCommand({ getParameter:1, sslMode:1 })
{ "sslMode" : "requireSSL", "ok" : 1 }

> use $external
switched to db $external

> show users
{
	"_id" : "$external.CN=root,O=kubedb:client",
	"userId" : UUID("fce9cd26-eeec-4ffa-b8f5-26b5bd4de4a7"),
	"user" : "CN=root,O=kubedb:client",
	"db" : "$external",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"external"
	]

> exit
bye
```

You can see here that, `sslMode` is set to `requireSSL` and also an user is created in `$external` with name `"CN=root,O=kubedb:client"`.

## TLS/SSL encryption in MongoDB Replicaset

Below is the YAML for MongoDB Replicaset. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `replicaset` (which is `requireSSL`) and [`spec.clusterAuthMode`](/docs/concepts/databases/mongodb.md#specclusterAuthMode) provides `clusterAuthMode` for mongodb replicaset nodes (which is `x509`).

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-rs-tls
  namespace: demo
spec:
  version: "4.1.13-v1"
  sslMode: requireSSL
  tls:
    issuerRef:
      name: mongo-ca-issuer
      kind: Issuer
      apiGroup: "cert-manager.io"
  clusterAuthMode: x509
  replicas: 4
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

### Deploy MongoDB Replicaset

```console
$ kubectl create -f ./docs/examples/mongodb/tls-ssl-encryption/tls-replicaset.yaml
mongodb.kubedb.com/mgo-rs-tls created
```

Now, wait until `mgo-rs-tls created` has status `Running`. i.e,

```console
$ kubectl get mg -n demo
NAME         VERSION     STATUS    AGE
mgo-rs-tls   4.1.13-v1   Running   3m28s
```

### Verify TLS/SSL in MongoDB Replicaset

Now, connect to this database through [mongo-shell](https://docs.mongodb.com/v4.0/mongo/) and verify if `SSLMode` and `ClusterAuthMode` has been set up as intended.

```console
$ kubectl describe secret -n demo mgo-rs-tls-client-cert
Name:         mgo-rs-tls-client-cert
Namespace:    demo
Labels:       <none>
Annotations:  cert-manager.io/alt-names:
                localhost,mgo-rs-tls,mgo-rs-tls.demo.svc,mgo-rs-tls-client-cert,mgo-rs-tls-client-cert.mgo-rs-tls-gvr-gvr,mgo-rs-tls-client-cert.mgo-rs-tl...
              cert-manager.io/certificate-name: mgo-rs-tls-client-cert
              cert-manager.io/common-name: root
              cert-manager.io/ip-sans: 127.0.0.1
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: mongo-ca-issuer
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
ca.crt:   1241 bytes
tls.crt:  1582 bytes
tls.key:  1679 bytes
```

```console
$ kubectl exec -it mgo-rs-tls-0 -n demo bash
mongodb@mgo-rs-tls-0: # you are into container

mongodb@mgo-rs-tls-0:/$ ls /var/run/mongodb/tls/
ca.crt  client.pem  mongo.pem

mongodb@mgo-rs-tls-0:/$ openssl x509 -in /var/run/mongodb/tls/client.pem -inform PEM -subject -nameopt RFC2253 -noout
subject=CN=root,O=kubedb:client

# Now use CN=root,O=kubedb:client as root 
mongodb@mgo-rs-tls-0:/$ mongo --ssl --sslCAFile /var/run/mongodb/tls/ca.crt --sslPEMKeyFile /var/run/mongodb/tls/client.pem admin --host localhost --authenticationMechanism MONGODB-X509 --authenticationDatabase='$external' -u "CN=root,O=kubedb:client"

MongoDB shell version v4.1.13
connecting to: mongodb://localhost:27017/admin?authMechanism=MONGODB-X509&authSource=%24external&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("dc15cbf7-699c-4892-9477-533c22aed175") }
MongoDB server version: 4.1.13
Welcome to the MongoDB shell.
---
rs0:PRIMARY> #you are connected to mongo cli

rs0:PRIMARY> db.adminCommand({ getParameter:1, sslMode:1 })
{
	"sslMode" : "requireSSL",
	"ok" : 1,
	"operationTime" : Timestamp(1564746095, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1564746095, 1),
		"signature" : {
			"hash" : BinData(0,"29XpLUr5ZaWhAjadOvzosW2CMsU="),
			"keyId" : NumberLong("6720531552222052354")
		}
	}
}

rs0:PRIMARY> db.adminCommand({ getParameter:1, clusterAuthMode:1 })
{
	"clusterAuthMode" : "x509",
	"ok" : 1,
	"operationTime" : Timestamp(1564746155, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1564746155, 1),
		"signature" : {
			"hash" : BinData(0,"D2kt9g/RsktDNJf0qM7NP9CUdX0="),
			"keyId" : NumberLong("6720531552222052354")
		}
	}
}


rs0:PRIMARY> use $external
switched to db $external

rs0:PRIMARY> show users
{
	"_id" : "$external.CN=root,O=kubedb:client",
	"userId" : UUID("a7f56f70-cd90-4ea8-846a-1a64736d44c8"),
	"user" : "CN=root,O=kubedb:client",
	"db" : "$external",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	]
}

rs0:PRIMARY> exit
bye
```

You can see here that, `sslMode` is set to `requireSSL` & `clusterAuthMode` is set to `x509` and also an user is created in `$external` with name `"CN=root,O=kubedb:client"`.

## TLS/SSL encryption in MongoDB Sharding

Below is the YAML for MongoDB Sharding. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `sharding` and [`spec.clusterAuthMode`](/docs/concepts/databases/mongodb.md#specclusterAuthMode) provides `clusterAuthMode` for sharding servers.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mongo-sh-tls
  namespace: demo
spec:
  version: "4.1.13-v1"
  sslMode: requireSSL
  tls:
    issuerRef:
      name: mongo-ca-issuer
      kind: Issuer
      apiGroup: "cert-manager.io"
  clusterAuthMode: x509
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
  updateStrategy:
    type: RollingUpdate
  storageType: Durable
  terminationPolicy: WipeOut

```

### Deploy MongoDB Sharding

```console
$ kubectl create -f ./docs/examples/mongodb/tls-ssl-encryption/tls-sharding.yaml
mongodb.kubedb.com/mongo-sh-tls created
```

Now, wait until `mongo-sh-tls created` has status `Running`. ie,

```console
$ kubectl get mg -n demo
NAME           VERSION   STATUS    AGE
mongo-sh-tls   3.6-v4    Running   9m34s
```

### Verify TLS/SSL in MongoDB Sharding

Now, connect to `mongos` component of this database through [mongo-shell](https://docs.mongodb.com/v4.0/mongo/) and verify if `SSLMode` and `ClusterAuthMode` has been set up as intended.

```console
$ kubectl describe secret -n demo mongo-sh-tls-client-cert
Name:         mongo-sh-tls-client-cert
Namespace:    demo
Labels:       <none>
Annotations:  cert-manager.io/alt-names:
                localhost,mongo-sh-tls,mongo-sh-tls.demo.svc,mongo-sh-tls-client-cert,mongo-sh-tls-client-cert.mongo-sh-tls-gvr-gvr,mongo-sh-tls-client-ce...
              cert-manager.io/certificate-name: mongo-sh-tls-client-cert
              cert-manager.io/common-name: root
              cert-manager.io/ip-sans: 127.0.0.1
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: mongo-ca-issuer
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
tls.key:  1679 bytes
ca.crt:   1241 bytes
tls.crt:  1610 bytes

```

```console
$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-tls-mongos
NAME                    READY   STATUS    RESTARTS   AGE
mongo-sh-tls-mongos-0   1/1     Running   0          10m
mongo-sh-tls-mongos-1   1/1     Running   0          10m

$ kubectl exec -it mongo-sh-tls-mongos-0 -n demo bash
mongodb@mongo-sh-tls-mongos-5958559b5d-5vrjm: # you are into container

mongodb@mongo-sh-tls-mongos-5958559b5d-5vrjm:/$ ls /var/run/mongodb/tls/
ca.crt  client.pem  mongo.pem
mongodb@mongo-sh-tls-mongos-5958559b5d-5vrjm:/$ openssl x509 -in /var/run/mongodb/tls/client.pem -inform PEM -subject -nameopt RFC2253 -noout
subject=CN=root,O=kubedb:client

# Now use CN=root,O=kubedb:client as root
mongodb@mongo-sh-tls-mongos-5958559b5d-5vrjm:/$ mongo --ssl --sslCAFile /var/run/mongodb/tls/ca.crt --sslPEMKeyFile /var/run/mongodb/tls/client.pem admin --host localhost --authenticationMechanism MONGODB-X509 --authenticationDatabase='$external' -u "CN=root,O=kubedb:client"
MongoDB shell version v4.1.13
connecting to: mongodb://localhost:27017/admin?authMechanism=MONGODB-X509&authSource=%24external&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("bda2f89b-5a1c-463d-a8e1-dd22fa2fef05") }
MongoDB server version: 4.1.13
Welcome to the MongoDB shell.

mongos> #you are connected to mongo cli

mongos> db.adminCommand({ getParameter:1, sslMode:1 })
{
	"sslMode" : "requireSSL",
	"ok" : 1,
	"operationTime" : Timestamp(1581415351, 2),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1581415351, 2),
		"signature" : {
			"hash" : BinData(0,"tdviL+PeK5/B0oyxGwN1q7d2X7Q="),
			"keyId" : NumberLong("6792112611048554513")
		}
	}
}

mongos> db.adminCommand({ getParameter:1, clusterAuthMode:1 })
{
	"clusterAuthMode" : "x509",
	"ok" : 1,
	"operationTime" : Timestamp(1581415371, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1581415371, 1),
		"signature" : {
			"hash" : BinData(0,"O9LLOIjRnsaCtKU3nMyTB/H1jMI="),
			"keyId" : NumberLong("6792112611048554513")
		}
	}
}

mongos> use $external
switched to db $external

mongos> show users
{
	"_id" : "$external.CN=root,O=kubedb:client",
	"userId" : UUID("bf1d46bf-e91d-419d-b633-6f68cc7189a2"),
	"user" : "CN=root,O=kubedb:client",
	"db" : "$external",
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"external"
	]
}
mongos> exit
bye
```

You can see here that, `sslMode` is set to `requireSSL` & `clusterAuthMode` is set to `x509` and also an user is created in `$external` with name `"CN=root,O=kubedb:client"`.

## Changing the SSLMode & ClusterAuthMode

User can update `sslMode` & `ClusterAuthMode` if needed. Some changes may be invalid from mongodb end, like using `sslMode: disabled` with `clusterAuthMode: x509`.

Good thing is, **KubeDB webhook will throw error for invalid SSL specs while creating/updating the mongodb crd object.** i.e.,

```console
$ kubectl patch -n demo mg/mgo-rs-tls -p '{"spec":{"sslMode": "disabled","clusterAuthMode": "x509"}}' --type="merge"
Error from server (Forbidden): admission webhook "mongodb.validators.kubedb.com" denied the request: can't have disabled set to mongodb.spec.sslMode when mongodb.spec.clusterAuthMode is set to x509
```

To **Upgrade from Keyfile Authentication to x.509 Authentication**, change the `sslMode` and `clusterAuthMode` in recommended sequence as suggested in [official documentation](https://docs.mongodb.com/manual/tutorial/upgrade-keyfile-to-x509/).  Each time after changing the specs, follow the procedure that is described above to verify the changes of `sslMode` and `clusterAuthMode` inside the database.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/mgo-rs-tls mg/mgo-tls mg/mongo-sh-tls -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mgo-rs-tls mg/mgo-tls mg/mongo-sh-tls

kubectl patch -n demo drmn/mgo-rs-tls drmn/mgo-tls drmn/mongo-sh-tls -p '{"spec":{"wipeOut":true}}' --type="merge"
kubectl delete -n demo drmn/mgo-rs-tls drmn/mgo-tls drmn/mongo-sh-tls

kubectl delete ns demo
```

## Next Steps

- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- [Snapshot and Restore](/docs/guides/mongodb/snapshot/backup-and-restore.md) process of MongoDB databases using KubeDB.
- Take [Scheduled Snapshot](/docs/guides/mongodb/snapshot/scheduled-backup.md) of MongoDB databases using KubeDB.
- Initialize [MongoDB with Script](/docs/guides/mongodb/initialization/using-script.md).
- Initialize [MongoDB with Snapshot](/docs/guides/mongodb/initialization/using-snapshot.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/mongodb/monitoring/using-coreos-prometheus-operator.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/mongodb/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/mongodb/private-registry/using-private-registry.md) to deploy MongoDB with KubeDB.
- Use [kubedb cli](/docs/guides/mongodb/cli/cli.md) to manage databases like kubectl for Kubernetes.
- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- Detail concepts of [Snapshot object](/docs/concepts/snapshot.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
