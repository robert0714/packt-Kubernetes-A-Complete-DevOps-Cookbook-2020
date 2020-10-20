# Monitoring suspicious application activities using Falco

Falco is a cloud-native runtime security toolset. Falco gains deep insight into system 
behavior through its runtime rule engine. It is used to detect intrusions and abnormalities 
in applications, containers, hosts, and the Kubernetes orchestrator.

In this section, we will cover the installation and basic usage of Falco on Kubernetes. 
## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest files in the chapter9 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter9
```
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.
## How to do it...
This section will show you how to configure and run Falco. This section is further divided 
into the following subsections to make this process easier:

* Installing Falco on Kubernetes
* Detecting anomalies using Falco
* Defining custom rules
## Installing Falco on Kubernetes
Falco can be installed in various ways, including directly on Linux hosts by deploying Falco 
as a DaemonSet or by using Helm. This recipe will show you how to install Falco as a DaemonSet.
Let's perform the following steps to get Falco deployed on our cluster:

1. Clone the Falco repository into your current working directory:
```
$ git clone https://github.com/falcosecurity/falco.git
$ cd falco && git checkout 0.22.1 && cd .. 
$ cd falco/integrations/k8s-using-daemonset/k8s-with-rbac
```
2. Create a Service Account for Falco. The following command will also create the ClusterRole and ClusterRoleBinding for it:
```
$ kubectl create -f falco-account.yaml
```
3. Create a service using the following command from the cloned repository location:
```
$ kubectl create -f falco-service.yaml
```
4. Create a config directory and copy the deployment configuration file and rule files in the config directory. We will need to edit these later:
```
$  mkdir config
$  cp ../../../falco.yaml config/
$  cp ../../../rules/falco_rules.* config/
$  cp ../../../rules/k8s_audit_rules.yaml config/
```
5. Create a ConfigMap using the config files in the config/ directory. 
Later, the DaemonSet will make the configuration available to Falco pods using the ConfigMap:
```
$ kubectl create configmap falco-config --from-file=config
```
6. Finally, deploy Falco using the following command:
```
$ kubectl create -f falco-daemonset-configmap.yaml
```
7. Verify that the DaemonSet pods have been successfully created. You should see one pod per schedulable worker node on the cluster. In our example, we used a 
Kubernetes cluster with four worker nodes:
```
$ kubectl get pods | grep
falco-daemonset-94p8w 1/1  Running 0 2m34s
falco-daemonset-c49v5 1/1  Running 0 2m34s 
falco-daemonset-htrxw 1/1  Running 0 2m34s
falco-daemonset-kwms5 1/1  Running 0 2m34s 
```
With that, Falco has been deployed and started monitoring behavioral activity to detect anomalous activities in our applications on our nodes.

## Detecting anomalies using Falco
Falco detects a variety of suspicious behavior. In this recipe, we will produce some 
activities that would be suspicious on a normal production cluster.

Let's perform the following steps to produce activities that would trigger a syscall event drop:

1. First, we need to review the full rules before we test some of the behaviors. Falco has two rules files. The default rules are located at
/etc/falco/falco_rules.yaml , while the local rules file is located at ***/etc/falco/falco_rules.local.yaml*** . Your custom rules and modifications
should be in the ***falco_rules.local.yaml*** file:
```
$ cat config/falco_rules.yaml
$ cat config/falco_rules.local.yaml
```

2. You will see a long list of default rules and macros. Some of them are as follows:
```
- rule: Disallowed SSH Connection
- rule: Launch Disallowed Container
- rule: Contact K8S API Server From Container
- rule: Unexpected K8s NodePort Connection
- rule: Launch Suspicious Network Tool in Container
- rule: Create Symlink Over Sensitive Files
- rule: Detect crypto miners using the Stratum protocol
```

3. Let's test that Falco is working by getting a bash shell into one of the Falco pods
and view the logs afterward. List the Falco pods:
$ kubectl get pods | grep
falco-daemonset-94p8w 1/1
falco-daemonset-c49v5 1/1
falco-daemonset-htrxw 1/1
falco-daemonset-kwms5 1/1
falco-daemonset
Running 0 2m34s
Running 0 2m34s
Running 0 2m34s
Running 0 2m34s
4. Get bash shell access to one of the Falco pods from the output of the preceding
command and view the logs:
$ kubectl exec -it falco-daemonset-94p8w bash
$ kubectl logs falco-daemonset-94p8w
