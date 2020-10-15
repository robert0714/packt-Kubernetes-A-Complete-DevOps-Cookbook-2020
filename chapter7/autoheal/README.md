# Auto-healing pods in Kubernetes
Kubernetes has self-healing capabilities at the cluster level. It restarts containers that fail, 
reschedules pods when nodes die, and even kills containers that don't respond to your 
user-defined health checks.

In this section, we will perform application and cluster scaling tasks. You will learn how to 
use liveness and readiness probes to monitor container health and trigger a restart action in 
case of failures.
## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources.
## How to do it...
This section is further divided into the following subsections to make this process easier:
*  Testing self-healing pods
*  Adding liveness probes to pods
## Testing self-healing pods
In this recipe, we will manually remove pods in our deployment to show how Kubernetes 
replaces them. Later, we will learn how to automate this using a user-defined health check. 
Now, let's test Kubernetes' self-healing for destroyed pods:

1. Create a deployment or StatefulSet with two or more replicas. As an example, we 
will use the MinIO application we used in the previous chapter, in the 
Configuring and managing S3 object storage using MinIO recipe. This example has 
four replicas:
```
$ cd src/chapter7/autoheal/minio
$ kubectl apply -f minio.yaml
```
2. List the MinIO pods that were deployed as part of the StatefulSet. You will see four pods:
```
$ kubectl  get  pods |grep minio
minio-0                          1/1     Running   0          4h16m
minio-1                          1/1     Running   0          4h15m
minio-2                          1/1     Running   0          4h14m
minio-3                          1/1     Running   0          4h14m

```
3. Delete a pod to test Kubernetes' auto-healing functionality and immediately list the pods again. You will see that the terminated pod will be quickly rescheduled 
and deployed:
```
$ kubectl delete pod minio-0
pod "minio-0" deleted
$ $ kubectl get pods |grep minio   
minio-0                          1/1     Running   0          30s
minio-1                          1/1     Running   0          4h17m
minio-2                          1/1     Running   0          4h16m
minio-3                          1/1     Running   0          4h15m
```
With this, you have tested Kubernetes' self-healing after manually destroying a pod in 
operation. Now, we will learn how to add a health status check to pods to let Kubernetes 
automatically kill non-responsive pods so that they're restarted.

## Adding liveness probes to pods
Kubernetes uses liveness probes to find out when to restart a container. Liveness can be 
checked by running a liveness probe command inside the container and validating that it 
returns 0 through TCP socket liveness probes or by sending an HTTP request to a specified path. In that case, if the path returns a success code, then kubelet will consider the container 
to be healthy. In this recipe, we will learn how to send an HTTP request method to the 
example application. Let's perform the following steps to add liveness probes:

1. Edit the ***minio.yaml*** file in the ***src/chapter7/autoheal/minio*** directory and 
add the following ***livenessProbe*** section right under the ***volumeMounts*** 
section, before ***volumeClaimTemplates*** . Your YAML manifest should look 
similar to the following. This will send an HTTP request to 
the ***/minio/health/live*** location every ***20*** seconds to validate its health:
```
...
    volumeMounts:
    - name: data
      mountPath: /data
#### Starts here
    livenessProbe:
      httpGet:
        path: /minio/health/live
        port: 9000
      initialDelaySeconds: 120
      periodSeconds: 20
#### Ends here
  # These are converted to volume claims by the controller
  # and mounted at the paths mentioned above.
  volumeClaimTemplates:
```
For liveness probes that use HTTP requests to work, an application needs to 
expose unauthenticated health check endpoints. In our example, MinIO 
provides this through the ***/minio/health/live*** endpoint. If your workload 
doesn't have a similar endpoint, you may want to use liveness commands 
inside your pods to verify their health.
2. Deploy the application. It will create four pods:
```
$ kubectl apply -f minio.yaml
```
3. Confirm the liveness probe by describing one of the pods. You will see a 
Liveness description similar to the following:
```
$ kubectl describe pod minio-0
...
    Liveness: http-get http://:9000/minio/health/live delay=120s timeout=1s period=20s #success=1 #failure=3
...
```
4. To test the liveness probe, we need to edit the ***minio.yaml*** file again. This time,
set the ***livenessProbe*** port to ***8000*** , which is where the application will not
able to respond to the HTTP request. Repeat Steps 2 and 3, redeploy the
application, and check the events in the pod description. You will see a ***minio failed liveness probe, will be restarted*** message in the events:
```
$ kubectl describe pod minio-0
```

