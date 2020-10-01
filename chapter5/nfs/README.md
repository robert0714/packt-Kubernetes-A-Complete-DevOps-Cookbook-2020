# Setting up NFS for shared storage on Kubernetes
Although it's not the best-performing solution, NFS is still used with cloud-native applications where multi-node write access is required. In this section, we will create an NFS-based persistent storage for this type of application. You will learn how to use OpenEBS and Rook to **ReadWriteMany (RWX)** accessible persistent volumes for stateful workloads that require shared storage on Kubernetes.
## Getting ready
For this recipe, we need to have either rook or openebs installed as an orchestrator. Make sure that you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.
## How to do it…
There are two popular alternatives when it comes to providing an NFS service. This section is sub-divided further into the following subsections to explain the process using Rook and
OpenEBS:
*  Installing NFS prerequisites
*  Installing an NFS provider using a Rook NFS operator
*  Using a Rook NFS operator storage class to create dynamic NFS PVs
*  Installing an NFS provider using OpenEBS
*  Using the OpenEBS operator storage class to create dynamic NFS PVs
##  Installing NFS prerequisites
To be able to mount NFS volumes, NFS client packages need to be preinstalled on all worker nodes where you plan to have NFS-mounted pods:  
1. If you are using Ubuntu, install nfs-common on all worker nodes:  
```
$ sudo apt install -y nfs-common
```  
2. If using CentOS, install nfs-common on all worker nodes:  
```
$ yum install nfs-utils
```  
Now we have nfs-utils installed on our worker nodes and are ready to get the NFS server to deploy.

##  Installing an NFS provider using a Rook NFS operator
Let's perform the following steps to get an NFS provider functional using the Rook NFS provider option:   
1. Clone the Rook repository:
```
$ git clone https://github.com/rook/rook.git
$ cd rook/cluster/examples/kubernetes/nfs/
```  
2. Deploy the Rook NFS operator:
```
$ kubectl create -f operator.yaml
```  
3. Confirm that the operator is running:
```
$ kubectl get pods -n rook-nfs-system
NAME                                      READY   STATUS    RESTARTS  AGE
rook-nfs-operator-54cf68686c-f66f5        1/1     Running   0         51s
rook-nfs-provisioner-79fbdc79bb-hf9rn     1/1     Running   0         51s
```  
4. Create a namespace, rook-nfs:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: rook-nfs
EOF
```
5. Make sure that you have defined your preferred storage provider as the default
storage class. In this recipe, we are using openebs-cstor-default, defined in
persistent storage using the OpenEBS recipe.
6. Create a PVC:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-default-claim
  namespace: rook-nfs
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```
7. Create the NFS instance:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: nfs.rook.io/v1alpha1
kind: NFSServer
metadata:
  name: rook-nfs
  namespace: rook-nfs
spec:
  serviceAccountName: rook-nfs
  replicas: 1
  exports:
  - name: share1
    server:
      accessMode: ReadWrite
      squash: "none"
    persistentVolumeClaim:
      claimName: nfs-default-claim
  annotations:
  # key: value
EOF
```  
8. Verify that the NFS pod is in the Running state:
```
$ kubectl get pod -l app=rook-nfs -n rook-nfs
NAME        READY STATUS    RESTARTS   AGE
rook-nfs-0  1/1   Running   0           2m
```   
By observing the preceding command, an NFS server instance type will be created.
##　　Using a Rook NFS operator storage class to create dynamic NFS PVs
NFS is used in the Kubernetes environment on account of its ReadWriteMany capabilities　for the application that requires access to the same data at the same time. In this recipe, we　will perform the following steps to dynamically create an NFS-based persistent volume:　　
1. Create Rook NFS storage classes using exportName, nfsServerName,and *nfsServerNamespace* from the Installing an NFS provider using a Rook NFS operator recipe:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    app: rook-nfs
  name: rook-nfs-share1
parameters:
  exportName: share1
  nfsServerName: rook-nfs
  nfsServerNamespace: rook-nfs
provisioner: rook.io/nfs-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```
2. Now, you can use the rook-nfs-share1 storage class to create PVCs for applications that require ReadWriteMany access:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rook-nfs-pv-claim
spec:
storageClassName: "rook-nfs-share1"
accessModes:
  - ReadWriteMany
resources:
  requests:
    storage: 1Mi
EOF
```
## Installing an NFS provisioner using OpenEBS
OpenEBS provides an NFS provisioner that is protected by the underlying storage engine options of OpenEBS. Let's perform the following steps to get an NFS service with OpenEBS up and running:
1. Clone the examples repository:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter5/openebs
```
2. In this recipe, we are using the openebs-jiva-default storage class. Review the directory content and apply the YAML file under the NFS directory:
```
$ kubectl apply -f nfs
```
3. List the PVCs and confirm that a PVC named openebspvc has been created:
```
$ kubectl get pvc
NAME        STATUS  VOLUME                                      CAPACITY  ACCESS MODES S  TORAGECLASS             AGE
openebspvc  Bound   pvc-9f70c0b4-efe9-4534-8748-95dba05a7327    110G      RWO             openebs-jiva-default    13m
```
## Using the OpenEBS NFS provisioner storage class to create dynamic NFS PVs
Let's perform the following steps to dynamically deploy an NFS PV protected by the OpenEBS storage provider:
1. List the storage classes, and confirm that openebs-nfs exists:
```
$ kubectl get sc
NAME                              PROVISIONER                   AGE
openebs-cstor-default (default)   openebs.io/provisioner-iscsi  14h
openebs-device openebs.io/local 15h
openebs-hostpath openebs.io/local 15h
openebs-jiva-default openebs.io/provisioner-iscsi 15h
openebs-nfs openebs.io/nfs 5s
openebs-snapshot-promoter volumesnapshot.externalstorage.
k8s.io/snapshot-promoter 15h
```
2. Now, you can use the openebs-nfs storage class to create PVCs for applications
that require ReadWriteMany access:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: openebs-nfs-pv-claim
spec:
storageClassName: "openebs-nfs"
accessModes:
- ReadWriteMany
resources:
requests:
storage: 1Mi
EOF

```
## See also
Rook NFS operator documentation: https://github.com/rook/rook/blob/master/Documentation/nfs. md
OpenEBS provisioning read-write-many PVCs: https://docs.openebs.io/ docs/next/rwm.html
