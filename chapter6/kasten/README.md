# Application backup and recovery using Kasten
In this section, we will create disaster recovery backups and migrate Kubernetes
applications and their persistent volumes in Kubernetes using **Kasten (K10)**.  
You will learn how to install and use K10, create standard and scheduled backups of
applications to an S3 target, and restore them back to the Kubernetes clusters.

## Getting ready
Make sure you have a Kubernetes cluster ready and **kubectl** and **helm** configured so that
you can manage the cluster resources. In this recipe, we will use a three-node Kubernetes
cluster on AWS.  
This recipe requires an existing stateful workload with presentable data to simulate a
disaster. To restore the data, we will use the **mytestapp** application we created in
the **Installing EBS CSI Driver to manage EBS volumes** recipe in **Chapter 5 , Preparing for
Stateful Workloads**.  
Clone the *k8sdevopscookbook/src* repository to your workstation to use the manifest
files under the *chapter6* directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter6
```
K10, by default, comes with a Starter Edition license that allows you to use the software on
a cluster with three worker nodes (at most) at no charge. K10 requires a backup target to be
configured.
## How to do it...
This section is further divided into the following subsections to make this process easier:
* Installing Kasten
* Accessing the Kasten dashboard
* Backing up an application
* Restoring an application
## Installing Kasten
Due to Kasten including PROMETHEUS , the minimum system requirements for PROMETHEUS is :  
* 4 GB RAM. 
* CPU on the level of 5th generation Pentium/Core i3 or equivalent.   
So the worker node need to adjust the Ram to 4 GB .  

Let's perform the following steps to install Kasten as a backup solution in our Kubernetes
cluster:  
1. Add the K10 helm repository:
```
$ helm repo add kasten https://charts.kasten.io/
```
2. Before we start, let's validate the environment. The following script will execute
some pre-installation tests to verify your cluster:
```
$ curl https://docs.kasten.io/tools/k10_preflight.sh | bash

Checking for tools
 --> Found kubectl
 --> Found helm
Checking access to the Kubernetes context kubernetes-admin@kubernetes
 --> Able to access the default Kubernetes namespace
Checking for required Kubernetes version (>= v1.12.0)
--> Kubernetes version (v1.15.3) meets minimum requirements
Checking if Kubernetes RBAC is enabled
--> Kubernetes RBAC is enabled
Checking if the Aggregated Layer is enabled
--> The Kubernetes Aggregated Layer is enabled
Checking if a default StorageClass is present
 --> A default storage class was found
Checking if the Kasten Helm repo is present
--> The Kasten Helm repo was found
Checking for required Helm Tiller version (>= v2.11.0)
--> Tiller version (v2.14.3) meets minimum requirements
All pre-flight checks succeeded!
```
3. Make sure your preferred storage class is set as the default; otherwise, define it
by adding the *-set persistence.storageClass* parameters to the following
command. In our example, we are using the *openebs-cstor-default storage*
class. Also, add your AWS access key and secret and install K10:
```
$ helm install kasten/k10 --name=k10 --namespace=kasten-io \
--set persistence.storageClass=openebs-cstor-default \
--set persistence.size=20Gi \
--set secrets.awsAccessKeyId="AWS_ACCESS_KEY_ID" \
--set secrets.awsSecretAccessKey="AWS_SECRET_ACCESS_KEY"
```
Since we’ve used OpenEBS as the persistent storage solution and its default storage class, our helm install command would be
```
# Modify the storageClass and pvc size according to the storage classes  present in your cluster.
$ helm install kasten/k10 --name=k10 --namespace=kasten-io \\
--set persistence.storageClass=openebs-jiva-default \\
--set persistence.size=20Gi
```
4. Confirm that the deployment status is DEPLOYED using the following helm
command:
```
$ helm ls
NAME  REVISION  UPDATED                   STATUS    CHART APP VERSION     NAMESPACE
k10   1         Tue Oct 29 07:36:19 2019  DEPLOYED  k10-1.1.56 1.1.56     kasten-io
```
All the pods should be deployed in around a minute after this step as Kasten exposes an
API based on Kubernetes CRDs. You can either use *kubectl* with the new CRDs (refer to
the *Kasten CLI commands* link in the See also section) or use the Kasten Dashboard by
following the next recipe, that is, the Accessing the Kasten Dashboard recipe.

## Accessing the Kasten Dashboard
Let's perform the following steps to access the Kasten Dashboard. This is where we will be
taking application backups and restoring them:
1. Create port forwarding using the following command. This step will forward the
Kasten Dashboard service on port 8000 to your local workstation on port 8080 :
```
$ export KASTENDASH_POD=$(kubectl get pods --namespace kasten-io -l "service=gateway" -o jsonpath="{.items[0].metadata.name}")
$ kubectl port-forward --namespace kasten-io $KASTENDASH_POD 8080:8000 >> /dev/null &
```
or use LoadBalancer /NodePort
```
kubectl -n kasten-io patch svc gateway --type='json' -p '[{"op":"replace","path":"/spec/type","value":"LoadBalancer"}]'
```
2. On your workstation, open http://127.0.0.1:8080/k10/# with your
browser:
```
$ firefox http://127.0.0.1:8080/k10/#
```
or 
```
$ firefox http://[[LoadBalancerIP]]/k10/#
```
3. Read and accept the end user license agreement:

With that, you have accessed the Kasten Dashboard. You can familiarize yourself with it by
clicking the main menus and referring to the Kasten documentation link in the See also section
for additional settings if needed.
## Backing up an application
Let's perform the following steps to take a backup of our application:  

1. If you have an application and persistent volumes associated with the backup
labeled already, you can skip to Step 5. Otherwise, create a namespace and a PVC
using the following example code:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: backup-example
  labels:
    app: app2backup
EOF
```

2. Create a PVC in the backup-example namespace:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2backup
  namespace: backup-example
  labels:
    app: app2backup
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-cstor-default
  resources:
    requests:
      storage: 4Gi
EOF
```

3. Create a pod that will use the PVC and write to the /data/out.txt file inside the pod using the sample myapp.yaml manifest under the src/chapter6/kasten directory:

```
$ kubectl apply -f - kasten/myapp.yaml
```

4. Verify that our myapp pod writes data to the volume:

```
$ kubectl exec -it myapp cat /data/out.txt -nbackup-example
Thu Sep 12 23:18:08 UTC 2019
```

5. On the Kasten Dashboard, click on Unmanaged applications:  

6. In the backup-example namespace, click on Create a policy:  

7. Enter a name and select the Snapshot action:  

8. Select Daily as the Action Frequency:  

9. Click on Create Policy:   

By following these steps, you will have created your first backup using the policy, as well
as a schedule for the following backup jobs.

##  Restoring an application
Let's perform the following steps to restore the application from an existing backup:  
1. Under Applications, from the list of compliant applications, click the arrow icon
next to backup-example and select Restore Application. If the application was
deleted, then the Removed option needs to be selected:   
2. Select a restore point to recover to:   
3. Select backup-example and click on Restore:   
4. Confirm that you want this to be restored:  
With that, you've learned how to restore an application and its volumes from its backup
using Kasten.  
## How it works...
This recipe showed you how to create disaster recovery backups, restore your application
and its data back from an S3 target, and how to create scheduled backups on Kubernetes.   

In the Backing up an application recipe, in Step 2, we created a pod that uses OpenEBS as a
storage vendor. In this case, Kasten uses a generic backup method that requires a sidecar to
your application that can mount the application data volume. The following is an example
that you can add to your pods and deployment when using non-standard storage options:
```
- name: kanister-sidecar
  image: kanisterio/kanister-tools:0.20.0
  command: ["bash", "-c"]
  args:
  - "tail -f /dev/null"
  volumeMounts:
  - name: data
    mountPath: /data
```
## See also
* The Kasten documentation , at https://docs.kasten.io/
* Kasten CLI commands, at https://docs.kasten.io/api/cli.html
* More on generic backup and restore using Kanister, at https://docs.kasten.io/kanister/generic.html#generic-kanister
