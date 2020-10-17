# Using Kubernetes CIS Benchmark for security auditing
Kubernetes CIS Benchmarks are the security configuration best practices that are accepted 
by industry experts. The CIS Benchmark guide can be download as a PDF file from the 
***Center for Internet Security (CIS)*** website at https://www.cisecurity.org/.kube-bench 
is an application that automates documented checks.

In this section, we will cover the installation and use of the open source *kube-bench* tool to 
run Kubernetes CIS Benchmarks for security auditing of Kubernetes clusters.

## Getting ready
For this recipe, we need to have a Kubernetes cluster ready and the Kubernetes command-line tool kubectl installed.

Clone the k8sdevopscookbook/src repository to your workstation to use the manifest files in the chapter9 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter9/cis
```

Some of the tests target Kubernetes nodes and can only be executed on fully self-managed 
clusters where you have control over the master nodes. Therefore, managed clusters such 
as EKS, GKE, AKS, and so on will not be able to execute all the tests and require different 
job descriptions or parameters to execute the tests. These will be mentioned when 
necessary.

## How to do it...
This section is further divided into the following subsections to make this process easier:
*  Running kube-bench on Kubernetes
*  Running kube-bench on managed Kubernetes services
*  Running kube-bench on OpenShift
*  Running kube-hunter

## Running kube-bench on Kubernetes
The CIS Benchmark has tests for both master and worker nodes. Therefore, the full scope of 
the test can only be completed on self-managed clusters where you have control over the 
master nodes. In this recipe, you will learn how to run kube-bench directly on the master 
and worker nodes.

Let's perform the following steps to run the CIS recommended tests:  
1. Download and install the kube-bench command-line interface on one of your master nodes and one of your worker nodes:
```
$ curl --silent --location "https://github.com/aquasecurity/kube-bench/releases/download/v0.4.0/kube-bench_0.4.0_linux_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/kube-bench /usr/local/bin
```
2. SSH into your Kubernetes master node and run the following command. It will 
quickly return the result of the test with an explanation and a list of additional 
manual tests that are recommended to be run after. Here, you can see that 31 
checks passed and 36 tests failed:
```
$ kube-bench master
...
== Summary ==
31 checks PASS
36 checks FAIL
24 checks WARN
1 checks INFO
```
3. To save the results, use the following command. After the test is complete, move 
the kube-bench-master.txt file to your localhost for further review:
```
$ kube-bench master > kube-bench-master.txt
```
4. Review the content of the kube-bench-master.txt file. You will see the status 
of the checks from the CIS Benchmark for the Kubernetes guide, similar to the 
following:
```
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
[PASS] 1.1.1 Ensure that the --anonymous-auth argument is set to false (Not Scored)
[FAIL] 1.1.2 Ensure that the --basic-auth-file argument is not set (Scored)
[PASS] 1.1.3 Ensure that the --insecure-allow-any-token argument is not set (Not Scored)
[PASS] 1.1.4 Ensure that the --kubelet-https argument is set to true (Scored)
[FAIL] 1.1.5 Ensure that the --insecure-bind-address argument is not set (Scored)
[FAIL] 1.1.6 Ensure that the --insecure-port argument is set to 0 (Scored)
[PASS] 1.1.7 Ensure that the --secure-port argument is not set to 0 (Scored)
[FAIL] 1.1.8 Ensure that the --profiling argument is set to false (Scored)
[FAIL] 1.1.9 Ensure that the --repair-malformed-updates argument is set to false (Scored)
[PASS] 1.1.10 Ensure that the admission control plugin AlwaysAdmit is not set (Scored)
...
```

Tests are split into categories that have been suggested in the CIS Benchmark 
guidelines, such as API Server, Scheduler, Controller Manager, Configuration 
Manager, etcd, General Security Primitives, and PodSecurityPolicies.

5. Follow the methods suggested in the Remediations section of the report to fix the 
failed issues and rerun the test to confirm that the correction has been made. You 
can see some of the remediations that were suggested by the preceding report 
here:
```
== Remediations ==
1.1.2 Follow the documentation and configure alternate mechanisms for authentication. Then,edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.manifest
on the master node and remove the --basic-auth-file=<filename> parameter.

1.1.5 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.manife$ on the master node and remove the --insecure-bind-address parameter.

1.1.6 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.manife$ apiserver.yaml on the master node and set the below parameter. --insecure-port=0

1.1.8 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.manife$ on the master node and set the below parameter. --profiling=false

1.2.1 Edit the Scheduler pod specification file /etc/kubernetes/manifests/kube-scheduler.manifest file on the master node and set the below parameter. --profiling=false
...
```
6. Let's take one of the issues from the preceding list. 1.2.1 suggests that we 
disable the profiling API endpoint. The reason for this is that highly 
sensitive system information can be uncovered by profiling data and the amount 
of data and load that's created by profiling your cluster could be put out of 
service (denial-of-service attack) by this feature.
Edit the *kube-scheduler.manifest* file and add *--profiling=false* right after the *kube-schedule* command, as shown in the following code:
```
...
spec:
  containers:
    - command:
      - /bin/sh
      - -c
      - mkfifo /tmp/pipe; (tee -a /var/log/kube-scheduler.log < /tmp/pipe & ) ; exec /usr/local/bin/kube-scheduler --profiling=False --kubeconfig=/var/lib/kube-scheduler/kubeconfig --leader-elect=true --v=2 > /tmp/pipe 2>&1
...
```
7. Run the test again and confirm that the issue on 1.2.1 has been corrected. Here, 
you can see that the number of passed tests has increased from 31 to 32 . One 
more check has been cleared:
```
$ kube-bench master
...
== Summary ==
32 checks PASS
35 checks FAIL
24 checks WARN
1 checks INFO
```
8. Run the test on the worker nodes by using the following command:
```
$ kube-bench node
...
== Summary ==
9 checks PASS
12 checks FAIL
2 checks WARN
1 checks INFO
```
9. To save the results, use the following command. After the test has 
completed, move the kube-bench-worker.txt file to your localhost for further 
review:
```
$ kube-bench node > kube-bench-worker.txt
```

10. Review the content of the kube-bench-worker.txt file. You will see the status 
of the checks from the CIS Benchmark for the Kubernetes guide, similar to the 
following:
```
[INFO] 2 Worker Node Security Configuration
[INFO] 2.1 Kubelet
[PASS] 2.1.1 Ensure that the --anonymous-auth argument is set to false (Scored)
[FAIL] 2.1.2 Ensure that the --authorization-mode argument is not set to AlwaysAllow (Scored)
[PASS] 2.1.3 Ensure that the --client-ca-file argument is set as appropriate (Scored)
...
```
Similarly, follow all the remediations until you've cleared all the failed tests on the master and worker nodes.

## Running kube-bench on managed Kubernetes services
The difference between managed Kubernetes services such as EKS, GKE, AKS, and so on is 
that you can't run the checks on the master. Instead, you have to either only follow the 
worker checks from the previous recipe or run a Kubernetes job to validate your 
environment. In this recipe, you will learn how to run kube-bench on managed Kubernetes 
service-based nodes and also in cases where you don't have direct SSH access to the nodes. 

Let's perform the following steps to run the CIS recommended tests:

1. For this recipe, we will use EKS as our Kubernetes service, but you can change 
the Kubernetes and container registry services to other cloud providers if you 
wish. First, create an ECR repository where we will host the kube-bench image:
```
$ aws ecr create-repository --repository-name k8sdevopscookbook/kube-bench --image-tag-mutability MUTABLE
```
2. Clone the kube-bench repository to your localhost:
```
$ git clone https://github.com/aquasecurity/kube-bench.git
```
3. Log in to your Elastic Container Registry (ECR) account. You need to be authenticated before you can push images to the registry:
```
$ $(aws ecr get-login --no-include-email --region us-west-2)
```
4. Build the kube-bench image by running the following command:
```
$ docker build -t k8sdevopscookbook/kube-bench
```
5. Replace <AWS_ACCT_NUMBER> with your AWS account number and execute it to 
push it to the ECR repository. The first command will create a tag, while the 
second command will push the image:
```
$ docker tag k8sdevopscookbook/kube-bench:latest
<AWS_ACCT_NUMBER>.dkr.ecr.us-west-2.amazonaws.com/k8s/kube-bench:latest
# docker push <AWS_ACCT_NUMBER>.dkr.ecr.us-west-2.amazonaws.com/k8s/kube-bench:latest
```
6. Edit the job-eks.yaml file and replace the image name on line 12 with the URI 
of the image you pushed in Step 5. It should look similar to the following, except 
you should use your AWS account number in the image URI: 
```
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      hostPID: true
      containers:
      - name: kube-bench
        # Push the image to your ECR and then refer to it here
        image: 316621595343.dkr.ecr.us-west-2.amazonaws.com/k8sdevopscookbook/kube-bench:latest
...
```
7. Run the job using the following command. It will be executed and completed shortly:
```
$ kubectl apply -f job-eks.yaml
```
8. List the kube-bench pods that were created in your cluster. It should show 
Completed as the status, similar to the following example:
```
$ kubectl get pods |grep kube-bench
kube-bench-7lxzn 0/1 Completed 0 5m
```
9. Replace the pod name with the output of the previous command and view the 
pod logs to retrieve the kube-bench results. In our example, the pod name is 
kube-bench-7lxzn :
```
$ kubectl logs kube-bench-7lxzn
```
## Running kube-bench on OpenShift
OpenShift has different command-line tools, so if we run the default test jobs, we won't be able to gather the required information on our cluster unless specified. In this recipe, you
will learn how to run kube-bench on OpenShift.

Let's perform the following steps to run the CIS recommended tests:

1. SSH into your OpenShift master node and run the following command using --version ocp-3.10 or ocp-3.11 based on your OpenShift version. Currently, 
only 3.10 and 3.11 are supported:
```
$ kube-bench master --version ocp-3.11
```
2. To save the results, use the following command. After the test has been 
completed, move the kube-bench-master.txt file to your localhost for further 
review:
```
$ kube-bench master --version ocp-3.11 > kube-bench-master.txt
```
3. SSH into your OpenShift worker node and repeat the first two steps of this 
recipe, but this time using the node parameter for the OpenShift version you are 
running. In our example, this is OCP 3.11:
```
$ kube-bench node --version ocp-3.11 > kube-bench-node.txt
```
Follow the Running kube-bench on Kubernetes recipe's instructions to patch security issues with the suggested remediations.

## How it works...
This recipe showed you how to quickly run CIS Kubernetes Benchmarks on your cluster 
using kube-bench.

In the Running kube-bench on Kubernetes recipe, in step 1, after you executed the checks, 
kube-bench accessed the configuration files that were kept in the following directories: 
*/var/lib/etcd* , */var/lib/kubelet* , */etc/systemd* , */etc/kubernetes* , and 
*/usr/bin* . Therefore, the user who runs the checks needs to provide root/sudo access to all the config files.

If the configuration files can't be found in their default directories, the checks will fail. The 
most common issue is the missing kubectl binary in the ***/usr/bin*** directory. kubectl is 
used to detect the Kubernetes version. You can skip this directory by specifying the 
Kubernetes version using ***--version*** as part of the command, similar to the following:
```
$ kube-bench master --version 1.14
```
Step 1 will return four different states. The ***PASS*** and ***FAIL*** states are self-explanatory as 
they indicate whether the tests were run successfully or failed. WARN indicates that the test 
requires manual validation, which means it requires attention. Finally, INFO means that no 
further action is required.

## See also
*  CIS Kubernetes Benchmarks: https://www.cisecurity.org/benchmark/kubernetes/
*  kube-bench repository: https://github.com/aquasecurity/kube-bench
*  How to customize the default configuration: https://github.com/aquasecurity/kube-bench/blob/master/docs/README.md#configuration-and-variables
*  Automating compliance checking for Kubernetes-based applications: https://github.com/cds-snc/security-goals
*  Hardening Kubernetes from Scratch: https://github.com/hardening-kubernetes/from-scratch
*  CNCF Blog on 9 Kubernetes Security Best Practices Everyone Must Follow:https://www.cncf.io/blog/2019/01/14/9-kubernetes-security-best-practices-everyone-must-follow/
*  Hardening Guide for Rancher https://rancher.com/docs/rancher/v2.x/en/security/hardening-2.2/
*  Must-have Kubernetes security audit tools:
    *  Kube-bench: https://github.com/aquasecurity/kube-bench
    *  Kube-hunter: https://kube-hunter.aquasec.com/
    *  Kubeaudit: https://github.com/Shopify/kubeaudit
    *  Kubesec: https://github.com/controlplaneio/kubesec
    *  Open Policy Agent: https://www.openpolicyagent.org/
    *  K8Guard: https://k8guard.github.io/
