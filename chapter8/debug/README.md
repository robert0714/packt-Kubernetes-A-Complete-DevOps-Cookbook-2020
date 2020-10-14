# Inspecting containers
In this section, we will troubleshoot problems related to pods stuck in Pending,
ImagePullBackOff, or CrashLoopBackOff states. You will learn how to inspect and debug
pods that are having deployment problems in Kubernetes.
## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the
cluster resources.

## How to do it...
This section is further divided into the following subsections to make the process easier:
* Inspecting pods in Pending status
* Inspecting pods in ImagePullBackOff status
* Inspecting pods in CrashLoopBackOff status

## Inspecting pods in Pending status
When you deploy applications on Kubernetes, it is inevitable that soon or later you will
need to get more information on your application. In this recipe, we will learn to inspect
common pods problem of pods stuck in Pending status:  
1. In the */src/chapter8* folder, inspect the content of the *mongo-sc.yaml* file and
deploy it running the following command. The deployment manifest includes
MongoDB Statefulset with three replicas, Service and will get stuck in Pending
state due mistake with a parameter and we will inspect it to find the source:   
```
$ cat debug/mongo-sc.yaml
$ kubectl apply -f debug/mongo-sc.yaml
```
2. List the pods by running the following command. You will notice that the status
is **Pending** for the **mongo-0** pod:
```
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mongo-0   0/2     Pending   0          9s
```
3. Get additional information on the pods using the ***kubectl describe pod***
command and look for the ***Events*** section. In this case, ***Warning*** is pointing to
an unbound ***PersistentVolumeClaim*** :
```
$ kubectl describe pod mongo-0
...(ommit)
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  8s (x3 over 95s)  default-scheduler  pod has unbound immediate PersistentVolumeClaims (repeated 4 times)
```
4. Now that we know that we need to look at the PVC status, thanks to the results
of the previous step, let's get the list of PVCs in order to inspect the issue. You
will see that PVCs are also stuck in the Pending state:  
```
$ kubectl get pvc
NAME                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-pvc-mongo-0   Pending                                      storageclass   2m28s
```
5. Get additional information on the PVCs using the ***kubectl describe pvc***
command, and look where the events are described. In this case, ***Warning*** is
pointing to a missing storage class named ***storageclass*** :
```
...(ommit)
Events:
  Type     Reason              Age                  From                         Message
  ----     ------              ----                 ----                         -------
  Warning  ProvisioningFailed  5s (x17 over 3m51s)  persistentvolume-controller  storageclass.storage.k8s.io "storageclass" not found
```
6. List the storage classes. You will notice that you don't have the storage class
named storageclass :
```
$ kubectl get sc
NAME                             PROVISIONER                                                AGE
openebs-device                   openebs.io/local                                           3h48m
openebs-hostpath                 openebs.io/local                                           3h48m
openebs-jiva-default (default)   openebs.io/provisioner-iscsi                               3h48m
openebs-snapshot-promoter        volumesnapshot.external-storage.k8s.io/snapshot-promoter   3h48m
```
7. Now we know that the manifest file we applied in step 1 used a storage class that
does not exist. In this case, you can either create the missing storage class or edit
the manifest to include an existing storage class to fix the issue.
Let's create the missing storage class from an existing default storage class like
shown in the example below gp2 :
```
$ kubectl create -f sc-gp2.yaml
```
8. List the pods by running the following command. You will notice that status is
now Running for all pods that were previously Pending in step 2:
```
$ kubectl get pods
NAME    READY   STATUS    RESTARTS    AGE
mongo-0 2/2     Running   0           2m18s
mongo-1 2/2     Running   0           88s
mongo-2 2/2     Running   0           50s
```
You have successfully learned how to inspect why a pod is pending and fix it.
