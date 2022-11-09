
# Vault and Consul via Kubernetes on Minikube

Aim of this repo is to setup a quick vault and consul environment for testing 
and development of IAC with terraform.

Expanded concepts coming:



- [Vault and Consul via Kubernetes on Minikube](#vault-and-consul-via-kubernetes-on-minikube)
  - [First Start](#first-start)
    - [Dependency install](#dependency-install)
    - [Start Minikube](#start-minikube)
    - [Install Consul, and Vault Helm charts](#install-consul-and-vault-helm-charts)
    - [Expose Consul](#expose-consul)
    - [Initialize Vault](#initialize-vault)
    - [Unseal Vault](#unseal-vault)
    - [Expose Vault UI by port forwarding](#expose-vault-ui-by-port-forwarding)
  - [Graceful shutdown and start after Initialization](#graceful-shutdown-and-start-after-initialization)
    - [Shutdown](#shutdown)
    - [Start](#start)
  - [Connect to Lens](#connect-to-lens)

## First Start

### Dependency install
```
brew install kubernetes-cli
brew install helm
# You need docker for minikube
brew install minikube
brew install jq

```

### Start Minikube
```
minikube start --driver docker

# Run dashboard to view ui
minikube dashboard
```

### Install Consul, and Vault Helm charts
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install Consul Via helm
helm install consul hashicorp/consul --values helm-consul-values.yml

# advised to view the pods and verify they come up
kubectl get pods

# Install Vault via Helm
helm install vault hashicorp/vault --values helm-vault-values.yml

# View Minikube services for verification 
minikube service list
|----------------------|----------------------------------|--------------|-----|
|      NAMESPACE       |               NAME               | TARGET PORT  | URL |
|----------------------|----------------------------------|--------------|-----|
| default              | consul-consul-connect-injector   | No node port |
| default              | consul-consul-controller-webhook | No node port |
| default              | consul-consul-dns                | No node port |
| default              | consul-consul-server             | No node port |
| default              | consul-consul-ui                 | http/80      |     |
| default              | kubernetes                       | No node port |
| default              | vault                            | No node port |
| default              | vault-active                     | No node port |
| default              | vault-agent-injector-svc         | No node port |
| default              | vault-internal                   | No node port |
| default              | vault-standby                    | No node port |
| kube-system          | kube-dns                         | No node port |
| kubernetes-dashboard | dashboard-metrics-scraper        | No node port |
| kubernetes-dashboard | kubernetes-dashboard             | No node port |
|----------------------|----------------------------------|--------------|-----|

```
### Expose Consul
```
# Expose consul by using minikube service
minikube service consul-consul-ui --namespace default                                                                                                           2 ↵ ──(Wed,Nov02)─┘
|-----------|------------------|-------------|---------------------------|
| NAMESPACE |       NAME       | TARGET PORT |            URL            |
|-----------|------------------|-------------|---------------------------|
| default   | consul-consul-ui | http/80     | http://192.168.49.2:30424 |
|-----------|------------------|-------------|---------------------------|
🏃  Starting tunnel for service consul-consul-ui.
|-----------|------------------|-------------|------------------------|
| NAMESPACE |       NAME       | TARGET PORT |          URL           |
|-----------|------------------|-------------|------------------------|
| default   | consul-consul-ui |             | http://127.0.0.1:50313 |
|-----------|------------------|-------------|------------------------|
🎉  Opening service default/consul-consul-ui in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

### Initialize Vault
```
# Get Pods 
kubectl get pods
NAME                                                 READY   STATUS    RESTARTS       AGE
consul-consul-client-rcqws                           1/1     Running   1 (75s ago)    169m
consul-consul-connect-injector-7565cf77db-4pwrc      1/1     Running   1 (75s ago)    169m
consul-consul-connect-injector-7565cf77db-lpk49      1/1     Running   2 (75s ago)    169m
consul-consul-controller-5df69f84dc-rbcjj            1/1     Running   2 (75s ago)    169m
consul-consul-server-0                               1/1     Running   1 (75s ago)    169m
consul-consul-webhook-cert-manager-bcdc6bfcf-66wsg   1/1     Running   1 (75s ago)    169m
vault-0                                              0/1     Running   1 (2m6s ago)   6h13m
vault-1                                              0/1     Running   1 (2m6s ago)   6h13m
vault-2                                              0/1     Running   1 (75s ago)    6h13m
vault-agent-injector-5b75fd986d-6pfg8                1/1     Running   4 (44s ago)    6h13m

# Notice vault[0-2] are not running, you can view the status in the minikube dashboard
# or see that it's ending in a error status
kubectl exec vault-0 -- vault status

# Viewing in the dashboard, we see that vault is sealed, we need to initialize
# Vault and unseal it from the created key share. This will output a token and
# unseal keys into a json file.

# First initialize the keys into a cluster-keys.json file
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json

```
### Unseal Vault
```
# We need to set a environmental variable with the unseal key, **Insecure operation**
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")

# We then unseal all vault instances using our unseal key via kubectl
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY

# check to ensure vault pods are running
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-server-0                  1/1     Running   0          10m
consul-consul-sxpbj                     1/1     Running   0          10m
vault-0                                 1/1     Running   0          5m49s
vault-1                                 1/1     Running   0          5m48s
vault-2                                 1/1     Running   0          5m47s
vault-agent-injector-5945fb98b5-vzbqv   1/1     Running   0          5m50s
```

### Expose Vault UI by port forwarding 
```
### In a browser window, navigate to the UI at http://localhost:8200/ui
kubectl port-forward vault-0 8200:8200
```


## Graceful shutdown and start after Initialization

We want our data to retain without corruption, and the default vault helm chart creates a PersistentVolumeClaim. 
### Shutdown

This will "shutdown" the cluster in a graceful manner 

```
minikube stop
```
### Start

Ensure Docker is running, start Minikube, with driver:
```bash
minikube start --driver docker
```

Repeat steps:

[Expose Consul](#expose-consul)

[Unseal Vault](#unseal-vault)

[Expose Vault UI](#expose-vault-ui-by-port-forwarding) 

## Connect to Lens

We can connect our minikube cluster to lens for more granular feedback on state and configuration. Make sure your minikube cluster is running and connected to kubectl currently.

Execute:
```
kubectl config view
```

Open lens, add cluster, and paste the contents.

Your cluster will now be connected to lens.

