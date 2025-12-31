# Azure Private Link Implementation: Secure Cross-VNet Connectivity

![Azure Private Link Banner](/images/privatelink_banner.png)

## Overview

Architected and deployed a production-grade, secure connectivity solution using **Azure Private Link** and **Infrastructure as Code (Bicep)** to expose internal applications across virtual networks without internet exposure. This implementation demonstrates enterprise-grade network security patterns suitable for compliance-heavy environments (PCI-DSS, HIPAA).

## Problem Statement

Organizations often need to share services between isolated networks while maintaining strict security boundaries. Traditional approaches either expose services to the public internet (increasing attack surface) or require complex VPN configurations. This project solves that challenge using Azure Private Link to create secure, private connectivity over the Microsoft backbone network.

## Solution Architecture

### High-Level Design

![Private Link Architecture Diagram](/images/privatelink_architecture.png)

The solution consists of two isolated Virtual Networks communicating exclusively through Azure Private Link:

**Provider Network (`WhizlabsVNet`)**
- Hosts the application (IIS on Windows Server 2019)
- Standard Load Balancer for high availability
- Private Link Service attached to load balancer frontend
- No public IP addresses exposed

**Consumer Network (`whizPEVnet`)**
- Simulates client environment
- Private Endpoint (IP: `10.0.0.5`) for service access
- All traffic routed through Microsoft backbone

### Architecture Diagram
```
┌─────────────────────────────────────┐      ┌──────────────────────────────────┐
│       Provider VNet                 │      │      Consumer VNet               │
│                                     │      │                                  │
│  ┌──────────────────────────────┐  │      │  ┌────────────────────────────┐ │
│  │  Standard Load Balancer      │  │      │  │  Consumer VM               │ │
│  │  Frontend IP: 10.1.0.x       │  │      │  │  Accesses: 10.0.0.5        │ │
│  └──────────┬───────────────────┘  │      │  └────────────┬───────────────┘ │
│             │                       │      │               │                 │
│  ┌──────────▼───────────────────┐  │      │  ┌────────────▼───────────────┐ │
│  │  Private Link Service        │◄─┼──────┼──┤  Private Endpoint          │ │
│  │  (Backend Pool)              │  │      │  │  IP: 10.0.0.5              │ │
│  └──────────┬───────────────────┘  │      │  └────────────────────────────┘ │
│             │                       │      │                                  │
│  ┌──────────▼───────────────────┐  │      └──────────────────────────────────┘
│  │  Backend VMs (IIS)           │  │               Microsoft Backbone
│  │  No Public IPs               │  │              (Private Traffic Only)
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

## Technical Implementation

### Technology Stack

![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat-square&logo=microsoft-azure&logoColor=white)
![Bicep](https://img.shields.io/badge/Bicep-0089D6?style=flat-square&logo=microsoft-azure&logoColor=white)
![IaC](https://img.shields.io/badge/IaC-Infrastructure_as_Code-blue?style=flat-square)

- **Azure Bicep** - Declarative Infrastructure as Code
- **Azure Private Link** - Secure service connectivity
- **Azure Private Endpoint** - Private network interface
- **Standard Load Balancer** - L4 traffic distribution
- **Azure CLI** - Deployment automation

### Infrastructure as Code

![Bicep Code Structure](/images/bicep_code.png)
*Figure 1: Defining the Private Link Service and Load Balancer relationships in Bicep*

Defined all infrastructure in a modular `main.bicep` template, orchestrating 15+ resources including VNets, NICs, VMs, and Private Link configurations.

**Key Bicep Resources:**
```bicep
// Private Link Service definition
resource privateLinkService 'Microsoft.Network/privateLinkServices@2023-04-01' = {
  name: 'pls-backend-service'
  location: location
  properties: {
    loadBalancerFrontendIpConfigurations: [
      {
        id: loadBalancer.properties.frontendIPConfigurations[0].id
      }
    ]
    ipConfigurations: [...]
  }
}

// Private Endpoint in consumer VNet
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: 'pe-backend-consumer'
  location: location
  properties: {
    subnet: {
      id: consumerSubnet.id
    }
    privateLinkServiceConnections: [...]
  }
}
```

### Deployment Process

**Step 1: Resource Group Creation**
```bash
az group create --name MyResourceGroup --location eastus
```

**Step 2: Automated Deployment**
```bash
az deployment group create \
  --resource-group MyResourceGroup \
  --template-file main.bicep \
  --parameters main.parameters.json
```

![Deployment Success](/images/deployment_success.png)
*Figure 2: Successful provisioning of resources via Azure CLI*

**Deployment Time:** ~8 minutes for complete infrastructure

### Validation & Testing

![Resource Group Overview](/images/resource_group.png)
*Figure 3: All resources successfully deployed in the Resource Group*

**Network Verification:**
- ✅ Private Link Service mapped to Load Balancer
- ✅ Private Endpoint status: `Approved`
- ✅ Private Endpoint IP: `10.0.0.5`
- ✅ No public IPs exposed on backend

**Connectivity Testing:**
1. Connected to Consumer VM via RDP
2. Accessed `http://10.0.0.5` via browser
3. **Result:** Successfully loaded IIS default page from Provider VM

![Connectivity Test](/images/connectivity_test.png)
*Figure 4: Successfully accessing the Provider Application via the Private Endpoint IP*

**Traffic Path Confirmation:**
- All traffic routed through Microsoft backbone
- Zero internet traversal
- Latency: <5ms (intra-region)

## Security Analysis

### Private Link vs. Service Endpoints

| Feature | Service Endpoints | Private Link |
|---------|------------------|--------------|
| **Traffic Path** | Microsoft Backbone | Microsoft Backbone |
| **Public IP Required** | No | No |
| **Data Exfiltration Risk** | Higher (entire service accessible) | **Lower (specific instance only)** |
| **Cross-Region Support** | No | **Yes** |
| **Consumer IP Type** | Public VNet IP | **Private (10.x.x.x)** |
| **Instance-Level Control** | No | **Yes** |

### Security Benefits

- **Zero Trust Architecture:** No public endpoints, reducing attack surface by 100%
- **Network Isolation:** Complete separation between provider and consumer networks
- **Compliance Ready:** Meets PCI-DSS, HIPAA, and SOC 2 requirements for private connectivity
- **Data Sovereignty:** Traffic never leaves Microsoft network infrastructure
- **Granular Access Control:** Private Endpoint approval workflow enforces least-privilege access

## Key Results

✅ **100% Infrastructure Automation** - Complete deployment via single Bicep template  
✅ **Zero Internet Exposure** - All traffic on Microsoft backbone network  
✅ **Secure Multi-VNet Connectivity** - Private IP-based service access (`10.0.0.5`)  
✅ **Compliance-Ready Pattern** - Suitable for regulated workloads (PCI-DSS/HIPAA)  
✅ **Repeatable Deployment** - Idempotent IaC enables consistent multi-region rollouts

## Cost Optimization Considerations

- **Standard Load Balancer:** Required for Private Link (higher cost than Basic SKU)
- **Private Link Data Processing:** ~$0.01 per GB processed
- **Recommendation:** Deallocate backend VMs in dev/test when not in use
- **Cost Savings:** Eliminates need for VPN Gateway (~$140/month)


## Skills Demonstrated

- Infrastructure as Code (Bicep/ARM)
- Azure Networking Architecture
- Private Link & Private Endpoints
- Load Balancing & High Availability
- Security & Compliance Design
- Automation & DevOps Practices

## Future Enhancements

- [ ] Implement Azure Monitor for Private Link metrics
- [ ] Add Azure Private DNS Zone integration
- [ ] Multi-region deployment with cross-region Private Link
- [ ] Terraform version for multi-cloud comparison
- [ ] CI/CD pipeline integration (GitHub Actions/Azure DevOps)

## Getting Started

### Prerequisites
- Azure subscription
- Azure CLI installed
- Contributor access to resource group

## Contact

**Praise Effiom** - Cloud Security Engineer  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/praiseeffiom)

---

⭐ Star this repository if you find it useful!