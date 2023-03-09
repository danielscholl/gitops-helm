# gitops-helm

This repo is a sample for gitops work.

## Prerequisites

You will need a Kubernetes cluster.  This can be run from Azure CloudShell.

```bash
export RESOURCE_GROUP="cluster-rg"
export NAME="cluster"
export AZURE_LOCATION=eastus

az group create -l $AZURE_LOCATION -n $RESOURCE_GROUP -onone && \
az deployment group create -g $RESOURCE_GROUP  --template-uri https://github.com/Azure/AKS-Construction/releases/download/0.9.10/main.json --parameters \
	resourceName=$NAME \
	JustUseSystemPool=true \
	nodePoolName=nodepool1 \
  agentCount=2 \
  agentVMSize=Standard_DS2_v2 \
	osDiskType=Managed \
	enableTelemetry=false \
	fluxGitOpsAddon=true

az k8s-configuration flux create -g $RESOURCE_GROUP \
	-c aks-${NAME} \
	-n gitops-helm \
	--namespace flux-system \
	-t managedClusters \
	--scope cluster \
	-u https://github.com/danielscholl/gitops-helm \
	--branch main  \
	--kustomization name=infra path=./infrastructure prune=true \
	--kustomization name=apps path=./apps/staging prune=true dependsOn=\["infra"\]
```

## Configuration

Create a Gitops Configuration.

```bash
# Deploy Application
export RESOURCE_GROUP="test-rg"
export NAME="test"
export AZURE_LOCATION=eastus

az k8s-configuration flux create -g $RESOURCE_GROUP \
	-c aks-${NAME} \
	-n gitops-helm \
	--namespace flux-system \
	-t managedClusters \
	--scope cluster \
	-u https://github.com/danielscholl/gitops-helm \
	--branch main  \
	--kustomization name=infra path=./infrastructure prune=true \
	--kustomization name=apps path=./apps/staging prune=true dependsOn=\["infra"\]
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools such as NGINX ingress controller and Helm repository definitions
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
    ├── cert-manager
    ├── istio
│   ├── nginx
│   ├── redis
│   └── sources
└── clusters
    ├── production
    └── staging
```

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/production/** dir contains the production Helm release values
- **apps/staging/** dir contains the staging values

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       └── release.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── staging
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

In **apps/base/podinfo/** dir we have a HelmRelease with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 5m
  values:
    cache: redis-master.redis:6379
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
```

In **apps/staging/** dir we have a Kustomize patch with the staging specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.staging
```

Infrastructure:

```
./infrastructure/
├── nginx
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── redis
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
└── sources
    ├── bitnami.yaml
    ├── istio-git.yaml
    ├── istio.yaml
    ├── jetstack.yaml
    ├── kustomization.yaml
    └── podinfo.yaml
```

In **infrastructure/sources/** dir we have the Helm repositories definitions:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: podinfo
spec:
  interval: 5m
  url: https://stefanprodan.github.io/podinfo
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
```

Note that with ` interval: 5m` we configure Flux to pull the Helm repository index every five minutes.
If the index contains a new chart version that matches a `HelmRelease` semver range, Flux will upgrade the release.
