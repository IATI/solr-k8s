# Solr with Kubernetes

Repo containing kubernetes deployment information for IATI Solr Production instance

## Deployment Steps

### Initial Setup / kubectl Context

```bash
# Create resource group
az group create --resource-group rg-solr-PROD --location uksouth
RG=rg-solr-PROD

# Creating AKS Cluster with 3 nodes across availability zones
az aks create \
    --resource-group $RG \
    --name aks-solr-PROD \
    --generate-ssh-keys \
    --node-vm-size "Standard_E2as_v4" \
    --vm-set-type VirtualMachineScaleSets \
    --load-balancer-sku standard \
    --node-count 3 \
    --zones 1 2 3

# Get context of your cluster and set that as your kubectl context
az aks get-credentials --resource-group $RG --name aks-solr-PROD

# list nodes in cluster and zones
kubectl get nodes -o custom-columns=NAME:'{.metadata.name}',REGION:'{.metadata.labels.topology\.kubernetes\.io/region}',ZONE:'{metadata.labels.topology\.kubernetes\.io/zone}'

# NAME                                REGION    ZONE
# aks-nodepool1-74884023-vmss000000   uksouth   uksouth-1
# aks-nodepool1-74884023-vmss000001   uksouth   uksouth-2
# aks-nodepool1-74884023-vmss000002   uksouth   uksouth-3
```

https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

### Ingress

```bash
# Create Public IP in Azure
POD_RG=$(az aks show --resource-group $RG --name aks-solr-PROD --query nodeResourceGroup -o tsv)
IP_ADDRESS=$(az network public-ip create --resource-group $POD_RG --name pip-solr-PROD --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv)

# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Use Helm to deploy an NGINX ingress controller
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP="$IP_ADDRESS" \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="aks-solr-prod" \
    --set-string controller.config.proxy-body-size="0"
    --set-string controller.config.large-client-header-buffers="4 128k"

# Check 
kubectl get pods -l app.kubernetes.io/name=ingress-nginx \
  --field-selector status.phase=Running

kubectl get services -o wide -w nginx-ingress-ingress-nginx-controller

az network public-ip list --resource-group $POD_RG --query "[?name=='pip-solr-PROD'].[dnsSettings.fqdn]" -o tsv
# aks-solr-prod.uksouth.cloudapp.azure.com
```

### Upgrade / Config Change NGINX

```zsh
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP="$IP_ADDRESS" \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="aks-solr-prod" \
    --set-string controller.config.proxy-body-size="0" \
    --set-string controller.config.large-client-header-buffers="4 512K" \
    --set-string controller.config.client-header-buffer-size="512K"
```

### Secrets

https://github.com/bitnami-labs/sealed-secrets#installation

#### Install Sealed Secrets
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/controller.yaml
```

#### Install kubeseal
- Follow above link

### Create a sealed secret

```bash
kubeseal -o yaml < my-secret.yaml > my-secret-sealed.yaml
kubectl apply -f my-secret-sealed.yaml
```
Commit my-secret-sealed.yaml into git with no worries. Delete my-secret.yaml

### Certificate Generation

Install cert-manager (if not already installed): https://apache.github.io/solr-operator/docs/solr-cloud/solr-cloud-crd.html#use-cert-manager-to-issue-the-certificate

Issue Certifiate Using Cloudflare DNS lookup challenge cert-manager
https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/

```bash
# Install cert-manager from helm chart
helm install cert-manager jetstack/cert-manager \
  --version v1.3.1 \
  --set installCRDs=true
```

```bash
# Create certificate ClusterIssuer
kubectl apply -f cert-issuer.yml

# Issue certificate
kubectl apply -f certificate.yml

# Check Progress
kubectl get certificates

# NAME         READY   SECRET             AGE
# tls-secret   False   solr-tls-staging   5s

kubectl describe certificate tls-secret

# Events:
#   Type    Reason     Age   From          Message
#   ----    ------     ----  ----          -------
#   Normal  Issuing    3m8s  cert-manager  Issuing certificate as Secret does not exist
#   Normal  Generated  3m7s  cert-manager  Stored new private key in temporary Secret resource "tls-secret-cgkjj"
#   Normal  Requested  3m7s  cert-manager  Created new CertificateRequest resource "tls-secret-b4dpp"
#   Normal  Issuing    26s   cert-manager  The certificate has been successfully issued

kubectl describe certificaterequests tls-secret-XXXX

kubectl describe order tls-secret-XXXX-YYYYYYYYY

```

### Renewing Certificates

`cert-manager` will auto renew certificates and save them into the Kubernetes secrets/keystore. 
The Solr pods won't pick up the new certificate until it's restarted. 
The below setting should force this restart, however if one of the Solr nodes is in a degraded state it may not allow a restart. 
So after leaving things for 3mo I had to delete pods to force a restart. 

```yml
solrTLS:  
    restartOnTLSSecretUpdate: true
```

https://apache.github.io/solr-operator/docs/solr-cloud/solr-cloud-crd.html#certificate-renewal-and-rolling-restarts

### Solr Cloud

https://apache.github.io/solr-operator/docs/running-the-operator

```bash
# Install operator helm repo
helm repo add apache-solr https://solr.apache.org/charts
helm repo update

# Install helm chart and then solr-operator
kubectl create -f https://solr.apache.org/operator/downloads/crds/v0.3.0/all-with-dependencies.yaml
helm upgrade --install solr-operator apache-solr/solr-operator \
  --version 0.3.0

# Check on whats running
kubectl get pod -l control-plane=solr-operator
kubectl describe pod -l control-plane=solr-operator

# Verify zookeeper
kubectl get pod -l component=zookeeper-operator

# Deploy
kubectl apply -f deployment.yml

# watch nodes come up
kubectl get solrclouds -w

# check ingress config for URL
kubectl get ingress
kubectl describe ingress iati-prod-solrcloud-common

# Get initial credentials for admin user, if password is changed using admin API, the secret is NOT updated.
kubectl get secret iati-prod-solrcloud-security-bootstrap \
  -o jsonpath='{.data.admin}' | base64 --decode

# Solr user pw
kubectl get secret iati-prod-solrcloud-security-bootstrap -o jsonpath='{.data.solr}' | base64 --decode

### HA

Check Pods are distributed to unique Nodes
```bash
kubectl get po -l solr-cloud=iati-prod,technology=solr-cloud \
  -o json | jq -r '.items | sort_by(.spec.nodeName)[] | [.spec.nodeName] | @tsv' | uniq | wc -l
# 3

kubectl get po -l solr-cloud=iati-prod,technology=zookeeper \
  -o json | jq -r '.items | sort_by(.spec.nodeName)[] | [.spec.nodeName] | @tsv' | uniq | wc -l
# 3
```

### Monitoring

Install Prometheus stack
https://apache.github.io/solr-operator/docs/solr-prometheus-exporter/#prometheus-stack

```bash
# Check
kubectl get pods -n monitoring

```

Set up SolrPrometheusExporter
```bash
# Apply
kubectl apply -f prom-exporter.yml

kubectl logs -f -l solr-prometheus-exporter=prom-exporter-prod

kubectl port-forward $(kubectl get pod -l solr-prometheus-exporter=prom-exporter-prod --no-headers -o custom-columns=":metadata.name") 8080

curl http://localhost:8080/metrics 
```

Set up service monitor
```bash
# Apply
kubectl apply -f monitor.yml

```

Grafana Dashboards
```bash
# Port forward from grafana pod
GRAFANA_POD_ID=$(kubectl get pod -l app.kubernetes.io/name=grafana --no-headers -o custom-columns=":metadata.name" -n monitoring)
kubectl port-forward -n monitoring $GRAFANA_POD_ID 3000
```

http://localhost:3000
username: admin
initial pw: prom-operator

Import Dashboard id: `12456`
Data source: Prometheus

```bash
# Add ingress rule to allow dashboard access at dashboard.solr.iatistandard.org
kubectl apply -f dashboard-ingress.yml
``` 
## Logs
Dump exceptions (+5 lines) from Solr - make sure you have the leader pod for Solr. Otherwise the errors are about syncing between the replicas and not the root error.
```bash
kubectl logs iati-prod-solrcloud-1 | grep -A 5 SolrException > logs.txt
```

## Maintenance Commands
https://apache.github.io/solr-operator/docs/solr-cloud/managed-updates.html

Logs of Solr Operator (shows Manage)
`kubectl logs solr-operator-*`
`kubectl logs solr-operator-6d4648fc56-zms94 --since=5m | grep ManagedUpdateSelector`

### Restarts

See https://github.com/IATI/IATI-Internal-Wiki/blob/main/IATI-Unified-Infra/Solr.md

## Upgrading Solr

Update tag to version in `deployment.yml`:
```yml
solrImage:
    repository: solr
    tag: 8.10.0
```

Apply:
`kubectl apply -f deployment.yml`

### Notes
- Upgrading from 8.8.2 to 8.9.0 took approximately ~4hrs. Recommended to take a downtime for this by Solr docs.

## Upgrade Solr Operator Helm Chart
https://artifacthub.io/packages/helm/apache-solr/solr-operator#upgrading-the-solr-operator

```
> kubectl replace -f https://solr.apache.org/operator/downloads/crds/v0.4.0/all-with-dependencies.yaml

customresourcedefinition.apiextensions.k8s.io/solrbackups.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/solrclouds.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/solrprometheusexporters.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/zookeeperclusters.zookeeper.pravega.io replaced

> helm repo update
> helm upgrade solr-operator apache-solr/solr-operator --version 0.4.0

Release "solr-operator" has been upgraded. Happy Helming!
NAME: solr-operator
LAST DEPLOYED: Tue Sep 28 15:48:25 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
Solr-Operator successfully installed!
```

### Notes
- v0.3.0 -> v0.4.0 
  - kicked off rolling restart of Solr and Zookeeper pods

## Helm
Show installed charts information
`helm list --namespace <namespace:default>`

## Clean up

## DELETE Entire RG
`az group delete --name rg-solr-PROD --yes --no-wait`

### Pods from a config

`kubectl delete -f <file.yml>`

### Operators

`helm list --namespace <namespace:default>`

`helm uninstall nginx-ingress`
`helm uninstall solr-operator`
`helm uninstall cert-manager`

## TODO

- Networking
  - [X] Static IP - https://docs.microsoft.com/en-ca/azure/aks/ingress-static-ip
  - [x] Ingress
  - [x] TLS - staging cert
  - [x] TLS - prod cert
- Auth
  - [x] Basic Auth - https://apache.github.io/solr-operator/docs/solr-cloud/solr-cloud-crd.html#authentication-and-authorization
- High Availability
  - [x] Azure Availability Zones - https://docs.microsoft.com/en-ca/azure/aks/availability-zones
  - [ ] Solr Level HA
      - [x] Pod Affinity/AntiAffinity Rules
      - [ ] Zone aware replica placement 
          - `-Davailability_zone=uksouth-2` is now set on pods, need to use this now! https://solr.apache.org/operator/articles/explore-v030-gke.html
- Performance Monitoring
  - [x] Prometheus/Grafana
  - [x] Allow internet access to Dashboard
- [ ] Backups
- [X] Sizing
  - Currently on "Standard_E2as_v4" (RAM Optimised) 16 GB RAM x 3 nodes  = ~$324/mo
- CI/CD
  - Can (or should) I GitHub Actions this?
- APIM Integration
  - https://docs.microsoft.com/en-us/azure/api-management/api-management-kubernetes?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Faks%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json
  - [X] - routing from APIM to public domain name. APIM does basic auth to Solr with a policy.

## Tips

Connecting to container locally using kubectl context:
`kubectl port-forward solr-0 28983:8983`

`http://localhost:28983/solr/#/~cloud?view=nodes`

## Resources

https://apache.github.io/solr-operator/docs/local_tutorial

https://lucidworks.com/post/running-solr-on-kubernetes-part-1/

https://docs.microsoft.com/en-ca/azure/aks/ingress-basic

https://solr.apache.org/operator/articles/explore-v030-gke.html

https://www.searchstax.com/docs/hc/how-many-solr-servers/
