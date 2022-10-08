Originated from https://github.com/openebs/community/tree/master/workshop/exercise-9
# Rollout your application with Blue/Green deployment

## What is Blue/Green Deployment?

Kubernetes has a controller object called [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). 
A Deployment controller provides declarative updates for Pods and ReplicaSets.
Deployments are used to provide rolling updates, change the desired state of the pods, rollback to an earlier revision, etc.

However, there are many traditional workloads that won't work with Kubernetes way of rolling updates. 
If your workload needs to deploy a new version and cut over to it immediately then you may need to perform blue/green deployment instead.

Using Blue/Green deployment approach, you label the current production as “Blue”. Create an identical production environment called “Green” - Redirect the services to “Green”. 
If services are functional in “Green” - destroy “Blue”. If “Green” is bad - rollback to “Blue”.

## Other Refernces
* [Zero Downtime Deployment in Kubernetes with Jenkins](https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/)
* [A Simple Guide to bluelue/green Deployment](https://codefresh.io/learn/software-deployment/what-is-blue-green-deployment)
* [Kubernetes blue/green deployment Examples](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/blue-green)
