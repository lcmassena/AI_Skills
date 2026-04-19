---
name: infra-azuredevops
description: Azure DevOps pipeline for deploying Bicep infrastructure templates.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

This guide explains how to create Azure DevOps pipelines for deploying Bicep infrastructure templates.

---

## Pipeline Structure

### YAML Pipeline Files

```
pipelines/
├── infra-deploy.yml      # Main infrastructure pipeline
├── infra-validate.yml  # Validation pipeline
└── infra-destroy.yml  # Cleanup pipeline
```

---

## Basic Pipeline: infra-deploy.yml

```yaml
trigger:
  paths:
    include:
      - infra/**

pr:
  paths:
    include:
      - infra/**

variables:
  azureServiceConnection: 'Azure-Connection'
  location: 'eastus'

stages:
  - stage: Build
    displayName: 'Validate Bicep'
    jobs:
      - job: Validate
        displayName: 'Validate Bicep Syntax'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'Validate Bicep'
            inputs:
              azureSubscription: $(azureServiceConnection)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infra/main.bicep

  - stage: Deploy_DEV
    displayName: 'Deploy to DEV'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - group: '<Product>-DEV'
    jobs:
      - deployment: DeployDEV
        displayName: 'Deploy DEV'
        pool:
          vmImage: 'ubuntu-latest'
        environment: '<Product>-DEV'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Deploy Bicep to DEV'
                  inputs:
                    azureSubscription: $(azureServiceConnection)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group rg-dev-<product>-001 \
                        --name dev-deployment-$(Build.BuildId) \
                        --template-file infra/main.bicep \
                        --parameters @infra/parameters/dev.bicepparam

  - stage: Deploy_HML
    displayName: 'Deploy to HML'
    dependsOn: Deploy_DEV
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - group: '<Product>-HML'
    jobs:
      - deployment: DeployHML
        displayName: 'Deploy HML'
        pool:
          vmImage: 'ubuntu-latest'
        environment: '<Product>-HML'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Deploy Bicep to HML'
                  inputs:
                    azureSubscription: $(azureServiceConnection)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group rg-hml-<product>-001 \
                        --name hml-deployment-$(Build.BuildId) \
                        --template-file infra/main.bicep \
                        --parameters @infra/parameters/qa.bicepparam

  - stage: Deploy_PROD
    displayName: 'Deploy to PROD'
    dependsOn: Deploy_HML
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - group: '<Product>-PROD'
    jobs:
      - deployment: DeployPROD
        displayName: 'Deploy PROD'
        pool:
          vmImage: 'ubuntu-latest'
        environment: '<Product>-PROD'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Deploy Bicep to PROD'
                  inputs:
                    azureSubscription: $(azureServiceConnection)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group rg-prod-<product>-001 \
                        --name prd-deployment-$(Build.BuildId) \
                        --template-file infra/main.bicep \
                        --parameters @infra/parameters/prod.bicepparam
```

---

## Validation Pipeline: infra-validate.yml

```yaml
trigger: none

pr:
  paths:
    include:
      - infra/**

variables:
  azureServiceConnection: 'Azure-Connection'

jobs:
  - job: Validate
    displayName: 'Validate Bicep'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - task: AzureCLI@2
        displayName: 'Build Bicep'
        inputs:
          azureSubscription: $(azureServiceConnection)
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            az bicep build --file infra/main.bicep --outfile infra/main.json

      - task: AzureCLI@2
        displayName: 'What-If Deployment'
        inputs:
          azureSubscription: $(azureServiceConnection)
          scriptType: 'bash'
          scriptLocation: 'inlineScript'
          inlineScript: |
            az deployment group what-if \
              --resource-group rg-validation-<product>-001 \
              --template-file infra/main.json \
              --parameters @infra/parameters/dev.bicepparam
```

---

## Variable Groups

### Azure DevOps Library

Create variable groups in **Pipelines** → **Library**:

| Variable Group | Variables |
|---------------|-----------|
| `<Product>-DEV` | `environment=dev`, `location=eastus` |
| `<Product>-HML` | `environment=hml`, `location=eastus` |
| `<Product>-PROD` | `environment=prd`, `location=eastus` |

---

## Service Connection

### Create Azure Resource Manager Service Connection

1. Go to **Project Settings** → **Service Connections**
2. Click **New service connection**
3. Select **Azure Resource Manager**
4. Choose **Service principal (automatic)**
5. Select subscription and resource group
6. Name: `Azure-Connection`

---

## Environment Approvals

### DEV Environment (Auto-deploy)

```yaml
environment:
  name: DEV
  # No approval required
```

### HML Environment (Manual Approval)

```yaml
environment:
  name: HML
  deployments:
    - queueForDeployerApproval: true
```

### PROD Environment (Manual Approval)

```yaml
environment:
  name: PROD
  deployments:
    - queueForDeployerApproval: true
  requiredApprover:
    - team-lead
```

---

## Outputs Pipeline

```yaml
- task: AzureCLI@2
  displayName: 'Get Outputs'
  inputs:
    azureSubscription: $(azureServiceConnection)
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      echo "##vso[task.setvariable variable=sbEndpoint;]$(az deployment group show --resource-group rg-dev-<product>-001 --name dev-deployment --query properties.outputs.serviceBusEndpoint.value -o tsv)"

- task: AzureCLI@2
  displayName: 'Save Outputs'
  inputs:
    azureSubscription: $(azureServiceConnection)
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      echo '$(sbEndpoint)' > $(Build.ArtifactStagingDirectory)/outputs.json

- task: PublishPipelineArtifact@1
  displayName: 'Publish Outputs'
  inputs:
    targetPath: $(Build.ArtifactStagingDirectory)/outputs.json
    artifact: infra-outputs
```

---

## Best Practices

1. **Separate Stages** - Build, DEV, HML, PROD
2. **Use Variable Groups** - Environment-specific settings
3. **Gate Approvals** - Manual for HML and PROD
4. **Publish Outputs** - Save for reference
5. **Use Naming Conventions** - Consistent deployment names

---

## Quick Reference

| Stage | Trigger | Approval | Environment |
|-------|---------|----------|-------------|
| Validate | PR | No | N/A |
| DEV | Push to main | No | Auto |
| HML | After DEV | Yes | Manual |
| PROD | After HML | Yes | Manual |