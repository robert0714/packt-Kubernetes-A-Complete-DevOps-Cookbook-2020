# Assigning applications to nodes
In this section, we will make sure that pods are not scheduled onto inappropriate nodes. 
You will learn how to schedule pods into Kubernetes nodes using node selectors, taints, 
toleration and by setting priorities.
## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources.
## How to do it...
This section is further divided into the following subsections to make this process easier:
* Labeling nodes
* Assigning pods to nodes using nodeSelector
* Assigning pods to nodes using node and inter-pod affinity
## Labeling nodes
Kubernetes labels are used for specifying the important attributes of resources that can be 
used to apply organizational structures onto system objects. In this recipe, we will learn 
about the common labels that are used for Kubernetes nodes and apply a custom label to be 
used when scheduling pods into nodes.

Let's perform the following steps to list some of the default labels that have been assigned 
to your nodes:
1. List the labels that have been assigned to your nodes. In our example, we will use 
a kops cluster that's been deployed on AWS EC2, so you will also see the relevant 
AWS labels, such as availability zones: 
```
$ kubectl get nodes --show-labels
NAME     STATUS   ROLES    AGE   VERSION    LABELS
k8s-m1   Ready    master   22h   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-m1,kubernetes.io/os=linux,node-role.kubernetes.io/master=
k8s-n1   Ready    <none>   22h   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-n1,kubernetes.io/os=linux
k8s-n2   Ready    <none>   22h   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-n2,kubernetes.io/os=linux,proxy=true
k8s-n3   Ready    <none>   22h   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-n3,kubernetes.io/os=linux,proxy=true
k8s-n4   Ready    <none>   22h   v1.15.12   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-n4,kubernetes.io/os=linux
```
2. Get the list of the nodes in your cluster. We will use node names to assign labels in the next step:
```
$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
k8s-m1   Ready    master   22h   v1.15.12
k8s-n1   Ready    <none>   22h   v1.15.12
k8s-n2   Ready    <none>   22h   v1.15.12
k8s-n3   Ready    <none>   22h   v1.15.12
k8s-n4   Ready    <none>   22h   v1.15.12
```

3. Label two nodes as ***production*** and ***development*** . Run the following 
command using your worker node names from the output of Step 2:
```
$ kubectl label nodes k8s-n1 environment=production
node/k8s-n1 labeled
$ kubectl label nodes k8s-n2 environment=production
node/k8s-n2 labeled
$ kubectl label nodes k8s-n3 environment=development
node/k8s-n3 labeled
$ kubectl label nodes k8s-n4 environment=development
node/k8s-n4 labeled
```
4. Verify that the new labels have been assigned to the nodes. This time, you should see ***environment*** labels on all the nodes except the node labeled ***role=master*** :
```
$ kubectl get nodes --show-labels
```
It is recommended to document labels for other people who will use your clusters. While 
they don't directly imply semantics to the core system, make sure they are still meaningful 
and relevant to all users.

## Assigning pods to nodes using nodeSelector
In this recipe, we will learn how to schedule a pod onto a selected node using the 
nodeSelector primitive:  

1. Create a copy of the Helm chart we used in the Manually scaling an application recipe in a new directory called todo-dev . We will edit the templates later in order to specify ***nodeSelector*** :
```
$ cd src/chapter7/charts
$ mkdir todo-dev
$ cp -a node/* todo-dev/
$ cd todo-dev
```
2. Edit the deployment.yaml file in the templates directory:
```
$ vi templates/deployment.yaml
```
3. Add ***nodeSelector***: and ***environment***: "{{ .Values.environment }}" right before the ***containers***: parameter. This should look as follows:
```
...(ommit)
        mountPath: {{ .Values.persistence.path }}
    {{- end }}
# Start of the addition
    nodeSelector:
      environment: "{{ .Values.environment }}"
# End of the addition
      containers:
      - name: {{ template "node.fullname" . }}
...(ommit)
```
The Helm installation uses templates to generate configuration files. As shown in 
the preceding example, to simplify how you customize the provided values, 
***{{expr}}*** is used, and these values come from the ***values.yaml*** file names. The 
***values.yaml*** file contains the default values for a chart.

ps. 
Although it may not be practical on large clusters, instead of using 
***nodeSelector*** and labels, you can also schedule a pod on one specific 
node using the ***nodeName*** setting. In that case, instead of the 
***nodeSelector*** setting, you add ***nodeName: yournodename*** to your 
deployment manifest.

4. Now that we've added the variable, edit the values.yaml file. This is where we 
will set the environment to the development label:
```
$ vi values.yaml
```
5. Add the environment: development line to the end of the files. It should look 
as follows:
```
...
## Affinity for pod assignment
## Ref:
https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
##
affinity: {}
environment: development
```
6. Edit the Chart.yaml file and change the chart name to its folder name. In this 
recipe, it's called todo-dev . After these changes, the first two lines should look 
as follows:
```
apiVersion: v1
name: todo-dev
...
```
7. Update the Helm dependencies and build them. The following commands will pull all the dependencies and build the Helm chart:
```
$ helm dep update & helm dep build
```
8. Examine the chart for issues. If there are any issues with the chart's files, the linting process will bring them up; otherwise, no failures should be found:
```
$ helm lint .
==> Linting .
Lint OK
1 chart(s) linted, no failures
```
9. Install the To-Do application example using the following command. This Helm chart will deploy two pods, including a Node.js service and a MongoDB service,
except this time the nodes are labeled as ***environment: development***:
```
$ helm install . --name my-app7-dev --set serviceType=LoadBalancer
```
10. Check that all the pods have been scheduled on the development nodes using the 
following command. You will find the my-app7-dev-todo-dev pod running on 
the node labeled environment: development :
```
$ for n in $(kubectl get nodes -l environment=development --no-
headers | cut -d " " -f1); do kubectl get pods --all-namespaces --
no-headers --field-selector spec.nodeName=${n} ; done
```
With that, you've learned how to schedule workload pods onto selected nodes using the ***nodeSelector*** primitive.

## Assigning pods to nodes using node and inter-pod Affinity
In this recipe, we will learn how to expand the constraints we expressed in the previous 
recipe, *Assigning pods to labeled nodes using nodeSelector*, using the affinity and anti-affinity 
features.
Let's use a scenario-based approach to simplify this recipe for different affinity selector 
options. We will take the previous example, but this time with complicated requirements:
*  todo-prod must be scheduled on a node with the environment:production label and should fail if it can't.
*  todo-prod should run on a node that is labeled with failure-domain.beta.kubernetes.io/zone=us-east-1a or us-east-1b but can run anywhere if the label requirement is not satisfied.
*  todo-prod must run on the same zone as mongodb , but should not run in the zone where todo-dev is running.

ps.
The requirements listed here are only examples in order to represent the use of some affinity definition functionality. This is not the ideal way to
configure this specific application. The labels may be completely different in your environment.

The preceding scenario will cover both types of node affinity options( requiredDuringSchedulingIgnoredDuringExecution and preferredDuringSchedulingIgnoredDuringExecution ). You will see these options 
later in our example. Let's get started:  
1. Create a copy of the Helm chart we used in the Manually scaling an 
application recipe to a new directory called todo-prod . We will edit the 
templates later in order to specify nodeAffinity rules: 

```
$ cd src/chapter7/charts
$ mkdir todo-prod
$ cp -a node/* todo-prod/
$ cd todo-prod
```
2. Edit the values.yaml file. To access it, use the following command:
```
$ vi values.yaml
```
3. Replace the last line, affinity: {} , with the following code. This change will 
satisfy the first requirement we defined previously, meaning that a pod can only 
be placed on a node with an environment label and whose value is production :
```
## Affinity for pod assignment
## Ref:
https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#
affinity-and-anti-affinity
# affinity: {}
# Start of the affinity addition #1
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
# End of the affinity addition #1
```
You can also specify more than one ***matchExpressions*** under the  
***nodeSelectorTerms*** . In this case, the pod can only be scheduled onto a node 
where all ***matchExpressions*** are satisfied, which may limit your successful scheduling chances.

ps.
Although it may not be practical on large clusters, instead of using 
***nodeSelector*** and labels, you can also schedule a pod on a specific node 
using the ***nodeName*** setting. In this case, instead of the ***nodeSelector*** 
setting, add ***nodeName***: ***yournodename*** to your deployment manifest.

4. Now, add the following lines right under the preceding code addition. This 
addition will satisfy the second requirement we defined, meaning that nodes 
with a label of ***failure-domain.beta.kubernetes.io/zone*** and whose value 
is ***us-east-1a*** or ***us-east-1b*** will be preferred:
```
          - production
# End of the affinity addition #1
# Start of the affinity addition #2
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 1
  preference:
    matchExpressions:
    - key: failure-domain.beta.kubernetes.io/zone
      operator: In
      values:
      - us-east-1a
      - us-east-1b
# End of the affinity addition #2
```
5. For the third requirement, we will use the inter-pod affinity and anti-affinity 
functionalities. They allow us to limit which nodes our pod is eligible to be 
scheduled based on the labels on pods that are already running on the node 
instead of taking labels on nodes for scheduling. The following podAffinity 
***requiredDuringSchedulingIgnoredDuringExecution*** rule will look for 
nodes where ***app***: ***mongodb*** exist and use ***failure-domain.beta.kubernetes.io/zone*** as a topology key to show us where the 
pod is allowed to be scheduled:
```
          - us-east-1b
# End of the affinity addition #2
# Start of the affinity addition #3a
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - mongodb
      topologyKey: failure-domain.beta.kubernetes.io/zone
# End of the affinity addition #3a
```
6. Add the following lines to complete the requirements. This time, the 
***podAntiAffinity preferredDuringSchedulingIgnoredDuringExecution*** 
rule will look for nodes where ***app***: ***todo-dev*** exists and use ***failure-domain.beta.kubernetes.io/zone*** as a topology key:
```
      topologyKey: failure-domain.beta.kubernetes.io/zone
# End of the affinity addition #3a
# Start of the affinity addition #3b
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - todo-dev
        topologyKey: failure-domain.beta.kubernetes.io/zone
# End of the affinity addition #3b
```
7. Edit the Chart.yaml file and change the chart name to its folder name. In this 
recipe, it's called todo-prod . After making these changes, the first two lines 
should look as follows:
```
apiVersion: v1
name: todo-prod
...
```
8. Update the Helm dependencies and build them. The following commands will 
pull all the dependencies and build the Helm chart:
```
$ helm dep update & helm dep build
```
9. Examine the chart for issues. If there are any issues with the chart files, the linting 
process will bring them up; otherwise, no failures should be found:
```
$ helm lint .
==> Linting .
Lint OK
1 chart(s) linted, no failures
```
10. Install the To-Do application example using the following command. This Helm 
chart will deploy two pods, including a Node.js service and a MongoDB service, 
this time following the detailed requirements we defined at the beginning of this 
recipe:
```
$ helm install . --name my-app7-prod --set serviceType=LoadBalancer
```
11. Check that all the pods that have been scheduled on the nodes are labeled as 
environment: production using the following command. You will find 
the my-app7-dev-todo-dev pod running on the nodes: 
```
$ for n in $(kubectl get nodes -l environment=production --no-headers | cut -d " " -f1); do kubectl get pods --all-namespaces --no-headers --field-selector spec.nodeName=${n} ; done
```
In this recipe, you learned about advanced pod scheduling practices while using a number 
of primitives in Kubernetes, including nodeSelector , node affinity, and inter-pod affinity. 
Now, you will be able to configure a set of applications that are co-located in the same 
defined topology or scheduled in different zones so that you have better ***service-levelagreement (SLA)*** times.

## How it works...
The recipes in this section showed you how to schedule pods on preferred locations, 
sometimes based on complex requirements.

In the Labeling nodes recipe, in Step 1, you can see that some standard labels have been applied to your nodes already. Here is a short explanation of what they mean and where they are used:   
*  ***kubernetes.io/arch*** : This comes from the runtime.GOARCH parameter and is applied to nodes to identify where to run different architecture container images, such as x86, arm, arm64, ppc64le, and s390x, in a mixed architecture cluster.
*  ***kubernetes.io/instance-type*** : This is only useful if your cluster is deployed on a cloud provider. Instance types tell us a lot about the platform, especially for AI and machine learning workloads where you need to run some pods on instances with GPUs or faster storage options.
*  ***kubernetes.io/os*** : This is applied to nodes and comes from runtime.GOOS . It is probably less useful unless you have Linux and Windows nodes in the same 
cluster.
*  ***failure-domain.beta.kubernetes.io/region and /zone*** : This is also more useful if your cluster is deployed on a cloud provider or your infrastructure is 
spread across a different failure-domain. In a data center, it can be used to define a rack solution so that you can schedule pods on separate racks for higher 
availability.
*  ***kops.k8s.io/instancegroup=nodes*** : This is the node label that's set to the name of the instance group. It is only used with kops clusters.
*  ***kubernetes.io/hostname*** : Shows the hostname of the worker.
*  ***kubernetes.io/role*** : This shows the role of the worker in the cluster. Some common values include node for representing worker nodes and master , which 
shows the node is the master node and is tainted as not schedulable for workloads by default.

In the Assigning pods to nodes using node and inter-pod affinity recipe, in Step 3, the node 
affinity rule says that the pod can only be placed on a node with a label whose key is 
environment and whose value is production . 

In Step 4, the ***affinity key***: ***value*** requirement is preferred 
( ***preferredDuringSchedulingIgnoredDuringExecution*** ). The ***weight*** field here can 
be a value between 1 and 100 . For every node that meets these requirements, a Kubernetes 
scheduler computes a sum. The nodes with the highest total score are preferred.

Another detail that's used here is the In parameter. Node Affinity supports the following 
operators: ***In , NotIn , Exists , DoesNotExist , Gt*** , and ***Lt*** . You can read more about the 
operators by looking at the ***Scheduler affinities through examples*** link mentioned in the See 
also section.

PS
If selector and affinity rules are not well planned, they can easily block 
pods getting scheduled on your nodes. Keep in mind that if you have 
specified both nodeSelector and nodeAffinity rules, both 
requirements must be met for the pod to be scheduled on the available 
nodes.

In Step 5, inter-pod affinity is used ( podAffinity ) to satisfy the requirement in PodSpec. In 
this recipe, podAffinity is requiredDuringSchedulingIgnoredDuringExecution .
Here, matchExpressions says that a pod can only run on nodes where failure-domain.beta.kubernetes.io/zone matches the nodes where other pods with the app: 
mongodb label are running.

In Step 6, the requirement is satisfied with ***podAntiAffinity*** using 
preferredDuringSchedulingIgnoredDuringExecution .   
Here, matchExpressions says that a pod can't run on nodes where ***failure-domain.beta.kubernetes.io/zone*** matches the nodes where other pods with the app:todo-dev label are running. The weight is increased by setting it to 100 .

## See also
*  List of known labels, annotations, and taints: https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/
*  Assigning Pods to Nodes in the Kubernetes documentation: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/
*  More on labels and selectors in the Kubernetes documentation: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
*  Scheduler affinities through examples: https://banzaicloud.com/blog/k8s-affinities/
*  Node affinity and NodeSelector design document: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md
*  Interpod topological affinity and anti-affinity design document: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md
