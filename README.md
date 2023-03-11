# gitops-helm

This repo is a sample for gitops work.

## Prerequisites

You will need a Kubernetes cluster.  This can be run from Azure CloudShell.

```bash
export RESOURCE_GROUP="cluster-rg"
export NAME="cluster"
export AZURE_LOCATION=eastus

az group create -l $AZURE_LOCATION -n $RESOURCE_GROUP -onone && \
az deployment group create --no-wait -g $RESOURCE_GROUP  --template-uri https://github.com/Azure/AKS-Construction/releases/download/0.9.10/main.json --parameters \
	resourceName=$NAME \
	JustUseSystemPool=true \
	nodePoolName=nodepool1 \
  agentCount=2 \
  agentVMSize=Standard_DS2_v2 \
	osDiskType=Managed \
	enableTelemetry=false \
	fluxGitOpsAddon=true

az deployment group list -g $RESOURCE_GROUP --query [].properties.provisioningState -otsv

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

Locate the public IP address of the ingress controller:

```bash
kubectl get svc -n ingress-nginx
```

Add the IP address to your hosts file:

```bash
echo "$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}') podinfo.staging" | sudo tee -a /etc/hosts
```

