# Managing Kubernetes Volume Snapshots and restore
In this section, we will create Volume Snapshots from our persistent volumes in Kubernetes. By following this recipe, you will learn how to enable the Volume Snapshot functionality, create snapshot storage classes, and restore from existing Volume Snapshots.
# Getting ready
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest files under the chapter6 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter6
```
Make sure the **Container Storage Interface (CSI)** driver from your preferred storage vendor is installed on your Kubernetes cluster and has implemented the snapshot functionality. We covered the installation of the AWS EBS, GCP PD, Azure Disk, Rook, and
OpenEBS CSI drivers in Chapter 5 , **Preparing for Stateful Workloads**.  
The instructions in this section work similarly with other vendors that support snapshots via CSI. You can find these additional drivers on the Kubernetes CSI documentation site at: [https://kubernetes-csi.github.io/docs/drivers.html].

## How to do it...
This section is further divided into the following subsections to make this process easier:
*  Enabling feature gates
*  Creating a volume snapshot via CSI
*  Restoring a volume from a snapshot via CSI
*  Cloning a volume via CSI

## Enabling feature gates
Some of the features that will be discussed here may be at different stages (alpha, beta, or
GA) at the moment. If you run into an issue, perform the following step:  
1. Set the following feature-gates flags to true for both kube-apiserver
and kubelet :
```
- --feature-gates=VolumeSnapshotDataSource=true
- --feature-gates=KubeletPluginsWatcher=true
- --feature-gates=CSINodeInfo=true
- --feature-gates=CSIDriverRegistry=true
- --feature-gates=BlockVolume=true
- --feature-gates=CSIBlockVolume=true  
```
>  Steps 1. find the kube-apiserver pod.
```
$  kubectl -n kube-system get pods
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-659f4b66fd-5wrkf   1/1     Running   0          14h
calico-node-2zbrh                          1/1     Running   0          14h
calico-node-67xf7                          1/1     Running   0          14h
calico-node-69kkw                          1/1     Running   0          14h
calico-node-c4856                          1/1     Running   0          14h
calico-node-fckpm                          1/1     Running   0          14h
coredns-5d4dd4b4db-9m4zg                   1/1     Running   0          14h
coredns-5d4dd4b4db-jb2dx                   1/1     Running   0          14h
etcd-k8s-m1                                1/1     Running   0          14h
kube-apiserver-k8s-m1                      1/1     Running   0          14h
kube-controller-manager-k8s-m1             1/1     Running   0          14h
kube-proxy-9r2nm                           1/1     Running   0          14h
kube-proxy-9vnwq                           1/1     Running   0          14h
kube-proxy-bfr9x                           1/1     Running   0          14h
kube-proxy-fhh4t                           1/1     Running   0          14h
kube-proxy-pjhcr                           1/1     Running   0          14h
kube-scheduler-k8s-m1                      1/1     Running   0          14h
tiller-deploy-74494fcb9c-ml7jb             1/1     Running   0          80m
```
>  Steps 2. edit the kube-apiserver pod.
```
$ kubectl -n kube-system edit pod kube-apiserver-k8s-m1   
```
And then ,edit it.
```
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.0.30
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --feature-gates=VolumeSnapshotDataSource=true
    image: k8s.gcr.io/kube-apiserver:v1.17.3
```
You can find the latest statuses for features and their states by going to the Kubernetes
Feature Gates link in the See also section.
## Creating a volume snapshot via CSI
A volume snapshot is a copy of the state taken from a PVC in the Kubernetes cluster. It is a
useful resource for bringing up a stateful application using existing data. Let's perform the
following steps to create a volume snapshot using CSI:  
1. Create a PVC or select an existing one. In our recipe, we'll use the AWS EBS CSI
driver and the aws-csi-ebs storage class we created in Chapter 5 , Preparing for
Stateful Workloads, in the Installing an EBS CSI driver to manage EBS volumes recipe:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: csi-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: aws-csi-ebs
  resources:
    requests:
      storage: 4Gi
EOF
```  
  
2. Create a pod that will write to the /data/out.txt file inside the **PersistentVolume (PV)**:  
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out.txt;sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: csi-ebs-pvc
EOF
```
3. Create a VolumeSnapshotClass . Make sure that the snapshot provider is set to
your CSI driver name [https://kubernetes-csi.github.io/docs/drivers.html]. In this recipe, this is ebs.csi.aws.com :  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshotClass
metadata:
  name: csi-ebs-vsc
snapshotter: ebs.csi.aws.com
EOF
```  
4. A PVC must be created using the CSI driver of a storage vendor. In our recipe,
we will use the PVC we created in the Installing EBS CSI driver to manage EBS
volumes recipe. Now, create a VolumeSnapshot using the PVC name ( csi-ebs-
pvc ) we set in Step 1:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshot
metadata:
  name: ebs-volume-snapshot
spec:
  snapshotClassName: csi-ebs-vsc
  source:
    name: csi-ebs-pvc
    kind: PersistentVolumeClaim
EOF
```  
5. List the Volume Snapshots:
```
$ kubectl get volumesnapshot
NAME                  AGE
ebs-volume-snapshot   18s
```  
6. Validate that the status is Ready To Use: true when checking the output of
the following command:
```
$ kubectl describe volumesnapshot ebs-volume-snapshot
```

## Restoring a volume from a snapshot via CSI
We can create snapshots in an attempt to restore other snapshots. Let's perform the
following steps to restore the snapshot we created in the previous recipe:  
1. Restore the volume from the snapshot with a PVC using the following command.
As you can see, a new PVC named csi-ebs-pvc-restored will be created
based on the ebs-volume-snapshot snapshot:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-ebs-pvc-restored
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: aws-csi-ebs
  resources:
  requests:
  storage: 4Gi
  dataSource:
  name: ebs-volume-snapshot
  kind: VolumeSnapshot
  apiGroup: snapshot.storage.k8s.io
EOF
```
2. Create another pod that will continue to write to the /data/out.txt file inside
the PV. This step will ensure that the volume is still accessible after it's been
created:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: newapp
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt;sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
    volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: csi-ebs-pvc-restored
EOF
```
3. Confirm that the newapp pod contains the restored data and the timestamps
from the Creating a volume snapshot recipe:
```
$ kubectl exec -it newapp cat /data/out.txt
```
With this, you've learned how to provision persistent volumes from an existing snapshot.
This is a very useful step in a CI/CD pipeline so that you can save time troubleshooting
failed pipelines.  

## Cloning a volume via CSI
While snapshots are a copy of a certain state of PVs, it is not the only way to create a copy
of data. CSI also allows new volumes to be created from existing volumes. In this recipe, we
will create a PVC using an existing PVC by performing the following steps:  
1. Get the list of PVCs. You may have more than one PVC. In this example, we will
use the PVC we created in the Creating a volume snapshot recipe. You can use
another PVC as long as it has been created using the CSI driver that supports
VolumePVCDataSource APIs:  
```
$ kubectl get pvc
NAME        STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
csi-ebs-pvc Bound   pvc-574ed379-71e1-4548-b736-7137ab9cfd9d  4Gi       RWO           aws-csi-ebs   23h
```
2. Create a PVC using an existing PVC (in this recipe, this is csi-ebs-pvc ) as the
dataSource . The data source can be either a VolumeSnapshot or PVC. In this
example, we used PersistentVolumeClaim to clone the data:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clone-of-csi-ebs-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: csi-ebs-pvc
EOF
```
With that, you've learned a simple way of cloning persistent data from an existing data
source.

## How it works...
This recipe showed you how to create snapshots, bring data back from a snapshot, and how
to instantly clone persistent volumes on Kubernetes.   
In the Restoring a volume from a snapshot via CSI and Cloning a volume via CSI recipes, we
added a dataSource to our PVC that references an existing PVC so that a completely
independent, new PVC is created. The resulting PVC can be attached, cloned, snapshotted,
or deleted independently if the source is deleted. The main difference is that right after
provisioning the PVC, instead of an empty PV, the backend device provisions an exact
duplicate of the specified volume.  
It's important to note that native cloning support is available for dynamic provisioners
using CSI drivers that have already implemented this feature. The CSI project is continuing
to evolve and mature, so not every storage vendor provides full CSI capabilities.  

## See also
*  List of Kubernetes CSI drivers, at [https://kubernetes-csi.github.io/docs/drivers.html]
*  Container Storage Interface (CSI) documentation , at [https://kubernetes-csi.github.io]
*  The CSI spec, at [https://github.com/container-storage-interface/spec]
*  Kubernetes Feature Gates, at [https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/]
*  Kubernetes Volume Cloning documentation, at [https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/]
*  Kubernetes Volume Snapshots documentation, at [https://kubernetes.io/docs/concepts/storage/volume-snapshots/]
