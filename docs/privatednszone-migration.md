# Private DNS Zone migration guide

## Background

As part of migrating away from Core subscriptions and into separate Environment subscriptions, the Private DNS Zones and all attached records need to be updated and changed to accommodate this requirement. This means that all product teams that are currently using Private DNS Zones in the Core subscription, will have to migrate to separate Private DNS Zones in separate environment subscriptions (Prod, Dev, Test). These Private DNS Zones have already been created, but need to be updated with correct DNS configuration for resources currently using Private Endpoints. They also need to be unlinked and linked to the correct Virtual Network. By creating new Private Endpoints for the existing resources, one step that would otherwise cause downtime is avoided.

## Overview

This guide provides a high-level overview of how to create a new private endpoint for your Azure resources and configure the required Private DNS Zones.
Each product team is responsible for implementing this based on their preferred infrastructure-as-code practices, but the implementation could also be performed by Team Platform if assistance is wanted.

## Migration strategy (Outline)

This outlines the overall strategy to minimize downtime. The Product Teams will only implement step 1, while Team Platform and Team Orange will perform the unlink and linking of the Private DNS Zones to the virtual networks.

1. Resources with existing Private Endpoints need to create new Private Endpoints with DNS configuration in the new Private DNS Zone.
2. Unlink the core Private DNS Zone to the Virtual Network
3. Link the environment Private DNS Zone to the Virtual Network

**NB: Step 2 will produce downtime!**

## Steps to Implement for each Product Team

### 1. Locate all existing Azure resources that needs new Private Endpoints

To begin the migration, identify all existing Azure resources currently using private endpoints linked to the Private DNS Zone in the Core subscription. These include storage accounts, SQL databases, and other services accessed via private endpoints. Document the resource names, types, and their associated private endpoints. This inventory will help ensure that all necessary resources are accounted for during the migration process.

### 2. Create a Private Endpoint and configure DNS

**NOTE: Do not delete the existing Private Endpoint, these will exist in paralell during the migration**

Repeat these steps for each resource:

- Identify the resource that currently uses a private endpoint with DNS configuration in the Core subscription private endpoint (e.g., Storage Account, SQL Database, etc.).
- Create a new Private Endpoint in the same subnet as the existing Private Endpoint
- Create DNS configuration for the new Private Endpoint to point to the **NEW** Private DNS Zone in the appropriate environment subscription
  - For Bicep, use the resource "_Microsoft.Network/privateEndpoints/privateDnsZoneGroups_"
  - For Terraform, use the resource "_azurerm_private_dns_zone_group_"

## Example

This example will illustrate how to create a new Private Endpoint with correct DNS configuration for a Key Vault in Production for the Product Team "Example"

### Overview of resources

_Current resource that needs a new Private Endpoint:_

| Resource Name | Resource Type | Private Endpoint | Subscription | Resource Group |
|---------------|---------------|----------------------|------------------------|--------------|
| `exampleprodkv` | Key Vault | `example-kv-prod-pe` | `example-prod Team Example - Prod` | `example-prod-nwe-infra-rg` |

The Key Vault has this **CURRENT** network configuration:

| Private Endpoint Name | Resource Group (Private DNS Zone) | Subscription (Private DNS Zone) | Private DNS Zone | VNet (subnet) | IP Address |
|---------------|-------------|---------------|----------------------|------------------------|--------------------------|
| `example-kv-prod-pe` | `example-core-nwe-platform-rg` | `example-core - Team Example - Core` | privatelink.vaultcore.azure.net  | `example-prod-nwe-infra-vnet/example-prod-nwe-infra-pep-snet` | `10.120.1.1` |

This will be the **NEW** configuration for the Key Vault, with a new Private Endpoint added:

| Private Endpoint Name | Resource Group (Private DNS Zone) | Subscription (Private DNS Zone) | Private DNS Zone | VNet (subnet) | IP Address |
|---------------|---------------|---------------|----------------------|------------------------|--------------------------|
| `example-kv-prod-pe` | `example-core-nwe-platform-rg` | `example-core - Team Example - Core` | privatelink.vaultcore.azure.net  | `example-prod-nwe-infra-vnet/example-prod-nwe-infra-pep-snet` | `10.120.1.1` |
| (`example-kv-prod-pe2` | `example-prod-nwe-privatedns-rg` | `example-prod - Team Example - Prod` |  privatelink.vaultcore.azure.net | `example-prod-nwe-infra-vnet/example-prod-nwe-infra-pep-snet` | `10.120.1.2`) |

### Terraform example

```markdown
# Create new Private Endpoint with reference to Prod Private DNS Zone
resource "azurerm_private_endpoint" "new_kv_pe" {
  name                = var.privateEndpointName
  location            = var.location
  resource_group_name = var.resourceGroupName
  subnet_id           = var.subnetId

  private_service_connection {
    name                           = var.privateEndpointName
    private_connection_resource_id = azurerm_key_vault.kv.id
    is_manual_connection           = false
    subresource_names              = ["vault"]
  }

   private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [azurerm_private_dns_zone.prod_dns_zone.id]
  }
}
```

### Bicep example

```bicep
# Create new Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2021-05-01' = {
  name: privateEndpointName
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: privateEndpointName
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
}

# Create new Private DNS Zone Group with reference to the new Private Endpoint
resource privateDnsZoneGroup  'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2024-05-01' = {
  parent: privateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'string'
        properties: {
          privateDnsZoneId: privateDnsZoneInProd.id
        }
      }
    ]
  }
}
```

For any questions about the migration, feel free to contact Team Platform.

---
