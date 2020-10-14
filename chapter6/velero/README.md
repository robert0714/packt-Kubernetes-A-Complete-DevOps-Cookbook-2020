# Application backup and recovery using Velero
In this section, we will create disaster recovery backups and migrate Kubernetes
applications and their persistent volumes in Kubernetes using VMware Velero (formerly
Heptio Ark).  
You will learn how to install Velero, create standard and scheduled backups of applications
with an S3 target, and restore them back to the Kubernetes clusters.

## Getting ready
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the
cluster resources.
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest
files under the chapter6 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter6
```
This recipe requires an existing stateful workload with presentable data so that we can
simulate a disaster and then restore the data. To do this, we will use
the mytestapp application we created during the Installing EBS CSI driver to manage EBS
volumes recipe in Chapter 5 , Preparing for Stateful Workloads.  
Velero also requires S3-compatible object storage to store the backups. In this recipe, we
will use the MinIO S3 target we deployed during the Configuring and managing S3 object
storage using MinIO recipe for storing our backups.  

## How to do it...
This section is further divided into the following subsections to make this process easier:
* Installing Velero
* Backing up an application
* Restoring an application
* Creating a scheduled backup
* Taking a backup of an entire namespace
* Viewing backups with MinIO
* Deleting backups and schedules

## Installing Velero
Velero is an open source project that's used to make backups, perform disaster recovery,
restore, and migrate Kubernetes resources and persistent volumes. In this recipe, we will
learn how to deploy Velero in our Kubernetes cluster by following these steps:  
1. Download the latest version of Velero:

```
$ wget https://github.com/vmware-tanzu/velero/releases/download/v1.5.1/velero-v1.5.1-linux-amd64.tar.gz
```
ps. At the time of writing this book, the latest version of Velero was v1.5.1.
Check the Velero repository at [https://github.com/vmware-tanzu/velero/] releases and update the link with the latest download link if it's
changed since this book's release.  

2. Extract the tarball:
```
$ tar -xvzf velero-v1.5.1-linux-amd64.tar.gz
$ sudo mv velero-v1.5.1-linux-amd64/velero /usr/local/bin/
```
3. Confirm that the velero command is executable:
```
$ velero version
Client:
	Version: v1.5.1
	Git commit: 87d86a45a6ca66c6c942c7c7f08352e26809426c
<error getting server version: no matches for kind "ServerStatusRequest" in version "velero.io/v1">
```
4. Create the credentials-velero file with the access key and secret key you used in the Configuring and managing S3 object storage using Minio recipe:
```
$ cat > credentials-velero <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```
5. Update the s3Url with the external IP of your MinIO service and install Velero Server:
```
$ velero install \
  --provider aws \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-restic \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://ac76d4a1ac72c496299b17573ac4cf2d-512600720.us-west-2.elb.amazonaws.com:9000
```  
or
```
$ kubectl apply -f ~/velero-v1.5.1-linux-amd64/examples/minio/00-minio-deployment.yaml
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.0.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-restic \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.16.35.17:9000 
```
or
```
$ kubectl apply -f ~/velero-v1.5.1-linux-amd64/examples/minio/00-minio-deployment.yaml
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.0.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-restic \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.16.35.17:9000 \
  --use-volume-snapshots=false \
  --snapshot-location-config region="default"
```
reference: 
*  https://velero.io/docs/v1.5/basic-install/
*  https://velero.io/docs/v1.5/supported-providers/
*  https://velero.io/docs/v1.5/contributions/minio/
6. Confirm that the deployment was successful:
```
$ kubectl   -n velero get deployments -l component=velero 
NAME    READY   UP-TO-DATE  AVAILABLE  AGE
velero  1/1     1           1           62s

$ kubectl   -n velero get deployments                     
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
minio    1/1     1            1           16h
velero   1/1     1            1           4m48s

$ kubectl logs deployment/velero -n velero
```  
With that, Velero has been configured on your Kubernetes cluster using MinIO as the
backup target.
##  Backing up an application
Let's perform the following steps to take a backup of an application and its volumes using
Velero. All the YAML manifest files we create here can be found under the
**/src/chapter6/velero** directory:  
1. If you have an application and volumes to back up labeled already, you can skip
to ***Step 5***. Otherwise, create a namespace and a PVC with the following
commands:  
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: backup-example
  labels:
    app: app2backup
EOF
```
2. Create a PVC in the **backup-example** namespace using your preferred
**storageClass** . In our example this is **aws-csi-ebs** :  
```
cat <<EOF | kubectl apply -f -
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
  storageClassName: aws-csi-ebs
  resources:
    requests:
      storage: 4Gi
EOF
```
3. Review the **myapp.yaml** file in the **src/chapter6/velero** directory and use it
to create a pod that will use the PVC and write to the **/data/out.txt** file inside
the pod:  
```
$ kubectl apply -f myapp.yaml
```
4. Verify that our **myapp** pod writes data to the volume:
```
$ kubectl -n  backup-example exec -it myapp cat /data/out.txt
Wed Oct 14 02:06:34 UTC 2020
```
5. Create a backup for all the objects with the **app=app2backup** label:  
```
$ velero backup create myapp-backup --selector app=app2backup
```
6. Confirm that the backup phase is completed:  
```
$ velero backup describe myapp-backup
Name: myapp-backup
Namespace: velero
Labels: velero.io/storage-location=default
Annotations: <none>
Phase: Completed
...
```
7. List all the available backups:
```
$ velero backup get
NAME          STATUS    CREATED                         EXPIRES   STORAGE LOCATION      SELECTOR
myapp-backup  Completed 2019-09-13 05:55:08 +0000 UTC   29d       default               app=app2backup
```
With that, you've learned how to create a backup of an application using labels.

## Restoring an application
Let's perform the following steps to restore the application from its backup:  
1. Delete the application and its PVC to simulate a data loss scenario:  
```
$ kubectl -n backup-example delete pvc pvc2backup
$ kubectl -n backup-example delete pod myapp
```
2. Restore your application from your previous backup called **myapp-backup** :  
```
$ velero restore create --from-backup myapp-backup
```
3. Confirm your application is running:
```
$ kubectl -n backup-example get pod
NAME  READY STATUS    RESTARTS  AGE
myapp 1/1   Running   0         10m
```

4. Confirm that our myapp pod writes data to the volume:  
```
$ kubectl -n backup-example exec -it myapp cat /data/out.txt
```
With that, you've learned how to restore an application and its volumes from its backup
using Velero.

## Creating a scheduled backup
Velero supports cron expressions to schedule backup tasks. Let's perform the following
steps to schedule backups for our application:  
1. Create a scheduled daily backup:
```
$ velero schedule create myapp-daily --schedule="0 0 1 * * ?" --selector app=app2backup
```
If you are not familiar with cron expressions, you can create a different schedule
using the Cron expression generator link in the See also section.  

ps.Note that the preceding schedule uses a cron expression. As an alternative, you can use a shorthand expression such as **--schedule="@daily"** or use an online cron maker to create a cron expression.

2. Get a list of the currently scheduled backup jobs:
```
$ velero schedule get
NAME        STATUS  CREATED                         SCHEDULE      BACKUP TTL LAST BACKUP   SELECTOR
myapp-daily Enabled 2019-09-13 21:38:36 +0000 UTC   0 0 1 * * ?   720h0m0s 2m ago          app=app2backup
```
3. Confirm that a backup has been created by the scheduled backup job:
```
$ velero backup get
NAME        			STATUS  	CREATED                        EXPIRES     STORAGE LOCATION    SELECTOR
myapp-daily-20190913205123  	Completed 	2019-09-13 20:51:24 +0000 UTC  29d         default             app=app2backup
```
With that, you've learned how to create scheduled backups of an application using Velero.

## Taking a backup of an entire namespace
When you take backups, you can use different types of selectors or even complete sources
in a selected namespace. In this recipe, we will include resources in a namespace by
performing the following steps:  
1. Take a backup of the entire namespace using the following command. This
example includes the backup-example namespace. Replace this namespace if
needed. The namespace and resources should exist before you can execute the
following command:
```
$ velero backup create fullnamespace --include-namespaces backup-example
```
2. If you need to exclude specific resources from the backup, add the backup:"false" label to them and run the following command:
```
$ velero backup create fullnamespace --selector 'backup notin (false)'
```
With that, you've learned how to create backups of resources in a given namespace using
Velero.
## Viewing backups with MinIO
Let's perform the following steps to view the content of the backups on the MinIO interface:  
1. Follow the instructions in the Accessing a MinIO web user interface recipe and
access the MinIO Browser.  
2. Click on the velero bucket:   
3. Open the backups directory to find a list of your Velero backups:  
4. Click on a backup name to access the content of the backup:  
With that, you've learned how to locate and review the content of Velero backups.
## Deleting backups and schedules
Velero backups can quickly grow in size if they're not maintained correctly. Let's perform
the following steps to remove an existing backup resource and clean up scheduled backups:
1. Delete the existing backup named myapp-backup :
```
$ velero backup delete myapp-backup
```
2. Delete all existing backups:
```
$ velero backup delete --all
```
3. Delete the scheduled backup job named myapp-daily :
```
$ velero schedule delete myapp-daily
```
## How it works...
This recipe showed you how to create disaster recovery backups, restore your application
and its data back from an S3 target, and how to create scheduled backup jobs on
Kubernetes.  
In the Backing up an application recipe, in Step 4, when you run velero backup create
myapp-backup --selector app=app2backup , the Velero client calls the Kubernetes API
server and creates a backup object.

ps. You can get a list of Custom Resource Definitions (CRDs) that have been
created by Velero by running the kubectl get crds |grep
velero command.

Velero's BackupController watches for a new object and when detected, it performs
standard validation and processes the backup. Velero's BackupController collects the
information to back up by asking resources from the API server. Then, it makes a call to the
default storage provider and uploads the backup files.

## See also
*  The Velero project repository, at https://github.com/vmware-tanzu/velero/
*  The Velero documentation, at https://velero.io/docs/master/index.html
*  Velero support matrix, at https://velero.io/docs/master/supported-providers/
*  Velero podcasts and community articles, at https://velero.io/resources/
*  Cron expression generator, at https://www.freeformatter.com/cron-expression-generator-quartz.html
