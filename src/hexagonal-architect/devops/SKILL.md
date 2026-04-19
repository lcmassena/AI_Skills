---
name: devops
description: Use when setting up CI/CD pipelines for .NET applications using GitHub Actions with Build Once Deploy Many strategy, multi-environment deployments (DEV/QA/PROD), and custom deployment targets. Do not use for Azure DevOps pipelines or non-.NET applications.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# DevOps Skill

## Use this skill when

- Setting up GitHub Actions for .NET applications
- Implementing CI/CD pipelines with Build Once Deploy Many
- Creating multi-environment deployments (DEV → QA → PROD)
- Configuring environment-specific settings
- Setting up deployment to IIS or custom hosting
- Managing secrets and variables across environments
- Implementing release versioning

## Do not use this skill when

- Working with Azure DevOps Pipelines (use azure-pipelines format)
- Deploying non-.NET applications
- Setting up only local development workflows
- Creating test-only pipelines without deployment

---

## Overview

This skill provides comprehensive guidance for setting up DevOps workflows using GitHub Actions with the **Build Once Deploy Many** strategy. This approach compiles the application once and deploys to multiple environments, ensuring consistency and reducing build times.

### Key Principles

1. **Build Once** - Application is compiled and packaged only once
2. **Deploy Many** - Same artifact deployed to DEV, QA, and PROD
3. **Environment Isolation** - Each environment has its own configuration
4. **Progressive Deployment** - DEV → QA → PROD flow

---

## Pipeline Architecture

### Multi-Stage Pipeline Flow

```
┌─────────────┐
│   BUILD    │  ← Compile once, create artifact
└─────────────┘
      │
      ▼
┌─────────────┐
│    DEV     │  ← Auto-deploy on commit to dev branch
└─────────────┘
      │
      ▼
┌─────────────┐
│    QA     │  ← Manual approval required
└─────────────┘
      │
      ▼
┌─────────────┐
│   PROD     │  ← Manual approval required
└─────────────┘
```

### Environment Strategy

| Environment | Branch Trigger | Approval Required | Purpose |
|------------|-------------|--------------|---------|
| DEV | Push to `dev/*` or feature branch | No | Development testing |
| QA | Pull request to `main` | Yes | Quality assurance |
| PROD | Push to `main` | Yes | Production release |

---

## GitHub Actions Structure

### Main Workflow File

**Location:** `.github/workflows/`

```
.github/workflows/
├── build.yml          # Continuous Integration
├── deploy-dev.yml     # DEV deployment
├── deploy-qa.yml    # QA deployment
└── deploy-prod.yml  # PROD deployment
```

### Build Pipeline (Continuous Integration)

```yaml
name: Build

on:
  push:
    branches: [main, dev, '**']
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  PROJECT_NAME: 'YourProject.Api'

jobs:
  build:
    runs-on: windows-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Restore dependencies
      run: dotnet restore ${{ env.PROJECT_NAME }}.csproj
    
    - name: Build
      run: dotnet build ${{ env.PROJECT_NAME }}.csproj --configuration Release
    
    - name: Test
      run: dotnet test ${{ env.PROJECT_NAME }}.csproj --configuration Release --verbosity normal
    
    - name: Publish
      run: |
        dotnet publish ${{ env.PROJECT_NAME }}.csproj \
          --configuration Release \
          --output ./publish \
          /p:Version=${{ github.run_number }}
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
        path: ./publish
        retention-days: 30
```

### DEV Deployment Pipeline

```yaml
name: Deploy DEV

on:
  workflow_run:
    workflows: [Build]
    types: [completed]
    branches: [dev, 'dev/**']
  push:
    branches: [dev, 'dev/**']

env:
  ENVIRONMENT: 'dev'

jobs:
  deploy-dev:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: windows-latest
    environment: dev
    
    steps:
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
    
    - name: Deploy to DEV
      run: |
        # Custom deployment script
        # Replace tokens in configuration
        # Deploy to hosting service
      env:
        DEPLOYMENT_TARGET: ${{ secrets.DEPLOYMENT_TARGET }}
        API_KEY: ${{ secrets.API_KEY }}
```

### QA Deployment Pipeline

```yaml
name: Deploy QA

on:
  workflow_run:
    workflows: [Build]
    types: [completed]
    branches: [main]
  workflow_dispatch:

env:
  ENVIRONMENT: 'qa'

jobs:
  deploy-qa:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: windows-latest
    environment: qa
    permissions:
      deployments: write
    
    steps:
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
    
    - name: Deploy to QA
      run: |
        # Custom deployment script
      env:
        DEPLOYMENT_TARGET: ${{ secrets.DEPLOYMENT_TARGET }}
        API_KEY: ${{ secrets.API_KEY }}
```

### PROD Deployment Pipeline

```yaml
name: Deploy PROD

on:
  workflow_run:
    workflows: [Build]
    types: [completed]
    branches: [main]
  workflow_dispatch:

env:
  ENVIRONMENT: 'prod'

jobs:
  deploy-prod:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: windows-latest
    environment: prod
    permissions:
      deployments: write
    
    steps:
    - name: Download artifact
      uses: actions/download-artifact@v4
      with:
        name: ${{ env.PROJECT_NAME }}
    
    - name: Deploy to PROD
      run: |
        # Custom deployment script
      env:
        DEPLOYMENT_TARGET: ${{ secrets.DEPLOYMENT_TARGET }}
        API_KEY: ${{ secrets.API_KEY }}
```

---

## Environment Configuration

### GitHub Environments

Create environments in GitHub repository settings:

1. Go to **Settings** → **Environments**
2. Create each environment: `dev`, `qa`, `prod`
3. Configure protection rules

### Environment Variables

Configure environment-specific variables:

```yaml
# dev environment
APP_ENV: development
LOG_LEVEL: Debug
ENABLE_SWAGGER: true

# qa environment
APP_ENV: qa
LOG_LEVEL: Information
ENABLE_SWAGGER: true

# prod environment
APP_ENV: production
LOG_LEVEL: Warning
ENABLE_SWAGGER: false
```

### Environment Secrets

Store sensitive information as secrets:

| Secret | Description | Used In |
|--------|------------|---------|
| `DEPLOYMENT_TARGET` | Hosting service URL | All environments |
| `API_KEY` | API authentication | All environments |
| `DATABASE_CONNECTION` | Database connection string | All environments |
| `AZURE_STORAGE_KEY` | Cloud storage credentials | All environments |

### Creating Environment Secrets

```bash
# Using GitHub CLI
gh secret set DATABASE_CONNECTION --env dev --body "Server=dev-db;..."
gh secret set DATABASE_CONNECTION --env qa --body "Server=qa-db;..."
gh secret set DATABASE_CONNECTION --env prod --body "Server=prod-db;..."
```

---

## Configuration Management

### appsettings.json Structure

```
appsettings.json           # Base configuration
appsettings.dev.json      # DEV overrides
appsettings.qa.json      # QA overrides
appsettings.prod.json   # PROD overrides
```

### Token Replacement

Use token replacement in deployment:

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "#{DATABASE_CONNECTION}#"
  },
  "AppSettings": {
    "Environment": "#{APP_ENV}#"
  }
}
```

### Deployment Script Example

```powershell
# deploy.ps1
param(
    [string]$Environment,
    [string]$ArtifactPath
)

# Load environment-specific configuration
$config = Get-Content "./config/appsettings.$Environment.json" | ConvertFrom-Json

# Replace tokens in artifacts
Get-ChildItem -Path $ArtifactPath -Recurse -Filter "*.json" | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $content = $content -replace '#{DATABASE_CONNECTION}#', $config.ConnectionStrings.DefaultConnection
    $content = $content -replace '#{APP_ENV}#', $config.AppSettings.Environment
    Set-Content -Path $_.FullName -Value $content
}

Write-Host "Deployed to $Environment successfully"
```

---

## Versioning Strategy

### Semantic Versioning

```
{MAJOR}.{MINOR}.{PATCH}
  │     │     └── Hotfix
  │     └────── Feature
  └────────── Breaking change
```

### Version Format in Pipeline

```yaml
env:
  VERSION: ${{ github.run_number }}.0.0
  
# or using git tags
VERSION: ${{ github.ref_name }}
```

### Version File Creation

```yaml
- name: Create version file
  run: |
    $version = "${{ github.run_number }}.0.0"
    $version | Out-File ./publish/VERSION -Encoding UTF8
```

---

## Deployment Customization

### Custom Deployment Scripts

Each organization has different hosting needs. The deployment step should be customized:

#### IIS Deployment

```yaml
- name: Deploy to IIS
  run: |
    # Stop application pool
    # Copy files
    # Update configuration
    # Start application pool
```

#### Azure Deployment

```yaml
- name: Deploy to Azure
  run: az webapp up --name $(APP_NAME) --resource-group $(RESOURCE_GROUP)
```

#### Kubernetes Deployment

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl apply -f ./k8s/deployment.yaml
    kubectl set image deployment/$(APP_NAME) app=$(IMAGE):latest
```

#### Container Registry

```yaml
- name: Build and push container
  run: |
    docker build -t ${{ env.REGISTRY }}/app:${{ env.VERSION }} .
    docker push ${{ env.REGISTRY }}/app:${{ env.VERSION }}
```

---

## Best Practices

### 1. Single Artifact Strategy

Build once, deploy to all environments:

```yaml
# Always use the same artifact
- name: Download artifact
  uses: actions/download-artifact@v4
  with:
    name: ${{ env.PROJECT_NAME }}
```

### 2. Environment Protection

Configure protection rules for each environment:

```yaml
environment:
  name: prod
  prevent_self_review: true
  required_reviewers:
    - team-lead
```

### 3. Secrets Management

Never hardcode secrets:

```yaml
# Correct
env:
  API_KEY: ${{ secrets.API_KEY }}

# Never do this
env:
  API_KEY: 'my-secret-key'
```

### 4. Idempotent Deployments

Ensure deployments can be run multiple times safely:

```yaml
- name: Deploy
  run: |
    # Use --force or similar flag
    kubectl apply --force -f ./deployment.yaml
```

### 5. Rollback Strategy

Have a rollback plan:

```yaml
- name: Rollback
  if: ${{ failure() }}
  run: |
    kubectl rollout undo deployment/$(APP_NAME)
```

---

## Monitoring and Notifications

### Deployment Notifications

```yaml
- name: Notify on failure
  if: ${{ failure() }}
  uses: slackapi/slack-github-action@v1.25
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Deployment Status Badge

Add to README.md:

```markdown
[![Deploy DEV](https://github.com/org/repo/actions/workflows/deploy-dev.yml/badge.svg)](https://github.com/org/repo/actions/workflows/deploy-dev.yml)
```

---

## Folder Structure

```
.github/
└── workflows/
    ├── build.yml         # CI: Build and test
    ├── deploy-dev.yml   # CD: DEV environment
    ├── deploy-qa.yml   # CD: QA environment
    └── deploy-prod.yml # CD: PROD environment

scripts/
├── deploy.ps1        # Common deployment script
└── config/         # Environment configs
    ├── appsettings.dev.json
    ├── appsettings.qa.json
    └── appsettings.prod.json
```

---

## Quick Reference

| Stage | Trigger | Approval | Artifact |
|-------|--------|----------|----------|
| Build | Push/PR | No | ✓ |
| DEV | Push to dev | No | Uses Build artifact |
| QA | PR to main | Yes | Uses Build artifact |
| PROD | Push to main | Yes | Uses Build artifact |

---

## Additional Resources

For templates and examples, see:

- `templates/` - Ready-to-use workflow templates
- `scripts/` - Deployment scripts
- Configuration management patterns

---

## Need Help?

If you're stuck:

1. Review GitHub Actions documentation
2. Check your hosting provider's deployment guide
3. Ensure environment secrets are configured
4. Verify GitHub environments are set up correctly