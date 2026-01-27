# Endpoint Examples

Complete, production-ready endpoint examples following Vertical Slice Architecture patterns.

## GET Endpoint Example

```csharp
public static class GetCarEndpoint
{
    public record Response(int Id, string Name, string Model, DateTime CreatedAt);
    
    public static RouteHandlerBuilder MapGetCar(this IEndpointRouteBuilder app)
        => app.MapGet("/{id:int}", Handler)
            .WithTags("Cars");
    
    private static async Task<Results<Ok<Response>, NotFound>> Handler(
        int id,
        ICarRepository repository,
        CancellationToken ct)
    {
        var car = await repository.GetByIdAsync(id, ct);
        return car is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(new Response(car.Id, car.Name, car.Model, car.CreatedAt));
    }
}
```

## POST Endpoint Example

```csharp
public static class CreateCarEndpoint
{
    public record Request(
        [property: Required, StringLength(100)] string Name,
        [property: Required, StringLength(100)] string Model
    );
    public record Response(int Id, string Name, string Model);

    public static RouteHandlerBuilder MapCreateCar(this IEndpointRouteBuilder app)
        => app.MapPost("/", Handler)
            .WithTags("Cars");

    private static async Task<Results<Created<Response>, BadRequest<ValidationProblemDetails>>> Handler(
        [AsParameters] Request request,
        ICarRepository repository,
        HttpContext httpContext,
        CancellationToken ct)
    {
        if (!httpContext.Request.IsValid(out var validationResult))
        {
            return TypedResults.BadRequest(validationResult);
        }

        var car = new Car { Name = request.Name, Model = request.Model };
        await repository.AddAsync(car, ct);
        return TypedResults.Created($"/cars/{car.Id}", new Response(car.Id, car.Name, car.Model));
    }
}
```

## PUT Endpoint Example

```csharp
public static class UpdateCarEndpoint
{
    public record Request(
        [property: Required, StringLength(100)] string Name,
        [property: Required, StringLength(100)] string Model
    );
    public record Response(int Id, string Name, string Model);

    public static RouteHandlerBuilder MapUpdateCar(this IEndpointRouteBuilder app)
        => app.MapPut("/{id:int}", Handler)
            .WithTags("Cars");

    private static async Task<Results<Ok<Response>, NotFound, BadRequest<ValidationProblemDetails>>> Handler(
        int id,
        [AsParameters] Request request,
        ICarRepository repository,
        HttpContext httpContext,
        CancellationToken ct)
    {
        if (!httpContext.Request.IsValid(out var validationResult))
        {
            return TypedResults.BadRequest(validationResult);
        }

        var car = await repository.GetByIdAsync(id, ct);
        if (car is null)
        {
            return TypedResults.NotFound();
        }

        car.Name = request.Name;
        car.Model = request.Model;
        await repository.UpdateAsync(car, ct);

        return TypedResults.Ok(new Response(car.Id, car.Name, car.Model));
    }
}
```

## DELETE Endpoint Example

```csharp
public static class DeleteCarEndpoint
{
    public static RouteHandlerBuilder MapDeleteCar(this IEndpointRouteBuilder app)
        => app.MapDelete("/{id:int}", Handler)
            .WithTags("Cars");
    
    private static async Task<Results<NoContent, NotFound>> Handler(
        int id,
        ICarRepository repository,
        CancellationToken ct)
    {
        var car = await repository.GetByIdAsync(id, ct);
        if (car is null)
        {
            return TypedResults.NotFound();
        }
        
        await repository.DeleteAsync(id, ct);
        return TypedResults.NoContent();
    }
}
```

## GET List Endpoint Example

```csharp
public static class ListCarsEndpoint
{
    public record Response(int Id, string Name, string Model);
    
    public static RouteHandlerBuilder MapListCars(this IEndpointRouteBuilder app)
        => app.MapGet("/", Handler)
            .WithTags("Cars");
    
    private static async Task<Ok<IEnumerable<Response>>> Handler(
        ICarRepository repository,
        CancellationToken ct)
    {
        var cars = await repository.GetAllAsync(ct);
        var response = cars.Select(c => new Response(c.Id, c.Name, c.Model));
        return TypedResults.Ok(response);
    }
}
```

## Endpoint with Query Parameters

```csharp
public static class SearchCarsEndpoint
{
    public record Response(int Id, string Name, string Model);
    
    public static RouteHandlerBuilder MapSearchCars(this IEndpointRouteBuilder app)
        => app.MapGet("/search", Handler)
            .WithTags("Cars");
    
    private static async Task<Ok<IEnumerable<Response>>> Handler(
        string? name,
        string? model,
        ICarRepository repository,
        CancellationToken ct)
    {
        var cars = await repository.SearchAsync(name, model, ct);
        var response = cars.Select(c => new Response(c.Id, c.Name, c.Model));
        return TypedResults.Ok(response);
    }
}
```

## Endpoint with Validation

```csharp
using System.ComponentModel.DataAnnotations;

public static class CreateCarEndpoint
{
    public record Request(
        [property: Required(ErrorMessage = "Name is required")]
        [property: MaxLength(100, ErrorMessage = "Name must be 100 characters or less")]
        string Name,
        [property: Required(ErrorMessage = "Model is required")]
        string Model
    );

    public record Response(int Id, string Name, string Model);

    public static RouteHandlerBuilder MapCreateCar(this IEndpointRouteBuilder app)
        => app.MapPost("/", Handler)
            .WithTags("Cars")
            .Produces<Response>(StatusCodes.Status201Created)
            .Produces<ValidationProblemDetails>(StatusCodes.Status400BadRequest);

    private static async Task<Results<Created<Response>, BadRequest<ValidationProblemDetails>>> Handler(
        [AsParameters] Request request,
        ICarRepository repository,
        CancellationToken ct,
        [FromServices] IServiceProvider serviceProvider 
    )
    {
        var validationContext = new ValidationContext(request, serviceProvider, null);
        var validationResults = new List<ValidationResult>();
        if (!Validator.TryValidateObject(request, validationContext, validationResults, true))
        {
            var errors = validationResults
                .GroupBy(r => r.MemberNames.FirstOrDefault() ?? string.Empty)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(r => r.ErrorMessage ?? "").ToArray()
                );
            return TypedResults.BadRequest(new ValidationProblemDetails(errors));
        }

        var car = new Car { Name = request.Name, Model = request.Model };
        await repository.AddAsync(car, ct);
        return TypedResults.Created($"/cars/{car.Id}", new Response(car.Id, car.Name, car.Model));
    }
}
```

## Expression Body Syntax Examples

**Simple GET handler:**

```csharp
private static Task<Results<Ok<Product>, NotFound>> Handler(
    int id,
    IProductRepository repository,
    CancellationToken ct)
    => repository.GetByIdAsync(id, ct) is { } product
        ? Task.FromResult(TypedResults.Ok(product))
        : Task.FromResult(TypedResults.NotFound());
```

**Simple Map method:**

```csharp
public static RouteHandlerBuilder MapGetCar(this IEndpointRouteBuilder app)
    => app.MapGet("/{id:int}", Handler)
        .WithTags("Cars");
```
