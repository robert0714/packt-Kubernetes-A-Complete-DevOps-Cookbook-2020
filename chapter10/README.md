# Logging with Kubernetes
In this chapter, we will discuss cluster logging for Kubernetes clusters. We will talk about 
setting up a cluster to ingest logs, as well as how to view them using both self-managed 
and hosted solutions.

In this chapter, we will cover the following recipes:

* Accessing Kubernetes logs locally
* Accessing application-specific logs
* Building centralized logging in Kubernetes using the EFK stack
* Logging with Kubernetes using Google Stackdriver
* Using a managed Kubernetes logging service
* Logging for your Jenkins CI/CD environment

# Technical requirements
The recipes in this chapter expect you to have a functional Kubernetes cluster deployed by 
following one of the recommended methods described in Chapter 1 , Building Production-Ready Kubernetes Clusters.

The Logging for your Jenkins CI/CD environment recipe in this chapter expects you to have a 
functional Jenkins server with an existing CI pipeline created by following one of the 
recommended methods described in Chapter 3 , Building CI/CD Pipelines.

The Kubernetes command-line tool kubectl will be used for the rest of the recipes in this 
chapter since it's the main command-line interface for running commands against 
Kubernetes clusters. We will also use helm where Helm charts are available in order to 
deploy solutions.

## Accessing Kubernetes logs locally
In Kubernetes, logs can be used for debugging and monitoring activities to a certain level. 
Basic logging can be used to detect configuration problems, but for cluster-level logging, an 
external backend is required to store and query logs. Cluster-level logging will be covered 
in the Building centralized logging in Kubernetes using the EFK stack and Logging Kubernetes 
using Google Stackdriver recipes.

In this section, we will learn how to access basic logs based on the options that are available in Kubernetes.

## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest files in the chapter10 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter10
```
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

## How to do it...
This section is further divided into the following subsections to make this process easier:
*  Accessing logs through Kubernetes
*  Debugging services locally using Telepresence

## Accessing logs through Kubernetes
This recipe will take you through how to access Kubernetes logs and debug services locally.

Let's perform the following steps to view logs by using the various options that are
available in Kubernetes:
1. Get the list of pods running in the kube-system namespace. The pods running 
in this namespace, especially kube-apiserver , kube-controller-manager , 
kube-dns , and kube-scheduler , play a critical role in the Kubernetes control plane:
```
$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-659f4b66fd-fhlpc   1/1     Running   2          6d7h
calico-node-4thwn                          1/1     Running   2          6d7h
calico-node-pm8q8                          1/1     Running   2          6d7h
calico-node-r4wkm                          1/1     Running   2          6d7h
calico-node-rg4lq                          1/1     Running   2          6d7h
calico-node-z2zxw                          1/1     Running   2          6d7h
coredns-5d4dd4b4db-6jcbm                   1/1     Running   2          6d7h
coredns-5d4dd4b4db-tlcll                   1/1     Running   2          6d7h
etcd-k8s-m1                                1/1     Running   2          6d7h
kube-apiserver-k8s-m1                      1/1     Running   3          6d4h
kube-controller-manager-k8s-m1             1/1     Running   5          6d7h
kube-proxy-5lnt5                           1/1     Running   2          6d7h
kube-proxy-d4dn4                           1/1     Running   2          6d7h
kube-proxy-l9xkx                           1/1     Running   2          6d7h
kube-proxy-lnzdf                           1/1     Running   2          6d7h
kube-proxy-nkks6                           1/1     Running   2          6d7h
kube-scheduler-k8s-m1                      1/1     Running   5          6d7h
kube-state-metrics-74b87488f-vdjmm         1/1     Running   2          5d8h
metrics-server-65779b66d-qnfgg             1/1     Running   2          5d8h
node-problem-detector-v0.1-7vtcl           1/1     Running   2          6d3h
node-problem-detector-v0.1-8nn6c           1/1     Running   2          6d3h
node-problem-detector-v0.1-bg9hp           1/1     Running   2          6d3h
node-problem-detector-v0.1-tl9kn           1/1     Running   2          6d3h
tiller-deploy-74494fcb9c-nm2b9             1/1     Running   2          6d7h
```
2. View the logs from a pod with a single container in the kube-system namespace. In this example, this pod is kube-apiserver . Replace the pod's
name and repeat this for the other pods as needed:
```
$ kubectl -n kube-system  logs  kube-apiserver-k8s-m1 
....(ommit)
```
As shown in the preceding output, you can find the time, source, and a short explanation of the event in the logs.

PS>  
The output of the logs can become long, though most of the time all you 
need is the last few events in the logs. If you don't want to get all the logs 
since you only need the last few events in the log, you can add ***-tail*** to 
the end of the command, along with the number of lines you want to look 
at. For example, ***kubectl logs <podname> -n <namespace> -tail 10***  would return the last 10 lines. Change the number as needed to limit
the output.

3. Pods can contain multiple containers. When you list the pods, the numbers under 
the Ready column show the number of containers inside the pod. Let's view a 
specific container log from a pod with multiple containers in the kube-system 
namespace. Here, the pod we're looking at is called  ***kube-dns*** . Replace the pod's 
name and repeat this for any other pods with multiple containers:
```
$ $ kubectl -n kube-system logs coredns-5d4dd4b4db-6jcbm 
.:53
2020-10-20T07:18:01.827Z [INFO] CoreDNS-1.3.1
2020-10-20T07:18:01.827Z [INFO] linux/amd64, go1.11.4, 6b56a9c
CoreDNS-1.3.1
linux/amd64, go1.11.4, 6b56a9c
2020-10-20T07:18:01.827Z [INFO] plugin/reload: Running configuration MD5 = 5d5369fbc12f985709b924e721217843
``` 
4. To view the logs after a specific time, use the --since-time parameter with a 
date, similar to what can be seen in the following code. You can either use an 
absolute time or request a duration. Only the logs after the specified time or 
within the duration will be displayed:
```
$ kubectl -n kube-system logs  coredns-5d4dd4b4db-6jcbm kubedns --since-time="2019-11-14T04:59:40.417Z"
...
I1114 05:09:13.309614 1 dns.go:601] Could not find endpoints for
service "minio" in namespace "default". DNS records will be created
once endpoints show up.
```
5. Instead of pod names, you can also view logs by label. Here, we're listing pods 
using the k8s-app=kube-dns label. Since the pod contains multiple containers, 
we can use the -c kubedns parameter to set the target container:
```
$ kubectl -n kube-system logs -l k8s-app=kube-dns -c kubedns
```
6. If the container has crashed or restarted, we can use the -p flag to retrieve logs from a previous instantiation of a container, as follows:
```
$ kubectl -n kube-system logs -l k8s-app=kube-dns -c kubedns -p
```
Now you know how to access pod logs through Kubernetes.

## Debugging services locally using Telepresence
When a build fails in your CI pipeline or a service running in a staging cluster contains a 
bug, you may need to run the service locally to troubleshoot it properly. However, 
applications depend on other applications and services on the cluster; for example, a 
database. Telepresence helps you run your code locally, as a normal local process, and then 
forwards requests to the Kubernetes cluster. This recipe will show you how to debug 
services locally while running a local Kubernetes cluster.

Let's perform the following steps to view logs through the various options that are 
available in Kubernetes:

1. On OSX, install the Telepresence binary using the following command:
```
$ brew cask install osxfuse
$ brew install datawire/blackbird/telepresence
```
On Windows, use Ubuntu on the Windows Subsystem for Linux (WSL). Then, 
on Ubuntu, download and install the Telepresence binary using the following 
command:
```
$ curl -s https://packagecloud.io/install/repositories/datawireio/telepresence/script.deb.sh | sudo bash
$ sudo apt install --no-install-recommends telepresence
```
2. Now, create a deployment of your application. Here, we're using a hello-world example:
```
$ kubectl run hello-world --image=datawire/hello-world --port=8000
```
3. Expose the service using an external LoadBalancer and get the service IP:
```
$ kubectl expose deployment hello-world --type=LoadBalancer --name=hello-world
$ kubectl get service hello-world 
NAME        TYPE            CLUSTER-IP        EXTERNAL-IP                                     PORT(S)          AGE
hello-world LoadBalancer    100.71.246.234    a643ea7bc0f0311ea.us-east-1.elb.amazonaws.com   8000:30744/TCP   8s
```
4. To be able to query the address, store the address in a variable using the following command:
```
$ export HELLOWORLD=http://$(kubectl get svc hello-world -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'):8000
```
5. Send a query to the service. This will return a Hello, world! message similar to the following:
```
$ curl $HELLOWORLD/
Hello, world!
```
6. Next, we will create a local web service and replace the Kubernetes service  hello-world message with the local web server service. First, create a directory
and a file to be shared using the HTTP server:
```
$ mkdir /tmp/local-test && cd /tmp/local-test
$ echo "hello this server runs locally on my laptop" > index.html
```
7. Create a web server and expose the service through port 8000 using the following command:
```
$ telepresence --swap-deployment hello-world --expose 8000 \
--run python3 -m http.server 8000 &
...
T: Forwarding remote port 8000 to local port 8000.
T: Guessing that Services IP range is 100.64.0.0/13. Services
started after this point will be inaccessible if are outside
T: this range; restart telepresence if you can't access a new
Service.
T: Setup complete. Launching your command.
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
The preceding command will start a proxy using the vpn-tcp method. Other
methods can be found in the Full list of Telepresence methods link in the See also
section.

PS>>   
When a service is exposed over the network, remember that your 
computer is exposed to all the risks of running a web server. When you 
expose a web service using the commands described here, make sure that 
you don't have any important files in the /tmp/local-test directory 
that you don't want to expose externally.

8. Send a query to the service. You will see that queries to the hello-world Service
will be forwarded to your local web server:
```
$ curl $HELLOWORLD/
hello this server runs locally on my laptop
```
9. To end the local service, use the fg command to bring the background 
Telepresence job in the current shell environment into the foreground. Then, use 
the Ctrl + C keys to exit it.

## How it works...
In this recipe, you learned how to access logs and debug service problems locally.
In the Debugging services locally using Telepresence recipe, in Step 7, we ran
the telepresence --swap-deployment command to replace the service with a local web
service.

Telepresence functions by building a two-way network proxy. The --swap-deployment 
flag is used to define the pod that will be replaced with a proxy pod on the cluster.
Telepresence starts a vpn-tcp process to send all requests to the locally exposed port, that 
is, 8000 . The --run python3 -m http.server 8000 & flag tells Telepresence to run 
an http.server using Python 3 in the background via port 8000 .

In the same recipe, in Step 9, the fg command is used to move the background service to
the foreground. When you exit the service, the old pod will be restored. You can learn 
about how Telepresence functions by looking at the How Telepresence works link in the See 
also section.

## See also
*  kubectl log commands: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs
*  Telepresence source code repository: https://github.com/telepresenceio/telepresence
*  Full list of Telepresence methods: https://telepresence.io/reference/methods.html
*  How Telepresence works: https://www.telepresence.io/discussion/how-it-works
*  How to use volume access support with Telepresence: https://telepresence.io/howto/volumes.html

  
# Accessing application-specific logs
In Kubernetes, pod and deployment logs that are related to how pods and containers are 
scheduled can be accessed through the kubectl logs command, but not all application 
logs and commands are exposed through Kubernetes APIs. Getting access to these logs and 
shell commands inside a container may be required.

In this section, we will learn how to access a container shell, extract logs, and update 
binaries for troubleshooting.

## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest 
files under the chapter10 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter10
```
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.

## How to do it...
This section is further divided into the following subsections to make this process easier:
*  Getting shell access in a container
*  Accessing PostgreSQL logs inside a container

## Getting shell access in a container
Let's perform the following steps to create a deployment with multiple containers and get a shell into running containers:

1. In this recipe, we will deploy PostgreSQL on OpenEBS persistent volumes to 
demonstrate shell access. Change the directory to the example files directory 
in ***src/chapter10/postgres*** , which is where all the YAML manifest for this 
recipe are stored. Create a ***ConfigMap*** with a database name and credentials 
similar to the following or review them and use the cm-postgres.yaml file:

```
$ cd postgres
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  labels:
    app: postgres
data:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: testuser
  POSTGRES_PASSWORD: testpassword123
EOF
```
2. Create the service for postgres :
```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: postgres
EOF
```
3. Review the postgres.yaml file and apply it to create the PostgreSQL 
StatefulSet. We can use this to deploy the pods and to auto-create the PV/PVC:
```
$ kubectl apply -f postgres.yaml
```
4. Get the pods with the postgres label:
```
$ kubectl get pods -l app=postgres
NAME        READY   STATUS    RESTARTS  AGE
postgres-0  1/1     Running   0         7m5s
postgres-1  1/1     Running   0         6m58s
```
5. Get a shell into the postgres-0 container:
```
$ kubectl exec -it postgres-0 -- /bin/bash
```
The preceding command will get you shell access to the running container.

## Accessing PostgreSQL logs inside a container
Let's perform the following steps to get the logs from the application running inside a container:

1. While you are in a shell, connect to the PostgreSQL database named postgresdb using the username testuser . You will see the PostgreSQL prompt, as follows:
```
$ psql --username testuser postgresdb
psql (12.1 (Debian 12.1-1.pgdg100+1))
Type "help" for help.
```
2. While on the PostgreSQL prompt, use the following command to create a table and add some data to it:
```
CREATE TABLE test (
  id int GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  a int NOT NULL,
  created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO test (a) SELECT * FROM generate_series(-1, -1000, -1);
```
3. Get the log's configuration details from ***postgresql.conf*** . You will see that the logs are stored in the ***/var/log/postgresql*** directory:
```
$ cat /var/lib/postgresql/data/postgresql.conf |grep log
```
4. List and access the logs in the /var/log/postgresql directory:
```
$ ls /var/log/postgresql
```
5. Optionally, while you're inside the container, you can create a backup of our example postgresdb database in the tmp directory using the following command:
```
$ pg_dump --username testuser postgresdb > /tmp/backup.sql
```
With that, you have learned how to get shell access into a container and how to access the locally stored logs and files inside the container.
