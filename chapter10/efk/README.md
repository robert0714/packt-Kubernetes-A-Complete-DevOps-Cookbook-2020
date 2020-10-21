# Building centralized logging in Kubernetes using the EFK stack

As described in the the Accessing Kubernetes logs locally section, basic logging can be used to 
detect configuration problems, but for cluster-level logging, an external backend is required 
to store and query logs. A cluster-level logging stack can help you quickly sort through and 
analyze the high volume of production log data that's produced by your application in the 
Kubernetes cluster. One of the most popular centralized logging solutions in the 
Kubernetes ecosystem is the Elasticsearch, ***Logstash, and Kibana (ELK)*** stack.

In the ELK stack, Logstash is used as the log collector. Logstash uses slightly more memory 
than Fluent Bit, which is a low-footprint version of Fluentd. Therefore, in this recipe, we 
will use the ***Elasticsearch, Fluent-bit, and Kibana (EFK)*** stack. If you have an application 
that has Logstash dependencies, you can always replace Fluentd/Fluent Bit with Logstash.

In this section, we will learn how to build a cluster-level logging system using the EFK 
stack to manage Kubernetes logs.

## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest 
files in the chapter10 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter10
```
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources.

## How to do it...
This section will show you how to configure an EFK stack on your Kubernetes cluster. This 
section is further divided into the following subsections to make this process easier: 

*  Deploying Elasticsearch Operator
*  Requesting an Elasticsearch endpoint
*  Deploying Kibana
*  Aggregating logs with Fluent Bit
*  Accessing Kubernetes logs on Kibana

## Deploying Elasticsearch Operator
Elasticsearch is a highly scalable open source full-text search and analytics engine.
Elasticsearch allows you to store, search, and analyze big volumes of data quickly. In this 
recipe, we will use it to store Kubernetes logs.

Let's perform the following steps to get Elastic Cloud on Kubernetes (ECK) deployed:

1. Deploy Elasticsearch Operator and its CRDs using the following command:
```
$ kubectl apply -f https://download.elastic.co/downloads/eck/1.0.0/all-in-one.yaml
```
2. Elasticsearch Operator will create its own CustomResourceDefinition (CRD).
We will use this CRD later to deploy and manage Elasticsearch instances on 
Kubernetes. List the new CRDs using the following command:
```
$ kubectl get crds |grep elastic.co
apmservers.apm.k8s.elastic.co                 2019-11-25T07:52:16Z
elasticsearches.elasticsearch.k8s.elastic.co  2019-11-25T07:52:17Z
kibanas.kibana.k8s.elastic.co                 2019-11-25T07:52:17Z
```
3. Create a new namespace called logging :
```
$ kubectl create ns logging
```
4. Create Elasticsearch using the default parameters in the logging namespace using the following command:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1beta1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: logging
spec:
  version: 7.4.2
  nodeSets:
  - name: default
    count: 3
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
EOF
5. Get the status of the Elasticsearch nodes:
$ kubectl get elasticsearch -n logging
NAME HEALTH NODES VERSION PHASE AGE
elasticsearch green 3 7.4.2 Ready 86s
6. You can also confirm the pod's status in the logging namespace using the
following command:
$ kubectl get pods -n logging
NAME
READY
elasticsearch-es-default-0 1/1
elasticsearch-es-default-1 1/1
elasticsearch-es-default-2 1/1
[ 508 ]
STATUS
Running
Running
Running
RESTARTS
0
0
0
AGE
2m24s
2m24s
2m24s
