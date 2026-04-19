---
name: devops-iac
description: Infrastructure as Code deployment with GitHub Actions, integrated with Build Once Deploy Many strategy.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

This guide explains how to deploy Azure infrastructure using Bicep templates within GitHub Actions pipelines, integrated with the DevOps Build Once Deploy Many strategy.

---

## Integration with Build Pipeline

The IaC deployment can be part of the main DevOps workflow or a separate workflow. The recommended approach is to trigger infrastructure deployment after the application build.

### Structure

```
.github/
└── workflows/
    ├── build.yml              # Application CI
    ├── deploy-app-dev.yml    # Application deployment
    └── deploy-infra.yml     # Infrastructure deployment
```

---

## Combined Workflow

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  PROJECT_NAME: '<Product>.Api'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Restore
      run: dotnet restore ${{ env.PROJECT_NAME }}.csproj
    
    - name: Build
      run: dotnet build ${{ env.PROJECT_NAME }}.csproj --configuration Release
    
    - name: Publish
      run: dotnet publish ${{ env.PROJECT_NAME }}.csproj --configuration Release --output ./publish
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
        path: ./publish

  deploy-infra:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy Infrastructure
      run: |
        az deployment group create \
          --resource-group rg-prod-<product>-001 \
          --name infra-deployment-${{ github.run_id }} \
          --template-file infra/main.bicep \
          --parameters @infra/parameters/prod.bicepparam
    
    - name: Get Service Bus Endpoint
      run: |
        sb_endpoint=$(az deployment group show \
          --resource-group rg-prod-<product>-001 \
          --name infra-deployment-${{ github.run_id }} \
          --query properties.outputs.serviceBusEndpoint.value -o tsv)
        echo "SB_ENDPOINT=$sb_endpoint" >> $GITHUB_ENV
    
    - name: Create App Settings
      run: |
        az webapp config appset set \
          --resource-group rg-prod-<product>-001 \
          --name func-<product>-prd-001 \
          --settings ServiceBusEndpoint=${{ env.SB_ENDPOINT }}

  deploy-app:
    needs: [build, deploy-infra]
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to Azure Functions
      uses: azure/functions-action@v1
      with:
        app-name: func-<product>-prd-001
        package: ./*.zip
```

---

## Separate Infrastructure Workflow

```yaml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'infra/**'
  workflow_dispatch:

env:
  ENVIRONMENT: prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Validate Bicep
      run: az bicep build --file infra/main.bicep
    
    - name: What-If
      run: |
        az deployment group what-if \
          --resource-group rg-${{ env.ENVIRONMENT }}-<product>-001 \
          --template-file infra/main.bicep \
          --parameters @infra/parameters/${{ env.ENVIRONMENT }}.bicepparam
    
    - name: Deploy
      run: |
        az deployment group create \
          --resource-group rg-${{ env.ENVIRONMENT }}-<product>-001 \
          --name infra-${{ github.run_id }} \
          --template-file infra/main.bicep \
          --parameters @infra/parameters/${{ env.ENVIRONMENT }}.bicepparam
    
    - name: Get Outputs
      run: |
        az deployment group show \
          --resource-group rg-${{ env.ENVIRONMENT }}-<product>-001 \
          --name infra-${{ github.run_id }} \
          --query properties.outputs
```

---

## Environment Matrix

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, hml, prod]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to ${{ matrix.environment }}
      run: |
        az deployment group create \
          --resource-group rg-${{ matrix.environment }}-<product>-001 \
          --name infra-deployment-${{ github.run_id }} \
          --template-file infra/main.bicep \
          --parameters @infra/parameters/${{ matrix.environment }}.bicepparam
```

---

## Secrets Required

| Secret | Description |
|--------|-------------|
| `AZURE_CREDENTIALS` | Service Principal JSON |
| `AZURE_SUBSCRIPTION_ID` | Subscription ID |

---

## Outputs Handling

### Save to Variable

```yaml
- name: Get Service Bus Connection
  run: |
    echo "SB_CONNECTION=$(az deployment group show \
      --resource-group rg-dev-<product>-001 \
      --name infra-deployment \
      --query properties.outputs.serviceBusConnectionString.value -o tsv)" \
      >> $GITHUB_ENV
```

### Save to File

```yaml
- name: Save Outputs
  run: |
    az deployment group show \
      --resource-group rg-dev-<product>-001 \
      --name infra-deployment \
      --query properties.outputs > outputs.json
    echo outputs.json > $GITHUB_ENV

- name: Upload Artifact
  uses: actions/upload-artifact@v4
  with:
    name: infra-outputs
    path: outputs.json
```

---

## Best Practices

1. **Infrastructure First** - Deploy infra before app
2. **Separation of Concerns** - Separate workflows
3. **Use What-If** - Always preview changes
4. **Store Outputs** - Save for reference
5. **Idempotent Deployments** - Use `--mode Incremental`

---

## Full DevOps Flow with IaC

```
┌─────────────┐
│    BUILD   │  ← Compile .NET application
└─────────────┘
      │
      ▼
┌─────────────┐
│   INFRA    │  ← Deploy Azure resources (Bicep)
└─────────────┘
      │
      ▼
┌─────────────┐
│    DEV    │  ← Auto-deploy app
└─────────────┘
      │
      ▼
┌─────────────┐
│    QA     │  ← Manual approval
└─────────────┘
      │
      ▼
┌─────────────┐
│   PROD     │  ← Manual approval
└─────────────┘
```

---

## Quick Reference

| Step | Command | Purpose |
|------|---------|---------|
| Login | `azure/login@v2` | Authenticate to Azure |
| Validate | `az bicep build` | Check syntax |
| What-If | `az deployment group what-if` | Preview changes |
| Deploy | `az deployment group create` | Deploy resources |
| Output | `az deployment group show` | Get outputs |