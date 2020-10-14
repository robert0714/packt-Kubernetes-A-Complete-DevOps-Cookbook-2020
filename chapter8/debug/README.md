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

## Inspecting pods in ImagePullBackOff status
Sometimes your manifest files may have a typo in the image name, or the image location
may have changed. As a result, when you deploy the application, the container image will
not be found and the deployment will get stuck. In this recipe, we will learn how to inspect
the common problem of pods becoming stuck in ImagePullBackOff status:  
1. In the ***/src/chapter8*** folder, inspect the contents of the ***mongo-image.yaml*** file and deploy it by running the following command. The deployment manifest includes MongoDB Statefulset with three replicas, Service and will get stuck in ImagePullBackOff state due to typo in the container image name and we will inspect it to find the source: 
```
$ cat debug/mongo-image.yaml
$ kubectl apply -f debug/mongo-image.yaml
```
2. List the pods by running the following command. You will notice that the status of the mongo-0 pod is ***ImagePullBackOff*** :
```
$ kubectl get pods
NAME      READY   STATUS              RESTARTS   AGE
mongo-0   0/2     ImagePullBackOff    0          29s
```
3. Get additional information on the pods using the kubectl describe pod command and look for the Events section. In this case, Warning is pointing to a failure to pull the mongi image:
```
$ kubectl describe pod mongo-0
...
Events:
Type    Reason  Age               From                                    Message
----    ------  ----              ----                                    -------
Warning Failed  25s (x3 over 68s) kubelet,ip-172-20-32-169.ec2.internal   Error: ErrImagePull
Warning Failed  25s (x3 over 68s) kubelet,ip-172-20-32-169.ec2.internal   Failed to pull image "mongi": rpc error: code = Unknown desc = Error response from daemon: pullaccess denied for mongi, repository does not exist or may require'docker login'
Normal  Pulling 25s (x3 over 68s) kubelet,ip-172-20-32-169.ec2.internal   Pulling image "mongi"
Normal BackOff 14s (x4 over 67s)  kubelet,ip-172-20-32-169.ec2.internal   Back-off pulling image "mongi"
Warning Failed 14s (x4 over 67s)  kubelet,ip-172-20-32-169.ec2.internal   Error: ImagePullBackOff
```
4. Now we know that we need to confirm the container image name. The correct  name is supposed to be mongo . Let's edit the manifest file, mongo-image.yaml ,and change the image name to mongo as follows:
```
...
spec:
  terminationGracePeriodSeconds: 10
  containers:
  - name: mongo
  image: mongo
  command:
...
```
5. Delete and redeploy the resource by running the following commands:
```
$ kubectl delete -f mongo-image.yaml
$ kubectl apply -f mongo-image.yaml
```  
6. List the pods by running the following command. You will notice that the status
is now Running for all pods that were previously in ImagePullBackOff status
in step 2:

```
$ kubectl get  pods
NAME    READY   STATUS      RESTARTS    AGE
mongo-0 2/2     Running     0           4m55s
mongo-1 2/2     Running     0           4m55s
mongo-2 2/2     Running     0           4m55s
```
You have successfully learned to inspect a pod with a status of ImagePullBackOff and troubleshoot it.

## Inspecting pods in CrashLoopBackOff status
Inspecting pods in ***CrashLoopBackOff*** status is fundamentally similar to inspecting  Pending pods, but might also require a bit more knowledge of the container workload you  are creating. ***CrashLoopBackOff*** occurs when the application inside the container keeps  crashing, the parameters of the pod are configured incorrectly, a liveness probe failed, or an  error occurred when deploying on Kubernetes.  
In this recipe, we will learn how to inspect the common problem of pods becoming stuck in  ***CrashLoopBackOff*** status:  
1. In the ***/src/chapter8*** folder, inspect the contents of the ***mongo-config.yaml*** file and deploy it running the following command. The deployment manifest includes a MongoDB statefulset with three replicas, Service and will get stuck in CrashLoopBackOff state due mistake with a missing configuration file and we will inspect it to find the source:  
```
$ cat debug/mongo-config.yaml
$ kubectl apply -f debug/mongo-config.yaml
```
2. List the pods by running the following command. You will notice that the status is CrashLoopBackOff or Error for the mongo-0 pod:
```
$ kubectl get pods
NAME    READY STATUS            RESTARTS AGE
mongo-0 1/2   CrashLoopBackOff  3        58s
```
3. Get additional information on the pods using the kubectl describe pod command and look for the Events section. In this case, the Warning shows that the container has restarted, but it is not pointing to any useful information:  
```
$ kubectl describe pod mongo-0
...
Events:
Type      Reason    Age               From                                  Message
----      ------    ----              ----                                  -------   ...
Normal    Pulled    44s (x4 over 89s) kubelet,ip-172-20-32-169.ec2.internal Successfully pulled image "mongo"
Warning   BackOff   43s (x5 over 87s) kubelet,ip-172-20-32-169.ec2.internal Back-off restarting failed container
```
4. When Events from the pods are not useful, you can use the kubectl logs 
command to get additional information from the pod. Check the messages in the 
pod's logs using the following command. The log message is pointing to a 
missing file; further inspection of the manifest is needed:  
```
$ kubectl logs mongo-0 mongo
/bin/sh: 1: cannot open : No such file
```
5. Inspect and have a closer look at the application manifest file, ***mongo-config.yaml*** , and you will see that the environmental variable ***MYFILE*** is missing in this case:   
```
...
  spec:
    terminationGracePeriodSeconds: 10
    containers:
      - name: mongo
        image: mongo
        command: ["/bin/sh"]
        args: ["-c", "sed \"s/foo/bar/\" < $MYFILE"]
...
```
6. To fix this issue, you can add a ***ConfigMap*** to your deployment. Edit the ***mongo-config.yaml*** file and add the missing file by adding the MYFILE parameter with a ***ConfigMap*** resource to the beginning of the file similar to following:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-env
data:
  MYFILE: "/etc/profile"
EOF
```
7. Delete and redeploy the resource by running the following commands:
```
$ kubectl delete -f mongo-image.yaml
$ kubectl apply -f mongo-image.yaml
```
8. List the pods by running the following command. You will notice that the status is now Running for all pods that were previously in CrashLoopBackOff status
in step 2.  
```
$ kubectl get pods
NAME    READY   STATUS    RESTARTS    AGE
mongo-0 2/2     Running   0           4m15s
mongo-1 2/2     Running   0           4m15s
mongo-2 2/2     Running   0           4m15s
```
You have successfully learned how to inspect a pod's CrashLoopBackOff issue and fix it.
## See also
*  Debugging init containers: https://kubernetes.io/docs/tasks/debug-application-cluster/debug-init-containers/
*  Debugging pods and ReplicationControllers https://kubernetes.io/docs/tasks/debug-application-cluster/debug-pod-replication-controller/
*  Debugging a statefulset: https://kubernetes.io/docs/tasks/debug-application-cluster/debug-stateful-set/
*  Determining the reason for pod failure: https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/
*  Squash, a debugger for microservices: https://github.com/solo-io/squash
