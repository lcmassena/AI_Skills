---
name: infra-deploy
description: Manual Bicep deployment guide using Azure CLI, Azure PowerShell, or Azure Portal.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

This guide explains how to manually deploy Azure infrastructure using Bicep templates via Azure CLI, Azure PowerShell, or Azure Portal.

---

## Prerequisites

### Azure CLI Requirements

```bash
# Install Azure CLI
az --version

# Sign in to Azure
az login

# Set subscription
az account set --subscription "<subscription-id>"
```

### Azure PowerShell Requirements

```powershell
# Install Az module
Install-Module -Name Az -AllowClobber -Scope CurrentUser

# Connect to Azure
Connect-AzAccount
```

---

## Deployment Methods

### Method 1: Azure CLI (Recommended)

#### Deploy with Inline Parameters

```bash
az deployment group create \
  --resource-group rg-<product>-dev-001 \
  --template-file ./infra/main.bicep \
  --parameters \
    environment=dev \
    location=eastus
```

#### Deploy with Parameter File

```bash
az deployment group create \
  --resource-group rg-<product>-dev-001 \
  --template-file ./infra/main.bicep \
  --parameters @./infra/parameters/dev.bicepparam
```

#### Deploy to Subscription Level

```bash
az deployment sub create \
  --location eastus \
  --template-file ./infra/main.bicep \
  --parameters environment=dev
```

### Method 2: Azure PowerShell

#### Deploy Resource Group

```powershell
New-AzResourceGroup -Name "rg-<product>-dev-001" -Location "eastus"

New-AzResourceGroupDeployment `
  -ResourceGroupName "rg-<product>-dev-001" `
  -TemplateFile "./infra/main.bicep" `
  -Environment "dev" `
  -Location "eastus"
```

#### Deploy with Parameter File

```powershell
$parameters = @{
  environment = 'dev'
  location = 'eastus'
}

New-AzResourceGroupDeployment `
  -ResourceGroupName "rg-<product>-dev-001" `
  -TemplateFile "./infra/main.bicep" `
  -TemplateParameterFile "./infra/parameters/dev.bicepparam"
```

### Method 3: Azure Portal

#### Steps

1. Go to [Azure Portal](https://portal.azure.com)
2. Navigate to **Deploy a custom template**
3. Select **Build your own template in the editor**
4. Paste or upload your Bicep file
5. Add parameters
6. Review and deploy

---

## What-If Validation

Always run what-if before deploying:

```bash
az deployment group what-if \
  --resource-group rg-<product>-dev-001 \
  --template-file ./infra/main.bicep \
  --parameters environment=dev
```

Expected output:
```
Resource changes:
ID: /subscriptions/.../Microsoft.ServiceBus/namespaces/...
Type: Microsoft.ServiceBus/namespaces
Action: Create
```

---

## Deployment by Environment

### DEV Environment

```bash
az deployment group create \
  --resource-group rg-<product>-dev-001 \
  --name dev-deployment \
  --template-file ./infra/servicebus/main.bicep \
  --parameters @./infra/parameters/dev.bicepparam
```

### QA Environment

```bash
az deployment group create \
  --resource-group rg-<product>-hml-001 \
  --name hml-deployment \
  --template-file ./infra/servicebus/main.bicep \
  --parameters @./infra/parameters/qa.bicepparam
```

### PROD Environment

```bash
az deployment group create \
  --resource-group rg-<product>-prd-001 \
  --name prd-deployment \
  --template-file ./infra/servicebus/main.bicep \
  --parameters @./infra/parameters/prod.bicepparam
```

---

## Deployment Outputs

After deployment, you can get outputs:

```bash
# Get outputs from deployment
az deployment group show \
  --resource-group rg-<product>-dev-001 \
  --name dev-deployment \
  --query properties.outputs
```

Returns:
```json
{
  "serviceBusConnectionString": {
    "type": "String",
    "value": "Endpoint=sb://..."
  },
  "serviceBusEndpoint": {
    "type": "String",
    "value": "sb://..."
  }
}
```

---

## Troubleshooting

### Check Deployment Status

```bash
az deployment group list \
  --resource-group rg-<product>-dev-001
```

### Get Deployment Errors

```bash
az deployment group show \
  --resource-group rg-<product>-dev-001 \
  --name dev-deployment \
  --query properties.error
```

### Redeploy with Fixes

```bash
az deployment group create \
  --resource-group rg-<product>-dev-001 \
  --template-file ./infra/main.bicep \
  --parameters @./infra/parameters/dev.bicepparam \
  --mode Incremental
```

---

## Best Practices

1. **Always use What-If** - Preview changes before deployment
2. **Use Parameter Files** - Keep configurations in files
3. **Test in DEV First** - Validate before production
4. **Track Deployments** - Use naming conventions
5. **Review Outputs** - Save connection strings securely

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `az deployment group create` | Deploy to resource group |
| `az deployment sub create` | Deploy to subscription |
| `az deployment group what-if` | Preview changes |
| `az deployment group show` | Get deployment details |