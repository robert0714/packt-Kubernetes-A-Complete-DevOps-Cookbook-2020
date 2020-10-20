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
***/etc/falco/falco_rules.yaml*** (https://github.com/falcosecurity/falco/blob/master/rules/falco_rules.yaml) , while the local rules file is located at ***/etc/falco/falco_rules.local.yaml*** (https://github.com/falcosecurity/falco/blob/master/rules/falco_rules.local.yaml) . Your custom rules and modifications
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

3. Let's test that Falco is working by getting a bash shell into one of the Falco pods and view the logs afterward. List the Falco pods:
```
$ kubectl get pods |grep falco-daemonset
falco-daemonset-446h5            1/1     Running   0          79m
falco-daemonset-5bqtc            1/1     Running   0          79m
falco-daemonset-c6tz6            1/1     Running   0          79m
falco-daemonset-sxxl8            1/1     Running   0          79m
```
4. Get bash shell access to one of the Falco pods from the output of the preceding
command and view the logs:
```
$ kubectl exec -it falco-daemonset-446h5 bash
$ kubectl logs falco-daemonset-446h5
```
5. In the logs, you will see that Falco detects our shell access to the pods:
```
{"output":"00:58:23.798345403: Notice A shell was spawned in a
container with an attached terminal (user=root k8s.ns=default
k8s.pod=falco-daemonset-94p8w container=0fcbc74d1b4c shell=bash
parent=docker-runc cmdline=bash terminal=34816
container_id=0fcbc74d1b4c image=falcosecurity/falco) k8s.ns=default
k8s.pod=falco-daemonset-94p8w container=0fcbc74d1b4c k8s.ns=default
k8s.pod=falco-daemonset-94p8w
container=0fcbc74d1b4c","priority":"Notice","rule":"Terminal shell
in container","time":"2019-11-13T00:58:23.798345403Z",
"output_fields":
{"container.id":"0fcbc74d1b4c","container.image.repository":"falcos
ecurity/falco","evt.time":1573606703798345403,"k8s.ns.name":"defaul
t","k8s.pod.name":"falco-
daemonset-94p8w","proc.cmdline":"bash","proc.name":"bash","proc.pna
me":"docker-runc","proc.tty":34816,"user.name":"root"}}
With that, you've learned how to use Falco to detect anomalies and suspicious behavior.
```
With that, you've learned how to use Falco to detect anomalies and suspicious behavior.

## Defining custom rules
Falco rules can be extended by adding our own rules. In this recipe, we will deploy a 
simple application and create a new rule to detect a malicious application accessing our 
database.

Perform the following steps to create an application and define custom rules for Falco:

1. Change to the src/chapter9/falco directory, which is where our examples are located:
```
$ cd src/chapter9/falco
```
2. Create a new falcotest namespace:
```
$ kubectl create ns falcotest
```
3. Review the YAML manifest and deploy them using the following commands. These commands will create a MySQL pod, web application, a client that we will
use to ping the application, and its services:
```
$ kubectl create -f mysql.yaml
$ kubectl create -f ping.yaml
$ kubectl create -f client.yaml
```
4. Now, use the client pod with the default credentials of bob/foobar to send a 
ping to our application. As expected, we will be able to authenticate and 
complete the task successfully:
```
$ kubectl exec client -n falcotest -- curl -F "s=OK" -F "user=bob" -F "passwd=foobar" -F "ipaddr=localhost" -X POST http://ping/ping.php
```
5. Edit the falco_rules.local.yaml file:
```
$ vim config/falco_rules.local.yaml
```
6. Add the following rule to the end of the file and save it:
```
- rule: Unauthorized process
  desc: There is a running process not described in the base template
  condition: spawned_process and container and k8s.ns.name=falcotest and k8s.deployment.name=ping and not proc.name in (apache2, sh, ping)
  output: Unauthorized process (%proc.cmdline) running in (%container.id)
  priority: ERROR
  tags: [process]
```
7. Update the ConfigMap that's being used for the DaemonSet and delete the pods to get a new configuration by running the following command:
```
$ kubectl delete -f falco-daemonset-configmap.yaml
$ kubectl create configmap falco-config --from-file=config --dry-run --save-config -o yaml | kubectl apply -f -
$ kubectl apply -f falco-daemonset-configmap.yaml
```
8. We will execute a SQL injection attack and access the file where our MySQL credentials are stored. Our new custom rule should be able to detect it:
```
$ kubectl exec client -n falcotest -- curl -F "s=OK" -F "user=bad" -F "passwd=wrongpasswd' OR 'a'='a" -F "ipaddr=localhost; cat /var/www/html/ping.php" -X POST http://ping/ping.php
```
9. The preceding command will return the content of the PHP file. You will be able to find the MySQL credentials there:
```
3 packets transmitted, 3 received, 0% packet loss, time 2044ms
rtt min/avg/max/mdev = 0.028/0.035/0.045/0.007 ms
<?php  $link = mysqli_connect("mysql", "root", "foobar", "employees"); ?>
```
10. List the Falco pods:
```
$ kubectl get pods |grep falco-daemonset
falco-daemonset-446h5            1/1     Running   0          79m
falco-daemonset-5bqtc            1/1     Running   0          79m
falco-daemonset-c6tz6            1/1     Running   0          79m
falco-daemonset-sxxl8            1/1     Running   0          79m
```
11. View the logs from a Falco pod:
```
$ kubectl exec -it falco-daemonset-446h5 bash
$ kubectl logs falco-daemonset-446h5
```
12. In the logs, you will see that Falco detects our shell access to the pods:
```
05:41:59.9275580001: Error Unauthorized process (cat
/var/www/html/ping.php) running in (5f1b6d304f99) k8s.ns=falcotest
k8s.pod=ping-74dbb488b6-6hwp6 container=5f1b6d304f99
```
With that, you know how to add custom rules using Kubernetes metadata such as ***k8s.ns.name*** and ***k8s.deployment.name*** . You can also use other filters. This is described in more detail in the Supported filters link in See also section.

## How it works...
This recipe showed you how to detect anomalies based on the predefined and custom rules 
of your applications when they're running on Kubernetes.

In ***the Installing Falco on Kubernetes recipe, in Step 5***, we created a ConfigMap to be used by 
the Falco pods. Falco has two types of rules files.

In Step 6, when we created the DaemonSet, all the default rules are provided through the 
***falco_rules.yaml*** file in the ConfigMap.These are placed in 
***/etc/falco/falco_rules.yaml*** inside the pods, while the local rules file 
, ***falco_rules.local.yaml***, can be found at ***/etc/falco/falco_rules.local.yaml*** .

The default rules file contains rules for many common anomalies and threats. All pieces of 
customization must be added to the ***falco_rules.local.yaml*** file, which we did in the 
***Defining custom rules*** recipe.

In the ***Defining custom rules*** recipe, in ***Step 6***, we created a custom rule file containing the 
***rules*** element. The Falco rule file is a YAML file that uses three kinds of elements: ***rules*** , 
***macros*** , and ***lists*** .

The rules define certain conditions to send alerts about them. A rule is a file that contains at 
least the following keys:

*  rule : The name of the rule
*  condition : An expression that's applied to events to check if they match the rule
*  desc : Detailed description of what the rule is used for
*  output : The message that is displayed to the user
*  priority : Either emergency, alert, critical, error, warning, notice, informational, or debug

You can find out more about these rules by going to the Understanding Falco Rules link that's provided in the See also section.

## See also
*  Falco documentation: https://falco.org/docs/
*  Falco repository and integration examples: https://github.com/falcosecurity/falco
*  Understanding Falco Rules: https://falco.org/dochttps://falco.org/docs/rules/s/rules/
*  Comparing Falco with other tools: https://sysdig.com/blog/selinux-seccomp-falco-technical-discussion/
*  Supported filters: https://github.com/draios/sysdig/wiki/Sysdig-User-Guide#all-supported-filters
