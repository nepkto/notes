# DevOps & CI/CD Infrastructure Reference Guide

This comprehensive reference document summarizes the architectural concepts, terminologies, and deployment workflows discussed regarding **GitHub Actions**, **Azure DevOps**, and **Azure Container Registry (ACR)**.

---

## 1. CI/CD Workers: Runners vs. Agents

In automation pipelines, the machine that compiles code, runs tests, and executes deployment scripts has different names depending on the platform, though their ultimate purpose is identical.

### Definitions
* **Runner**: The nomenclature utilized by [GitHub Actions](https://github.com/features/actions) and **GitLab CI/CD**.
* **Agent**: The nomenclature utilized by [Azure Pipelines](https://azure.microsoft.com/en-us/products/devops/pipelines), **Jenkins**, and **TeamCity**.

### Technical Architectural Differences

| Feature | Runner (GitHub / GitLab) | Agent (Azure DevOps / Jenkins) |
| :--- | :--- | :--- |
| **Communication Model** | **Pull (Outbound Polling):** The runner client periodically polls the central server over HTTPS to fetch jobs. | **Push or Pull:** Modern agents poll via outbound HTTPS, but legacy setups (like Jenkins SSH) rely on the master server opening inbound ports to trigger jobs. |
| **Firewall Requirements** | Highly secure; requires no inbound open ports. Only outbound network access is needed. | Varies; legacy push models require complex inbound port configurations. |
| **State & Isolation** | Heavily emphasizes ephemeral execution (fresh virtual machines or distinct containers per step). | Historically multi-threaded on a single persistent machine workspace, though moving toward containerized architectures. |

---

## 2. Infrastructure Grouping: Pools vs. Groups

When scale demands managing multiple workers, platforms group individual machines to simplify access control and job routing.

* **Azure DevOps "Agent Pool"** $\equiv$ **GitHub Actions "Runner Group"**

### How Workloads are Routed

#### Azure DevOps (Explicit Pools)
Jobs target a specific named pool directly in the YAML definition. Azure randomly assigns the workload to an idle agent inside that designated collective.
```yaml
pool: 'Windows-Enterprise-Pool'
```

#### GitHub Actions (Label-Based Routing)
Instead of matching a group directly, jobs target specific **Labels** assigned to individual runners. GitHub scans authorized **Runner Groups** to find a machine containing every requested label.
```yaml
runs-on: [self-hosted, linux, gpu]
```
*Note: Runner Groups in GitHub serve primarily as an administrative security boundary at the Organization or Enterprise level to control which repositories can access specific hardware.*

---

## 3. Azure Container Registry (ACR) Deployment

An **Azure Container Registry** manages and stores private Docker container images. 

### SKU Tier Comparison

| SKU | Intended Use Case | Key Features |
| :--- | :--- | :--- |
| **Basic** | Learning, testing, and small personal projects. | Cost-optimized, lower storage limits, basic throughput. |
| **Standard** | Production scenarios for most business applications. | Balanced cost, increased storage capacity, and higher throughput. |
| **Premium** | Enterprise-grade deployments. | Includes advanced capabilities like **Geo-replication**, private endpoints, and availability zones. |

### Deployment Methods

#### Method A: Azure CLI (Recommended for Speed)
Execute the following to establish a resource group and spin up an ACR instance. *(Note: The registry name must be globally unique across all of Azure).*

```bash
# 1. Create a Resource Group
az group create --name MyResourceGroup --location eastus

# 2. Create the Container Registry
az acr create   --resource-group MyResourceGroup   --name myuniqueacr2026   --sku Basic
```

#### Method B: Bicep (Infrastructure as Code)
For automated pipeline deployments, create a declarative file named `main.bicep`:

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

Deploy using the following command:
```bash
az deployment group create --resource-group MyResourceGroup --template-file main.bicep
```

---

## 4. Post-Deployment Authentication & Usage

To securely connect your machine or CI/CD worker to the registry to push or pull images, invoke the registry login wrapper:

```bash
az acr login --name myuniqueacr2026
```