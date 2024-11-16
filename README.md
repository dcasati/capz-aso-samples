# capz-aso-samples

```bash
#!/usr/bin/env bash

###############################################################################################################
# variables
export RESOURCE_GROUP=rg-capz-cluster
export AZURE_LOCATION=westus3
export CLUSTER_NAME=capz-cluster
export AZURE_SUBSCRIPTION_ID=$(az account show -o tsv --query id)
export AZURE_TENANT_ID=$(az account show -o tsv --query tenantId)
export ASO_CREDENTIAL_SECRET_NAME="aso-controller-settings"
export CLUSTER_IDENTITY_NAME="cluster-identity"
export AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE="default"

az group create -n $RESOURCE_GROUP -l $AZURE_LOCATION
az aks create -g $RESOURCE_GROUP -n $CLUSTER_NAME --enable-oidc-issuer 
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME 

###############################################################################################################
# managed id
export MI_RESOURCE_GROUP="rg-mi-aso"
export MI_NAME="mi-aso"
az group create --name ${MI_RESOURCE_GROUP} --location ${AZURE_LOCATION}
az identity create --name ${MI_NAME} --resource-group ${MI_RESOURCE_GROUP} > mi-aso.json

export AZURE_CLIENT_ID=$(cat mi-aso.json|jq -r .clientId)
export AZURE_CLIENT_ID_USER_ASSIGNED_IDENTITY=$AZURE_CLIENT_ID
export SERVICE_ACCOUNT_ISSUER=$(az aks show -n ${CLUSTER_NAME} -g ${RESOURCE_GROUP} --query "oidcIssuerProfile.issuerUrl" -o tsv)

az role assignment create --assignee ${AZURE_CLIENT_ID_USER_ASSIGNED_IDENTITY} --role Contributor --scope /subscriptions/$AZURE_SUBSCRIPTION_ID
az identity federated-credential create \
  --name aso-federated-credential \
  --identity-name ${MI_NAME} \
  --resource-group ${MI_RESOURCE_GROUP}\
  --issuer ${SERVICE_ACCOUNT_ISSUER}\
  --subject "system:serviceaccount:capz-system:azureserviceoperator-default" \
  --audiences "api://AzureADTokenExchange"

cat <<EOF  > aso-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
 name: aso-credentials
 namespace: default
stringData:
 AZURE_SUBSCRIPTION_ID: "$AZURE_SUBSCRIPTION_ID"
 AZURE_TENANT_ID: "$AZURE_TENANT_ID"
 AZURE_CLIENT_ID: "$AZURE_CLIENT_ID"
 USE_WORKLOAD_IDENTITY_AUTH: "true"
EOF

kubectl apply -f aso-credentials.yaml

###############################################################################################################
# Initialize the management cluster - CAPZ
clusterctl init --infrastructure azure

###############################################################################################################
# cluster
cat <<EOF > cluster.yaml
apiVersion: resources.azure.com/v1api20200601
kind: ResourceGroup
metadata:
  name: rg-cluster-aso
  namespace: default
  annotations:
    serviceoperator.azure.com/credential-from: aso-credentials
spec:
  location: ${AZURE_LOCATION}
---
apiVersion: containerservice.azure.com/v1api20231102preview
kind: ManagedCluster
metadata:
  name: sample-managedcluster-20231102preview
  namespace: default
  annotations:
    serviceoperator.azure.com/credential-from: aso-credentials
spec:
  location: ${AZURE_LOCATION}
  sku:
    name: Base
    tier: Standard
  metricsProfile:
    costAnalysis:
      enabled: true
  owner:
    name: rg-cluster-aso
  dnsPrefix: aso
  agentPoolProfiles:
    - name: pool1
      count: 1
      vmSize: Standard_DS2_v2
      osType: Linux
      mode: System
  identity:
    type: SystemAssigned

EOF
```
kubectl apply -f cluster.yaml
