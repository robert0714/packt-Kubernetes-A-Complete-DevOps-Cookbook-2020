# Configuring and managing persistent storage using Rook  
Rook is a cloud-native, open source storage orchestrator for Kubernetes. Rook provides self-managing, self-scaling, and self-healing distributed storage systems in Kubernetes. In this section, we will create multiple storage providers using the Rook storage orchestrator for your applications in Kubernetes. You will learn to create a Ceph provider for your stateful applications that require persistent storage.  
## Getting ready
Make sure that you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.
## How to do it…  
This section is sub-divided further into the following subsections to facilitate the process:
Installing a Ceph provider using Rook
*  Creating a Ceph cluster
*  Verifying a Ceph cluster's health
*  Create a Ceph block storage class
*  Using a Ceph block storage class to create dynamic PVs

## Installing a Ceph provider using Rook
Let's perform the following steps to get a Ceph scale-out storage solution up and running using the Rook project:
1. Clone the Rook repository:
```
$ git clone https://github.com/rook/rook.git
$ cd rook/cluster/examples/kubernetes/ceph/
```   
2. Deploy the Rook Operator:
```
$ kubectl create -f common.yaml
$ kubectl create -f operator.yaml
```  
3. Verify the Rook Operator:
```
$ kubectl get pod -n rook-ceph
NAME                                  READY   STATUS    RESTARTS  AGE
rook-ceph-operator-6b66859964-vnrfx   1/1     Running     0       2m12s
rook-discover-8snpm                   1/1     Running     0       97s
rook-discover-mcx9q                   1/1     Running     0       97s
rook-discover-mdg2s                   1/1     Running     0       97s
```
Now you have learned how to deploy the Rook orchestration components for the Ceph provider running on Kubernetes.

## Creating a Ceph cluster  
Let's perform the following steps to deploy a Ceph cluster using the Rook Operator:
1. Create a Ceph cluster:
```bash
$ cat <<EOF | kubectl apply -f -
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
cephVersion:
  image: ceph/ceph:v14.2.3-20190904
dataDirHostPath: /var/lib/rook
mon:
  count: 3
dashboard:
  enabled: true
storage:
  useAllNodes: true
  useAllDevices: false
  directories:
  - path: /var/lib/rook
EOF
```  
2. Verify that all pods are running:  
```  
$ kubectl get pod -n rook-ceph
```  
Within a minute, a fully functional Ceph cluster will be deployed and ready to be used. You can read more about Ceph in the Rook Ceph Storage Documentation link in the See also
section.

## Verifying a Ceph cluster's health
The Rook toolbox is a container with common tools used for rook debugging and testing. Let's perform the following steps to deploy the Rook toolbox to verify cluster health:  
1. Deploy the Rook toolbox:
```bash
$ kubectl apply -f toolbox.yaml
```
2. Verify that the toolbox is running:
```
$ kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
NAME                              READY   STATUS    RESTARTS  AGE
rook-ceph-tools-6fdfc54b6d-4kdtm  1/1     Running   0         109s
```
3. Connect to the toolbox:
```
$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```
4. Verify that the cluster is in a healthy state (HEALTH_OK):
```
# ceph status
cluster:
  id: 6b6e4bfb-bfef-46b7-94bd-9979e5e8bf04
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum a,b,c (age 12m)
  mgr: a(active, since 12m)
  osd: 3 osds: 3 up (since 11m), 3 in (since 11m)
data:
  pools: 0 pools, 0 pgs
  objects: 0 objects, 0 B
  usage: 49 GiB used, 241 GiB / 291 GiB avail
  pgs:
```
5. When you are finished troubleshooting, remove the deployment using the following command:
```
$ kubectl -n rook-ceph delete deployment rook-ceph-tools
```
Now you know how to deploy the Rook toolbox with its common tools that are used to debug and test Rook.

## Create a Ceph block storage class
Let's perform the following steps to create a storage class for Ceph storage.:
1. Create CephBlockPool:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
EOF
```
2. Create a Rook Ceph block storage class:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Delete
EOF
```  
3. Confirm that the storage class has been created:
```
$ kubectl get sc
NAME                PROVISIONER                   AGE
default (default)   kubernetes.io/azure-disk      6h27m
rook-ceph-block     rook-ceph.rbd.csi.ceph.com    3s
```
As you can see from the preceding provisioner name, rook-ceph.rbd.csi.ceph.com,Rook also uses CSI to interact with Kubernetes APIs. This driver is optimized for RWO pod
access where only one pod may access the storage.

## Using a Ceph block storage class to create dynamic PVs
In this recipe, we will deploy Wordpress using dynamic persistent volumes created by the Rook Ceph block storage provider. Let's perform the following steps:
1. Clone the examples repository:
```bash
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter5/rook/
```
2. Review both mysql.yaml and wordpress.yaml. Note that PVCs are using the rook-ceph-block storage class:
```
$ cat mysql.yaml && cat wordpress.yaml
```
3. Deploy MySQL and WordPress:
```
$ kubectl apply -f mysql.yaml
$ kubectl apply -f wordpress.yaml
```
4. Confirm the persistent volumes created:
```
$ kubectl get pv
NAME                                     CAPACITY ACCESS  MODES   RECLAIM   POLICY STATUS CLAIM     STORAGECLASS REASON   AGE
pvc-eb2d23b8-d38a-11e9-88a2-a2c82783dcda 20Gi     RWO     Delete  Bound     default/mysql-pv-claim  rook-ceph-block       38s
pvc-eeab1ebc-d38a-11e9-88a2-a2c82783dcda 20Gi     RWO     Delete  Bound     default/wp-pv-claim     rook-ceph-block       38s
```
5. Get the external IP of the WordPress service:
```
$ kubectl get service
NAME            TYPE          CLUSTER-IP      EXTERNAL-IP   PORT(S)       AGE
kubernetes      ClusterIP     10.0.0.1        <none>        443/TCP       6h34m
wordpress       LoadBalancer  10.0.102.14     13.64.96.240  80:30596/TCP  3m36s
wordpress-mysql ClusterIP     None            <none>        3306/TCP      3m42s
```
6. Open the external IP of the WordPress service in your browser to access your Wordpress deployment:

Now you know how to get the popular WordPress service, with persistent storage stored on Rook-based Ceph storage, up and running.

## See also
*  Rook documentation: https://rook.io/docs/rook/master/
*  Rook Ceph storage documentation: https://rook.io/docs/rook/master/cephstorage.html
*  Rook community slack channel: https://slack.rook.io/
