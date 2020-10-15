# Creating an external load balancer
The load balancer service type is a relatively simple service alternative to ingress that uses a 
cloud-based external load balancer. The external load balancer service type's support is 
limited to specific cloud providers but is supported by the most popular cloud providers,including AWS, GCP, Azure, Alibaba Cloud, and OpenStack.

In this section, we will expose our workload ports using a load balancer. We will learn how 
to create an external GCE/AWS load balancer for clusters on public clouds, as well as for 
your private cluster using inlet-operator .

## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources. In this recipe, we are using a cluster that's been deployed on 
AWS using kops , as described in Chapter 1 , Building Production-Ready Kubernetes Clusters,
in the Amazon Web Services recipe. The same instructions will work on all major cloud providers.

To access the example files, clone the k8sdevopscookbook/src repository to your workstation to use the configuration files in the src/chapter7/lb directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter7/lb/
```
After you've cloned the examples repository, you can move on to the recipes.
## How to do it...
This section is further divided into the following subsections to make this process easier:
* Creating an external cloud load balancer
* Finding the external address of the service
## Creating an external cloud load balancer
When you create an application and expose it as a Kubernetes service, you usually need the 
service to be reachable externally via an IP address or URL. In this recipe, you will learn 
how to create a load balancer, also referred to as a cloud load balancer.

In the previous chapters, we have seen a couple of examples that used the load balancer 
service type to expose IP addresses, including the Configuring and managing S3 object storage 
using MinIO and Application backup and recovery using Kasten recipes in the previous chapter, 
as well as the To-Do application that was provided in this chapter in the Assigning 
applications to nodes recipe.

Let's use the MinIO application to learn how to create a load balancer. Follow these steps to 
create a service and expose it using an external load balancer service:

1. Review the content of the minio.yaml file in the examples directory 
in src/chapter7/lb and deploy it using the following command. This will 
create a StatefulSet and a service where the MinIO port is exposed internally to 
the cluster via port number 9000 . You can choose to apply the same steps and 
create a load balancer for your own application. In that case, skip to Step 2:
```
$ kubectl apply -f minio.yaml
```

2. List the available services on Kubernetes. You will see that the MinIO service 
shows ClusterIP as the service type and none under the EXTERNAL-IP field: 
```
$ kubectl get svc
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes           ClusterIP      10.96.0.1        <none>         443/TCP        23h
minio                ClusterIP      None             <none>         9000/TCP       6s
```
3. Create a new service with the TYPE set to LoadBalancer . The following command will expose port: 9000 of our MinIO application at targetPort:
9000 using the TCP protocol, as shown here:
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  type: LoadBalancer
  ports:
    - port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    app: minio
EOF
```
The preceding command will immediately create the Service object, but the actual load 
balancer on the cloud provider side may take 30 seconds to a minute to be completely 
initialized. Although the object will state that it's ready, it will not function until the load 
balancer is initialized. This is one of the disadvantages of cloud load balancers compared to 
ingress controllers, which we will look at in the next recipe, Creating an ingress service and 
service mesh using Istio.

As an alternative to Step 3, you can also create the load balancer by using the following 
command:
```
$ kubectl expose rc example --port=9000 --target-port=9000 --name=minio-service --type=LoadBalancer
```
## Finding the external address of the service
Let's perform the following steps to get the externally reachable address of the service:

1. List the services that use the LoadBalancer type. The EXTERNAL-IP column will show you the cloud vendor-provided address:
```
$ kubectl get svc |grep LoadBalancer
minio-service        LoadBalancer   10.99.142.247    192.16.35.18   9000:32107/TCP   78s

```
2. If you are running on a cloud provider service such as AWS, you can also use the 
following command to get the exact address. You can copy and paste this into a 
web browser:
```
$ SERVICE_IP=http://$(kubectl get svc minio-service \
-o jsonpath='{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[].targetPort}')
$ echo $SERVICE_IP
```
3. If you are running on a bare-metal server, then you probably won't have a 
hostname entry. As an example, if you are running MetalLB ( https://metallb.universe.tf/ ), a load balancer for bare-metal Kubernetes clusters, or SeeSaw
( https://github.com/google/seesaw ), a Linux Virtual Server (LVS)-based load balancing platform, you need to look for the ip entry instead:
```
$ SERVICE_IP=http://$(kubectl get svc minio-service \
-o jsonpath='{.status.loadBalancer.ingress[0].ip}:{.spec.ports[].targetPort}')
$ echo $SERVICE_IP
```
The preceding command will return a link similar to https://containerized.me.us-east-1.elb.amazonaws.com:9000 .

## How it works...
This recipe showed you how to quickly create a cloud load balancer to expose your services
with an external address.

In the Creating a cloud load balancer recipe, in Step 3, when a load balancer service is created 
in Kubernetes, a cloud provider load balancer is created on your behalf without you having 
to go through the cloud service provider APIs separately. This feature helps you easily 
manage the creation of load balancers outside of your Kubernetes cluster, but at the same 
takes a bit of time to complete and requires a separate load balancer for every service, so 
this might be costly and not very flexible.

To give load balancers flexibility and add more application-level functionality, you can use 
ingress controllers. Using ingress, traffic routing can be controlled by rules defined in the 
ingress resource. You will learn more about popular ingress gateways in the next two 
recipes, Creating an ingress service and service mesh using Istio and Creating an ingress service 
and service mesh using Linkerd.

## See also
* Kubernetes documentation on the load balancer service type: https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer
* Using a load balancer on Amazon EKS: https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html
* Using a load balancer on AKS: https:/​ / ​ docs.​ microsoft.​ com/​ en-​ us/​ azure/​ aks/load-​ balancer-standard
* Using a load balancer on Alibaba Cloud: https:/​ / ​ www.​ alibabacloud.​ com/​ help/doc-​ detail/​ 53759.​ htm
* Load balancer for your private Kubernetes cluster: https:/​ / ​ blog.​ alexellis.​ io/ingress-​ for-​ your-​ local-​ kubernetes-​ cluster/​
