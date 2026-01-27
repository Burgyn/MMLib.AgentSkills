# OpenAPI Paths in .NET WebAPI

## Standard OpenAPI Endpoints

### Modern .NET (ASP.NET Core with Microsoft.AspNetCore.OpenApi)

When using `app.MapOpenApi()` in `Program.cs`, the standard OpenAPI JSON endpoint is:

- **Path**: `/openapi/v1.json`
- **Full URL example**: `http://localhost:5000/openapi/v1.json`

### Legacy Swashbuckle/Swagger

Older projects using Swashbuckle.AspNetCore may expose OpenAPI at:

- **Path**: `/swagger/v1/swagger.json`
- **Full URL example**: `http://localhost:5000/swagger/v1/swagger.json`

### Alternative Paths

Some projects may customize the OpenAPI path. Common alternatives:

- `/openapi.json`
- `/api/openapi.json`
- `/swagger.json`
- `/api/swagger.json`

## Discovery Strategy

When discovering OpenAPI documentation:

1. **Try standard path first**: `/openapi/v1.json`
2. **Try legacy path**: `/swagger/v1/swagger.json`
3. **Try common alternatives**: `/openapi.json`, `/swagger.json`
4. **Check Program.cs**: Look for `MapOpenApi()` or `MapSwagger()` calls to determine custom paths

## Program.cs Configuration Examples

### Modern OpenAPI Setup

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi();

var app = builder.Build();
app.MapOpenApi(); // Exposes /openapi/v1.json
```

### Custom OpenAPI Path

```csharp
app.MapOpenApi("/custom/openapi.json");
```

### Swashbuckle Setup (Legacy)

```csharp
builder.Services.AddSwaggerGen();
app.UseSwagger();
app.UseSwaggerUI();
// Exposes /swagger/v1/swagger.json
```

## Response Format

OpenAPI JSON follows the OpenAPI 3.0 specification structure:

```json
{
  "openapi": "3.0.1",
  "info": {
    "title": "API Title",
    "version": "v1"
  },
  "paths": {
    "/api/cars": {
      "get": { ... },
      "post": { ... }
    }
  },
  "components": {
    "schemas": { ... }
  }
}
```

## Key Fields for Request Generation

- **paths**: Object containing all API endpoints
  - Each path has HTTP methods: `get`, `post`, `put`, `delete`, `patch`
  - Each method contains: `parameters`, `requestBody`, `responses`
- **parameters**: Array of query, path, or header parameters
- **requestBody**: Schema for POST/PUT request bodies
- **components.schemas**: Reusable schema definitions
