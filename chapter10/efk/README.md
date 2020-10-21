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
```
5. Get the status of the Elasticsearch nodes（ You need to wait for a while） :
```
$ kubectl get elasticsearch -n logging
NAME            HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch   green    3       7.4.2     Ready   23m
```
6. You can also confirm the pod's status in the logging namespace using the
following command:
```
$ kubectl get pods -n logging
NAME                         READY   STATUS    RESTARTS   AGE
elasticsearch-es-default-0   1/1     Running   0          24m
elasticsearch-es-default-1   1/1     Running   0          24m
elasticsearch-es-default-2   1/1     Running   0          24m
```
A three-node Elasticsearch cluster will be created. By default, the nodes we created here are all of the following types: master-eligible, data, and ingest. As your Elasticsearch cluster grows, it is recommended to create dedicated master-eligible, data, and ingest nodes.

## Requesting the Elasticsearch endpoint
When an Elasticsearch cluster is created, a default user password is generated and stored in a Kubernetes secret. You will need the full credentials to request the Elasticsearch endpoint.

Let's perform the following steps to request Elasticsearch access:

1. Get the password that was generated for the default elastic user:
```
$ PASSWORD=$(kubectl get secret elasticsearch-es-elastic-user \
-n logging -o=jsonpath='{.data.elastic}' | base64 --decode)
```
2. Request the Elasticsearch endpoint address (notice elasticsearch-es-http is remote domain name):
```
$ curl -u "elastic:$PASSWORD" -k "https://elasticsearch-es-http:9200"  \
  { \
    "name" : "elasticsearch-es-default-2",  \
    "cluster_name" : "elasticsearch",  \
    "cluster_uuid" : "E_ATzAz8Th6oMvd4D_QocA", \
    "version" : {...}, \
    "tagline" : "You Know, for Search" \
  }
```

If you are accessing the Kubernetes cluster remotely, you can create a port-forwarding service and use localhost, similar to what can be seen in the following code:
```
$ kubectl  -n  logging  port-forward   service/elasticsearch-es-http  9200
Forwarding from 127.0.0.1:9200 -> 9200
Forwarding from [::1]:9200 -> 9200
```
And then use th new termainal
```
$ curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
{
  "name" : "elasticsearch-es-default-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "lbJhggNkTD6ManGGm2brtw",
  "version" : {
    "number" : "7.4.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2f90bbf7b93631e52bafb59b3b049cb44ec25e96",
    "build_date" : "2019-10-28T20:40:44.881551Z",
    "build_snapshot" : false,
    "lucene_version" : "8.2.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
Now, we have access to our three-node small Elasticsearch cluster that we deployed on Kubernetes. Next, we need to deploy Kibana to complete the stack.

# Deploying Kibana
Kibana is an open source data visualization dashboard that lets you visualize your Elasticsearch data.

Let's perform the following steps to get Kibana deployed:

1. Create a Kibana instance associated with the Elasticsearch cluster we created previously:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1beta1
kind: Kibana
metadata:
  name: mykibana
  namespace: logging
spec:
  version: 7.4.2
  count: 1
  elasticsearchRef:
    name: elasticsearch
EOF
```
2. Get the status of the Kibana node:
```
$  kubectl get elasticsearch -n logging
NAME            HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch   green    1       7.4.2     Ready   55m
```
3. You can also confirm the pod's status in the logging namespace using the following command:
```
$ kubectl get pods -n logging
NAME                          READY   STATUS              RESTARTS   AGE
elasticsearch-es-default-0    1/1     Running             0          56m
elasticsearch-es-default-1    1/1     Running             0          56m
elasticsearch-es-default-2    1/1     Running             0          56m
mykibana-kb-8f6bd95cb-chrm8   1/1     Running             0          3m36s
```
With that, you have both Elasticsearch and Kibana nodes deployed. Next, we will deploy fluent-bit to forward container logs to our Elasticsearch deployment.

## Aggregating logs with Fluent Bit

Let's perform the following steps to get fluent-bit deployed:

1. Get the password for the default elastic user:
```
$ kubectl get secret elasticsearch-es-elastic-user \
-n logging -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

2. Copy the output of Step 1 and edit the ***fluent-bit-values.yaml*** file in the ***/src/chapter10/efk*** directory. Replace the ***http_passwd*** value with the output of Step 1 and save the file:
```
backend:
  type: es
  es:
    host: elasticsearch-es-http
    port: 9200
    http_user: elastic
    http_passwd: m2zr9fz49zqbkbpksprf4r76
    # Optional TLS encryption to ElasticSearch instance
    tls: "on"
    tls_verify: "off"
```
3. Deploy fluent-bit using the Helm chart:
```
$ helm install stable/fluent-bit --name=fluent-bit --namespace=logging -f fluent-bit-values.yaml
```
4. Confirm the pod's status in the logging namespace using the following command:
```
$ kubectl get pods -n logging
NAME                          READY   STATUS    RESTARTS   AGE
elasticsearch-es-default-0    1/1     Running   0          3h1m
elasticsearch-es-default-1    1/1     Running   0          3h1m
elasticsearch-es-default-2    1/1     Running   0          3h1m
fluent-bit-966ck              1/1     Running   0          120m
fluent-bit-dcm76              1/1     Running   0          120m
fluent-bit-pz8c7              1/1     Running   0          120m
fluent-bit-rn89k              1/1     Running   0          120m
mykibana-kb-8f6bd95cb-chrm8   1/1     Running   0          126m
```
With that, you have deployed all the components of the EFK stack. Next, we will connect to the Kibana dashboard.
## Accessing Kubernetes logs on Kibana
Let's perform the following steps to connect to the Kibana dashboard:

1. Confirm that the Kibana service has been created. By default, a ClusterIP service will be created:
```
$ kubectl get service mykibana-kb-http -n logging
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mykibana-kb-http   ClusterIP   10.104.168.228   <none>        5601/TCP   127m

```
2. Before we connect to the dashboard, get the password for the default elastic user:
```
$ kubectl get secret elasticsearch-es-elastic-user \
-n logging -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```
3. Create a port-forwarding service to access the Kibana dashboard from your workstation:
```
$ kubectl   -n logging port-forward service/mykibana-kb-http 5601
```
4. Open the Kibana dashboard at https://localhost:5601 in your browser. Enter elastic as the username and the password from the output of Step 2:

5. On the home page, click on the ***Connect to your Elasticsarch index*** button, as shown in the following screenshot:

6. Kibana will search for Elasticsearch index patterns. Define the index pattern that matches your results. In our example, we used *kubernetes_cluster-** . Click
on Next step to continue:

7. Specify ***Time Filter field name*** as *@timestamp* and click on the ***Create index pattern*** button, as shown in the following screenshot:

8. Click on the ***Discover*** menu. It is the first icon from the top:

9. On the ***Discover*** page, use the search field to look for keywords and filters:

10. If the keyword you are looking for can't be found in the current time frame, you 
need to change the date range by clicking on the calendar icon next to the search 
field and clicking on the ***Apply*** button after the new range has been selected:

With that, you've learned how to configure an EFK stack on your Kubernetes cluster in
order to manage and visualize cluster-wide logs.

## See also
* Elastic Cloud on Kubernetes (ECK): https:/​ / ​ github.​ com/​ elastic/​ cloud-​ on-k8s
* Deployment instructions on Red Hat OpenShift: https:/​ / ​ www.​ elastic.​ co/guide/​ en/​ cloud-​ on-​ k8s/​ 0.​ 9/​ k8s-​ openshift.​ html
* Elasticsearch Service documentation: https:/​ / ​ www.​ elastic.​ co/​ guide/​ en/cloud/​ current/​ index.​ html
* Introduction to Kibana: https:/​ / ​ www.​ elastic.​ co/​ guide/​ en/​ kibana/​ 7.​ 4/introduction.​ html#introduction
* Fluentd documentation: https:/​ / ​ docs.​ fluentd.​ org/​
* Fluent Bit documentation: https:/​ / ​ docs.​ fluentbit.​ io/​ manual/​
* Rancher Elastic Stack Kubernetes Helm Charts: https:/​ / ​ github.​ com/​ rancher/charts/​ tree/​ master/​ charts/​ efk/​ v7.​ 3.​ 0
* Kudo Elastic Operator: https:/​ / ​ github.​ com/​ kudobuilder/​ operators/​ tree/master/​ repository/​ elastic
