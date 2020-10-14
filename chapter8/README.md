# Observability and Monitoring on Kubernetes
In this chapter, we will discuss the built-in Kubernetes tools and the popular third-party
monitoring options for your containerized DevOps environment. You will learn how to
monitor metrics for performance analysis, and also how to monitor and manage the real-
time cost of Kubernetes resources.
By the end of this chapter, you should have knowledge of the following:
* Monitoring in Kubernetes
* Inspecting containers
* Monitoring using Amazon CloudWatch
* Monitoring using Google Stackdriver
* Monitoring using Azure Monitor
* Monitoring Kubernetes using Prometheus and Grafana
* Monitoring and performance analysis using Sysdig
* Managing the cost of resources using Kubecost

## Technical requirements
The recipes in this chapter assume that you have deployed a functional Kubernetes cluster
following one of the recommended methods described in Chapter 1 , Building Production-
Ready Kubernetes Clusters.

Kubernetes' command-line tool, kubectl , will be used for the rest of the recipes in this
chapter since it's the main command-line interface for running commands against
Kubernetes clusters. We will also use Helm where Helm charts are available to deploy
solutions.

## Monitoring in Kubernetes
In this section, we will configure our Kubernetes cluster to get core metrics, such as CPU
and memory. You will learn how to monitor Kubernetes metrics using the built-in
Kubernetes tools both in the CLI and on the UI.
## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the
cluster resources.
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest
files in the chapter8 directory:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd /src/chapter8
```
The Monitoring metrics using Kubernetes Dashboard recipe requires Kubernetes Dashboard
v2.0.0 or later to function. If you want to add metric functionality to the dashboard, make
sure that you have Kubernetes Dashboard installed by following the instructions in
the Deploying Kubernetes Dashboard recipe in Chapter 1 , Building Production-Ready Kubernetes
Clusters.

## How to do it...
This section is further divided into the following subsections to make the process easier:
* Adding metrics using Kubernetes Metrics Server
* Monitoring metrics using the CLI
* Monitoring metrics using Kubernetes Dashboard
* Monitoring Node Health

# Adding metrics using Kubernetes Metrics Server
Getting core system metrics such as CPU and memory not only provides useful
information, but is also required by extended Kubernetes functionality such as Horizontal
Pod Autoscaling, which we mentioned in ***Chapter 7 , Scaling and Upgrading Applications***:

1. Install kube-state-metrics.  

| kube-state-metrics | **Kubernetes 1.14** |  **Kubernetes 1.15** | **Kubernetes 1.16** |  **Kubernetes 1.17** |  **Kubernetes 1.18** |  **Kubernetes 1.19** |
|--------------------|---------------------|----------------------|---------------------|----------------------|----------------------|----------------------|
| **v1.7.2**         |         ✓           |          ✓           |         -           |          -           |          -           |          -           |
| **v1.8.0**         |         ✓           |          ✓           |         -           |          -           |          -           |          -           |
| **v1.9.7**         |         -           |         ✓            |         ✓           |          -           |          -           |          -           |
| **v2.0.0-alpha.1** |         -           |         -            |         -           |          ✓           |          ✓           |          ✓           |

- `✓` Fully supported version range.
- `-` The Kubernetes cluster has features the client-go library can't use (additional API objects, deprecated APIs, etc).

```
$ git clone https://github.com/kubernetes/kube-state-metrics.git
$ cd kube-state-metrics  && git checkout release-1.8 && kubectl apply -f kubernetes
```

1. Clone the Metrics Server repository to your client by running the following
command:
```
$ git clone https://github.com/kubernetes-incubator/metrics-server.git
```
2. Deploy the Metrics Server by applying the manifest in the metrics-server/deploy/1.8+ directory by running the following command:
```
$ kubectl apply -f metrics-server/deploy/1.8+
```
Or reference [https://computingforgeeks.com/how-to-deploy-metrics-server-to-kubernetes-cluster/]  
```
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
$ vi components.yaml
```
Modify like below:
```
...............
containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.7
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
          - --kubelet-insecure-tls
          - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
```
And then
```
$ kubectl apply -f  components.yaml
```
This command will create the resources required in the kube-space namespace.

## Monitoring metrics using the CLI
As part of the Metrics Server, the Resource Metrics API provides access to CPU and
memory resource metrics for pods and nodes. Let's use the Resource Metrics API to access
the metrics data from the CLI:  
1. First, let's display node resource utilization:
```
$ kubectl top nodes
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-m1   174m         8%     1944Mi          52%       
k8s-n1   94m          4%     1556Mi          42%       
k8s-n2   118m         5%     1426Mi          38%       
k8s-n3   116m         5%     1513Mi          41%       
k8s-n4   85m          4%     1318Mi          35%  
```
The command will return utilized CPU and memory on all your Kubernetes
nodes.  
ps. There are a couple of ways to use the metrics information. First of all, at
any given time, usage of both CPU and memory should be below your
desired threshold, otherwise new nodes need to be added to your cluster
to handle services smoothly. Balanced utilization is also important, which
means that if the percentage of memory usage is higher than the average
percentage of CPU usage, you may need to consider changing your cloud
instance type to use better-balanced VM instances.  

2. Display pod resource utilization in any namespace. In this example, we are
listing the pods in the openebs namespace:
```
$ kubectl top pods -n openebs
NAME                                          CPU(cores)   MEMORY(bytes)   
maya-apiserver-6c5d8f8f49-6sjnc               1m           23Mi            
openebs-admission-server-86cfdc9685-x5g65     1m           14Mi            
openebs-localpv-provisioner-9fbcd5c58-qkml7   2m           5Mi             
openebs-ndm-8fcwm                             1m           14Mi            
openebs-ndm-8jhm8                             1m           20Mi            
openebs-ndm-operator-5976f76d45-9lfcg         1m           14Mi            
openebs-ndm-p4zrh                             1m           20Mi            
openebs-ndm-rw5wp                             1m           16Mi            
openebs-provisioner-79d59d4b87-5kvwz          4m           5Mi             
openebs-snapshot-operator-65f88f68f8-j2bnb    5m           21Mi
```
The command should return the utilized CPU and memory on all your pods. Kubernetes
features such as Horizontal Pod Scaler can utilize this information to scale your pods.

## Monitoring metrics using Kubernetes Dashboard
By default, Kubernetes Dashboard doesn't display detailed metrics unless Kubernetes
Metrics Server is installed and the kubernetes-metrics-scraper sidecar container is
running.
Let's first verify that all the necessary components are running, and then we will see how to
access the metrics data from Kubernetes Dashboard:  
1. Verify that the kubernetes-metrics-scraper pod is running. If not, install
Kubernetes Dashboard by following the instructions in the Deploying the
Kubernetes Dashboard recipe in Chapter 1 , Building Production-Ready Kubernetes
Clusters:
```
$ kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-76679bc5b9-gzv68   1/1     Running   0          4h5m
kubernetes-dashboard-65bb64d6cb-tbfp6        1/1     Running   0          4h5m
```
2. On Kubernetes Dashboard, select Namespaces and click on the Overview menu.
This view shows pods in that namespace with their CPU and memory utilization:   

3. On Kubernetes Dashboard, select a namespace and click on Pods in the
Overview menu. This view shows the overall CPU and memory utilization of the
workloads within the selected namespace:   

4. Select Nodes under the Cluster menu. This view shows nodes in the cluster with
CPU and memory utilization:  
If the requests and limits are set very high, then they can take up more than their expected
share of the cluster.

## Monitoring node health
In this recipe, we will learn how to create a DaemonSet in the Kubernetes cluster to monitor
node health. The node problem detector will collect node problems from daemons and will
report them to the API server as NodeCondition and Event:   
1. From the /src/chapter8 folder first, inspect the content of the node-problem-
detector.yaml file and create the DaemonSet to run the node problem detector:
```
$ cat debug/node-problem-detector.yaml
$ kubectl apply -f debug/node-problem-detector.yaml
```
2. Get a list of the nodes in the cluster. This command will return both worker and
master nodes:
```
$ kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
k8s-m1   Ready    master   4h7m   v1.15.12
k8s-n1   Ready    <none>   4h6m   v1.15.12
k8s-n2   Ready    <none>   4h5m   v1.15.12
k8s-n3   Ready    <none>   4h4m   v1.15.12
k8s-n4   Ready    <none>   4h3m   v1.15.12
```
3. Describe a node's status by replacing the node name in the following command
with one of your node names and running it. In the output, examine
the Conditions section for error messages. Here's an example of the output:

```
$ kubectl describe node k8s-n1 | grep -i condition -A 20 | grep Ready -B 20
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Wed, 14 Oct 2020 12:26:11 +0800   Wed, 14 Oct 2020 12:26:11 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Wed, 14 Oct 2020 16:32:16 +0800   Wed, 14 Oct 2020 12:24:55 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Wed, 14 Oct 2020 16:32:16 +0800   Wed, 14 Oct 2020 12:24:55 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Wed, 14 Oct 2020 16:32:16 +0800   Wed, 14 Oct 2020 12:24:55 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Wed, 14 Oct 2020 16:32:16 +0800   Wed, 14 Oct 2020 12:25:46 +0800   KubeletReady                 kubelet is posting ready status
```
4. Additionally, you can check for KernelDeadlock , MemoryPressure ,
and DiskPressure conditions by replacing the last part of the command with
one of the conditions. Here is an example for KernelDeadlock :   
```
$ kubectl get node  k8s-n1 -o yaml | grep -B5 KernelDeadlock

- lastHeartbeatTime: "2019-10-18T23:58:53Z"
lastTransitionTime: "2019-10-18T23:49:46Z"
message: kernel has no deadlock
reason: KernelHasNoDeadlock
status: "False"
type: KernelDeadlock
```
The Node Problem Detector can detect unresponsive runtime daemons; hardware issues
such as bad CPU, memory, or disk; kernel issues including kernel deadlock conditions;
corrupted filesystems; unresponsive runtime daemons; and also infrastructure daemon
issues such as NTP service outages.

##  See also
* Kubernetes Metrics Server Design Document: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/metrics-server.md
* Configuring and using the monitoring stack in OpenShift Container Platform: https://access.redhat.com/documentation/en-us/openshift_container_platform/4.2/html/monitoring/index
* Krex, a Kubernetes Resource Explorer: https://github.com/kris-nova/krex
