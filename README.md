# Solr with Kubernetes

Repo containing kubernetes deployment information for IATI Solr Production instance

## Current Cluster

NOTE: the current cluster is named `aks-solr-test` in resource group `rg-solr-test`. The below instructions assume you're creating a NEW cluster called `aks-solr-new-dev`. 

https://github.com/IATI/IATI-Internal-Wiki/blob/main/IATI-Unified-Infra/Solr.md#dev

```bash
# Get context of this cluster and set that as your kubectl context
az aks get-credentials --resource-group aks-solr-test --name aks-solr-test
```

## New Cluster Deployment Steps

### Initial Setup / kubectl Context

```bash
# Set ENV
ENV=new-dev

# Create resource group
az group create --resource-group rg-solr-$ENV --location uksouth
RG=rg-solr-$ENV

# Creating AKS Cluster with 1 node (dev sized)
az aks create \
    --resource-group $RG \
    --name aks-solr-$ENV \
    --generate-ssh-keys \
    --node-vm-size "standard_B4ms" \
    --vm-set-type VirtualMachineScaleSets \
    --load-balancer-sku standard \
    --node-count 1 \
    --auto-upgrade-channel stable # auto-upgrade dev cluster ONLY

# Get context of your cluster and set that as your kubectl context
az aks get-credentials --resource-group $RG --name aks-solr-$ENV

# list nodes in cluster and zones
kubectl get nodes -o custom-columns=NAME:'{.metadata.name}',REGION:'{.metadata.labels.topology\.kubernetes\.io/region}',ZONE:'{metadata.labels.topology\.kubernetes\.io/zone}'

# NAME                                REGION    ZONE
# aks-nodepool1-19045045-vmss000004   uksouth   0
```

https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

### Ingress

```bash
# Create Public IP in Azure
POD_RG=$(az aks show --resource-group $RG --name aks-solr-$ENV --query nodeResourceGroup -o tsv)
IP_ADDRESS=$(az network public-ip create --resource-group $POD_RG --name pip-solr-$ENV --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv)

# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Use Helm to deploy an NGINX ingress controller
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
    --set controller.replicaCount=1 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP="$IP_ADDRESS" \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="aks-solr-$ENV" \
    --set-string controller.config.proxy-body-size="0" \
    --set-string controller.config.large-client-header-buffers="4 128k" \
    --set-string controller.config.client-body-buffer-size="50M"

# Check 
kubectl get pods -l app.kubernetes.io/name=ingress-nginx \
  --field-selector status.phase=Running

kubectl get services -o wide -w nginx-ingress-ingress-nginx-controller

az network public-ip list --resource-group $POD_RG --query "[?name=='pip-solr-$ENV'].[dnsSettings.fqdn]" -o tsv
# aks-solr-$ENV.uksouth.cloudapp.azure.com
```

### Upgrade / Config Change NGINX

https://github.com/kubernetes/ingress-nginx/

```zsh
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
    # Above Values to re-use, or try --reuse-values
    # ADD MORE
```

Upgrade with same values
```zsh
helm upgrade --reuse-values nginx-ingress ingress-nginx/ingress-nginx
```

### Secrets

https://github.com/bitnami-labs/sealed-secrets#installation

#### Install Sealed Secrets
```bash
helm install sealed-secrets sealed-secrets \
--repo https://bitnami-labs.github.io/sealed-secrets
```

#### Install kubeseal cli tool
- Follow above link

### Create a sealed secret

```bash
kubeseal --controller-name sealed-secrets --controller-namespace default -o yaml < my-secret.yaml > my-secret-sealed.yaml
kubectl apply -f my-secret-sealed.yaml
```
Commit my-secret-sealed.yaml into git with no worries. Delete my-secret.yaml

### Certificate Generation

Install cert-manager (if not already installed): 
https://artifacthub.io/packages/helm/cert-manager/cert-manager

Issue Certificate Using Cloudflare DNS lookup challenge cert-manager
https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/

```bash
# Install CRDs separately to allow you to easily uninstall and reinstall cert-manager without deleting your installed custom resources.
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.crds.yaml
# Install cert-manager from helm chart
helm install cert-manager --version v1.8.0 jetstack/cert-manager
```

Create cloudflare api token secret
Follow https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/ to get API and an put in .yaml file. Then seal the secret following instructions above for using kubseal.

```bash
# Create certificate ClusterIssuer
cd infrastructure/certificates/secrets
kubeseal --controller-name sealed-secrets --controller-namespace default -o yaml < cloudflare-api-token-secret.yaml > cloudflare-api-token-sealed.yaml
kubectl apply -f cloudflare-api-token-sealed.yaml # api token secret, needs to be re-created per cluster
kubectl apply -f ../cert-issuer.yml

# Issue certificate
kubectl apply -f pkcs12-keystore-password-dev.yaml # password, needs to be re-created per cluster, sealed doesn't work for Solr.
kubectl apply -f ../certificate.yml

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

### Upgrading cert-manager
https://cert-manager.io/docs/installation/upgrading/

```bash
helm upgrade --version v1.8.0 cert-manager jetstack/cert-manager
```

### Solr Cloud

https://apache.github.io/solr-operator/docs/running-the-operator

```bash
# Install operator helm repo
helm repo add apache-solr https://solr.apache.org/charts
helm repo update

# Install helm chart and then solr-operator
kubectl create -f https://solr.apache.org/operator/downloads/crds/v0.5.1/all-with-dependencies.yaml
helm install solr-operator apache-solr/solr-operator --version 0.5.1

# Check on whats running
kubectl get pod -l control-plane=solr-operator
kubectl describe pod -l control-plane=solr-operator

# Verify zookeeper
kubectl get pod -l component=zookeeper-operator

# Create service permissions for zone setting on node
k apply -f infrastructure/auth/service-account.yml

# Deploy
kubectl apply -f solr/deployment.yml

# watch nodes come up
kubectl get solrclouds -w

# check ingress config for URL
kubectl get ingress
kubectl describe ingress iati-$ENV-solrcloud-common

# Get initial credentials for admin user, if password is changed using admin API, the secret is NOT updated.
kubectl get secret iati-$ENV-solrcloud-security-bootstrap \
  -o jsonpath='{.data.admin}' | base64 --decode

# Solr user pw
kubectl get secret iati-$ENV-solrcloud-security-bootstrap -o jsonpath='{.data.solr}' | base64 --decode

```

### Monitoring (Grafana)

Install Prometheus stack
https://apache.github.io/solr-operator/docs/solr-prometheus-exporter/#prometheus-stack

Install w/ helm and make persistent, set root_url
```
helm upgrade mon prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --install \
  --set grafana.enabled=true \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi \
  --set grafana.'grafana\.ini'.server.root_url=https://dashboard.solr-dev.iatistandard.org/
```

```bash
# Check
kubectl get pods -n monitoring

```

Set up SolrPrometheusExporter
```bash
# Apply
kubectl apply -f monitoring/prom-exporter.yml

kubectl logs -f -l solr-prometheus-exporter=prom-exporter-$ENV

kubectl port-forward $(kubectl get pod -l solr-prometheus-exporter=prom-exporter-$ENV --no-headers -o custom-columns=":metadata.name") 8080

curl http://localhost:8080/metrics 
```

Set up service monitor
```bash
# Apply
kubectl apply -f monitoring/monitor.yml

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

#### Login and change password, or use Grafana CLI. CLI can be used to reset pw if forgotten

```bash
k exec -n monitoring <grafana-pod-name> -c grafana -- grafana-cli admin reset-admin-password <pw>
```

#### Ingress for dashboard

```bash
# Add ingress rule to allow dashboard access at dashboard.solr-$ENV.iatistandard.org
kubectl apply -f infrastructure/ingress/dashboard-ingress.yml
``` 
#### Importing a Dashboard

##### From the Source
Import Dashboard id: `12456`
Data source: Prometheus

##### From existing IATI Dashboard
Use this to keep our custom charts and alerts

Import > Upload JSON File > `monitoring/Grafana_Dashboard_Dev.json`

### Monitoring (Azure Monitor)

- Follow instructions [here](https://docs.microsoft.com/en-us/azure/aks/monitor-aks#configure-monitoring)


## Logs
Dump exceptions (+5 lines) from Solr - make sure you have the leader pod for Solr. Otherwise the errors are about syncing between the replicas and not the root error.
```bash
kubectl logs iati-prod-solrcloud-0 | grep -A 5 SolrException > logs.txt
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
`kubectl apply -f solr/deployment.yml`

### Notes
- Upgrading from 8.8.2 to 8.9.0 took approximately ~4hrs. Recommended to take a downtime for this by Solr docs.

## Upgrade Solr Operator Helm Chart
https://artifacthub.io/packages/helm/apache-solr/solr-operator#upgrading-the-solr-operator

```
> kubectl replace -f https://solr.apache.org/operator/downloads/crds/v0.5.0/all-with-dependencies.yaml

customresourcedefinition.apiextensions.k8s.io/solrbackups.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/solrclouds.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/solrprometheusexporters.solr.apache.org replaced
customresourcedefinition.apiextensions.k8s.io/zookeeperclusters.zookeeper.pravega.io replaced

> helm repo update
> helm upgrade solr-operator apache-solr/solr-operator --version 0.5.0

Release "solr-operator" has been upgraded. Happy Helming!
NAME: solr-operator
LAST DEPLOYED: Mon Nov 22 14:28:48 2021
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
  - Took a few hours

- v0.4.0 -> v0.5.0 `dev`
  - restarted the solr pod
  - was back up in ~2min
  - Collections still there, integration tests passed

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
