## Welcome to The RBAC Demo

### Sections of this Demo
- Basic Role 
- Cluster Role 
- Terraform Automation 

## Purpose of the Demo
RBAC = Role-based access control (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within an enterprise.
      
RBAC can be both intimidating and confusing all at the same time if you are new to kubernetes or RBAC in general. Think of RBAC as a way to map users to roles to permissions, sounds simple right? Well you will see in this demo it is ..

In this demo we will :
- create a new namespace for our user
- create a new user
- a cert for that user leveraging the local CA
- rbac role for that user 
- testing the permissions for that user.

## How to read the console output
```
~/github/rbac-demo  master ✗ system    <-- Directory in which the command was executed from                                                                                                                                                                                               19m ⚑ ✚ ◒  
▶ openssl version                      <-- Command that is execute                                                                                                                                
LibreSSL 2.6.5                         <-- Output from the command
```

## Items you will need to complete this demo
- IDE or Text Editor (Recommend VSCode or Atom)
- Docker Desktop Installed with Kubernetes Enabled
- kubectl  
- openssl 
- basic knowledge of kubernetes is beneficial but not required

## Before we get start, check a few things that you need

- Verify kubectl is installed

```
~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   18m ⚑ ✚ ◒  
▶ kubectl version                                                          
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.6", GitCommit:"96fac5cd13a5dc064f7d9f4f23030a6aeface6cc", GitTreeState:"clean", BuildDate:"2019-08-19T11:05:16Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

- Verify openssl is installed

```
~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   19m ⚑ ✚ ◒  
▶ openssl version                                                                                                                                                      
LibreSSL 2.6.5
```

- Verify you have access to your local kubernetes cluster

```
~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   18m ⚑ ✚ ◒  
▶ kubectl config get-contexts                  
CURRENT   NAME                 CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop       docker-desktop   docker-desktop   
          docker-for-desktop   docker-desktop   docker-desktop   

~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   18m ⚑ ✚ ◒  
▶ kubectl get nodes                     
NAME             STATUS   ROLES    AGE    VERSION
docker-desktop   Ready    master   101s   v1.14.6
```

## Basic Role

### Creating the new User (This is our test user we shall call him George)

First lets setup some local dirs to hold the certs and other items (I am running OSX so if running another os you commands may vary)

- Create the directory for the user certs

```
~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   14m ⚑ ✚ ◒  
▶ mkdir .certs
```

- Create the directory for the CA certs

```
~/github/rbac-demo  master ✗ system                                                                                                                                                                                                   14m ⚑ ✚ ◒  
▶ mkdir .ca_certs
```

- Create the user private key (we will use this and the private cert in the next step to give the user access to the cluster)

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              20m ⚑ ✚ ◒  
▶ openssl genrsa -out george.key 2048   

Generating RSA private key, 2048 bit long modulus
.............................................+++
...........................................+++
e is 65537 (0x10001)
```

- create the user cert (the CN should be the name of the user and O would be the name of your awesome company)

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              20m ⚑ ✚ ◒  
▶ openssl req -new -key george.key -out george.csr -subj "/CN=george/O=tiddlywigs"  
```

- Getting the CA Key for you local kubernetes instance (you will need this and the crt to sign the user)

```
github/rbac-demo/.ca_certs  master ✗ system                                                                                                                                                                                          24m ⚑ ✚ ◒  ⍉
▶ kubectl cp kube-apiserver-docker-desktop:run/config/pki/ca.key -n kube-system ca.key       
```
- Getting the CA Cert for you local kubernetes instance

```
github/rbac-demo/.ca_certs  master ✗ system                                                                                                                                                                                           25m ⚑ ✚ ◒  
▶ kubectl cp kube-apiserver-docker-desktop:run/config/pki/ca.crt -n kube-system ca.crt
```

- Signing the user cert with the CA

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                             28m ⚑ ✚ ◒  ⍉
▶ openssl x509 -req -in george.csr -CA ../.ca_certs/ca.crt -CAkey ../.ca_certs/ca.key -CAcreateserial -out george.crt -days 500

Signature ok
subject=/CN=george/O=tiddlywigs
Getting CA Private Key
```

Congradulations you have now create a new user and signed the cert, next will we will set the context in the kube config and see what access we have.

### Setting the kube config

Setting the kube config "~/.kube/conig" is where kubectl looks by default for contexts. Context specify how and who is tring to access the cluster.
More information here [Config](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

- We need to check out the current contexts  and you should see something similar to the following the asterisk marks the current in use context

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              28m ⚑ ✚ ◒  
▶ kubectl config get-contexts                                                         
CURRENT   NAME                 CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop       docker-desktop   docker-desktop   
          docker-for-desktop   docker-desktop   docker-desktop
```

- Now add george to the config

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              32m ⚑ ✚ ◒  
▶ kubectl config set-credentials george --client-certificate=george.crt  --client-key=george.key                                                  

User "george" set.
```

- Set george to point to a certain cluster in this case docker-desktop

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              33m ⚑ ✚ ◒  
▶ kubectl config set-context rbac-demo --cluster=docker-desktop  --user=george                     
Context "rbac-demo" created.
```

- At this point george is in the config and has a set context to point the docker-desktop cluster but we are still active on the old context, change that

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              35m ⚑ ✚ ◒  
▶ kubectl config use-context rbac-demo                                        
Switched to context "rbac-demo".

github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              36m ⚑ ✚ ◒  
▶ kubectl config get-contexts         
CURRENT   NAME                 CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop       docker-desktop   docker-desktop   
          docker-for-desktop   docker-desktop   docker-desktop   
*         rbac-demo            docker-desktop   george  
```

- What access george has at this time?

```
github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                              36m ⚑ ✚ ◒  
▶ kubectl get pods                    
Error from server (Forbidden): pods is forbidden: User "george" cannot list resource "pods" in API group "" in the namespace "default"

github/rbac-demo/.certs  master ✗ system                                                                                                                                                                                             38m ⚑ ✚ ◒  ⍉
▶ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "george" cannot list resource "nodes" in API group "" at the cluster scope
```

You can see george can authenticate to the cluster but doesn have permssions to do anything not even read, this is where RBAC comes in.

### RBAC Readonly

George wants to read / view items in the cluster and we feel that he should so how do we do that. Well RBAC is the solution, with RBAC we can assign permissions to George by leveraging Roles and Role Binding to give him the access we want.

Files need for this exercise:
- rbac-readonly/role.yaml
- rbac-readonly/role_binding.yaml
- rbac-fullperms/role_binding.yaml


- we first need to switch back to a context that has access to create roles and bindings

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       47m ⚑ ✚ ◒  
▶ kubectl config use-context docker-desktop    
Switched to context "docker-desktop".

github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       47m ⚑ ✚ ◒  
▶ kubectl config get-contexts              
CURRENT   NAME                 CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop       docker-desktop   docker-desktop   
          docker-for-desktop   docker-desktop   docker-desktop   
          rbac-demo            docker-desktop   george  
```

- now we can apply the role.yaml file

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       47m ⚑ ✚ ◒  
▶ kubectl apply -f role.yaml                   
role.rbac.authorization.k8s.io/readonly created
```

- and then the binding to bind the user to the role

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       48m ⚑ ✚ ◒  
▶ kubectl apply -f role_binding.yaml 
rolebinding.rbac.authorization.k8s.io/readonly-role-binding created
```

- switch back to george and see what happens now

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                      49m ⚑ ✚ ◒  ⍉
▶ kubectl config use-context rbac-demo
Switched to context "rbac-demo".

github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       49m ⚑ ✚ ◒  
▶ kubectl get pods                    
No resources found.

github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                       50m ⚑ ✚ ◒  
▶ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "george" cannot list resource "nodes" in API group "" at the cluster scope

github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                      50m ⚑ ✚ ◒  ⍉
▶ kubectl get pods --all-namespaces   
Error from server (Forbidden): pods is forbidden: User "george" cannot list resource "pods" in API group "" at the cluster scope
```

Great george can see pods if there were any in the default namespace but not nodes and pods across all namespaces why? Nodes and the term "--all-namespaces" are cluster scoped  and we have created a role, Roles and ClusterRoles are not the same, we will cover Cluster Roles later

see if George can deploy the the default namespace a simple hello world app.

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                      1h3m ⚑ ✚ ◒  
▶ kubectl create deployment --image nginx my-nginx
Error from server (Forbidden): deployments.apps is forbidden: User "george" cannot create resource "deployments" in API group "apps" in the namespace "default"
```

Well thats no good I want george to deploy that app, but maybe not in the default namespace as it is bad practice so create a new space for george to do his deployment.

### RBAC Full Permissions

- yet again switch back to a user that has the access we need for creating objects

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                     1h3m ⚑ ✚ ◒  ⍉
▶ kubectl config use-context docker-desktop       
Switched to context "docker-desktop".
```

- now we need to create a new namespace for george to do his deployments , call it georges-awesome-app

```
github/rbac-demo/rbac-readonly  master ✗ system                                                                                                                                                                                      1h5m ⚑ ✚ ◒  
▶ kubectl create namespace georges-awesome-app    
namespace/georges-awesome-app created
```

- before we switch back to george we need to ensure that we give george a role that can deploy to that namespace but since we want george to own this namespace we will give him full permissions on it.

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                     1h9m ⚑ ✚ ◒  
▶ kubectl apply -f role.yaml                  
role.rbac.authorization.k8s.io/fullperms created
```

- now we bind that role to george

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h10m ⚑ ✚ ◒  
▶ kubectl apply -f role_binding.yaml                 
rolebinding.rbac.authorization.k8s.io/fullperms-role-binding created
```

- switch back to george

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                   1h12m ⚑ ✚ ◒  ⍉
▶ kubectl config use-context rbac-demo
Switched to context "rbac-demo".
```
- see if we can at least read the namespace first

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h17m ⚑ ✚ ◒  
▶ kubectl get pods --namespace georges-awesome-app
No resources found.
```
- now its time to try the deployment

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h17m ⚑ ✚ ◒  
▶ kubectl create deployment --image nginx my-nginx --namespace georges-awesome-app
deployment.apps/my-nginx created

github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h18m ⚑ ✚ ◒  
▶ kubectl get pods --namespace georges-awesome-app                                
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-5d998f947f-twxmr   1/1     Running   0          6s
```

GREAT! George can now deploy to his own namespace but did we take away his access to default to read?

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h18m ⚑ ✚ ◒  
▶ kubectl get pods                                
No resources found.
```

No we didnt? but why? Well in short we create two roles that George can assume, one that has read in default namespace but one in the "georges-awesome-app" namespace that he has full permissions on even delete.

```
github/rbac-demo/rbac-fullperms  master ✗ system                                                                                                                                                                                    1h19m ⚑ ✚ ◒  
▶ kubectl delete deployment my-nginx --namespace georges-awesome-app
deployment.extensions "my-nginx" deleted
```

### Explaining the files

First make a note that RBAC is inclusive and not exclusive. In English means you can not say all but this, you can only say just this..

Role YAML
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default <-- namespace the role is bound 
  name: readonly <-- name of the role , to be user in the role binding
rules:
  - apiGroups: ["*"] <-- API group access "*" all apis
    resources: ["*"] <-- Resource access examples, pods, configmaps ect.. and sub resources such as pod/logs
    verbs: ["get", "list", "watch"] <-- what actions are allowed against the resources and api
```

Role Binding YAML
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: readonly-role-binding <-- name of the binding
  namespace: default <-- namespace the role is bound needs to match the role
subjects:
  - kind: User <-- what is it binding to, could be serviceaccount pending your setup
    name: george <-- name of the bind, in this case matches our cert CN of George
    apiGroup: "" <-- api Group see RBAC docs for more information, but for users it is always ""
roleRef:
  kind: Role <-- can be cluster role or role
  name: readonly <-- name of the role , from the above role yaml
  apiGroup: "" <-- api Group see RBAC docs for more information

```

## Cluster Role

## Terraform Automation

## Task List
    - [ ] Add Cluster Role demo
    - [ ] Add Terraform Automation docs
    

## Outside links for reference
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)


