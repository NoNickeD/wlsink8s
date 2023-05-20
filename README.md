# WebLogic in Kubernetes.

Deployment of a WebLogic Server in Kubernetes Using the WebLogic Kubernetes Operator.

## Prerequisites

- Install [Docker](https://docs.docker.com/desktop/install/)
- Install [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- Install [Helm](https://helm.sh/docs/intro/install/)
- Create account to [OCR](https://container-registry.oracle.com/)

## Creating local Kubernetes Cluster

To build the Kubernetes cluster locally we will use Docker and minikube.

```bash
# Check the Minikube version
minikube version

# Configure the amount of memory and number of CPUs for Minikube.
minikube start --memory 4096 --cpus 2

# Get the status of a Minikube cluster
minikube status
```

```bash
# Check the nodes of Local K8s Cluster.
kubectl get nodes -o wide   

# Output
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
minikube   Ready    control-plane   23s   v1.24.1   192.168.49.2   <none>        Ubuntu 20.04.4 LTS   5.15.49-linuxkit   docker://20.10.17
```

#### Useful minikube commands

```bash
# Access the Kubernetes Dashboard
minikube dashboard

# Get the IP address of the Minikube VM
minikube ip

# List enabled addons
minikube addons list

# Enable addons
minikube addons enable <addon-name>

# Disable addons
minikube addons disable <addon-name>
```

## Install WebLogic Operator

Establish Helm configuration by indicating the location of the operator's Helm chart as follows.

```bash
helm repo add weblogic-operator https://oracle.github.io/weblogic-kubernetes-operator/charts --force-update
```

##### Prepare the enviroment:

The Kubernetes system differentiates between user accounts and service accounts due to several considerations. The primary differentiation is that user accounts are intended for human users, whereas service accounts are designed for processes operating within pods. Service accounts are also a requirement for the operator. When a service account isn't explicitly stated, it will automatically use the default service account of the namespace. If there's a need to utilize a different service account, it's mandatory to establish both the operator's namespace and the service account prior to deploying the operator's Helm chart.

- Therefore, you should pre-create the operator's namespace.

```bash
kubectl create namespace weblogic-operator-ns-local

# Check if the namespace created successfully
kubectl get ns
```

-  Also, create and the service account:

```bash
kubectl -n weblogic-operator-ns-local create serviceaccount weblogic-operator-sa-local

# Check if the service account created successfully
kubectl -n weblogic-operator-ns-local get sa
```

Install the operator using helm.

```bash
helm install weblogic-operator weblogic-operator/weblogic-operator \
--namespace weblogic-operator-ns-local \
--set "serviceAccount=weblogic-operator-sa-local" \
--set "enableClusterRoleBinding=true" \
--set "domainNamespaceSelectionStrategy=LabelSelector" \
--set "domainNamespaceLabelSelector=weblogic-operator\=enabled" \
--wait
```

The output will be similar to the following:

```bash
NAME: weblogic-operator
LAST DEPLOYED: Sat May 20 10:58:56 2023
NAMESPACE: weblogic-operator-ns-local
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Confirm the operator's pod is operational by enumerating the pods in the operator's specific namespace. There should be two visible pods - one for the operator itself and another for the conversion webhook. The conversion webhook is a unique deployment in your Kubernetes cluster that seamlessly performs automatic upgrades to domain resources.

```bash
kubectl -n weblogic-operator-ns-local get pods
```

Validate the operator's running status by examining the pod's log.

```bash
kubectl -n weblogic-operator-ns-local logs -c weblogic-operator deployments/weblogic-operator
```

The output will be similar to the following:

```bash
Launching Oracle WebLogic Server Kubernetes Operator...
VM settings:
    Max. Heap Size (Estimated): 1.92G
    Using VM: OpenJDK 64-Bit Server VM

{"timestamp":"2023-05-20T07:59:15.668463937Z","thread":1,"fiber":"","namespace":"","domainUID":"","level":"INFO","class":"oracle.kubernetes.operator.helpers.HealthCheckHelper","method":"createAndValidateKubernetesVersion","timeInMillis":1684569555668,"message":"Kubernetes version is: v1.24.1","exception":"","code":"","headers":{},"body":""}
{"timestamp":"2023-05-20T07:59:16.163876085Z","thread":1,"fiber":"","namespace":"","domainUID":"","level":"INFO","class":"oracle.kubernetes.operator.OperatorMain$MainDelegateImpl","method":"logStartup","timeInMillis":1684569556163,"message":"Oracle WebLogic Kubernetes Operator, version: 4.0.6, implementation: ca01017ffee4c8a4c0fb4b17b3cde1583a2fcfc8.ca01017, build time: 2023-04-08T10:54:43-0400","exception":"","code":"","headers":{},"body":""}
....
```

## Create a Traefik ingress controller.

Establish Helm settings based on the location of the Traefik Helm chart using this specified format.

```bash
helm repo add traefik https://helm.traefik.io/traefik --force-update
```

Establish a dedicated namespace for the ingress controller.

```bash
kubectl create namespace traefik-ns-local
```

This operation will set up the Traefik controller with the plaintext node port at 30701, the SSL node port at 30443, and a specific configuration for kubernetes.namespaces.

```bash
helm install traefik traefik/traefik  --namespace traefik-ns-local
```

The output will be similar to the following:

```bash
NAME: traefik-operator
LAST DEPLOYED: Sat May 20 11:24:58 2023
NAMESPACE: traefik-ns-local
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Traefik Proxy v2.10.1 has been deployed successfully
on traefik-ns-local namespace !
```

```bash
kubectl -n traefik-ns-local get pods
```

## Make preparations for a domain.

1. Establish and assign a label to a namespace capable of hosting one or multiple domains.

```bash
# Create ns
kubectl create namespace wls-test-domain-ns

# Label ns
kubectl label ns wls-test-domain-ns weblogic-operator=enabled
```

Confirm the creation of namespace with proper labels.

```bash
kubectl get ns --show-labels
```

2. Confirm your acceptance of the WebLogic Server images license agreement.
	- Navigate to the [Oracle Container Registry (OCR)](https://container-registry.oracle.com/)using any web browser and authenticate yourself using the Oracle Single Sign-On (SSO) service. If you don't have an SSO account, click on the 'Sign In' button located on the top right corner of the webpage to create one.
	- Input 'weblogic' into the search bar and choose 'weblogic' from the results that appear.
	- Use the drop-down menu to select the language of your preference, and then proceed by clicking on the 'Continue' button.
	- Review and agree to the terms and conditions of the license agreement.


3. Initiate a docker-registry secret to enable the download of the example WebLogic Server image from the registry.

```bash
kubectl create secret docker-registry weblogic-repo-credentials \
     --docker-server=container-registry.oracle.com \
     --docker-username='test@mail.me' \
     --docker-password='********' \
     --docker-email='test@mail.me' \
     -n wls-test-domain-ns
```

Replace the docker-username, docker-email & docker-password with YOUR details for access the registry.

## Create a Domain

Choose a username and password for the WebLogic domain administrator. Then, use these credentials to create a Kubernetes Secret for the domain.

```bash
kubectl -n wls-test-domain-ns create secret generic wls-test-domain-weblogic-credentials \
  --from-literal=username='weblogic' --from-literal=password='welcome1'
```

Set up an encryption secret for the runtime domain.

```bash
kubectl -n wls-test-domain-ns create secret generic \
  wls-test-domain-runtime-encryption-secret \
   --from-literal=password='weblogic1'
```

Establish the **wls-test-domain** domain resource along with its associated **wls-test-domain-cluster-1** cluster resource by employing a singular YAML resource file that outlines both resources. Instead of replacing the conventional WebLogic configuration files, the domain and cluster resources work in conjunction with them to articulate the Kubernetes artifacts related to the respective domain.

```bash
kubectl apply -f domain-resource.yaml
```

Verify that the operator has initiated the servers for the domain.

```bash
kubectl -n wls-test-domain-ns describe domain wls-test-domain
```

The status of the domain can be obtained using this command.

```bash
kubectl -n wls-test-domain-ns get domain wls-test-domain -o jsonpath='{.status}'
```

Shortly, you will observe the Administration Server and Managed Servers are in active state.

```bash
kubectl -n wls-test-domain-ns get pods

# Output
NAME                              READY   STATUS    RESTARTS   AGE
wls-test-domain-admin-server      1/1     Running   0          7m14s
wls-test-domain-managed-server1   1/1     Running   0          6m31s
wls-test-domain-managed-server2   1/1     Running   0          6m31s
```

In addition, all the Kubernetes Services associated with the domain should be discernible.

```bash
kubectl -n wls-test-domain-ns get svc

# Output will be like the following:
NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
wls-test-domain-admin-server        ClusterIP   None             <none>        7001/TCP   9m40s
wls-test-domain-cluster-cluster-1   ClusterIP   10.111.241.125   <none>        8001/TCP   8m57s
wls-test-domain-managed-server1     ClusterIP   None             <none>        8001/TCP   8m57s
wls-test-domain-managed-server2     ClusterIP   None             <none>        8001/TCP   8m57s
```

### Create an ingress route for the domain.

Formulate an ingress pathway for the specific domain within the corresponding domain namespace, utilizing the subsequent YAML file.

```bash
kubectl -n wls-test-domain-ns apply -f ingress-route.yaml
```

By default, when you create a service of type LoadBalancer in a Kubernetes cluster, it's the responsibility of the cloud provider's network infrastructure to provision a network load balancer and to create routes so that the service can be accessed via the external IP address of the load balancer.

However, when you're running Kubernetes locally with Minikube, there's no underlying cloud provider network infrastructure to do this. As a result, while you can create a LoadBalancer service in Minikube, it won't get an external IP by default. It will stay in the `pending` state indefinitely if you try to check the external IP.

```bash
kubectl -n traefik-ns-local get svc
```

To solves this problem by creating a network route on your machine to services deployed with type `LoadBalancer` and provides them with a real IP address as if they were running in a cloud provider's network. 

```bash
# Open a new tab in your terminal
minikube tunnel
```

This allows you to run and test services locally in an environment that is more similar to a full-scale, hosted Kubernetes environment.

You would use `minikube tunnel` whenever you want to expose a LoadBalancer service running in your Minikube cluster to be accessible from your local machine. Note that running minikube tunnel requires administrative privileges because it configures network routing rules on your machine.

Access to the WebLogic Server Administration Console.

```bash
open http://localhost/console
```

## Clean up

If you want to delete the cluster entirely, including all data and state, you can use the following command.

```bash
minikube delete