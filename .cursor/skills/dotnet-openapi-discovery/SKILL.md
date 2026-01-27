---
name: dotnet-openapi-discovery
description: Discover current OpenAPI documentation for .NET WebAPI endpoints. Use when agent needs to find OpenAPI documentation, create requests in .http files, or test endpoints. The skill can start the service if not running, download OpenAPI JSON, and generate VS Code REST Client format requests or curl commands for testing.
---

# .NET OpenAPI Discovery

## Overview

This skill enables the agent to discover current OpenAPI documentation for .NET WebAPI projects. It can start the service if needed, download OpenAPI JSON, and use it to create requests in .http files or test endpoints via curl.

## When to Use

This skill activates when:
- Agent explicitly requests OpenAPI documentation
- Agent wants to create requests in .http files
- Agent wants to test specific endpoints
- User query contains keywords: "openapi", "swagger", "api documentation", ".http", "test endpoint"

## Workflow

Follow these steps sequentially:

### Step 1: Detect .NET WebAPI Project

1. Search for `.csproj` files in the workspace root
2. Read the `.csproj` file to verify it contains ASP.NET Core references:
   - Look for `Microsoft.AspNetCore.App` framework reference
   - Or `Microsoft.AspNetCore.App` package reference
   - Or `Microsoft.NET.Sdk.Web` SDK
3. If no WebAPI project is found, inform the user and end workflow
4. Search for `launchSettings.json` in:
   - `Properties/launchSettings.json` (preferred location)
   - `launchSettings.json` (root directory)

### Step 2: Determine Base URL and Launch Profile

1. Read and parse `launchSettings.json` as JSON
2. Extract the `profiles` object
3. Select a launch profile in this priority order:
   - Profile named `"http"` (if exists)
   - Profile named `"https"` (if exists)
   - First profile in the profiles object
4. Extract the base URL:
   - Extract the first URL from `applicationUrl` (split by semicolon if multiple URLs)
   - If `applicationUrl` contains multiple URLs separated by semicolons, prefer HTTP over HTTPS
   - Remove any trailing slashes
   - Example: `http://localhost:5000` or `https://localhost:5001`

### Step 3: Check if Service is Running

1. Extract the port number from the base URL (from Step 2)
2. Check if the service is running using one of these methods:

   **Method A: HTTP Request (preferred)**
   - Use `curl -f --max-time 2 <base-url>` or similar HTTP request
   - If the request succeeds (exit code 0), the service is running
   - If it fails or times out, the service is not running

   **Method B: Port Check (fallback)**
   - On macOS/Linux: `lsof -i :<port>` or `netstat -an | grep :<port>`
   - On Windows: `netstat -an | findstr :<port>`
   - If port is in use, service may be running (but verify with HTTP request)

3. If service is not running:
   - Start the service using `dotnet run --launch-profile <profile-name>` in background
   - Wait 3-5 seconds for startup
   - Verify service is responding (repeat check)

### Step 4: Discover OpenAPI Endpoint

1. Try standard OpenAPI paths in this order (see [openapi-paths.md](references/openapi-paths.md) for details):
   - `/openapi/v1.json` (modern .NET with `app.MapOpenApi()`)
   - `/swagger/v1/swagger.json` (legacy Swashbuckle)
   - `/openapi.json`
   - `/swagger.json`
2. For each path, attempt GET request: `curl -f --max-time 5 <base-url><path>`
3. If successful, proceed to Step 5
4. If all paths fail:
   - Check `Program.cs` for custom OpenAPI configuration
   - Look for `MapOpenApi()` or `MapSwagger()` calls to determine custom paths
   - If still not found, inform the user that OpenAPI endpoint could not be discovered

### Step 5: Download and Parse OpenAPI JSON

1. Download OpenAPI JSON from the discovered endpoint
2. Parse the JSON structure:
   - Extract `paths` object (contains all endpoints)
   - For each path, extract HTTP methods (`get`, `post`, `put`, `delete`, etc.)
   - Extract `parameters` (query params, path params, headers)
   - Extract `requestBody` for POST/PUT requests
   - Extract `responses` for response schemas
3. Store parsed data in memory for use in Step 6

### Step 6: Use OpenAPI Documentation

Choose the appropriate action based on user request:

#### Option A: Create .http Request File

1. Determine target file:
   - If user specified a file, use that
   - Otherwise, create or append to `api.http` or `requests.http` in workspace root
2. Format requests using VS Code REST Client format:
   ```http
   ### GET /api/cars/{id}
   GET http://localhost:5000/api/cars/1
   
   ### POST /api/cars
   POST http://localhost:5000/api/cars
   Content-Type: application/json
   
   {
     "name": "Toyota",
     "model": "Camry"
   }
   ```
3. For each endpoint:
   - Add comment with method and path: `### GET /api/cars`
   - Add request line with full URL
   - Add query parameters if present (append `?param=value`)
   - Add headers if needed (Content-Type, Authorization)
   - Add request body for POST/PUT requests (use example from schema if available)
4. Replace path parameters with example values (e.g., `{id}` → `1`)

#### Option B: Test Endpoint via curl

1. Identify the endpoint to test from user request
2. Construct curl command based on OpenAPI specification:
   ```bash
   curl -X GET http://localhost:5000/api/cars/1 \
     -H "Content-Type: application/json"
   ```
3. For POST/PUT requests, include request body:
   ```bash
   curl -X POST http://localhost:5000/api/cars \
     -H "Content-Type: application/json" \
     -d '{"name":"Toyota","model":"Camry"}'
   ```
4. Execute the curl command and display results
5. Verify response status code matches expected values from OpenAPI spec

## Edge Cases

### No launchSettings.json Found

- Use default ports:
  - HTTP: `http://localhost:5000`
  - HTTPS: `https://localhost:5001` (if HTTPS is preferred)
- Use profile name `"http"` or `"https"` when running `dotnet run`
- If neither profile exists, run `dotnet run` without `--launch-profile`

### Service Already Running

- If Step 3 detects the service is running, proceed directly to Step 4
- Do not attempt to start another instance

### OpenAPI Endpoint Not Found

- Try all standard paths (Step 4)
- Check `Program.cs` for custom configuration
- Inform user if OpenAPI endpoint cannot be discovered
- Suggest checking if OpenAPI is configured in the project

### Startup Failure

- If `dotnet run` fails, inform the user about the error
- Show the error message from the command output
- Do not proceed with OpenAPI discovery if service cannot start

### Invalid OpenAPI JSON

- If JSON parsing fails, inform the user
- Display error message
- Suggest verifying the OpenAPI endpoint returns valid JSON

### Missing Request Body Schema

- For POST/PUT requests without request body schema, use generic JSON structure
- Include placeholder values based on parameter names
- Add comment indicating schema was not available

## Technical Notes

### OpenAPI Path Discovery

See [openapi-paths.md](references/openapi-paths.md) for detailed information about standard OpenAPI paths in .NET WebAPI projects.

### VS Code REST Client Format

- Use `###` for request separators
- Include method and path in separator comment
- Full URL on request line
- Headers on separate lines
- Empty line before request body
- JSON body indented with 2 spaces

### Parsing OpenAPI JSON Structure

- `paths`: Object where keys are endpoint paths (e.g., `/api/cars`)
- Each path contains HTTP methods as keys (`get`, `post`, etc.)
- `parameters`: Array of parameter objects with `name`, `in` (query/path/header), `required`, `schema`
- `requestBody.content`: Object with content types as keys (e.g., `application/json`)
- `requestBody.content[type].schema`: JSON schema for request body
- `responses`: Object with status codes as keys (e.g., `200`, `400`)

### Service Check Commands

**HTTP Request:**
```bash
curl -f --max-time 2 http://localhost:5000
```

**Port Check (macOS/Linux):**
```bash
lsof -i :5000
# or
netstat -an | grep :5000
```

**Port Check (Windows):**
```bash
netstat -an | findstr :5000
```

### Starting Service

```bash
# From project root directory
dotnet run --launch-profile http
```

## Resources

### references/

- **openapi-paths.md**: Documentation of standard OpenAPI paths in .NET WebAPI projects, including discovery strategies and configuration examples.
