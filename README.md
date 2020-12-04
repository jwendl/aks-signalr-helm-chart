# An Example Helm Chart to Deploy Signal R to AKS

This is an example Helm Chart to Deploy Signal R to an AKS cluster. This project assumes that you have setup an AKS cluster using Azure similar to the [walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough).

> Note: this repository is a work in progress and it will be a bit until I get a chance to clean it up.

## Running the Example

It's assumed that you have helm installed using the following [instructions](https://helm.sh/docs/intro/install/).

Once installed use the following helm commands to install the required charts to the cluster for this example to work.

``` bash
clusterName=$ClusterName
resourceGroupName="$ResourceGroupName"

echo "Retrieving location..."
location=$(az aks show --resource-group $resourceGroupName --name "${ClusterName}" --query location --output tsv)

echo "Retrieving user identity..."
userIdentityId=$(az identity show --resource-group "MC_${resourceGroupName}_${ClusterName}_${location}" --name "${ClusterName}-agentpool" --query clientId --output tsv)

echo "Adding helm repros"
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm repo add azure-marketplace https://marketplace.azurecr.io/helm/v1/repo 
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo update

echo "Installing / upgrading helm pods for redis"
helm upgrade --install redis-backplane azure-marketplace/redis --namespace redis-backplane --create-namespace
helm upgrade --install redis-cache azure-marketplace/redis --namespace redis-cache --create-namespace
helm upgrade --install redis-cache-api azure-marketplace/redis --namespace redis-cache-api --create-namespace

echo "Installing / upgrading helm pods for nginx"
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace nginx --create-namespace --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="${resourcePrefix}aks"

echo "Installing / upgrading helm pods for cert-manager"
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer

echo "Installing / upgrading secrets provider"
helm upgrade --install csi-secrets-store-provider-azure csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace csi-secrets --create-namespace
```

An example of installing your application (to be run under the helm folder)

``` bash
helm upgrade --install ${pubSubContainerName} ./ --namespace ${pubSubContainerName} --create-namespace \
    --set environment=development \
    --set emailAddress=$emailAddress \
    --set apphost=k8s \
    --set label.name=apiservice \
    --set azure.resourceGroupName=$resourceGroupName \
    --set container.name=${pubSubContainerName} \
    --set container.pullPolicy=Always \
    --set container.image=${resourcePrefix}acr.azurecr.io/${pubSubContainerName} \
    --set container.tag=v1 \
    --set container.port=7100 \
    --set replicas=2 \
    --set ingress.hostName=${resourcePrefix}aks.${location}.cloudapp.azure.com \
    --set redis.backplane.host=redis-backplane-master.redis-backplane.svc.cluster.local \
    --set redis.cache.host=redis-cache-master.redis-cache.svc.cluster.local \
    --set aadOptions.authority=$Authority \
    --set aadOptions.validIssuer=$ValidIssuer \
    --set aadOptions.validAudieence=$ValidAudience \
    --set service.port=7100 \
    --set service.type=ClusterIP \
    --set keyVault.userIdentityId=$userIdentityId \
    --set keyVault.keyVaultName=${resourcePrefix}kv \
    --set keyVault.resourceGroup=$resourceGroupName \
    --set keyVault.subscriptionId=$subscriptionId \
    --set keyVault.tenantId=$tenantId
```
