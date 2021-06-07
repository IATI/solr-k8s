# Solr with Kubernetes
Repo containing kubernetes deployment information for IATI Solr Production instance

## Deployment Steps

### Initial Setup / kubectl Context
https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

```bash
# Install azure kubernetes service (aks) CLI
az aks install-cli

# Get context of your cluster and set that as your kubectl context
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify
kubectl get nodes

# NAME                                STATUS   ROLES   AGE    VERSION
# aks-nodepool1-13026912-vmss000000   Ready    agent   3d5h   v1.19.11
```

### Ingress
```bash
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Use Helm to deploy an NGINX ingress controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace default \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux

# Deploy configuration yml
kubectl apply -f ingress.yml

# Check on config
kubectl describe ingress solr-ingress
```

### Solr Cloud
https://apache.github.io/solr-operator/docs/running-the-operator

```bash
# Install operator helm repo
helm repo add apache-solr https://solr.apache.org/charts
helm repo update

# Install helm charts
kubectl create -f https://solr.apache.org/operator/downloads/crds/v0.3.0/all-with-dependencies.yaml
helm install solr-operator apache-solr/solr-operator --version 0.3.0

# Check on whats running
kubectl get all

# Deploy solr yml
kubectl apply -f deployment.yml
```

## Clean up

### Pods
`kubectl delete -f <file.yml>`

### Operators
`helm list --namespace <namespace:default>`

`helm uninstall nginx-ingress`
`helm uninstall solr-operator`

## TODO
- Access/Auth
   - Static IP - https://docs.microsoft.com/en-us/azure/aks/static-ip
   - Auth - https://apache.github.io/solr-operator/docs/solr-cloud/solr-cloud-crd.html#authentication-and-authorization
- CI/CD
   - Can (or should) I GitHub Actions this?

## Resources
https://apache.github.io/solr-operator/docs/local_tutorial

https://lucidworks.com/post/running-solr-on-kubernetes-part-1/

Connecting to container locally using kubectl context:
`kubectl port-forward solr-0 28983:8983`

`http://localhost:28983/solr/#/~cloud?view=nodes`

https://docs.microsoft.com/en-ca/azure/aks/ingress-basic