---
name: postman-agent
description: Agent behavior for automatic Postman collection generation from Azure Functions HTTP triggers.
license: MIT License. See LICENSE file for details.
metadata:
  author: Lucas Massena
  version: "1.0"
  contact: lucas@massena.com.br
  github: https://github.com/lcmassena
---

## Overview

This document describes the behavior of an Agent that automatically discovers Azure Functions and generates Postman collection files.

## Agent Responsibilities

### 1. Discover HTTP Trigger Functions

The Agent must scan the codebase for Azure Functions with HTTP triggers:

**Search Locations:**
- `Functions/API/` - API project
- Any project containing `[Function(...)]` attribute with HttpTrigger

**Search Pattern:**
```csharp
[HttpTrigger(AuthorizationLevel.Anonymous, "<method>", Route = "<route>")]
```

### 2. Extract Endpoint Information

For each HTTP trigger function, extract:

| Information | Source |
|-------------|--------|
| Endpoint name | `Function` attribute or class name |
| HTTP method | HttpTrigger first parameter |
| Route path | `Route` parameter |
| Headers | Function parameters or class |
| Request body | Command/Request model |
| Response | IActionResult or result type |

### 3. Generate Request Files

Create YAML files in the format:

```yaml
$kind: http-request
name: <route>
url: "{{ServerUrl}}/api/<route>"
method: <METHOD>
headers:
  x-functions-key: "{{FunctionsKey}}"
settings:
  followOriginalHttpMethod: true
description: >
  <description>
body:
  type: json
  content: |
    <JSON body template>
responses:
  - statusCode: 200
    content-type: application/json
    description: Success
```

## Agent Workflow

### Step 1: Scan Functions Projects

```
For each .cs file in Functions/API/:
  Find classes with [Function] attribute
    Find methods with [HttpTrigger] attribute
      Extract route, method, parameters
```

### Step 2: Map to Postman Format

```
Http method → POST
Route "v1/entity" → URL "/api/v1/entity"
Request model → JSON template
```

### Step 3: Create Collection Structure

```
.postman/collections/<ProjectName> APIs/
├── .resources/
│   └── definition.yaml
├── v1/
│   ├── v1-entity1.request.yaml
│   └── v1-entity2.request.yaml
└── api-health.request.yaml
```

### Step 4: Update Collection Definition

Add new requests to `.resources/definition.yaml`

### Step 5: Update Environments

Ensure environment files have required variables

## File Naming Convention

### Request Files

- Use kebab-case: `v1-risco-emissao.request.yaml`
- Include version prefix: `v1-`, `v2-`
- Use `.request.yaml` extension

### Collection Folders

- Use project name with " APIs" suffix
- Group by version: `v1/`, `v2/`

## Example: Agent Execution

### Input: Azure Function

```csharp
[Function("post")]
[HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "v1/lotes")]
[OpenApiRequestBody(contentType: "application/json", bodyType: typeof(LoteRequest))]
public async Task<IActionResult> Post([HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req)
{
    var contentBody = await new StreamReader(req.Body).ReadToEndAsync();
    var modelRequest = Deserialize<LoteRequest>(contentBody);
    // ...
}
```

### Output: Postman Request

```yaml
$kind: http-request
name: v1/lotes
url: "{{ServerUrl}}/api/v1/lotes"
method: POST
headers:
  x-functions-key: "{{FunctionsKey}}"
settings:
  followOriginalHttpMethod: true
description: >
  Creates a new lote.
body:
  type: json
  content: |
    {
      "codCiaSusep": "",
      "codLoteCia": ""
    }
responses:
  - statusCode: 200
    content-type: application/json
    description: Success
  - statusCode: 400
    content-type: application/json
    description: BadRequest
```

## Exclusion Rules

### Skip Non-HTTP Triggers

The Agent should skip:

- TimerTrigger functions
- QueueTrigger functions
- BlobTrigger functions
- ServiceBusTrigger functions
- Internal/private functions

### Skip Patterns

```csharp
// Skip these:
[TimerTrigger("%TimerSchedule%")]
[QueueTrigger("%QueueName%")]
[BlobTrigger("%Container%")]
```

## Response Documentation

### Standard Responses

The Agent should add standard response documentation:

```yaml
responses:
  - statusCode: 200
    content-type: application/json
    description: Success
  - statusCode: 400
    content-type: application/json
    description: BadRequest
  - statusCode: 401
    description: Unauthorized
  - statusCode: 500
    description: InternalServerError
```

## Manual Review Required

After generating files, the Agent should:

1. **Mark as draft** - Requests need body customization
2. **Add descriptions** - Document business logic
3. **Include examples** - Add real test data
4. **Verify routes** - Ensure paths are correct

## Error Handling

### Missing Information

If the Agent cannot determine:

- Request body → Use empty object `{}`
- Response codes → Use standard set
- Authentication → Log warning

## Output Location

The Agent should create files in:

```
.postman/collections/<ProjectName> APIs/v1/
```

## Configuration

### Agent Settings

```yaml
agent:
  version_prefix: "v1"
  api_path_prefix: "/api"
  include_health_check: true
  skip_triggers:
    - TimerTrigger
    - QueueTrigger
    - BlobTrigger
```

---

## Best Practices

### 1. Idempotent Runs

The Agent should be able to run multiple times without duplicating requests:

- Check if request already exists before creating
- Update existing requests if structure changed

### 2. Version Detection

Detect API version from route:
- `/v1/*` → `v1/` folder
- `/v2/*` → `v2/` folder

### 3. Authentication Detection

Detect auth type from HttpTrigger:
- `AuthorizationLevel.Function` → Requires `x-functions-key`
- `AuthorizationLevel.Anonymous` → No key required (but recommended)

### 4. Ordering

Create requests in logical order:
1. Health check first
2. CRUD operations (Create, Read, Update, Delete)
3. Business operations

---

## Human Review Checklist

After Agent execution, Human should verify:

- [ ] All HTTP endpoints discovered
- [ ] Routes correctly mapped
- [ ] Request bodies have valid JSON
- [ ] Headers configured correctly
- [ ] Environments updated
- [ ] Collection definition updated

---

## Need Help?

1. Check Function attributes for triggers
2. Verify route patterns
3. Review generated YAML syntax