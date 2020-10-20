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
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            n/a
HA Enabled         false
```
7. Initialize the Vault instance. The following command will return an unseal key 
and root token:
```
$ $ kubectl exec -it vault-0 -nvault -- vault operator init -n 1 -t 1
Unseal Key 1: C7VKyiFNE83VDokKU3XqMt/lPJUmWQ9CYIMak6tj0xU=

Initial Root Token: s.tz6BvcPXJxrbe1WsgX2tPf1a

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.

```

8. Unseal Vault using the unseal key from the output of the following command:
```
$ kubectl exec -it vault-0 -nvault -- vault operator unseal C7VKyiFNE83VDokKU3XqMt/lPJUmWQ9CYIMak6tj0xU=
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.3.1
Cluster Name    vault-cluster-204422bf
Cluster ID      c72cf7d0-b2e7-351c-03d6-e56d8485f422
HA Enabled      false
```


9. Verify the pod's status. You will see that the readiness probe has been validated and that the pod is ready:
```
$ kubectl -n vault get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          3m56s
vault-agent-injector-77d8f6d9d4-jkqxc   1/1     Running   0          3m56s
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
or 
```
kubectl -n vault get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
vault                      ClusterIP   10.111.174.214   <none>        8200/TCP,8201/TCP   5m35s
vault-agent-injector-svc   ClusterIP   10.101.117.90    <none>        443/TCP             5m35s
$ kubectl -n vault edit svc vault
service/vault edited
$ kubectl -n vault get svc
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                         AGE
vault                      LoadBalancer   10.111.174.214   192.16.35.17   8200:31098/TCP,8201:31034/TCP   6m12s
vault-agent-injector-svc   ClusterIP      10.101.117.90    <none>         443/TCP                         6m12s

```
2. Once forwarding is complete, you can access the UI at http://localhost:8200  (or http://192.16.35.17:8200/ui/vault/auth):

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
Key               Value
---               -----
refresh_interval  768h
value             bar
```
or 
```
$ kubectl exec -it  vault-0 -nvault -- vault login s.tz6BvcPXJxrbe1WsgX2tPf1a && vault  write secret/foo value=bar 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.tz6BvcPXJxrbe1WsgX2tPf1a
token_accessor       RYZwtIJyWKST8epldXbV5Yka
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
-bash: vault: command not found

```
3. Let's configure Vault's Kubernetes authentication backend. First, create a ServiceAccount:
```
$ kubectl -n vault create serviceaccount vault-k8s
```
4. Create a RoleBinding for the vault-k8s ServiceAccount:
```
$ cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-k8s
  namespace: default
EOF
```
5. Get the token:
```
$ SECRET_NAME=$(kubectl -n vault get serviceaccount vault-k8s -o jsonpath={.secrets[0].name})
$ ACCOUNT_TOKEN=$(kubectl -n vault get secret ${SECRET_NAME} -o jsonpath={.data.token} | base64 --decode; echo)
$ export VAULT_SA_NAME=$(kubectl get sa -n vault vault-k8s -o jsonpath=”{.secrets[*].name}”)
$ export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -n vault -o jsonpath={.data.'ca\.crt'} | base64 --decode; echo)
```
6. Enable the Kubernetes auth backend in the vault:
```
$ vault auth enable kubernetes
$ vault write auth/kubernetes/config
kubernetes_host=”https://MASTER_IP:6443"
kubernetes_ca_cert=”$SA_CA_CRT”
token_reviewer_jwt=$TR_ACCOUNT_TOKEN
```
7. Create a new policy called vault-policy from the example repository using the policy.hcl : file:
```
$ vault write sys/policy/vault-policy policy=@policy.hcl
```
8. Next, create a Role for the ServiceAccount:
```
$ vault write auth/kubernetes/role/demo-role \
bound_service_account_names=vault-coreos-test \
bound_service_account_namespaces=default \
policies=demo-policy \
ttl=1h
```
9. Authenticate with the Role by running the following command:
```
$ DEFAULT_ACCOUNT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -n
vault -o jsonpath={.data.token} | base64 — decode; echo )
```
10. Log in to the Vault with the token by running the following command:
```
$ vault write auth/kubernetes/login role=demo-role
jwt=${DEFAULT_ACCOUNT_TOKEN}
```
11. Create a secret at the secret/demo path:
```
$ vault write secret/demo/foo value=bar
```
With that, you've learned how to create a Kubernetes auth backend with Vault and use Vault to store Kubernetes secrets.

## See also
*  Hashicorp Vault documentation: https://www.vaultproject.io/docs/
*  Hashicorp Vault repository: https://github.com/hashicorp/vault-helm
*  Hands-on with Vault on Kubernetes: https://github.com/hashicorp/hands-on-with-vault-on-kubernetes
