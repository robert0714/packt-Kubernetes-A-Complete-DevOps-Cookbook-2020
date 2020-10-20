# Securing credentials using HashiCorp Vault
HashiCorp Vault is a popular tool for securely storing and accessing secrets such as 
credentials, API keys, and certificates. Vault provides secure secret storage, on-demand 
dynamic secrets, data encryption, and support for secret revocation.

In this section, we will cover the installation and basic use case of accessing and storing 
secrets for Kubernetes.

## Getting ready
Clone the k8sdevopscookbook/src repository to your workstation to use the manifest 
files in the chapter9 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter9
```
Make sure you have a Kubernetes cluster ready and kubectl and helm configured to 
manage the cluster resources.

## How to do it...
This section is further divided into the following subsections to make this process easier: 
*  Installing Vault on Kubernetes
*  Accessing the Vault UI
*  Storing credentials on Vault

## Installing Vault on Kubernetes
This recipe will show you how to get a Vault service on Kubernetes. Let's perform the 
following steps to get Vault installed using Helm charts:

1. Clone the chart repository:
```
$ git clone https://github.com/hashicorp/vault-helm.git
$ cd vault-helm
```
2. Check out the latest stable release:
```
$ git checkout v$(curl --silent "https://api.github.com/repos/hashicorp/vault-helm/releases/latest"| \
grep '"tag_name":' | \
sed -E 's/.*"v([^"]+)".*/\1/')
```
3. If you would like to install a highly available Vault, skip to Step 4; otherwise, install the standalone version using the Helm chart parameters shown here:
```
$ helm install --name vault --namespace vault ./
```
4. To deploy a highly available version that uses an HA storage backend such as Consul, use the following Helm chart parameters. This will deploy Vault using a StatefulSet with three replicas:
```
$ helm install --name vault --namespace vault --set='server.ha.enabled=true' ./
```
If you see the error Error: apiVersion 'v2' is not valid. The value must be "v1" , you need to use Helm v.3 (vault v0.4.0+ use helm v3)

5. Verify the status of the pods. You will notice that the pods aren't ready since the readiness probe requires Vault to be initialized first:
```
$ kubectl -n vault get pods
```
6. Check the initialization status. It should be false :
```
$ kubectl exec -it vault-0 -nvault -- vault status
```
7. Initialize the Vault instance. The following command will return an unseal key 
and root token:
```
$ kubectl exec -it vault-0 -nvault -- vault operator init -n 1 -t 1
Unseal Key 1: lhLeU6SRdUNQgfpWAqWknwSxns1tfWP57iZQbbYtFSE=
Initial Root Token: s.CzcefEkOYmCt70fGSbHgSZl4
Vault initialized with 1 key shares and a key threshold of 1.
Please securely distribute the key shares printed above. When the Vault is re-sealed, restarted, or stopped, you must supply at least 1 of these keys to
unseal it before it can start servicing requests.
```

8. Unseal Vault using the unseal key from the output of the following command:
```
$ kubectl exec -it vault-0 -nvault -- vault operator unseal
```

9. Verify the pod's status. You will see that the readiness probe has been validated and that the pod is ready:
```
$ kubectl get pods -n vault
```
Vault is ready to be used after it is initialized. Now, you know how to get Vault running on Kubernetes.

## Accessing the Vault UI
By default, the Vault UI is enabled when using a Helm chart installation. Let's perform the 
following steps to access the Vault UI:
1. Since access to Vault is a security concern, it is not recommended to expose it  
with a service. Use port-forwarding to access the Vault UI using the following 
command:
```
$ kubectl port-forward vault-0 -nvault 8200:8200
```
2. Once forwarding is complete, you can access the UI at http://localhost:8200 :

Now, you have access to the web UI. Take the Vault Web UI tour to familiarize yourself with its functionality.

# Storing credentials on Vault
This recipe will show you how to use Vault in Kubernetes and retrieve secrets from Vault.
Let's perform the following steps to enable the Kubernetes authentication method in Vault:
1. Log in to Vault using your token:
```
$ vault login <root-token-here>
```
2. Write a secret to Vault:
```
$ vault write secret/foo value=bar
Success! Data written to: secret/foo
$ vault read secret/foo
Key
Value
---
-----
refresh_interval 768h
value
bar
```
3. Let's configure Vault's Kubernetes authentication backend. First, create a 
ServiceAccount:
```
$ kubectl -n vault create serviceaccount vault-k8s
```
4. Create a RoleBinding for the vault-k8s ServiceAccount:
```
$ cat <<EOF | kubectl apply -f -
```
