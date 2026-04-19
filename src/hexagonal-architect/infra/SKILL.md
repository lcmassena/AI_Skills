---
name: infra
description: Use when deploying Azure infrastructure using Bicep IaC templates, following Cloud Adoption Framework naming conventions, resource organization, and environment-specific configurations. Do not use for Terraform, ARM JSON, or non-Azure cloud providers.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Infrastructure as Code (IaC) Skill

## Use this skill when

- Creating Azure infrastructure with Bicep
- Following Cloud Adoption Framework (CAF) naming conventions
- Organizing infrastructure by resource type
- Setting up multi-environment deployments (DEV, QA, PROD)
- Configuring Azure resources with parameters
- Using modular Bicep templates

## Do not use this skill when

- Using Terraform for infrastructure
- Using ARM JSON templates (deprecated)
- Deploying to non-Azure clouds (AWS, GCP)
- Single resource without modular structure

---

## Overview

This skill provides guidance for creating Azure infrastructure using Bicep, following the Microsoft Cloud Adoption Framework (CAF) naming conventions and best practices.

### Key Principles

1. **Modular Design** - Separate files per resource type
2. **Naming Conventions** - Follow CAF resource naming
3. **Parameterization** - Environment-specific values
4. **Outputs** - Export connection strings and keys
5. **Composition** - Main file composes modules

---

## Folder Structure

### Recommended Structure

```
infra/
├── main.bicep                      # Main deployment file
├── parameters/
│   ├── dev.bicepparam              # DEV parameters
│   ├── qa.bicepparam              # QA parameters
│   └── prod.bicepparam             # PROD parameters
├── modules/
│   ├── servicebus/
│   │   ├── main.bicep              # Service Bus module
│   │   └── queues.bicep            # Queue definitions
│   ├── storage/
│   │   └── main.bicep              # Storage account module
│   ├── functions/
│   │   └── main.bicep              # Functions app module
│   ├── keyvault/
│   │   └── main.bicep              # Key Vault module
│   └── appinsights/
│       └── main.bicep              # Application Insights module
└── README.md
```

### Example Structure (Project)

```
infra/
├── servicebus/
│   ├── main.bicep                  # Service Bus namespace + queues
│   └── queues.bicep                # Queue definitions
├── storage/
│   └── main.bicep                  # Blob storage
└── ...
```

---

## Cloud Adoption Framework Naming

### Resource Naming Convention

```
<Prefix>-<Environment>-<Service>-<Region>-<Number>
```

| Component | Description | Example |
|-----------|-------------|---------|
| Prefix | Company/Project identifier | `<Company>`, `<Product>` |
| Environment | Environment code | `dev`, `hml`, `prd` |
| Service | Service type | `sb`, `func`, `st` |
| Region | Primary region | `eus`, `cus` |
| Number | Unique identifier | `001`, `002` |

### Environment Codes

| Environment | Code | Description |
|-------------|------|-------------|
| Development | `dev` | Development |
| homologation | `hml` | Testing/QA |
| Production | `prd` | Production |

### Service Prefixes

| Service | Prefix | Example |
|---------|--------|---------|
| Azure Functions | `func` | `func-<product>-eus-001` |
| Service Bus | `sb` | `sb-<product>-eus-001` |
| Storage Account | `st` | `st<product><product>001` |
| Key Vault | `kv` | `kv-<product>-eus-001` |
| App Insights | `ai` | `ai-<product>-eus-001` |
| API Management | `apim` | `apim-<product>-eus-001` |

### Resource ID Format

```
/subscriptions/<subscriptionId>/resourceGroups/<rgName>/providers/<resourceProvider>/<resourceType>/<resourceName>
```

Example:
```
/subscriptions/12345678-1234-1234-1234-123456789012/resourceGroups/rg-<product>-eus-001/providers/Microsoft.Web/sites/func-<product>-eus-001
```

---

## Main Bicep File

### Basic Structure

```bicep
// Main Bicep deployment script for <ProjectName>
// Deploys Azure infrastructure for <Environment>

// Parameters
@description('Environment name')
param environment string = 'dev'

@description('Azure region')
param location string = 'eastus'

@description('Environment tag')
param envTag object = {
  environment: environment
}

// Environment-specific naming
var resourceGroupName = 'rg-${environment}-<service>-001'
var environmentCode = environment == 'dev' ? 'dev' : (environment == 'hml' ? 'hml' : 'prd')

// Main module deployment
module serviceBus './modules/servicebus/main.bicep' = {
  name: 'serviceBusDeployment'
  params: {
    environment: environmentCode
    location: location
    envTag: envTag
  }
}

// Outputs
output serviceBusConnectionString string = serviceBus.outputs.serviceBusConnectionString
output serviceBusEndpoint string = serviceBus.outputs.serviceBusEndpoint
```

---

## Service Bus Module Example

### main.bicep

```bicep
// Service Bus Namespace Module
// Deploys Azure Service Bus with queues

@description('Environment (dev/hml/prd)')
param environment string

@description('Azure region')
param location string

@description('Environment tags')
param envTag object

// Naming
var serviceBusNamespaceName = 'sbns-${environment}-<project>-001'

// Service Bus SKU
@allowed(['Basic', 'Standard', 'Premium'])
param serviceBusSkuName string = 'Standard'

// Service Bus Namespace Resource
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2021-06-01-preview' = {
  name: serviceBusNamespaceName
  location: location
  tags: envTag
  sku: {
    name: serviceBusSkuName
    tier: serviceBusSkuName
  }
  properties: {
    minimumTlsVersion: 'TLS1_2'
    publicNetworkAccess: 'Enabled'
  }
}

// Queue parameters
param maxQueueSizeInMegabytes int = 5120
param maxDeliveryCount int = 10
param messageTimeToLiveInDays int = 2
param lockDurationInSeconds int = 300

// Default queues
var defaultQueues = [
  'queue1'
  'queue2'
  'queue3'
]

// Deploy queues
resource queues 'Microsoft.ServiceBus/namespaces/queues@2021-06-01-preview' = [for queueName in defaultQueues: {
  parent: serviceBusNamespace
  name: queueName
  location: location
  properties: {
    maxQueueSizeInMegabytes: maxQueueSizeInMegabytes
    maxDeliveryCount: maxDeliveryCount
    defaultMessageTimeToLive: 'P${messageTimeToLiveInDays}D'
    lockDuration: 'PT${lockDurationInSeconds}S'
    enablePartitioning: true
  }
}]

// Outputs
output serviceBusConnectionString string = listKeys(serviceBusNamespace.id, serviceBusNamespace.apiVersion).primaryConnectionString
output serviceBusEndpoint string = serviceBusNamespace.properties.serviceBusEndpoint
output queueCount int = length(defaultQueues)
```

### queues.bicep (Separated)

```bicep
// Queue definitions module
// Can be imported by main or deployed separately

param serviceBusNamespaceName string
param location string = resourceGroup().location
param queueNames array = [
  'queue1'
  'queue2'
]

// Queue configuration
param maxQueueSizeInMegabytes int = 5120
param maxDeliveryCount int = 10
param messageTimeToLiveInDays int = 2
param lockDurationInSeconds int = 300
param enablePartitioning bool = false

// Reference existing namespace
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2021-06-01-preview' existing = {
  name: serviceBusNamespaceName
}

// Deploy queues
resource queue 'Microsoft.ServiceBus/namespaces/queues@2021-06-01-preview' = [for queueName in queueNames: {
  parent: serviceBusNamespace
  name: queueName
  location: location
  properties: {
    maxQueueSizeInMegabytes: maxQueueSizeInMegabytes
    maxDeliveryCount: maxDeliveryCount
    defaultMessageTimeToLive: 'P${messageTimeToLiveInDays}D'
    lockDuration: 'PT${lockDurationInSeconds}S'
    enablePartitioning: enablePartitioning
  }
}]

output queueNames array = queueNames
output queueCount int = length(queueNames)
```

---

## Parameter Files

### dev.bicepparam

```bicep
using './main.bicep'

param environment = 'dev'
param location = 'eastus'
```

### qa.bicepparam

```bicep
using './main.bicep'

param environment = 'hml'
param location = 'eastus'
```

### prod.bicepparam

```bicep
using './main.bicep'

param environment = 'prd'
param location = 'eastus'
```

---

## Common Azure Resources

### Resource Provider Namespaces

| Service | Namespace |
|--------|-----------|
| Azure Functions | `Microsoft.Web/sites` |
| Service Bus | `Microsoft.ServiceBus/namespaces` |
| Storage Account | `Microsoft.Storage/storageAccounts` |
| Key Vault | `Microsoft.KeyVault/vaults` |
| App Insights | `Microsoft.Insights/components` |
| API Management | `Microsoft.ApiManagement/service` |
| SQL Database | `Microsoft.Sql/servers/databases` |
| Cosmos DB | `Microsoft.DocumentDB/databaseAccounts` |

### Common Resource Configuration

| Resource | Required Properties |
|----------|-------------------|
| Service Bus | `sku.name`, `sku.tier` |
| Storage | `sku.name`, `kind` |
| Functions | `serverFarmId`, `httpsAlwaysOn` |
| Key Vault | `sku.name`, `tenantId` |
| App Insights | `Application_Type` |

---

## Best Practices

### 1. Use Modules

```bicep
// Instead of single file
module serviceBus './servicebus/main.bicep' = {
  name: 'serviceBusDeployment'
  params: {
    environment: environment
  }
}
```

### 2. Use Parameters for Environment

```bicep
// Environment as parameter
param environment string = 'dev'

var resourceName = 'sbns-${environment}-project-001'
```

### 3. Use Outputs for Connection

```bicep
// Export connection strings
output serviceBusConnectionString string = listKeys(ns.id, ns.apiVersion).primaryConnectionString
```

### 4. Tag Resources

```bicep
// Apply tags
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2021-06-01-preview' = {
  name: serviceBusNamespaceName
  location: location
  tags: {
    environment: environment
    project: '<ProjectName>'
    managedBy: 'Bicep'
  }
}
```

### 5. Use What-If Before Deploy

```bash
az deployment group what-if \
  --resource-group rg-<product>-dev-001 \
  --template-file main.bicep \
  --parameters environment=dev
```

### 6. Use Semantic Versioning for Outputs

```bicep
output resourceId string = resource.id
output resourceName string = resource.name
output resourceType string = resource.type
```

---

## Deployment Commands

### Deploy to Resource Group

```bash
az deployment group create \
  --resource-group rg-<product>-dev-001 \
  --template-file main.bicep \
  --parameters @dev.bicepparam
```

### Deploy with What-If

```bash
az deployment group what-if \
  --resource-group rg-<product>-dev-001 \
  --template-file main.bicep \
  --parameters environment=dev
```

### Deploy to Subscription

```bash
az deployment sub create \
  --location eastus \
  --template-file main.bicep \
  --parameters environment=dev
```

---

## Security Considerations

### 1. Use Managed Identity

```bicep
resource functionApp 'Microsoft.Web/sites' = {
  identity: {
    type: 'SystemAssigned'
  }
}
```

### 2. Enable TLS

```bicep
properties: {
  minimumTlsVersion: 'TLS1_2'
}
```

### 3. Use Private Endpoints

```bicep
resource privateEndpoint 'Microsoft.Network/privateEndpoints' = {
  properties: {
    privateLinkServiceConnections: [
      {
        privateLinkServiceId: serviceBus.id
      }
    ]
  }
}
```

---

## Quick Reference

| Concept | Syntax |
|---------|-------|
| Parameter | `param name type` |
| Variable | `var name = value` |
| Module | `module name './module.bicep'` |
| Resource | `resource name 'ns@version'` |
| Output | `output name type = value` |
| Loop | `[for item in items: {...}]` |
| Condition | `[if condition: {...}]` |

---

## Azure Resource ID Reference

| Service | Resource ID Format |
|----------|-----------------|
| Service Bus | `/namespaces/{name}` |
| Queue | `/namespaces/{ns}/queues/{name}` |
| Storage | `/storageAccounts/{name}` |
| Blob | `/storageAccounts/{name}/blobServices/default/containers/{name}` |
| Functions | `/sites/{name}` |
| Key Vault | `/vaults/{name}` |

---

## Need Help?

1. Review Microsoft Bicep documentation
2. Check CAF naming conventions
3. Verify resource provider API versions