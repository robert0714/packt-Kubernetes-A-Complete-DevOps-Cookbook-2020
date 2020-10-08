# Configuring and managing persistent storage using OpenEBS
OpenEBS is a popular open source, cloud-native storage (CNS) project with a large community. In this section, we will install an OpenEBS persistent storage provider. You
will learn how to create volumes using different types of storage engine options for stateful workloads on Kubernetes.

## Getting ready
For this recipe, we need to have helm and kubectl installed. Make sure you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.

## How to do it…
This section is sub-divided further into the following subsections to facilitate the process:
*  Installing iSCSI client prerequisites
*  Installing OpenEBS
*  Using ephemeral storage to create persistent volumes
*  Creating storage pools
*  Creating OpenEBS storage classes
*  Using an OpenEBS storage class to create dynamic PVs
## Installing iSCSI client prerequisites
The OpenEBS storage provider requires that the iSCSI client runs on all worker nodes:  
1. On all your worker nodes, follow the steps to install and enable open-iscsi:
```
## ubuntu
$ sudo apt-get update && sudo apt-get install open-iscsi && sudo service open-iscsi restart

## centos
$ sudo yum install iscsi-initiator-utils -y && sudo  sudo systemctl enable --now iscsid
```  
reference: [https://docs.openebs.io/docs/next/quickstart.html]  
2. Validate that the iSCSI service is running:  
```
$ systemctl status iscsid
● iscsid.service - iSCSI initiator daemon (iscsid)
Loaded: loaded (/lib/systemd/system/iscsid.service; enabled; vendor
preset: enabled)
Active: active (running) since Sun 2019-09-08 07:40:43 UTC; 7s ago
Docs: man:iscsid(8)
```  
3. If the service status is showing as inactive, then enable and start the iscsid service:
```
$ sudo systemctl enable iscsid && sudo systemctl start iscsid
```  
After installing the iSCSI service, you are ready to install OpenEBS on your cluster.

## Installing OpenEBS
Let's perform the following steps to quickly get the OpenEBS control plane installed:  
1. Install OpenEBS services by using the operator:  
```
$ kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```  
2. Confirm that all OpenEBS pods are running:  
```
$ kubectl get pods --namespace openebs
NAME                                        READY STATUS  RESTARTS  AGE
maya-apiserver-dcbc87f7f-k99fz              0/1   Running 0         88s
openebs-admission-server-585c6588d-j29ng    1/1   Running 0         88s
openebs-localpv-provisioner-cfbd49877-jzjxl 1/1   Running 0         87s
openebs-ndm-fcss7                           1/1   Running 0         88s
openebs-ndm-m4qm5                           1/1   Running 0         88s
openebs-ndm-operator-bc76c6ddc-4kvxp        1/1   Running 0         88s
openebs-ndm-vt76c                           1/1   Running 0         88s
openebs-provisioner-57bbbd888d-jb94v        1/1   Running 0         88s
openebs-snapshot-operator-7dd598c655-2ck74  2/2   Running 0         88s
```  
OpenEBS consists of the core components listed here. Node Disk Manager (NDM) is one of the important pieces of OpenEBS that is responsible for detecting disk changes and runs as DaemonSet on your worker nodes.
## Using ephemeral storage to create persistent volumes 
OpenEBS currently provides three storage engine options (Jiva, cStor, and LocalPV). The first storage engine option, Jiva, can create replicated storage on top of the ephemeral storage. Let's perform the following steps to get storage using ephemeral storage configured:  
1. List the default storage classes:
```
$ kubectl get sc
NAME                      PROVISIONER                                             AGE
openebs-device            openebs.io/local                                        25m
openebs-hostpath          openebs.io/local                                        25m
openebs-jiva-default      openebs.io/provisioner-iscsi                            25m
openebs-snapshot-promoter volumesnapshot.externalstorage.k8s.io/snapshot-promoter 25m
```  
2. Describe the openebs-jiva-default storage class:
```
$ kubectl describe sc openebs-jiva-default
Name: openebs-jiva-default
IsDefaultClass: No
Annotations: cas.openebs.io/config=- name: ReplicaCount
  value: "3"
- name: StoragePool
  value: default
```
3. Create a persistent volume claim using openebs-jiva-default:  
```
$ cat <<EOF | kubectl apply -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: demo-vol1-claim
spec:
  storageClassName: openebs-jiva-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G
EOF
```  
4. Confirm that the PVC status is BOUND:
```
$ kubectl get pvc
NAME            STATUS  VOLUME                                    CAPACITY  ACCESS  MODES STORAGECLASS      AGE
demo-vol1-claim Bound   pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5  5G        RWO     openebs-jiva-default    4s
```  
5. Now, use the PVC to dynamically provision a persistent volume:  
```
$ kubectl apply -f https://raw.githubusercontent.com/openebs/openebs/master/k8s/demo/percona/percona-openebs-deployment.yaml
```  
6. Now list the pods and make sure that your workload, OpenEBS controller, and replicas are all in the running state:  
```
$ kubectl get pods
NAME                                                          READY   STATUS  RESTARTS  AGE
percona-767db88d9d-2s8np                                      1/1     Running 0         75s
pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-ctrl-54d7fd794-s8svt 2/2     Running 0         2m23s
pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-2b9q4 1/1     Running 1         2m18s
pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-nkbfq 1/1     Running 0         2m18s
pvc-cb7485bc-6d45-4814-adb1-e483c0ebbeb5-rep-647458f56f-x7s9b 1/1     Running 0         2m18s
```  
Now you know how to get highly available, cloud-native storage configured for your stateful applications on Kubernetes.
## Creating storage pools
In this recipe, we will use raw block devices attached to your nodes to create a storage pool.These devices can be AWS EBS volumes, GCP PDs, Azure Disk, virtual disks, or vSAN volumes. Devices can be attached to your worker node VMs, or basically physical disks if you are using a bare-metal Kubernetes cluster. Let's perform the following steps to create a storage pool out of raw block devices:  
1. List unused and unclaimed block devices on your nodes:  
```
$ kubectl get blockdevices -n openebs
NAME                                            NODENAME                                    SIZE            CLAIMSTATE  STATUS  AGE
blockdevice-24d9b7652893384a36d0cc34a804c60c    ip-172-23-1-176.uswest-2.compute.internal   107374182400    Unclaimed   Active  52s
blockdevice-8ef1fd7e30cf0667476dba97975d5ac9    ip-172-23-1-25.uswest-2.compute.internal    107374182400    Unclaimed   Active  51s
blockdevice-94e7c768ef098a74f3e2c7fed6d82a5f    ip-172-23-1-253.uswest-2.compute.internal   107374182400    Unclaimed   Active  52s
```
In our example, we have a three-node Kubernetes cluster on AWS EC2 with one additional EBS volume attached to each node.  
2. Create a storage pool using the unclaimed devices from Step 1:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk-pool
  annotations:
    cas.openebs.io/config: |
      - name: PoolResourceRequests
      value: |-
          memory: 2Gi
      - name: PoolResourceLimits
          value: |-
              memory: 4Gi
spec:
  name: cstor-disk-pool
  type: disk
  poolSpec:
    poolType: striped
  blockDevices:
    blockDeviceList:
    - blockdevice-24d9b7652893384a36d0cc34a804c60c
    - blockdevice-8ef1fd7e30cf0667476dba97975d5ac9
    - blockdevice-94e7c768ef098a74f3e2c7fed6d82a5f
EOF
```
3. List the storage pool claims:  
```
$ kubectl get spc
NAME              AGE
cstor-disk-pool   29s
```  
4. Verify that a cStor pool has been created and that its status is Healthy:  
```
$ kubectl get csp
NAME                  ALLOCATED   FREE    CAPACITY  STATUS  TYPE    AGE
cstor-disk-pool-8fnp  270K        99.5G   99.5G     Healthy striped 3m9s
cstor-disk-pool-nsy6  270K        99.5G   99.5G     Healthy striped 3m9s
cstor-disk-pool-v6ue  270K        99.5G   99.5G     Healthy striped 3m10s
```
5. Now we can use the storage pool in storage classes to provision dynamic volumes.
## Creating OpenEBS storage classes
Let's perform the following steps to create a new storage class to consume StoragePool,which we created previously in the Creating storage pools recipe:  
1. Create an OpenEBS cStor storage class using the cStor StoragePoolClaim name, cstor-disk-pool, with three replicas:  
```
$ cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
name: openebs-cstor-default
annotations:
  openebs.io/cas-type: cstor
  cas.openebs.io/config: |
    - name: StoragePoolClaim
      value: "cstor-disk-pool"
    - name: ReplicaCount
      value: "3"
provisioner: openebs.io/provisioner-iscsi
EOF
```
2. List the storage classes:  
```
$ kubectl get sc
NAME                      PROVISIONER                                             AGE
default                   kubernetes.io/aws-ebs                                   25m
gp2 (default)             kubernetes.io/aws-ebs                                   25m
openebs-cstor-default     openebs.io/provisioner-iscsi                            6s
openebs-device            openebs.io/local                                        20m
openebs-hostpath          openebs.io/local                                        20m
openebs-jiva-default      openebs.io/provisioner-iscsi                            20m
openebs-snapshot-promoter volumesnapshot.externalstorage.k8s.io/snapshot-promoter 20m
ubun
```
3. Set the gp2 AWS EBS storage class as the non-default option:  
```
$ kubectl patch storageclass gp2 -p '{"metadata":{"annotations":{"storageclass.beta.kubernetes.io/is-defaultclass":"false"}}}'
```
4. Define openebs-cstor-default as the default storage class:  
```
$ kubectl patch storageclass openebs-cstor-default -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-defaultclass":"true"}}}'
```  
Make sure that the previous storage class is no longer set as the default and that you only have one default storage class.

# Using an OpenEBS storage class to create dynamic PVs
Let's perform the following steps to deploy dynamically created persistent volumes using the OpenEBS storage provider:  
1. Clone the examples repository:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter5/openebs/
```  
2. Review minio.yaml and note that PVCs are using the openebs-stordefault storage class.  
3. Deploy Minio:  
```
$ kubectl apply -f minio.yaml
deployment.apps/minio-deployment created
persistentvolumeclaim/minio-pv-claim created
service/minio-service created
```
4. Get the Minio service load balancer's external IP:  
```
$ kubectl get service
NAME            TYPE            CLUSTER-IP  EXTERNAL-IP                                                             PORT(S)         AGE
kubernetes      ClusterIP       10.3.0.1    <none>                                                                  443/TCP         54m
minio-service   LoadBalancer    10.3.0.29   adb3bdaa893984515b9527ca8f2f8ca6-1957771474.us-west-2.elb.amazonaws.com 9000:32701/TCP  3s
```
5. Add port 9000 to the end of the address and open the external IP of the Minio service in your browser:  
6. Use the username minio, and the password minio123 to log in to the Minio deployment backed by persistent OpenEBS volumes:
You have now successfully deployed a stateful application that is deployed on the OpenEBS cStor storage engine.
## How it works...
This recipe showed you how to quickly provision a persistent storage provider using OpenEBS.
In the Using ephemeral storage to create persistent volumes recipe, in Step 6, when we deployed a workload using the openebs-jiva-default storage class, OpenEBS launched OpenEBS volumes with three replicas. 
To set one replica, as is the case with a single-node Kubernetes cluster, you can create a new storage class (similar to the one we created in the Creating OpenEBS storage class recipe) and set the ReplicaCount variable value to 1:
```
apiVersion: openebs.io/v1alpha1
kind: StoragePool
metadata:
  name: my-pool
  type: hostdir
spec:
  path: "/my/openebs/folder"
```
When ephemeral storage is used, the OpenEBS Jiva storage engine uses the /var/openebs directory on every available node to create replica sparse files. If you would like to change the default or create a new StoragePool resource, you can create a new storage pool and set a custom path.

## See also
* OpenEBS documentation: https://docs.openebs.io/
* Beyond the basics: OpenEBS workshop: https://github.com/openebs/community/tree/master/workshop
* OpenEBS Community Slack channel: https://openebs.io/join-our-slackcommunity
* OpenEBS enterprise platform: https://mayadata.io/product
* OpenEBS director for managing stateful workloads: https://account.mayadata.io/login
