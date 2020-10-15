# Scaling and Upgrading Applications
TERNAIn this chapter, we will discuss the methods and strategies that we can use to 
dynamically scale containerized services running on Kubernetes to handle the changing 
traffic needs of our service. After following the recipes in this chapter, you will have the 
skills needed to create load balancers to distribute traffic to multiple workers and increase 
bandwidth. You will also know how to handle upgrades in production with minimum 
downtime.
In this chapter, we will cover the following recipes:

* Scaling applications on Kubernetes
* Assigning applications to nodes with priority
* Creating an external load balancer
* Creating an ingress service and service mesh using Istio
* Creating an ingress service and service mesh using Linkerd
* Auto-healing pods in Kubernetes
* Managing upgrades through blue/green deployments
## Scaling applications on Kubernetes
In this section, we will perform application and cluster scaling tasks. You will learn how to 
manually and also automatically scale your service capacity up or down in Kubernetes to 
support dynamic traffic.
## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest 
files in the chapter7 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd /src/chapter7/
```
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources.

## How to do it...
This section is further divided into the following subsections to make this process easier: 
*  Validating the installation of Metrics Server
*  Manually scaling an application
*  Autoscaling applications using Horizontal Pod Autoscaler
## Validating the installation of Metrics Server
The ***Autoscaling applications using the Horizontal Pod Autoscaler*** recipe in this section also 
requires Metrics Server to be installed on your cluster. Metrics Server is a cluster-wide 
aggregator for core resource usage data. Follow these steps to validate the installation of 
Metrics Server:

1. Confirm if you need to install Metrics Server by running the following command:
```
$ kubectl top node
error: metrics not available yet
```
2. If it's been installed correctly, you should see the following node metrics:
```
$ kubectl top nodes
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-m1   205m         10%    2277Mi          61%       
k8s-n1   140m         7%     735Mi           19%       
k8s-n2   136m         6%     936Mi           25%       
k8s-n3   129m         6%     633Mi           17%       
k8s-n4   100m         5%     626Mi           16%
```
If you get an error message stating ****metrics not available yet**** , then you need to 
follow the steps provided in the next chapter in the Adding metrics using the Kubernetes 
Metrics Server recipe to install Metrics Server.

## Manually scaling an application
When the usage of your application increases, it becomes necessary to scale the application 
up. Kubernetes is built to handle the orchestration of high-scale workloads. 
Let's perform the following steps to understand how to manually scale an application:  
1. Change directories to /src/chapter7/charts/node , which is where the local 
clone of the example repository that you created in the Getting ready section can 
be found:
```
$ cd /charts/node/
```
2. Install the To-Do application example using the following command. This Helm chart will deploy two pods, including a Node.js service and a MongoDB service:
```
$ helm install . --name my-ch7-app
```
3. Get the service IP of my-ch7-app-node to connect to the application. The following command will return an external address for the application:
```
$ export SERVICE_IP=$(kubectl get svc --namespace default my-ch7-app-node --template "{{range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
$ echo http://$SERVICE_IP/
http://mytodoapp.us-east-1.elb.amazonaws.com/
```
4. Open the address from ***Step 3*** in a web browser. You will get a fully functional To-Do application:

5. Check the status of the application using helm status . You will see the number 
of pods that have been deployed as part of the deployment in the Available 
column:
```
$ helm status my-ch7-app
LAST DEPLOYED: Thu Oct 15 09:22:50 2020
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                READY  UP-TO-DATE  AVAILABLE  AGE
my-ch7-app-mongodb  1/1    1           1          18m
my-ch7-app-node     1/1    1           1          18m
...(ommit)
```
6. Scale the node pod to 3 replicas from the current scale of a single replica:
```
$ kubectl scale --replicas 3 deployment/my-ch7-app-node 
deployment.extensions/my-ch7-app-node scaled
```
7. Check the status of the application again and confirm that, this time, the number 
of available replicas is ***3*** and that the number of my-ch7-app-node pods in 
the ***v1/Pod*** section has increased to ***3*** :
```
$ helm status my-ch7-app
...(ommit)
RESOURCES:
==> v1/Deployment
NAME                READY  UP-TO-DATE  AVAILABLE  AGE
my-ch7-app-mongodb  1/1    1           1          21m
my-ch7-app-node     1/3    3           1          21m

...(ommit)
==> v1/Pod(related)
NAME                                 READY  STATUS    RESTARTS  AGE
my-ch7-app-mongodb-56b5ff86dc-rct7t  1/1    Running   0         21m
my-ch7-app-node-849fb8745d-c49c5     1/1    Running   0         21m
my-ch7-app-node-849fb8745d-gc2jf     0/1    Init:1/2  0         77s
my-ch7-app-node-849fb8745d-hnl7g     0/1    Init:1/2  0         77s
...(ommit)
```
8. To scale down your application, repeat ***Step 5***, but this time with ***2*** replicas:
```
$ kubectl scale --replicas 2 deployment/my-ch7-app-node
deployment.extensions/my-ch7-app-node scaled
```
With that, you've learned how to scale your application when needed. Of course, your Kubernetes cluster resources should be able to support growing workload capacities as 
well. You will use this knowledge to test the service healing functionality in the ***Auto-healing pods in Kubernetes*** recipe.

The next recipe will show you how to autoscale workloads based on actual resource consumption instead of manual steps.

## Autoscaling applications using a Horizontal Pod Autoscaler
In this recipe, you will learn how to create a ***Horizontal Pod Autoscaler (HPA)*** to automate 
the process of scaling the application we created in the previous recipe. We will also test the 
HPA with a load generator to simulate a scenario of increased traffic hitting our services.  
Follow these steps:  
1. First, make sure you have the sample To-Do application deployed from 
the Manually scaling an application recipe. When you run the following command, 
you should get both MongoDB and Node pods listed:
```
$ kubectl get pods | grep my-ch7-app
my-ch7-app-mongodb-56b5ff86dc-rct7t   1/1     Running   0          26m
my-ch7-app-node-849fb8745d-c49c5      1/1     Running   0          26m
my-ch7-app-node-849fb8745d-hnl7g      1/1     Running   0          5m45s
```
2. Create an HPA declaratively using the following command. This will automate 
the process of scaling the application between ***1*** to ***5*** replicas when 
the ***targetCPUUtilizationPercentage*** threshold is reached. In our example, 
the mean of the pods' CPU utilization target is set to ***50*** percent usage. When the 
utilization goes over this threshold, your replicas will be increased:
```
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-ch7-app-autoscaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-ch7-app-node
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
EOF
```

Although the results may be the same most of the time, a declarative 
configuration requires an understanding of the Kubernetes object configuration 
specs and file format. As an alternative, ***kubectl*** can be used for the imperative 
management of Kubernetes objects.

ps.  
Note that you must have a request set in your deployment to use 
autoscaling. If you do not have a request for CPU in your deployment, the 
HPA will deploy but will not work correctly.
You can also create the same ***HorizontalPodAutoscaler*** imperatively
by running the ***$ kubectl autoscale deployment my-ch7-app-node --cpu-percent=50 --min=1 --max=5**** command.  
```
$ kubectl delete -n default horizontalpodautoscaler my-ch7-app-node
```

3. Confirm the number of current replicas and the status of the HPA. When you run 
the following command, the number of replicas should be ***1*** :
```
$  kubectl get hpa
NAME                    REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
my-ch7-app-autoscaler   Deployment/my-ch7-app-node   0%/50%   1         5         2          3m45s

```

4. Get the service IP of my-ch7-app-node so that you can use it in the next step:
```
$ export SERVICE_IP=$(kubectl get svc --namespace default my-ch7-app-node --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
$ echo http://$SERVICE_IP/
http://mytodoapp.us-east-1.elb.amazonaws.com/
```

5. Start a new Terminal window and create a load generator to test the HPA. Make 
sure that you replace ***YOUR_SERVICE_IP*** in the following code with the actual 
service IP from the output of ***Step 4***. This command will generate traffic to your 
To-Do application:  
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh 
while true; do wget -q -O- YOUR_SERVICE_IP; done
```
You will see the below: 
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh                                             
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
/ #  while true; do wget -q -O- mytodoapp.us-east-1.elb.amazonaws.com; done

```
6. Wait a few minutes for the Autoscaler to respond to increasing traffic. While the 
load generator is running on one Terminal, run the following command on a 
separate Terminal window to monitor the increased CPU utilization. In our 
example, this is set to 210% :

```
$  kubectl get hpa
NAME                    REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
my-ch7-app-autoscaler   Deployment/my-ch7-app-node   210%/50%         1         5         2          23m45s
```
7. Now, check the deployment size and confirm that the deployment has been  
resized to 5 replicas as a result of the increased workload:
```
$ kubectl get deployment my-ch7-app-node
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
my-ch7-app-node   5/5     5            5           5h23m
```
8. On the Terminal screen where you run the load generator, press Ctrl + C to  
terminate the load generator. This will stop the traffic coming to your  
application.
9. Wait a few minutes for the Autoscaler to adjust and then verify the HPA status  
by running the following command. The current CPU utilization should be  
lower. In our example, it shows that it went down to 0% :  
```
$  kubectl get hpa
NAME                    REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
my-ch7-app-autoscaler   Deployment/my-ch7-app-node   0%/50%         1         5         1         34m45s
```
10. Check the deployment size and confirm that the deployment has been scaled  
down to 1 replica as the result of stopping the traffic generator:
```
$ kubectl get deployment my-ch7-app-node
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
my-ch7-app-node   1/1     1            1           5h33m
```

In this recipe, you learned how to automate how an application is scaled dynamically based 
on changing metrics. When applications are scaled up, they are dynamically scheduled on 
existing worker nodes.
## How it works...
This recipe showed you how to manually and automatically scale the number of pods in a 
deployment dynamically based on the Kubernetes metric.   
In this recipe, in Step 2, we created an Autoscaler that adjusts the number of replicas 
between the defined minimum using ***minReplicas: 1*** and ***maxReplicas: 5*** . As shown 
in the following example, the adjustment criteria are triggered by the 
***targetCPUUtilizationPercentage: 50*** metric:  
```
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-ch7-app-node
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```
***targetCPUUtilizationPercentage*** was used with the ***autoscaling/v1*** APIs. You will 
soon see that ***targetCPUUtilizationPercentage*** will be replaced with an array called 
metrics.  
To understand the new metrics and custom metrics, run the following command. This will 
return the manifest we created with V1 APIs into a new manifest using V2 APIs:  
```
$ kubectl get hpa.v2beta2.autoscaling my-ch7-app-node -o yaml
```
This enables you to specify additional resource metrics. By default, CPU and memory are 
the only supported resource metrics. In addition to these resource metrics, v2 APIs enable 
two other types of metrics, both of which are considered custom metrics: per-pod custom 
metrics and object metrics. You can read more about this by going to the ***Kubernetes HPA documentation*** link mentioned in the See also section.
## See also
*  Kubernetes pod Autoscaler using custom metrics: https://sysdig.com/blog/kubernetes-autoscaler/
*  Kubernetes HPA documentation: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
*  Declarative Management of Kubernetes Objects Using Configuration  
Files: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/
*  Imperative Management of Kubernetes Objects Using Configuration  
Files: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/
