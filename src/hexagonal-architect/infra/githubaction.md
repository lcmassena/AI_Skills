---
name: infra-githubaction
description: GitHub Actions pipeline for deploying Azure infrastructure using Bicep templates.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

This guide explains how to create a GitHub Actions pipeline for deploying Azure infrastructure using Bicep templates.

---

## Workflow Structure

### Location

```
.github/
└── workflows/
    ├── infra-deploy.yml        # Main infra deployment workflow
    └── infra-destroy.yml      # Cleanup/destroy workflow
```

---

## Basic Workflow: infa-deploy.yml

```yaml
name: Deploy Infrastructure

on:
  push:
    branches:
      - main
    paths:
      - 'infra/**'
      - '.github/workflows/infra-deploy.yml'
  workflow_dispatch:

env:
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  RESOURCE_GROUP: rg-${{ vars.ENVIRONMENT }}-<product>-001
  LOCATION: eastus

jobs:
  deploy-infra:
    runs-on: ubuntu-latest
    environment: ${{ vars.ENVIRONMENT || 'dev' }}
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy Bicep
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: ${{ env.RESOURCE_GROUP }}
        template: ${{ github.workspace }}/infra/main.bicep
        parameters: ${{ github.workspace }}/infra/parameters/${{ vars.ENVIRONMENT || 'dev' }}.bicepparam
        location: ${{ env.LOCATION }}
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: infra-deployment-outputs
        path: outputs.json
```

---

## Multi-Environment Workflow

```yaml
name: Deploy Infrastructure

on:
  push:
    branches:
      - main
    paths:
      - 'infra/**'
  pull_request:
    branches:
      - main
    paths:
      - 'infra/**'
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options:
          - dev
          - hml
          - prd
        description: Select environment

env:
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

jobs:
  validate:
    runs-on: ubuntu-latest
    name: Validate Bicep
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Validate Bicep syntax
      run: |
        az bicep build --file infra/main.bicep
    
    - name: What-If deployment
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: rg-validation
        template: infra/main.bicep
        parameters: infra/parameters/dev.bicepparam
        mode: ValidationOnly
        deploymentName: validate-${{ github.run_id }}

  deploy-dev:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: dev
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy DEV
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: rg-dev-<product>-001
        template: infra/main.bicep
        parameters: infra/parameters/dev.bicepparam
        location: eastus

  deploy-hml:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: hml
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy HML
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: rg-hml-<product>-001
        template: infra/main.bicep
        parameters: infra/parameters/qa.bicepparam
        location: eastus

  deploy-prod:
    needs: deploy-hml
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: prod
    permissions:
      deployments: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy PROD
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: rg-prod-<product>-001
        template: infra/main.bicep
        parameters: infra/parameters/prod.bicepparam
        location: eastus
```

---

## GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `AZURE_SUBSCRIPTION_ID` | Azure Subscription ID |
| `AZURE_CREDENTIALS` | Service Principal credentials |
| `TENANT_ID` | Azure AD Tenant ID |

---

## GitHub Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `ENVIRONMENT` | Target environment | `dev`, `hml`, `prd` |
| `LOCATION` | Azure region | `eastus` |

---

## Environment Protection

```yaml
environment:
  name: prod
  prevent_self_review: true
  required_reviewers:
    - team-lead
```

---

## Outputs Handling

```yaml
- name: Get deployment outputs
  run: |
    echo "serviceBusEndpoint=${{ steps.deploy.outputs.serviceBusEndpoint }}" >> $GITHUB_ENV
    
- name: Save outputs
  run: |
    echo '${{ steps.deploy.outputs }}' > outputs.json
```

---

## Best Practices

1. **Validate First** - Always validate Bicep syntax
2. **Use What-If** - Preview deployment effects
3. **Environment Branches** - Match environments to branches
4. **Approvals for PROD** - Require manual approval
5. **Idempotent Deployments** - Use `--mode Incremental`

---

## Quick Reference

| Trigger | Environment | Action |
|---------|-------------|---------|
| Push to main | Dev | Auto-deploy |
| Push to main | HML | Auto-deploy |
| Push to main | PROD | Requires approval |
| Manual | Any | Manual trigger |