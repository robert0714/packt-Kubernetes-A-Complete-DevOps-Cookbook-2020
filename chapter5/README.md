# Troubleshooting storage issues
In this section, you will learn how to solve the most common storage issues associated with Kubernetes. After following the recipes in this chapter, you will gain the basic skills required to troubleshoot persistent volumes stuck in pending or termination states.
## Getting ready
Make sure that you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.
## How to do it…
This section is sub-divided further into the following subsections to facilitate the process:
*  Persistent volumes in the pending state
*  A PV is stuck once a PVC has been deleted
## Persistent volumes in the pending state
You have deployed an application, but both pods and persistent volume claims are stuck in the pending state, similar to the following:
```
$ kubectl get pvc
NAME            STATUS  VOLUME CAPACITY ACCESS MODES STORAGECLASS   AGE
mysql-pv-claim  Pending rook-ceph-block                             28s
```
Let's perform the following steps to start troubleshooting:  
1. First, describe the PVC to understand the root cause:  
```
$ kubectl describe pvc mysql-pv-claim
...
Events:
Type Reason Age From Message
---- ------ ---- ---- -------
Warning ProvisioningFailed 3s (x16 over 3m42s) persistentvolumecontroller
storageclass.storage.k8s.io "rook-ceph-block" not found
```  
2. A PVC is stuck due to an incorrect or non-existing storage class. We need to change the storage class with a valid resource. List the storage classes as follows:
```
$ kubectl get sc
NAME                            PROVISIONER                                                 AGE
default                         kubernetes.io/aws-ebs                                       102m
gp2                             kubernetes.io/aws-ebs                                       102m
openebs-cstor-default (default) openebs.io/provisioner-iscsi                                77m
openebs-device                  openebs.io/local                                            97m
openebs-hostpath                openebs.io/local                                            97m
openebs-jiva-default            openebs.io/provisioner-iscsi                                97m
openebs-snapshot-promoter       volumesnapshot.externalstorage.k8s.io/snapshot-promoter     97m
```
3. Delete the deployment using kubectl delete -f <deployment.yaml>.  
4. Edit the deployment and replace the storageClassName field with a valid  storage class from the output of the previous step, in our case, openebs-cstordefault.  
5. Redeploy the application using kubectl apply -f <deployment.yaml>.   
6. Confirm that the PVC status is Bound:  
```
$ kubectl get pvc
NAME            STATUS    VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS          AGE
mysql-pv-claim  Bound     pvc-bbf2b01e-2a69-4c4c-b9c2-48921959c363  20Gi      RWO           openebs-cstor-default 5s
```   
Now you have successfully troubleshooted PVC issues caused by a missing StorageClass resource.
##  A PV is stuck once a PVC has been deleted
You have deleted a PVC. However, either the PVC or PV deletion is stuck in the terminating state, similar to the following:
```
$ kubectl get pv
NAME                                      CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS CLAIM      STORAGECLASS            REASON  AGE
pvc-bbf2b01e-2a69-4c4c-b9c2-48921959c363  20Gi      RWO           Delete          Terminating       default/mysql-pv-claim  gp2     7m45s
```  
Edit the stuck PVs or PVCs in the terminating state:
```
$ kubectl edit pv <PV_Name>
```
Remove finalizers similar to - kubernetes.io/pv-protection, and save the changes.
