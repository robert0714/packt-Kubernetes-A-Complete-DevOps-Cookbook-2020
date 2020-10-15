# Using RBAC to harden cluster security

In a complex system such as Kubernetes, authorization mechanisms are used to set who is allowed to make what changes to the cluster resources and manipulate them. ***Role-based access control (RBAC)*** is a mechanism that's highly integrated into Kubernetes that grants users and applications granular access to Kubernetes APIs.  

As good practice, you should use the Node and RBAC authorizers together with the 
NodeRestriction admission plugin.

In this section, we will cover getting RBAC enabled and creating Roles and RoleBindings to grant applications and users access to the cluster resources.

##  Getting ready
Make sure you have an RBAC-enabled Kubernetes cluster ready (since Kubernetes 1.6, RBAC is enabled by default) and that kubectl and helm have been configured so that you can manage the cluster resources. Creating private keys will also require that you have the openssl tool before you attempt to create keys for users.

Clone the k8sdevopscookbook/src repository to your workstation to use the manifest files IN the chapter9 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter9/rbac
```
RBAC is enabled by default starting with Kubernetes 1.6. If it is disabled for any 
reason, start the API server with --authorization-mode=RBAC to enable RBAC.
##  How to do it...
This section is further divided into the following subsections to make this process easier:
* Viewing the default Roles
* Creating user accounts
* Creating Roles and RoleBindings
* Testing the RBAC rules

##  Viewing the default Roles
RBAC is a core component of the Kubernetes cluster that allows us to create and grant roles 
to objects and control access to resources within the cluster. This recipe will help you 
understand the content of roles and role bindings.

Let's perform the following steps to view the default roles and role bindings in our cluster:

1. View the default cluster roles using the following command. You will see a long 
mixed list of ***system***: , ***system:controller***: , and a few other prefixed 
***roles. system***:* roles are used by the infrastructure, ***system:controller ***
roles are used by a Kubernetes controller manager, which is a control loop that 
watches the shared state of the cluster. In general, they are both good to know 
about when you need to troubleshoot permission issues, but they're not 
something we will be using very often:
```
$ kubectl get clusterroles
$ kubectl get clusterrolebindings
```
2. View one of the system roles owned by Kubernetes to understand their purpose 
and limits. In the following example, we're looking at ***system:node*** , which 
defines the permission for kubelets. In the output in Rules, apiGroups: indicates 
the core API group, ***resources*** indicates the Kubernetes resource type, 
and ***verbs*** indicates the API actions allowed on the role:
```
$ kubectl get clusterroles system:node -o yaml
```
3. Let's view the default user-facing roles since they are the ones we are more 
interested in. The roles that don't have the system: prefix are intended to be 
user-facing roles. The following command will only list the non-system: prefix 
roles. The main roles that are intended to be granted within a specific namespace 
using RoleBindings are the admin , edit , and view roles:
```
$ kubectl  get clusterroles |grep -v '^system'
NAME                                                                   AGE
admin                                                                  28h
calico-kube-controllers                                                28h
calico-node                                                            28h
cluster-admin                                                          28h
edit                                                                   28h
kube-keepalived-vip                                                    27h
kube-state-metrics                                                     6h5m
kubernetes-dashboard                                                   28h
openebs-maya-operator                                                  28h
view                                                                   28h
```
4. Now, review the default cluster binding, that is, ***cluster-admin*** , using the 
following command. You will see that this binding gives the ***system:masters*** 
group cluster-wide superuser permissions with the ***cluster-admin*** role:
```
$ kubectl get clusterrolebindings/cluster-admin -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
...(ommit)
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```
Since the Kubernetes 1.6 release, RBAC is enabled by default and new users can be created 
and start with no permissions until permissions are assigned by an admin user to a specific 
resource. Now, you know about the available default roles.

In the following recipes, you will learn how to create new Roles and RoleBindings and 
grant accounts the permissions that they need.

## Creating user accounts
As explained in the Kubernetes docs, Kubernetes doesn't have objects to represent normal 
user accounts. Therefore, they need to be managed externally (check the ***Kubernetes 
Authentication*** documentation in the ***See also*** section for more details). This recipe will show 
you how to create and manage user accounts using private keys.

Let's perform the following steps to create a user account:

1. Create a private key for the example user. In our example, the key file is user3445.key :
```
$ openssl genrsa -out user3445.key 2048
```
2. Create a ***certificate sign request (CSR)*** called ***user3445.csr*** using the private 
key we created in Step 1. Set the username ( /CN ) and group name ( /O ) in the ***-subj*** parameter. In the following example, the username is ***john.geek*** , while
the group is ***development*** :
```
$ openssl req -new -key user3445.key \
-out user3445.csr \
-subj "/CN=john.geek/O=development"
```
3. To use the built-in signer, you need to locate the cluster-signing certificates for 
your cluster. By default, the *ca.crt* and *ca.key* files should be in the 
*/etc/kubernetes/pki/* directory.If you are using kops to deploy, your cluster 
signing keys can be downloaded from 
**s3://$BUCKET_NAME/$KOPS_CLUSTER_NAME/pki/private/ca/*.key** and
**s3://$BUCKET_NAME/$KOPS_CLUSTER_NAME/pki/issued/ca/*.crt** . Once you've located the keys, change the CERT_LOCATION mentioned in the following 
code to the current location of the files and generate the final signed certificate:
```
$ openssl x509 -req -in user3445.csr \
-CA CERT_LOCATION/ca.crt \
-CAkey CERT_LOCATION/ca.key \
-CAcreateserial -out user3445.crt \
-days 500
```
You'll see it .
```
$ sudo openssl x509 -req -in user3445.csr \
 -CA     /etc/kubernetes/pki/ca.crt  \
 -CAkey  /etc/kubernetes/pki/ca.key  \
 -CAcreateserial -out user3445.crt  \
 -days 500
 
Signature ok
subject=/CN=john.geek/O=development
Getting CA Private Key
```
generating user3445.crt  
4. If all the files have been located, the command in Step 3 should return an output similar to the following:
```
Signature ok
subject=CN = john.geek, O = development
Getting CA Private Key
```
Before we move on, make sure you store the signed keys in a safe directory. As an 
industry best practice, using a secrets engine or Vault storage is recommended. 
You will learn more about Vault storage later in this chapter IN the ***Securing credentials using HashiCorp Vault*** recipe.

5. Create a new context using the new user credentials:
```
$ kubectl config set-credentials user3445 --client-certificate=user3445.crt --client-key=user3445.key
$ kubectl config set-context user3445-context --cluster=local --namespace=secureapp --user=user3445
```
If you saw  error :cannot access /home/vagrant/.kube/config.lock: No such file or directory.   
Solution Beside, steps specified in [kubernetes documentation|https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#more-information].
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Add change permissions on $HOME/.kube/ directory.
```
$ sudo chown $(id -u):$(id -g) $HOME/.kube/
```
6. List the existing context using the following comment. You will see that the new user3445-context has been created:
```
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
          user3445-context              local        user3445           secureapp
```
7. Now, try to list the pods using the new user context. You will get an access 
denied error since the new user doesn't have any roles and new users don't come 
with any roles assigned to them by default:
```
$ kubectl --context=user3445-context get pods
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
8. Optionally, you can **base64** encode all three files ( **user3445.crt** , 
**user3445.csr** , and **user3445.key** ) using the openssl base64 -in 
<infile> -out <outfile> command and distribute the populated **config-user3445.yml** file to your developers. An example file can be found in this
book's GitHub repository in the **src/chapter9/rbac** directory. There are many 
ways to distribute user credentials. Review the example using your text editor:
```
$ cat config-user3445.yaml
```
With that, you've learned how to create new users. Next, you will create roles and assign 
them to the user.

## Creating Roles and RoleBindings
Roles and RolesBindings are always used in a defined namespace, meaning that the 
permissions can only be granted for the resources that are in the same namespace as the 
Roles and the RoleBindings themselves compared to the ClusterRoles and 
ClusterRoleBindings that are used to grant permissions to cluster-wide resources such as 
nodes.

Let's perform the following steps to create an example Role and RoleBinding in our cluster:   
1. First, create a namespace where we will create the Role and RoleBinding. In our 
example, the namespace is secureapp :
```
$ kubectl create ns secureapp
```
2. Create a role using the following rules. This role basically allows all operations to 
be performed on deployments, replica sets, and pods for the deployer role in 
the secureapp namespace we created in Step 1. Note that any permissions that 
are granted are only additive and there are no deny rules: 
```
$ cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secureapp
  name: deployer
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch","delete"]
EOF
```
3. Create a RoleBinding using the deployer role and for the username john.geek 
in the secureapp namespace. We're doing this since a RoleBinding can only 
reference a Role that exists in the same namespace: 
```
$ cat <<EOF | kubectl apply -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: deployer-binding
  namespace: secureapp
subjects:
- kind: User
  name: john.geek
  apiGroup: ""
roleRef:
  kind: Role
  name: deployer
  apiGroup: ""
EOF
```
With that, you've learned how to create a new Role and grant permissions to a user using RoleBindings.
## Testing the RBAC rules
Let's perform the following steps to test the Role and RoleBinding we created earlier:  
1. Deploy a test pod in the secureapp namespace where the user has access:
```
$ cat <<EOF | kubectl --context=user3445-context apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: secureapp
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
EOF
```
List the pods in the new user's context. The same command that failed in the Creating user accounts recipe in Step 7 should now execute successfully:
```
$ kubectl --context=user3445-context get pods
NAME                            READY   STATUS    RESTARTS   AGE
busybox 1/1                     1/1     Running   1          8h
```
If you try to create the same pod in a different namespace, you will see that the command 
will fail to execute.

## How it works...
This recipe showed you how to create new users in Kubernetes and quickly create Roles 
and RoleBindings to grant permission to user accounts on Kubernetes.

Kubernetes clusters have two types of users:

*  ***User accounts***: User accounts are normal users that are managed externally.
*  ***Service accounts***: Service accounts are the users who are associated with the Kubernetes services and are managed by the Kubernetes API with its own resources.

You can read more about service accounts by looking at the Managing service accounts link in the *See also* section.

In the Creating Roles and RoleBindings recipe, in Step 1, we created a Role named deployer .
Then, in Step 2, we granted the rules associated with the deployer Role to the user account *john.geek* .

RBAC uses the *rbac.authorization.k8s.io* API to make authorization decisions. This 
allows admins to dynamically configure policies using the Kubernetes APIs. If you wanted 
to use the existing Roles and give someone cluster-wide superuser permission, you could 
use the *cluster-admin* ClusterRole with a ClusterRoleBinding instead. ClusterRoles don't 
have namespace limits and can execute commands in any namespace with the granted 
permissions. Overall, you should be careful while assigning the *cluster-admin* 
ClusterRole to users. ClusterRoles can be also limited to namespaces, similar to Roles if 
they are used with RoleBindings to grant permissions instead.

## See also
*  RBAC Authorization in Kubernetes documentation: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding
*  More on the default roles and role bindings: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings
*  Autogenerating RBAC policies based on Kubernetes audit logs: https://github.com/liggitt/audit2rbac
*  Kubernetes Authentication: https://kubernetes.io/docs/reference/access-authn-authz/authentication/
*  Managing Service Accounts: https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
*  The kubectl-bindrole tool for finding Kubernetes Roles bound to a specified ServiceAccount: https://github.com/Ladicle/kubectl-bindrole
