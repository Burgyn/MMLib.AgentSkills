---
name: dotnet-vertical-slice-architecture
description: Vertical Slice Architecture patterns and conventions for .NET Minimal API projects. Use when organizing endpoints, creating handlers, structuring modules, grouping endpoints, or implementing feature-based architecture. Covers endpoint organization, Request/Response DTOs, Setup.cs pattern, Results handlers with strongly-typed return values, MapGroup usage, and expression body syntax.
---

# Vertical Slice Architecture

## Overview

This skill defines the Vertical Slice Architecture pattern implementation for .NET projects using Minimal APIs. Organize code by features (vertical slices) rather than technical layers, where each feature contains all code from endpoint to database.

## Architecture Overview

Vertical Slice Architecture organizes code by feature rather than by technical layers. Instead of separating concerns into horizontal layers (UI, business logic, data access), each vertical slice contains all code for a specific feature from endpoint to database.

**Benefits:**
- Self-contained features that are easier to understand
- Reduced indirection and object transformation overhead
- Features are independently testable
- Easier to maintain and scale

**Contrast with N-Tier Architecture:**
- Traditional N-Tier: Single feature scattered across multiple folders/projects
- Vertical Slice: All code for a feature lives together

## Project Directory Structure

Organize your project by features (modules) rather than technical layers. Each feature/module is self-contained in its own folder, containing all code from endpoint to database.

### Typical Project Structure

```text
ProjectRoot/
├── Program.cs
├── ProjectName.csproj
├── Features/
│   ├── Cars/
│   │   ├── Setup.cs
│   │   ├── CreateCarEndpoint.cs
│   │   ├── GetCarEndpoint.cs
│   │   ├── UpdateCarEndpoint.cs
│   │   ├── DeleteCarEndpoint.cs
│   │   ├── Car.cs (domain model)
│   │   ├── ICarRepository.cs
│   │   └── CarRepository.cs
│   ├── Users/
│   │   ├── Setup.cs
│   │   ├── CreateUserEndpoint.cs
│   │   ├── GetUserEndpoint.cs
│   │   ├── User.cs
│   │   ├── IUserRepository.cs
│   │   └── UserRepository.cs
│   └── Orders/
│       ├── Setup.cs
│       ├── CreateOrderEndpoint.cs
│       ├── GetOrderEndpoint.cs
│       ├── Order.cs
│       ├── IOrderRepository.cs
│       └── OrderRepository.cs
└── Infrastructure/
    ├── Database/
    │   └── AppDbContext.cs (if using EF Core)
    ├── Logging/
    └── ...
```

### Structure Guidelines

**Root Level:**
- `Program.cs` - Application entry point and configuration
- `ProjectName.csproj` - Project file

**Features Folder:**
- Contains all feature/module folders (Cars, Users, Orders, etc.)
- Each module folder is a vertical slice containing:
  - `Setup.cs` - Module configuration (DI and endpoint mapping)
  - Endpoint files - One file per endpoint operation (CreateCarEndpoint.cs, GetCarEndpoint.cs, etc.)
  - Domain models - Entity classes (Car.cs, User.cs, etc.)
  - Repository interfaces - Module-specific repository contracts (ICarRepository.cs)
  - Repository implementations - Data access implementations (CarRepository.cs)
  - Module-specific services - Any additional services needed by the module

**Infrastructure Folder:**
- Contains truly cross-cutting concerns:
  - Database configuration (DbContext, migrations if using EF Core)
  - Logging configuration
  - Shared utilities and helpers
  - External service integrations
- **Important**: Infrastructure should not contain business logic or domain models specific to features

### Key Principles

- **Self-Contained Modules**: Each feature/module folder contains all code needed for that feature
- **Co-location**: Related code (endpoints, models, repositories) lives together in the same module folder
- **Setup.cs Location**: Each module has its own `Setup.cs` at the module level, not at the root
- **One File Per Endpoint**: Each endpoint operation is its own file (CreateCarEndpoint.cs, not CarsEndpoints.cs)
- **Infrastructure for Cross-Cutting**: Use Infrastructure folder only for truly shared, cross-cutting concerns

## Cross-Module Dependencies

Vertical Slice Architecture follows the principle: **"Minimize coupling between slices, maximize coupling within a slice."** While features should be self-contained, there are legitimate cases where modules need to interact.

## Minimal API as Application Layer

Use **Minimal APIs instead of Controllers**. Treat the Minimal API endpoint as the application layer itself, not just a thin entry point.

**Key principles:**
- Avoid unnecessary layering and abstraction
- Reduce object transformation overhead
- Keep structure simple and easy to navigate
- Use only the layers necessary for your application

## Endpoint Structure Pattern

Each endpoint is its own vertical slice, organized as a static class:

```csharp
public static class CreateCarEndpoint
{
    public record Request(string Name, string Model);
    public record Response(int Id, string Name, string Model);

    public static RouteHandlerBuilder MapCreateCar(this IEndpointRouteBuilder app)
        => app.MapPost("/cars", Handler);

    private static async Task<Results<Created<Response>, BadRequest>> Handler(
        Request request,
        ICarRepository repository,
        CancellationToken ct)
    {
        var car = new Car { Name = request.Name, Model = request.Model };
        await repository.AddAsync(car, ct);
        return TypedResults.Created($"/cars/{car.Id}", new Response(car.Id, car.Name, car.Model));
    }
}
```

**Key points:**
- Static class named after the operation (e.g., `CreateCarEndpoint`)
- Request and Response DTOs as nested records in the same class
- Extension method `Map{Operation}` uses expression body syntax (`=>`)
- Private static `Handler` method with business logic
- Handler receives dependencies via dependency injection
- **Always use expression body syntax (`=>`) for methods where possible**

For complete endpoint examples, see [endpoint-examples.md](references/endpoint-examples.md).

## Module Organization Pattern

Each module (e.g., Cars, Users, Orders) has a `Setup.cs` class:

```csharp
public static class Setup
{
    public static IServiceCollection AddCars(this IServiceCollection services)
    {
        services.AddScoped<ICarRepository, CarRepository>();
        return services;
    }

    public static IEndpointRouteBuilder MapCars(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/cars")
            .WithTags("Cars")
            .RequireAuthorization();

        group.MapCreateCar();
        group.MapGetCar();
        group.MapUpdateCar();
        group.MapDeleteCar();

        return app;
    }
}
```

**Key points:**
- `Add{Module}` method for dependency injection configuration
- `Map{Module}` method for endpoint mapping
- Use MapGroup to create a route group with common properties
- Register all endpoints from the module in the Map method
- **Prefer expression body syntax (`=>`) for simple methods**

For complete module examples, see [module-examples.md](references/module-examples.md).

## Handler Return Types - Strongly Typed Results

Use strongly-typed discriminant union types for handler return values:

```csharp
private static async Task<Results<Ok<Product>, NotFound>> Handler(
    int id,
    IProductRepository repository,
    CancellationToken ct)
{
    var product = await repository.GetByIdAsync(id, ct);
    return product is null 
        ? TypedResults.NotFound() 
        : TypedResults.Ok(product);
}
```

**Expression body syntax for simple methods:**

For any function or method that can be written as a single expression (including handlers and utility methods), use the expression-bodied member syntax (`=>`):

```csharp
// Good: Simple Map method
public static RouteHandlerBuilder MapGetCar(this IEndpointRouteBuilder app)
    => app.MapGet("/{id:int}", Handler);

// Good: Simple synchronous method
public int CalculateTotal(int a, int b) => a + b;
```

**Important: Expression body syntax and async methods:**

**Do NOT use expression body syntax for async methods that require await**. Pattern matching on `Task<T>` does not work - you must await the task first.

```csharp
// BAD: This does NOT work - pattern matching on Task<Product?>
private static Task<Results<Ok<Product>, NotFound>> Handler(
    int id,
    IProductRepository repository,
    CancellationToken ct)
    => repository.GetByIdAsync(id, ct) is { } product  // ERROR: Task<Product?> is not Product?
        ? Task.FromResult<Results<Ok<Product>, NotFound>>(TypedResults.Ok(product))
        : Task.FromResult<Results<Ok<Product>, NotFound>>(TypedResults.NotFound());

// GOOD: Use async/await syntax for async methods
private static async Task<Results<Ok<Product>, NotFound>> Handler(
    int id,
    IProductRepository repository,
    CancellationToken ct)
{
    var product = await repository.GetByIdAsync(id, ct);
    return product is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(product);
}
```

**Guidelines:**
- Use expression body syntax (`=>`) for simple synchronous methods and Map methods
- Use standard async/await syntax for async methods that require await
- Expression body syntax can be used for async methods that return `Task.FromResult(...)` without await

**More patterns are supported; these are the most common:**
- `Results<Ok<T>, NotFound>` – most common pattern (e.g., details or fetch by ID)
- `Results<Ok<T>, NotFound, BadRequest>` – multiple possible results (e.g., validation failure + not found)
- `Results<Created<T>, BadRequest, Conflict>` – typical for create scenarios
- ...

Use the static `TypedResults` class to create result instances.

Declare all possible results explicitly in the return type. For additional result patterns, see the documentation.

**Benefits:**
- Type-safe error handling
- Clear contract of possible responses
- Better OpenAPI/Swagger documentation
- Compile-time checking of return types

For comprehensive Results patterns, see [results-patterns.md](references/results-patterns.md).

## Endpoint Grouping with MapGroup

Use `MapGroup` to apply common properties to groups of endpoints:

```csharp
var carsGroup = app.MapGroup("/api/cars")
    .WithTags("Cars")
    .WithOpenApi()
    .RequireAuthorization("AdminPolicy")
    .WithSummary("Car management endpoints");

carsGroup.MapCreateCar();
carsGroup.MapGetCar();
```

**Common properties applied via MapGroup:**
- **Route prefix**: `/api/cars` applied to all endpoints
- **Tags**: For OpenAPI documentation grouping
- **Authorization**: Common authorization policies
- **OpenAPI metadata**: Summary, description, examples
- **Rate limiting**: Apply rate limits to entire group
- **CORS**: Group-specific CORS policies
- **And more**: Any additional shared endpoint filters or conventions (e.g., logging, custom headers, versioning, request/response transformations, etc.)

**Pattern:**
- Create group in `Setup.Map{Module}` method
- Pass group to each endpoint's `Map` method
- Endpoints inherit group properties automatically

## Best Practices

- **Expression Body Syntax**: Always use expression body syntax (`=>`) for methods where possible, especially for Map methods and simple handlers
- **Feature Cohesion**: Keep all code for a feature together (endpoint, handler, validation, domain logic)
- **Minimal Layers**: Use only necessary layers, avoid over-engineering
- **Self-Contained Slices**: Each slice should be independently testable
- **Clear Naming**: Endpoint classes clearly indicate their operation
- **Repository Pattern**: For data manipulation, use the repository pattern, but do not create large or overly generic repositories—instead, define specific, focused repository interfaces per domain or feature
- **Dependency Injection**: Inject dependencies (including repositories) directly into handlers
- **Cancellation Tokens**: Always accept `CancellationToken` in async handlers
- **Validation**: Use new Data Annotations on Request DTOs
- **Error Handling**: Use Results pattern instead of exceptions for expected errors
- **OpenAPI Documentation**: Leverage WithOpenApi() and Produces attributes

## Integration with Other Skills

- Works with `dotnet-technology-stack` skill for technology choices
- Uses Minimal API features from .NET 10+
- Follows C# modern features (records, file-scoped namespaces, expression body syntax, etc.)

## Reference Files and Templates

This skill includes detailed examples and templates:

- **Reference Files** (loaded into context when needed):
  - See [endpoint-examples.md](references/endpoint-examples.md) for complete endpoint examples
