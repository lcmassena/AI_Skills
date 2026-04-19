---
name: postman
description: Use when creating or managing Postman collections for Azure Functions APIs, including collection structure, environment configuration, and automated API discovery. Do not use for general API testing without Azure Functions context.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

# Postman Skill

## Use this skill when

- Creating Postman collections for Azure Functions
- Setting up environment variables for different environments (Local, Dev, QA, Prod)
- Documenting API endpoints for team collaboration
- Maintaining API collections in source control
- Automating API discovery from Azure Functions

## Do not use this skill when

- Testing non-Azure Functions APIs
- Creating integration tests without documentation focus
- Using Postman for load testing (use separate tooling)
- Manual API exploration only

---

## Overview

This skill provides guidance for creating and maintaining Postman collections for Azure Functions APIs. The structure follows the Postman Local workspace format with YAML-based collections and environments that can be version-controlled alongside the codebase.

---

## Folder Structure

### Project Structure

```
.postman/
├── resources.yaml                           # Workspace configuration
├── collections/
│   └── <ProjectName> APIs/
│       ├── .resources/
│       │   └── definition.yaml          # Collection variables & auth
│       ├── v1/
│       │   ├── v1-endpoint1.request.yaml
│       │   ├── v1-endpoint2.request.yaml
│       │   └── ...
│       └── api-health.request.yaml
└── environments/
    ├── <ProjectName> APIs - Local.environment.yaml
    ├── <ProjectName> APIs - Dev.environment.yaml
    ├── <ProjectName> APIs - QA.environment.yaml
    └── <ProjectName> APIs - Prod.environment.yaml
```

### Example Structure

```
.postman/
├── resources.yaml
├── collections/
│   └── <Product> APIs/
│       ├── .resources/
│       │   └── definition.yaml
│       ├── v1/
│       │   ├── v1-risco-emissao.request.yaml
│       │   ├── v1-parcela-emitida.request.yaml
│       │   └── v1-saldo-contabil.request.yaml
│       ├── Legado/
│       │   └── api-auth.request.yaml
│       └── api-health.request.yaml
└── environments/
    ├── <Product>APIs - Local.environment.yaml
    ├── <Product>APIs - Dev.environment.yaml
    └── <Product>APIs - QA.environment.yaml
```

---

## Collection Definition

### .resources/definition.yaml

The collection definition file contains variables and authentication configuration:

```yaml
$kind: collection
variables:
  <Product>APIsToken: ""
auth:
  - id: auth-id
    type: bearer
    name: bearer auth
    credentials:
      token: "{{<Product>APIsToken}}"
```

### Collection Variables

Common variables used across requests:

| Variable | Description | Example |
|----------|-------------|---------|
| `ServerUrl` | API base URL | `localhost:7071` or `dev-project.azurewebsites.net` |
| `FunctionsKey` | Azure Functions key | Generated in Azure portal |
| `<Product>APIsToken` | Bearer token | Obtained from authentication |

---

## Request File Format

### Basic HTTP Request

```yaml
$kind: http-request
name: v1/<endpoint-path>
url: "{{ServerUrl}}/api/v1/<endpoint-path>"
method: POST
headers:
  x-functions-key: "{{FunctionsKey}}"
settings:
  followOriginalHttpMethod: true
description: >
  Description of what this endpoint does.
body:
  type: json
  content: |
    {
      "property1": "value1",
      "property2": "value2"
    }
responses:
  - statusCode: 200
    content-type: application/json
    description: Success response
  - statusCode: 400
    content-type: application/json
    description: Bad request
  - statusCode: 500
    description: Internal server error
```

### Request Properties

| Property | Description |
|----------|-------------|
| `$kind` | Must be `http-request` |
| `name` | Request name with endpoint path |
| `url` | Full URL with variable interpolation |
| `method` | HTTP method (GET, POST, PUT, DELETE) |
| `headers` | Request headers |
| `settings.followOriginalHttpMethod` | Preserve original method on redirects |
| `body.content` | JSON request body |
| `responses` | List of expected responses |

---

## Environment Files

### Environment Structure

```yaml
name: <ProjectName> - <Environment>
values:
  - key: ServerUrl
    value: '<environment-url>'
    enabled: true
  - key: FunctionsKey
    value: '<functions-key>'
    enabled: true
  - key: <Product>APIsToken
    value: ''
    enabled: true
color: <color-code>
```

### Environment Color Codes

| Color | Code |
|-------|------|
| Red | 0 |
| Green | 50 |
| Blue | 75 |
| Purple | 100 |
| Cyan | 120 |
| Yellow | 150 |
| Pink | 175 |
| Gray | 200 |

### Environment Variables by Stage

**Local Environment:**
```yaml
- ServerUrl: 'localhost:7071'
- FunctionsKey: ''  # Not needed locally
```

**Dev Environment:**
```yaml
- ServerUrl: 'dev-project.azurewebsites.net'
- FunctionsKey: '<generated-key>'
```

**QA Environment:**
```yaml
- ServerUrl: 'qa-project.azurewebsites.net'
- FunctionsKey: '<generated-key>'
```

**Prod Environment:**
```yaml
- ServerUrl: 'project.azurewebsites.net'
- FunctionsKey: '<generated-key>'
```

---

## Resources Configuration

### resources.yaml

This file maps local Postman files to Postman Cloud IDs:

```yaml
workspace:
  id: <workspace-id>

cloudResources:
  collections:
    ..\postman\collections\<CollectionName>: <cloud-collection-id>
  environments:
    ..\postman\environments\<EnvironmentName>.environment.yaml: <cloud-environment-id>
```

---

## Creating Requests from Azure Functions

### HTTP Trigger Pattern

When creating Postman requests from Azure Functions, follow this mapping:

| Azure Functions | Postman Request |
|----------------|---------------|
| `[HttpTrigger(AuthorizationLevel.Anonymous, "get")]` | GET method |
| `[HttpTrigger(AuthorizationLevel.Anonymous, "post")]` | POST method |
| `[HttpTrigger(AuthorizationLevel.Anonymous, "put")]` | PUT method |
| `[HttpTrigger(AuthorizationLevel.Anonymous, "delete")]` | DELETE method |
| `[Route = "v1/entity"]` | URL path: `v1/entity` |

### Example: Function to Request

**Azure Function:**
```csharp
[Function("get")]
[HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "v1/lotes/{codLoteCia?}")]
public async Task<IActionResult> Get([HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequest req)
```

**Postman Request:**
```yaml
$kind: http-request
name: v1/lotes
url: "{{ServerUrl}}/api/v1/lotes"
method: GET
headers:
  x-functions-key: "{{FunctionsKey}}"
```

### Headers for Azure Functions

| Header | Description | Required |
|--------|-------------|----------|
| `x-functions-key` | Azure Functions key | For functions with key authentication |
| `Authorization` | Bearer token | When using JWT |
| `Content-Type` | Request content type | For POST/PUT |

---

## Best Practices

### 1. Organize by Version

```
collections/
└── <ProjectName> APIs/
    ├── v1/
    │   ├── endpoint1.request.yaml
    │   └── endpoint2.request.yaml
    └── v2/
        └── ...
```

### 2. Document Responses

Always document expected responses:

```yaml
responses:
  - statusCode: 200
    content-type: application/json
    description: Success
  - statusCode: 400
    content-type: application/json
    description: Validation error
```

### 3. Use Environment Scoping

Use separate environment files for each stage:

- Local → Development testing
- Dev → Integration testing
- QA → QA validation
- Prod → Production releases

### 4. Authentication

Configure bearer token authentication at collection level:

```yaml
auth:
  - type: bearer
    credentials:
      token: "{{<Product>APIsToken}}"
```

### 5. Include Health Check

Always include a health check endpoint:

```yaml
$kind: http-request
name: health
url: "{{ServerUrl}}/api/health"
method: GET
```

---

## Version Control

### Git Integration

Add to `.gitignore`:

```
# Postman
.postman/local/
.postman/cache/
*.log
```

### Postman Local vs Cloud

- **Local (recommended):** Files stored in `.postman/` folder in project
- **Cloud:** Stored in Postman cloud, harder to version control

---

## Additional Resources

For the Agent that automates discovery, see:

- `agent.md` - Agent behavior for automatic API discovery

---

## Quick Reference

| File | Purpose |
|------|---------|
| `resources.yaml` | Workspace & cloud mapping |
| `.resources/definition.yaml` | Collection auth & variables |
| `*.request.yaml` | API requests |
| `*.environment.yaml` | Environment configuration |

---

## Need Help?

1. Review existing collections in `.postman/collections/`
2. Check Azure Function attributes for endpoint mapping
3. Verify environment variables are configured