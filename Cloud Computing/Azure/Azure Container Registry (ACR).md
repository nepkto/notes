# Azure Container Registry (ACR) Reference Guide

An Azure Container Registry (ACR) is a managed, private Docker registry service based on the open-source Docker Registry 2.0. It allows you to store and manage container images and related artifacts for all types of container deployments.

## Prerequisites
Before creating a registry, you must have an active Azure Resource Group. If needed, create one using:
```bash
az group create --name MyResourceGroup --location eastus
```

---

## Deployment Methods

### 1. Azure CLI (Recommended for Automation)
The registry name must be **globally unique** across Azure and contain only alphanumeric characters.

```bash
az acr create \
  --resource-group MyResourceGroup \
  --name myuniqueacr2026 \
  --sku Basic
```

### 2. Infrastructure as Code (Bicep)
For automated pipelines (e.g., GitHub Actions), define the resource in a `main.bicep` file:

```bicep
param acrName string = 'myuniqueacr2026'
param location string = resourceGroup().location

resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: acrName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
}
```

Deploy via CLI:
```bash
az deployment group create --resource-group MyResourceGroup --template-file main.bicep
```

### 3. Azure Portal
1. Navigate to the **Azure Portal**.
2. Search for **Container registries** and click **Create**.
3. Select your **Subscription**, **Resource Group**, and enter a unique **Registry name**.
4. Choose your **Location** and **SKU**.
5. Click **Review + create**, then **Create**.

---

## Registry SKUs Comparison

| SKU | Best For | Features |
| :--- | :--- | :--- |
| **Basic** | Learning, testing, and small hobby projects | Cost-optimized, lower storage limits, entry-level throughput. |
| **Standard** | Most production scenarios | Balanced cost, increased storage limits, and higher throughput. |
| **Premium** | Enterprise-grade deployments | High throughput, [Geo-replication](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication), Private Endpoints, Customer-managed keys. |

---

## Post-Deployment Authentication

To push or pull images locally after creation, authenticate using the Azure CLI:
```bash
az acr login --name myuniqueacr2026
```