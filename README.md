```markdown
# Azure Private Link Implementation: Secure Cross-VNet Connectivity using Bicep

![Azure Private Link Banner](/images/privatelink_banner.png)

## Introduction

In this project, I architected and deployed a secure, private connectivity solution using **Azure Private Link** and **Infrastructure as Code (Bicep)**. The primary goal was to expose an internal business application hosted in a "Provider" virtual network to a "Consumer" network without ever traversing the public internet.

By leveraging **Azure Bicep**, I automated the provisioning of the entire stack—including a Standard Load Balancer, Private Link Service, and Private Endpoints—ensuring a repeatable and error-free deployment. This architecture eliminates public attack vectors by keeping all traffic strictly within the Microsoft backbone network, a critical requirement for compliance-heavy environments (PCI-DSS, HIPAA).

## Objectives

- **Automate Infrastructure:** Deploy a multi-VNet environment using a modular Azure Bicep template.
- **Implement Private Connectivity:** Configure Azure Private Link Service to expose backend resources securely.
- **Eliminate Public Exposure:** Ensure the backend application has no public IP addresses.
- **Secure Consumption:** Deploy a Private Endpoint in the consumer network to access the service via a private 10.x IP.
- **High Availability:** Configure a Standard SKU Load Balancer to front-end the application logic.

## Tech Stack

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Bicep](https://img.shields.io/badge/Bicep-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Azure Private Link](https://img.shields.io/badge/Azure_Private_Link-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Load Balancer](https://img.shields.io/badge/Standard_Load_Balancer-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)

- **Azure Bicep** - Declarative Infrastructure as Code (IaC).
- **Azure Private Link** - Service tunneling over Microsoft Backbone.
- **Azure Private Endpoint** - Network Interface for private consumption.
- **Standard Load Balancer** - L4 Traffic distribution required for Private Link.
- **Azure CLI** - Deployment execution.

## Architecture Overview

### Deployment Architecture

![Private Link Architecture Diagram](/images/privatelink_architecture.png)

The architecture consists of two isolated Virtual Networks:

1.  **Provider Network (`WhizlabsVNet`):** Hosts the application (IIS on Windows Server 2019) behind a **Standard Load Balancer**. A **Private Link Service** is attached to the Load Balancer's frontend IP.
2.  **Consumer Network (`whizPEVnet`):** Simulates a client environment. It accesses the provider application ONLY via a **Private Endpoint** (a local NIC with IP `10.0.0.5`), which tunnels traffic via the Microsoft Backbone.

## Implementation Steps

### Phase 1: Infrastructure as Code (Bicep Definition)

I defined the entire infrastructure in a single `main.bicep` file to ensure consistency. This template orchestrates the creation of 15+ resources including VNets, NICs, VMs, and the Private Link configuration.

**Key Bicep Components:**
* `Microsoft.Network/privateLinkServices`: Defines the service exposure.
* `Microsoft.Network/privateEndpoints`: Connects the consumer VNet to the service.
* `Microsoft.Network/loadBalancers`: Standard SKU (Required for PLS).

![Bicep Code Structure](/images/bicep_code.png)
*Figure 1: Defining the Private Link Service and Load Balancer relationships in Bicep*

### Phase 2: Automated Deployment

Executed the deployment using Azure CLI. This demonstrates the power of IaC—deploying a complex, multi-tier network architecture in under 10 minutes with a single command.

```bash
az group create --name MyResourceGroup --location eastus
az deployment group create --resource-group MyResourceGroup --template-file main.bicep

```

*Figure 2: Successful provisioning of resources via Azure CLI*

### Phase 3: Network Verification

Post-deployment, I verified that the **Private Link Service** was correctly mapped to the Load Balancer and that the **Private Endpoint** was approved and active in the Consumer VNet.

* **Private Endpoint IP:** `10.0.0.5`
* **Connection Status:** `Approved`

*Figure 3: All resources successfully deployed in the Resource Group*

### Phase 4: Connectivity Testing

To validate the "Private" nature of the link:

1. I logged into the **Consumer VM** via RDP.
2. Opened a browser and navigated to `http://10.0.0.5` (The Private Endpoint IP).
3. **Result:** The IIS default page from the Provider VM loaded successfully.

*Figure 4: Successfully accessing the Provider Application via the Private Endpoint IP*

## Security & Strategic Analysis

### Why use Bicep?

Using Bicep instead of manual portal click-ops ensures **idempotency**. I can redeploy this exact secure architecture to a new region or subscription without risk of configuration drift or human error.

### Security Benefits (Private Link vs. Public Service Endpoints)

| Feature | Service Endpoints | Private Link (This Project) |
| --- | --- | --- |
| **Traffic Path** | Microsoft Backbone | Microsoft Backbone |
| **Public IP Required?** | No | **No** |
| **Data Exfiltration Risk** | Higher (Access entire service) | **Low (Mapped to specific instance)** |
| **Cross-Region?** | No | **Yes (Global Reach)** |
| **Consumer IP** | Public IP of VNet | **Private IP (10.x.x.x)** |

### Cost Optimization

* **Standard Load Balancer:** While more expensive than Basic, it is strictly required for Private Link Services.
* **Data Processing:** Private Link incurs a cost per GB processed (ingress/egress).
* **Recommendation:** For dev/test environments, deallocate the backend VMs when not in use, as the Private Link infrastructure costs are minimal when idle.

## Key Results

* **100% Automated Deployment:** Provisioned a full Producer-Consumer network topology using Azure Bicep.
* **Zero Internet Exposure:** Successfully exposed an IIS application solely via the Microsoft Backbone network.
* **Secure Consumption:** Validated that a consumer client could access the service using a private, internal IP address (`10.0.0.5`).
* **Compliance Ready:** Established a pattern suitable for PCI-DSS/HIPAA environments where public endpoints are strictly prohibited.

---

## Repository Contents

* `/bicep/` - The `main.bicep` source code.
* `/images/` - Architecture diagrams and proof-of-concept screenshots.
* `/docs/` - Deployment parameters and configuration guides.

## About This Project

**Role:**
Cloud Network Engineer

**Skills Demonstrated:**
Infrastructure as Code (Bicep), Azure Networking, Private Link, Load Balancing, Automation.

```

```