# Configuring and managing S3 object storage using MinIO
In this section, we will create an S3 object storage using MinIO to store artifacts or configuration files created by your applications in Kubernetes. You will learn how to create deployment manifest files, deploy an S3 service, and provide an external IP address for other applications or users to consume the service.
## Getting ready
Clone the ***k8sdevopscookbook/src*** repository to your workstation to use manifest files under the ***chapter6*** directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter6
```
Make sure you have a Kubernetes cluster ready and kubectl configured so that you can manage the cluster resources.
## How to do it...
This section is further divided into the following subsections to make this process easier:
* Creating a deployment YAML manifest
* Creating a MinIO S3 service
* Accessing the MinIO web user interface
## Creating a deployment YAML manifest
All Kubernetes resources are created in a declarative way by using YAML manifest files.Let's perform the following steps to create an example file we will use later to deploy an application in Kubernetes:  
1. For this recipe, we will use MinIO to create a couple of resources that we can use to understand the file format and later help us deploy the fully functional
application. Open the MinIO download website by going to [https://min.io/download#/kubernetes] .   
2. On the MinIO website from the list of available download options, click on the ***Kubernetes*** button and select the ***Kubernetes CLI*** tab. This page will help us generate the YAML content required for the MinIO application based on our preferences:  

3. Enter your access key and secret key pair. In our example, we used ***minio / minio123*** . This will be used in place of a username and password when you access your MinIO service. Select ***Distributed*** as the deployment model and enter 4 for the number of nodes. This option will create a StatefulSet with four replicas. Enter 10 GB as the size. In our example, we'll use the values shown on the following configuration screen:
4. Click on the ***Generate*** button and examine the file's content. You will notice three different resources stored in the YAML manifest, including service, StatefulSet,and second service, which will create a cloud load balancer to expose the first service ports to the external access.
5. Copy the content and save it as minio.yaml on your workstation.
## Creating a MinIO S3 service
Let's perform the following steps to create the necessary resources to get a functional S3 service using MinIO:  
1. Deploy MinIO using the YAML manifest you created in the ***Creating a deployment YAML manifest*** recipe:
```
$ kubectl apply -f minio.yaml
```

As an alternative method, you can use the sample YAML file saved under the ***/src/chapter6/minio*** directory in the example repository using the ***$ kubectl apply -f minio/minio.yaml*** command.

2. Verify StatefulSet. You should see 4 out of 4 replicas deployed, similar to the following output. Note that if you deployed as standalone, you will not have StatefulSets:
```
$ kubectl get statefulsets
NAME    READY   AGE
minio   4/4     2m17s
```

Now, you have a MinIO application that's been deployed. In the next recipe, we will learn how to discover its external address to access the service.

## Accessing the MinIO web user interface
As part of the deployment process, we have MinIO create a cloud load balancer to expose the service to external access. In this recipe, we will learn how to access the MinIO interface to upload and download files to the S3 backend. To do so, we will perform the following steps:   
1. Get the ***minio-service*** LoadBalancer's external IP using the following command. You will see the exposed service address under the ***EXTERNAL-IP*** column, similar to the following output:
```
$ kubectl get service
NAME          TYPE          CLUSTER-IP  EXTERNAL-IP                       PORT(S)         AGE
minio         ClusterIP     None        <none>                            9000/TCP        2m49s
minio-service LoadBalancer  10.3.0.4    abc.us-west-2.elb.amazonaws.com   9000:30345/TCP  2m49s
```
2. As you can see, the output service is exposed via port ***9000*** . To access the service,we also need to add port 9000 to the end of the address ( ***http://[externalIP]:9000*** ) and open the public address of the MinIO service in our browser.
3. You need to have permissions to access the Dashboard. Use the default username of ***minio*** and the default password of ***minio123*** we created earlier to log in to the Minio deployment. After you've logged in, you will be able to access the MinIO Browser, as shown in the following screenshot:

MinIO is compatible with the Amazon S3 cloud storage service and is best suited for storing unstructured data such as photos, log files, and backups. Now that you have access to the MinIO user interface, you can create bucks, upload your files, and access them through S3 APIs, similar to how you would access a standard Amazon S3 service to store your backups. You can learn more about MinIO by going to the ***MinIO Documentation*** link in the ***See also*** section.

## How it works...
This recipe showed you how to provision a completely Amazon S3 API-compatible service using MinIO deployed on Kubernetes. This service will be used later for disaster recovery and backing up applications running on Kubernetes.

In the Creating a MinIO S3 service recipe, in Step 1, when we deploy MinIO, it creates a LoadBalancer service at port 9000 . Since we set the number of nodes to 4 , a StatefulSet will be created with four replicas. Each will use the information set under the volumeClaimTemplates section to create a PVC. If storageClassName is not defined specifically, then the default storage class will be used. As a result, you will see four instance of **PersistentVolumesClaim (PVC)** created on the cluster to provide a highly available MinIO service.

## See also
*  The MinIO documentation,at: [https://docs.min.io/docs/minio-quickstart-guide.html]
*  MinIO Operator for Kubernetes at: [https://github.com/minio/minio-operator]
*  The MinIO Erasure Code QuickStart Guide at: [https://docs.min.io/docs/minio-erasure-code-quickstart-guide]
*  Using MinIO Client, at: [https://docs.min.io/docs/minio-client-quickstart-guide]
