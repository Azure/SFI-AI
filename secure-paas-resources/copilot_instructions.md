# Secure PaaS Resources SFI Compliance Instructions

## Objective

Implement SFI compliance for securing Platform-as-a-Service (PaaS) resources by implementing either private endpoints/private links OR Network Security Perimeters (NSP) based on access requirements.

## Reference Implementation

- **Primary Reference**: `toma3233/servicehub-validation` GitHub repository
- Focus on private endpoint configurations for VNet-contained traffic
- Study Network Security Perimeter implementations for external access scenarios
- Reference deployment script configurations and DNS zone setup patterns

## Implementation Steps

### 1. Scope Identification

The following PaaS resources are in scope for securing:

- **Storage Accounts**
- **Key Vaults**
- **SQL Database Servers**
- **Cosmos DB Accounts**

### 2. Approach Selection: Private Endpoints vs Network Security Perimeter

**Choose your implementation approach based on access requirements:**

#### Use Private Endpoints/Private Links when:
- All traffic to the PaaS resource originates from within your VNet
- Resources like AKS pods, deployment scripts can access via private connectivity
- Example: SQL Database accessed only by AKS cluster workloads
- **Result**: Complete network isolation with `publicNetworkAccess: 'Disabled'`

#### Use Network Security Perimeter (NSP) when:
- External services like EV2 extensions need access to the resource
- Traffic cannot be completely secured within a single VNet
- Multiple different sources need controlled access
- Example: Global Key Vault that EV2 must access for certificate operations
- **Result**: Controlled access through NSP rules while maintaining security

### 3. Implementation Requirements by Approach

#### Private Endpoint Approach:
- **Disable Public Access**: Set `publicNetworkAccess: 'Disabled'`
- **Private Endpoint**: Create in dedicated `pe-subnet`
- **Private DNS Zone**: Configure for proper name resolution within VNet
- **VNet Integration**: Ensure all consumers are VNet-connected

#### Network Security Perimeter Approach:
- **NSP Creation**: Deploy Network Security Perimeter resource
- **Access Rules**: Define specific service tags and IP ranges
- **Resource Association**: Choose access mode (Learning vs Enforced)
- **Public Access**: May remain enabled but controlled by NSP rules

### 3. Virtual Network Prerequisites

Ensure your shared resources include:

- **VNet Configuration**: Properly configured virtual network with multiple subnets
- **PE Subnet**: Dedicated subnet for private endpoints (`pe-subnet`)
- **AKS Subnet**: Separate subnet for AKS node pools (`aks-subnet`) 
- **Script Subnet**: Subnet for deployment scripts with container instance delegation
- **Subnet Configuration**: Set `defaultOutboundAccess: false` on all subnets

### 4. Bicep Template Implementation

#### Private Endpoint Implementation Examples:

**SQL Database Servers (Private Endpoint)**:
- Set `publicNetworkAccess: 'Disabled'` on the SQL server resource
- Create private endpoint with `service: 'sqlServer'`
- Reference existing private DNS zone: `privatelink${environment().suffixes.sqlServerHostname}`
- Configure private DNS zone group for proper name resolution

**Key Vaults (Private Endpoint)**:
- Set `publicNetworkAccess: 'Disabled'` for local/regional Key Vaults
- Create private endpoint with `service: 'vault'`
- Reference private DNS zone: `privatelink.vaultcore.azure.net`
- Ensure AKS Secret Store CSI driver can access via private connectivity

**Storage Accounts (Private Endpoint)**:
- Disable public network access where possible
- Create private endpoints for required services (blob, file, etc.)
- Set `AllowSharedKeyAccess: true` if used by deployment scripts
- Implement cleanup scripts to delete temporary storage accounts

#### Network Security Perimeter Implementation Examples:

**Global Key Vaults (NSP)**:
- Create Network Security Perimeter resource with appropriate access rules
- Configure inbound rules for specific service tags (e.g., EV2 IP ranges)
- Associate Key Vault with NSP using appropriate access mode
- Keep public network access enabled but controlled by NSP rules

**NSP Access Rules Configuration**:
- **Inbound Rules**: Allow specific service tags (e.g., `AzureCloud.${location}` for deployment scripts)
- **EV2 Access**: Include EV2 IP address ranges from central configuration
- **Profile Management**: Create profiles with appropriate access rules for different scenarios

#### Deployment Scripts (Private Network):
- Create script subnet with `Microsoft.ContainerInstance/containerGroups` delegation
- Provision storage account with shared key access enabled
- Set up private endpoints for blob and file services
- Configure proper role assignments for script identity

### 5. Network Security Perimeter (NSP) Detailed Implementation

**When to Use NSP vs Private Endpoints:**
- **NSP**: External services (EV2, Geneva extensions) need access alongside internal VNet traffic
- **Private Endpoints**: All traffic can be contained within VNet boundaries

**NSP Implementation Steps:**

- **NSP Creation**: Deploy Network Security Perimeter resource
- **Access Rules**: Configure inbound/outbound rules for specific service tags and IP ranges
- **Resource Association**: Associate PaaS resources with appropriate access mode
- **Access Modes**: 
  - `Learning`: Considers both NSP rules AND resource-level publicNetworkAccess settings
  - `Enforced`: ONLY considers NSP rules (required for HVT services)
- **Traffic Analysis**: Enable NSP logging for rule tuning and access pattern analysis

**Example Use Cases:**
- **Global Key Vault**: EV2 extensions need certificate access for Geneva account creation
- **Storage Account**: Multiple external services require controlled access
- **Cosmos DB**: External APIs need read access while maintaining security

### 6. Key Components by Approach

#### Private Endpoint Approach Components:
- **VNet with PE Subnet**: Dedicated subnet for private endpoints
- **Private DNS Zones**: Proper name resolution within VNet
- **Private Endpoints**: Per-service connectivity (vault, sqlServer, blob, file)
- **Role Assignments**: Identity permissions for private resource access

#### NSP Approach Components:
- **Network Security Perimeter**: Central security control resource
- **Access Rules**: Service tag and IP-based access controls
- **Profile Configuration**: Different access patterns for different scenarios
- **Resource Associations**: Link PaaS resources to security perimeter

## Best Practices

- Always reference the `toma3233/servicehub-validation` repository for current implementation patterns
- Pay attentation to how ev2.ipaddresses is referenced from the EV2 central configuration, and replicate it exactly if needed
- **Decision Matrix**: Use private endpoints when all access is VNet-contained; use NSP when external services need access  
- **DNS Configuration**: Ensure proper private DNS zone configuration for private endpoint resolution
- **Storage Account Cleanup**: Implement cleanup scripts for temporary storage accounts used by deployment scripts
- **NSP Logging**: Enable Network Security Perimeter logging to analyze traffic patterns and optimize rules
- **Access Mode Selection**: Use `Learning` mode initially, then move to `Enforced` for production (required for HVT services)
- **Role Assignments**: Configure minimal required permissions for service identities accessing private resources
- **Subnet Planning**: Properly design VNet subnets with `defaultOutboundAccess: false` and appropriate delegations