# gitops-helm

This repo is a sample for gitops work.

## Prerequisites

Register the preview features

```bash
# Register
az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

# Show
az feature show --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

# Refresh
az provider register --namespace Microsoft.ContainerService
```

## Deploy

You will need a Kubernetes cluster.  This can be run from Azure CloudShell.

```bash
export RESOURCE_GROUP="cluster-rg"
export NAME="cluster"
export AZURE_LOCATION="eastus"
export DNS_DOMAIN="<your_domain>"

############
# STEP 1: create Azure Resource Group
############################################
az group create -l $AZURE_LOCATION -n $RESOURCE_GROUP


############
# STEP 2: create Azure DNS Zone (Optional)
############################################
az network dns zone create \
  --resource-group ${RESOURCE_GROUP} \
  --name ${DNS_DOMAIN}


############
# STEP 3: create Azure Kubernetes Service cluster
############################################
az deployment group create --no-wait -g $RESOURCE_GROUP  --template-uri https://github.com/Azure/AKS-Construction/releases/download/0.9.10/main.json --parameters \
	resourceName=$NAME \
	JustUseSystemPool=true \
	nodePoolName=nodepool1 \
  agentCount=2 \
  agentVMSize=Standard_DS2_v2 \
	osDiskType=Managed \
	keyVaultAksCSI=true \
	keyVaultCreate=true \
	keyVaultOfficerRolePrincipalId=$(az ad signed-in-user show --query id --out tsv) \
	enableTelemetry=false \
  oidcIssuer=true \
	workloadIdentity=true \
	fluxGitOpsAddon=true

az deployment group list -g $RESOURCE_GROUP --query [].properties.provisioningState -otsv


############ 
# STEP 4: retrieve KUBECONFIG to access AKS cluster
############################################
az aks get-credentials \
  --resource-group ${RESOURCE_GROUP} \
  --name aks-${NAME}



#######
# STEP 5: Assign Role to kubelet identity. (TEST ONLY)
#############
export DNS_SCOPE=$(
  az network dns zone list \
    --query "[?name=='$DNS_DOMAIN'].id" \
    --output table | tail -1
)

export PRINCIPAL_ID=$(
  az aks show -g $RESOURCE_GROUP -n aks-${NAME} \
    --query "identityProfile.kubeletidentity.objectId" \
    --output tsv
)

# Create
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "DNS Zone Contributor" \
  --scope "$DNS_SCOPE"

# Verify
az role assignment list --assignee $PRINCIPAL_ID --all \
  --query '[].{roleDefinitionName:roleDefinitionName, provider:scope}' \
  --output table | sed 's|/subscriptions.*providers/||' | cut -c -80


VAULT=$(az keyvault list -g $RESOURCE_GROUP --query [].name -otsv)
az keyvault secret set --name TENANT-ID --vault-name $VAULT --value $(az account show --query homeTenantId -otsv)
az keyvault secret set --name SUBSCRIPTION-ID --vault-name $VAULT --value $(az account show --query id -otsv)
az keyvault secret set --name RESOURCE-GROUP --vault-name $VAULT --value $RESOURCE_GROUP
az keyvault secret set --name DNS-DOMAIN --vault-name $VAULT --value $DNS_DOMAIN


# Create multiple YAML objects from stdin
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: externaldns-values
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "nginx"
data:
  tenant_id: $(az account show --query homeTenantId -otsv)
  subscription_id: $(az account show --query id -otsv)
  resource_group: ${RESOURCE_GROUP}
  dns_domain: ${DNS_DOMAIN}
  principal_id: $(az aks show -g $RESOURCE_GROUP -n aks-${NAME} --query "identityProfile.kubeletidentity.objectId" -otsv)
EOF

############ 
# STEP 5: configure AKS cluster
############################################
az k8s-configuration flux create -g $RESOURCE_GROUP \
	-c aks-${NAME} \
	-n gitops-helm \
	--namespace flux-system \
	-t managedClusters \
	--scope cluster \
	-u https://github.com/danielscholl/gitops-helm \
	--branch main  \
	--kustomization name=components path=./components prune=true \
	--kustomization name=configurations path=./configurations prune=true dependsOn=\["components"\] \
	--kustomization name=apps path=./apps/staging prune=true dependsOn=\["configurations"\]
```

## Access PodInfo

```bash
curl --header "Host: podinfo.staging" https://$(kubectl get svc -n nginx nginx-ingress-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```