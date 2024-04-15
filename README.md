# Two GitOps Titans, One Powerful Solution: Flamingo (Flux + Argo)!

![License](https://img.shields.io/badge/license-MIT-green.svg)

- Session @ Open Source Summit NA 2024: [<https://sched.co/1aBNh>]([https://sched.co/1aBNh)

## Overview

The landscape of Kubernetes and containerized applications has evolved rapidly in recent years, demanding innovative solutions to manage and orchestrate deployments seamlessly. Flamingo, the Flux subsystem for ArgoCD, is a powerful integration of these two GitOps titans, to supercharge your workflows. This dynamic duo creates a seamless and powerful GitOps-driven platform for managing Kubernetes applications and infrastructure. Join us for a deep dive into Flamingo and discover how it can revolutionize your GitOps workflows This lab will guide you step-by-step through the process of getting started with a Flamingo management cluster and two edge clusters to deploy and manage a sample application.

## Architectural Diagram

TO-DO

## Step 1: Install Prerequisites

To get started with the lab, we will need to meet some tool prerequisites. For our lab, we will be using Azure Kubernetes Service (AKS), but you may use a cloud provider of your choice. Simply replace the commands with the equivalent ones needed to create a Kubernetes cluster and merge them into your existing kubeconfig.

```bash
# Install Azure CLI
brew install azure-cli

# Install Flux CLI 
brew install fluxcd/tap/flux

# Install Flamingo CLI
brew install flux-subsystem-argo/tap/flamingo
```

## Step 2: Create AKS management and edge clusters

```bash
# Set your Azure Subscription ID and necessary variables for cluster creation
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
az account set --subscription $AZURE_SUBSCRIPTION_ID

export CLUSTER_RG=oss24
export MGMT_CLUSTER_NAME=flamingo-management
export LOCATION=southcentralus
export NODE_COUNT=2
export AZ_AKS_VERSION=1.27.9

# Create a resource group for the AKS clusters
az group create --name $CLUSTER_RG --location $LOCATION

# Create an AKS management cluster
az aks create -k $AZ_AKS_VERSION -y -g $CLUSTER_RG \
    -s Standard_B4ms -c $NODE_COUNT \
    --network-plugin kubenet -n $MGMT_CLUSTER_NAME

# Connect to the cluster and merge the kubeconfig
az aks get-credentials --resource-group $CLUSTER_RG --name $MGMT_CLUSTER_NAME --overwrite-existing

# Create edge clusters and merge them into the kubeconfig.
for LOCATION in eastus2 westus2; do \
  EDGE_CLUSTER_NAME="edge-dev-$LOCATION"; \
  az aks create -k $AZ_AKS_VERSION -y -g $CLUSTER_RG \
      -s Standard_B4ms -c $NODE_COUNT \
      --network-plugin kubenet -n $EDGE_CLUSTER_NAME \
      --location $LOCATION; \
      az aks get-credentials --resource-group $CLUSTER_RG --name $EDGE_CLUSTER_NAME --overwrite-existing
done
```

## Step 3: Install Flamingo in Management Cluster

```bash

# Switch Context to Management Cluster
kubectl config use-context $MGMT_CLUSTER_NAME

# Install Argo CD + Flamingo (You can also add the --export flag to generate the YAML files and manually apply them using kubectl apply)
flamingo install

# Verify the pods are running correctly
kubectl get pods -n argocd

# Obtain the initial password by running the following command to login. The default username is admin.
flamingo show-init-password

# Portforward to http://localhost:8080
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

> Access the Flamingo UI: [http://localhost:8080](http://localhost:8080).

## Step 4: Install Flux Controllers in each edge cluster

```bash
export GITHUB_TOKEN=<update with your github PAT>

for LOCATION in eastus2 westus2; do \
  EDGE_CLUSTER_NAME="edge-dev-$LOCATION"; \
  kubectl config use-context "$EDGE_CLUSTER_NAME"; \
  flux bootstrap github --owner="joaquinrz" --repository="flamingo-oss24" --private=false --personal=true --path="clusters/$EDGE_CLUSTER_NAME"; \
done

git pull
```

## Step 5: Deploy podinfo-kustomize in each edge cluster

```bash

for LOCATION in eastus2 westus2; do \
  EDGE_CLUSTER_NAME="edge-dev-$LOCATION"; \
  kubectl config use-context "$EDGE_CLUSTER_NAME"; \
  cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo-kustomize
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: podinfo
  namespace: podinfo-kustomize
spec:
  interval: 10m
  url: oci://ghcr.io/stefanprodan/manifests/podinfo
  ref:
    tag: latest
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: podinfo-kustomize
spec:
  interval: 10m
  targetNamespace: podinfo-kustomize
  prune: true
  sourceRef:
    kind: OCIRepository
    name: podinfo
  path: ./
EOF
done
```

## Step 6: Deploy podinfo-helm in each edge cluster

```bash

for LOCATION in eastus2 westus2; do \
  EDGE_CLUSTER_NAME="edge-dev-$LOCATION"; \
  kubectl config use-context "$EDGE_CLUSTER_NAME"; \
  cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo-helm
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: podinfo
  namespace: podinfo-helm
spec:
  interval: 10m
  type: oci
  url: oci://ghcr.io/stefanprodan/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo-helm
spec:
  interval: 10m
  targetNamespace: podinfo-helm
  chart:
    spec:
      chart: podinfo
      version: '*'
      sourceRef:
        kind: HelmRepository
        name: podinfo
      verify:
        provider: cosign # keyless
EOF
done
```

## Step 7: Register Edge Clusters in Flamingo

```bash
kubectl config use-context $MGMT_CLUSTER_NAME

flamingo list-clusters

for LOCATION in eastus2 westus2; do \
  EDGE_CLUSTER_NAME="edge-dev-$LOCATION"; \
  flamingo add-cluster $EDGE_CLUSTER_NAME --insecure; \
done

flamingo list-clusters

```

## Step 8: Register the Flux applications in Flamingo

```bash
kubectl config use-context $MGMT_CLUSTER_NAME
flamingo generate-app --app-name podinfo-ks-eastus2 edge-dev-eastus2/ks/podinfo -n podinfo-kustomize
flamingo generate-app --app-name podinfo-ks-westus2 edge-dev-westus2/ks/podinfo -n podinfo-kustomize
flamingo generate-app --app-name podinfo-helm-eastus2 edge-dev-eastus2/hr/podinfo -n podinfo-helm
flamingo generate-app --app-name podinfo-helm-westus2 edge-dev-westus2/hr/podinfo -n podinfo-helm
```
