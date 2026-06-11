# Moyo Architecture Guide: Building Apps & Modules

**Purpose**: Comprehensive guide on how apps/modules are built in Moyo, how they interact with the gateway, and how endpoints are structured.

**Examples Reference**: StockChecking (Query-based) and SupplyChain (Command-based) apps

**Target Audience**: Developers implementing new features (e.g., LeadTimeMatrix module)

**Last Updated**: 2026

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Clean Architecture Layers](#clean-architecture-layers)
3. [App Project Structure](#app-project-structure)
4. [Endpoint Patterns](#endpoint-patterns)
5. [CQRS & Wolverine Messaging](#cqrs--wolverine-messaging)
6. [Gateway Interaction Pattern](#gateway-interaction-pattern)
7. [Dependency Injection & Registration](#dependency-injection--registration)
8. [Caching Strategy](#caching-strategy)
9. [Authorization & Middleware](#authorization--middleware)
10. [Full Request Flow Example](#full-request-flow-example)
11. [Key Patterns & Best Practices](#key-patterns--best-practices)
12. [Creating a New Endpoint](#creating-a-new-endpoint)

---

## Architecture Overview

Moyo follows a **modular microservices + distributed monolith hybrid architecture** using:

- **Clean Architecture** (Domain → Application → Infrastructure → API)
- **CQRS Pattern** (Commands for writes, Queries for reads)
- **Wolverine Messaging** (In-process & distributed messaging via RabbitMQ)
- **Dependency Injection** (Microsoft.Extensions.DependencyInjection)
- **Gateway Pattern** (Apps delegate to centralized Amtrack gateway for data)

### Key Technologies

| Layer | Technology |
|-------|-----------|
| API | ASP.NET Core 8 + Wolverine |
| Messaging | Wolverine (RabbitMQ backend) |
| Caching | FusionCache with Redis backplane |
| Auth | Azure AD (JWT Bearer) |
| Serialization | System.Text.Json |
| Validation | FluentValidation |
| ORM | Entity Framework Core |
| Mapping | AutoMapper |

---

## Clean Architecture Layers

Every app has **4 project layers**:

### 1. **Domain Layer** (`StockChecking.Domain`)
- **Entities**: Core business models
- **Value Objects**: Immutable, identityless objects
- **Interfaces**: Domain service contracts
- **Exceptions**: Domain-specific exceptions
- **No External Dependencies**: Pure business logic

**Example**:
```csharp
namespace Moyo.Apps.StockChecking.Domain.Entities;

public class Communication
{
	public int Id { get; set; }
	public string EmailAddress { get; set; }
	public string Name { get; set; }
	public DateTime CreatedDate { get; set; }
}
```

### 2. **Application Layer** (`StockChecking.Application`)
- **Commands/Queries**: Request objects (CQRS)
- **Handlers**: Business logic orchestrators
- **Validators**: Fluent validation rules
- **DTOs**: Data transfer objects
- **Gateway Interfaces**: Contracts for external data
- **Cache Utilities**: Caching logic

**Example Command**:
```csharp
namespace Moyo.Apps.StockChecking.Application.Endpoints.StockQuotes.Commands.CreateStockQuote;

public record CreateStockQuoteCommand(
	int VariantId,
	string CustomerEmail,
	int Quantity
);

public class CreateStockQuoteCommandHandler
{
	public static async Task<int> Handle(
		CreateStockQuoteCommand command,
		IStockQuoteRepository repository,
		IMediator mediator,
		CancellationToken cancellationToken
	)
	{
		// Business logic
		var stockQuote = new StockQuote(command.VariantId, command.CustomerEmail, command.Quantity);
		return await repository.SaveAsync(stockQuote, cancellationToken);
	}
}
```

### 3. **Infrastructure Layer** (`StockChecking.Infrastructure`)
- **Database**: EF Core DbContext, migrations, configurations
- **Gateway Services**: Implementation of gateway interfaces
- **Repositories**: Data access patterns
- **External Services**: Email, file storage, etc.
- **Data Seeders**: Test data generators

**Example Gateway Service**:
```csharp
namespace Moyo.Apps.StockChecking.Infrastructure.Services.Gateway;

public partial class GatewayService(
	IMoyoDataGatewayClient client, 
	IHttpHeaderContext headerContext
) : IGatewayService
{
	public async Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken)
	{
		var result = await client.Colors.ExecuteAsync(cancellationToken);
		HandleErrors(result);
		return result.Data?.SegmentColors?.Select(...).ToList() ?? [];
	}
}
```

### 4. **API Layer** (`StockChecking.Api`)
- **Endpoints**: HTTP route handlers
- **Program.cs**: Service configuration & middleware setup
- **Authorization**: Role-based access control
- **Custom Middleware**: Request/response interceptors
- **Exception Handlers**: Global error handling

**Example Endpoint**:
```csharp
namespace StockChecking.Api.Endpoints.Colors;

public static partial class StockCheckingEndpoints
{
	[Authorize]
	[WolverineGet($"{Routes.Base}/Colors")]
	public static Task<List<ColorDto>> GetColors(
		[FromQuery] GetColorsQuery query,
		IMessageBus messageBus,
		CancellationToken cancellationToken
	) => messageBus.InvokeAsync<List<ColorDto>>(query, cancellationToken);
}
```

---

## App Project Structure

### Standard Folder Hierarchy

```
src/Apps/StockChecking/
├── StockChecking.Api/                          # HTTP Layer
│   ├── Program.cs                              # Startup configuration
│   ├── Endpoints/                              # Route handlers
│   │   ├── Colors/
│   │   │   └── GetColors.cs                    # Endpoint definition
│   │   ├── Variants/
│   │   │   ├── GetBrandingCalculatorPrices.cs
│   │   │   └── GetBrandingCalculatorPositions.cs
│   │   └── LeadTimeEnquiry/
│   │       ├── GetLeadTimeEnquiryNextWorkingDay.cs
│   │       └── SubmitLeadTimeEnquiry.cs
│   ├── Authorization/
│   │   └── [Custom auth policies]
│   └── appsettings.json
│
├── StockChecking.Application/                  # Business Logic Layer
│   ├── DependencyInjection.cs                  # Service registration
│   ├── CacheDefaults.cs                        # Shared cache config
│   ├── Endpoints/
│   │   ├── Colors/
│   │   │   └── Queries/
│   │   │       └── GetColors/
│   │   │           ├── GetColorsQuery.cs       # Query definition
│   │   │           ├── GetColorsQueryHandler.cs
│   │   │           ├── GetColorsQueryValidator.cs
│   │   │           └── ColorDto.cs
│   │   ├── Variants/
│   │   │   ├── Queries/
│   │   │   │   ├── GetBrandingCalculatorPositions/
│   │   │   │   └── GetBrandingCalculatorPrices/
│   │   │   └── [DTOs]
│   │   └── StockQuotes/
│   │       └── Commands/
│   │           └── CreateStockQuote/
│   │               ├── CreateStockQuoteCommand.cs
│   │               ├── CreateStockQuoteCommandHandler.cs
│   │               ├── CreateStockQuoteCommandValidator.cs
│   │               └── CreateStockQuoteDto.cs
│   ├── Interfaces/
│   │   └── Gateway/
│   │       ├── GatewayInterface/               # Gateway method contracts
│   │       │   ├── Colors.cs                   # IGatewayService.Colors method
│   │       │   ├── Customers.cs
│   │       │   └── Variants.cs
│   │       └── Dtos/
│   │           ├── ColorGatewayDto.cs          # Gateway response DTOs
│   │           ├── CustomerGatewayDto.cs
│   │           └── VariantGatewayDto.cs
│   └── Cache/
│       └── CacheExtensions.cs
│
├── StockChecking.Infrastructure/               # Data & Integration Layer
│   ├── Data/
│   │   ├── ApplicationDbContext.cs
│   │   ├── Configurations/                     # EF Core mappings
│   │   │   └── CommunicationConfiguration.cs
│   │   ├── Migrations/
│   │   │   ├── 20251204053927_Initial.cs
│   │   │   └── [other migrations]
│   │   └── DataGenerators/
│   ├── Services/
│   │   ├── Gateway/
│   │   │   ├── Base.cs                         # Base gateway service
│   │   │   ├── Colors.cs                       # IGatewayService.Colors impl
│   │   │   ├── Customers.cs
│   │   │   └── Constants.cs
│   │   └── Email/
│   │       └── Base.cs
│   └── Repositories/                           # Data access patterns
│       └── IStockQuoteRepository.cs
│
└── StockChecking.Domain/                       # Business Entities Layer
	├── Entities/
	│   ├── Communication.cs
	│   └── StockQuote.cs
	└── Exceptions/
		└── DomainException.cs
```

---

## Endpoint Patterns

### Query Endpoint (GET - Read-Only)

**Purpose**: Retrieve data without side effects

**Pattern**:
1. **Endpoint** receives query object via URL/body
2. **MessageBus** routes to handler
3. **Handler** calls gateway/database, applies business logic
4. **Response** returned to client

**Example: GetColors (Simple Query)**

```csharp
// 1. ENDPOINT DEFINITION (Api/Endpoints/Colors/GetColors.cs)
namespace StockChecking.Api.Endpoints.Colors;

public static partial class StockCheckingEndpoints
{
	[Authorize]
	[WolverineGet($"{Routes.Base}/Colors")]
	public static Task<List<ColorDto>> GetColors(
		[FromQuery] GetColorsQuery query,
		IMessageBus messageBus,
		CancellationToken cancellationToken
	) => messageBus.InvokeAsync<List<ColorDto>>(query, cancellationToken);
}

// 2. QUERY DEFINITION (Application/Endpoints/Colors/Queries/GetColors/GetColorsQuery.cs)
namespace Moyo.Apps.StockChecking.Application.Endpoints.Colors.Queries.GetColors;

public record GetColorsQuery();  // Simple marker query

// 3. VALIDATOR (Application/Endpoints/Colors/Queries/GetColors/GetColorsQueryValidator.cs)
public class GetColorsQueryValidator : AbstractValidator<GetColorsQuery>
{
	public GetColorsQueryValidator()
	{
		// Validation rules here if needed
	}
}

// 4. HANDLER (Application/Endpoints/Colors/Queries/GetColors/GetColorsQueryHandler.cs)
public static class GetColorsQueryHandler
{
	private const string CacheKey = "stock-checking:colors";

	public static async Task<List<ColorDto>> Handle(
		GetColorsQuery _,
		IGatewayService gatewayService,
		IFusionCacheWithKeyTracking cache,
		CancellationToken cancellationToken
	)
	{
		return await cache.GetOrSetAsync(
			CacheKey,
			async ct =>
			{
				// Call gateway to fetch data
				var list = await gatewayService.Colors(ct);

				// Transform to DTO
				return list
					.Select(x => new ColorDto(x.Code, x.Name, x.Id, x.InternalId))
					.OrderBy(x => x.Name)
					.ToList();
			},
			TimeSpan.FromHours(1),
			cancellationToken
		);
	}
}

// 5. DTO (Application/Endpoints/Colors/Queries/GetColors/ColorDto.cs)
public record ColorDto(string Code, string Name, string? Id, string? InternalId);

// 6. GATEWAY INTERFACE (Application/Interfaces/Gateway/GatewayInterface/Colors.cs)
public partial interface IGatewayService
{
	Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken);
}

// 7. GATEWAY DTO (Application/Interfaces/Gateway/Dtos/ColorGatewayDto.cs)
public record ColorGatewayDto(string Code, string Name, string? Id, string? InternalId);

// 8. GATEWAY IMPLEMENTATION (Infrastructure/Services/Gateway/Colors.cs)
public partial class GatewayService : IGatewayService
{
	public async Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken)
	{
		var result = await ExecuteWithHeaders(
			DefaultHeaders,
			async () => await client.Colors.ExecuteAsync(cancellationToken)
		);
		HandleErrors(result);

		var data = result.Data?.SegmentColors ?? [];
		return data
			.Where(x => x.Code != null && x.Name != null)
			.Select(x => new ColorGatewayDto(x.Code!, x.Name!, x.Id, x.InternalId))
			.ToList();
	}
}
```

### Command Endpoint (POST/PUT/DELETE - Write Operations)

**Purpose**: Modify data with side effects

**Pattern**: Same as Query, but with command object containing mutation details

**Example: ApproveShippingContainer (SupplyChain)**

```csharp
// 1. ENDPOINT DEFINITION
namespace SupplyChain.Api.Endpoints.ShippingContainers;

public static partial class SupplyChainEndpoints
{
	[Authorize]
	[WolverinePut($"{Routes.Base}/ShippingContainers/Approve")]
	public static async Task ApproveShippingContainer(
		[FromBody] ApproveShippingContainerCommand command,
		IMessageBus messageBus,
		CancellationToken cancellationToken
	) => await messageBus.InvokeAsync(command, cancellationToken);
}

// 2. COMMAND DEFINITION
public record ApproveShippingContainerCommand(
	int ShippingContainerId,
	string ApprovedBy,
	DateTime ApprovedDate
);

// 3. VALIDATOR
public class ApproveShippingContainerCommandValidator 
	: AbstractValidator<ApproveShippingContainerCommand>
{
	public ApproveShippingContainerCommandValidator()
	{
		RuleFor(x => x.ShippingContainerId).GreaterThan(0);
		RuleFor(x => x.ApprovedBy).NotEmpty();
	}
}

// 4. HANDLER
public class ApproveShippingContainerCommandHandler
{
	public static async Task Handle(
		ApproveShippingContainerCommand command,
		IShippingContainerRepository repository,
		IMediator mediator,
		CancellationToken cancellationToken
	)
	{
		var container = await repository.GetByIdAsync(command.ShippingContainerId, cancellationToken);
		if (container == null)
			throw new EntityNotFoundException(nameof(ShippingContainer), command.ShippingContainerId);

		container.Approve(command.ApprovedBy, command.ApprovedDate);
		await repository.SaveAsync(container, cancellationToken);

		// Publish domain events
		await mediator.PublishAsync(container.DomainEvents, cancellationToken);
	}
}
```

### Endpoint Decorators

| Decorator | HTTP Method | Purpose |
|-----------|-----------|---------|
| `[WolverineGet(...)]` | GET | Retrieve data |
| `[WolverinePost(...)]` | POST | Create resource |
| `[WolverinePut(...)]` | PUT | Update resource |
| `[WolverinePatch(...)]` | PATCH | Partial update |
| `[WolverineDelete(...)]` | DELETE | Delete resource |
| `[Authorize]` | N/A | Require authentication |
| `[Authorize(Roles = "Admin")]` | N/A | Require specific role |

---

## CQRS & Wolverine Messaging

### What is CQRS?

**Command Query Responsibility Segregation**: Separate read and write operations.

- **Commands**: Mutate state (Create, Update, Delete)
- **Queries**: Read state (no side effects)

### Wolverine Message Bus

Moyo uses **Wolverine** for in-process & distributed messaging:

```csharp
// In API endpoint, inject IMessageBus
public static Task<Result> MyEndpoint(
	[FromBody] MyCommand command,
	IMessageBus messageBus,  // ← Wolverine message bus
	CancellationToken cancellationToken
)
{
	// Route command to handler + wait for result
	return messageBus.InvokeAsync<Result>(command, cancellationToken);
}
```

### Handler Discovery

Wolverine automatically discovers handlers via reflection:

```csharp
// Wolverine looks for methods matching these patterns:
public static class GetColorsQueryHandler
{
	// Pattern 1: public static Task<TResult> Handle(TRequest, ...)
	public static async Task<List<ColorDto>> Handle(
		GetColorsQuery query,
		IGatewayService gateway,  // Dependencies injected!
		IFusionCacheWithKeyTracking cache,
		CancellationToken cancellationToken
	) { ... }
}

// Pattern 2: public static void Handle(TCommand, ...)
public class CreateOrderCommandHandler
{
	public static void Handle(CreateOrderCommand command, IRepository repo) { ... }
}

// Pattern 3: public class MyHandler : IMessageHandler<TRequest>
public class SpecialHandler : IMessageHandler<SpecialRequest>
{
	public async Task Handle(SpecialRequest request, ...) { ... }
}
```

### Dependency Injection in Handlers

Wolverine automatically injects dependencies based on type:

```csharp
public static async Task<Result> Handle(
	MyCommand command,              // The command/query itself
	IGatewayService gateway,        // Interface (resolved from DI)
	IFusionCacheWithKeyTracking cache,
	IMediator mediator,             // MediatR for domain events
	CancellationToken cancellationToken
)
{
	// All injected automatically
}
```

---

## Gateway Interaction Pattern

### What is the Gateway?

The **Gateway** is a centralized service that provides access to legacy Amtrack data.

- **Single Source of Truth**: All read access to Amtrack goes through gateway
- **Cached**: Responses are cached to reduce load
- **Type-Safe**: Generated client from GraphQL/OpenAPI schema
- **Error Handling**: Unified error handling for external calls

### Gateway Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  StockChecking App                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Handler                                                         │
│    ↓                                                              │
│  IGatewayService (Interface)                                     │
│    ↓                                                              │
│  GatewayService (Implementation)                                 │
│    ↓                                                              │
│  IMoyoDataGatewayClient (Generated GraphQL Client)               │
│    ↓                                                              │
│  HTTP → Amtrack Gateway API                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Gateway Flow

#### 1. Define Gateway Interface (in Application Layer)

```csharp
// Application/Interfaces/Gateway/GatewayInterface/Colors.cs
public partial interface IGatewayService
{
	Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken);
}
```

#### 2. Define Gateway DTOs

```csharp
// Application/Interfaces/Gateway/Dtos/ColorGatewayDto.cs
public record ColorGatewayDto(
	string Code,
	string Name,
	string? Id,
	string? InternalId
);
```

#### 3. Implement Gateway Service

```csharp
// Infrastructure/Services/Gateway/Colors.cs
public partial class GatewayService : IGatewayService
{
	public async Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken)
	{
		// Call generated GraphQL client
		var result = await ExecuteWithHeaders(
			DefaultHeaders,
			async () => await client.Colors.ExecuteAsync(cancellationToken)
		);

		// Error handling
		HandleErrors(result);

		// Transform to DTO
		var data = result.Data?.SegmentColors ?? [];
		return data
			.Where(x => x.Code != null && x.Name != null)
			.Select(x => new ColorGatewayDto(x.Code!, x.Name!, x.Id, x.InternalId))
			.ToList();
	}
}
```

#### 4. Use in Handler

```csharp
public static async Task<List<ColorDto>> Handle(
	GetColorsQuery _,
	IGatewayService gatewayService,  // ← Injected
	IFusionCacheWithKeyTracking cache,
	CancellationToken cancellationToken
)
{
	return await cache.GetOrSetAsync(
		"stock-checking:colors",
		async ct =>
		{
			var gatewayDtos = await gatewayService.Colors(ct);  // ← Call
			return gatewayDtos
				.Select(x => new ColorDto(x.Code, x.Name, x.Id, x.InternalId))
				.OrderBy(x => x.Name)
				.ToList();
		},
		TimeSpan.FromHours(1),
		cancellationToken
	);
}
```

### Key Gateway Service Methods

```csharp
public partial class GatewayService : IGatewayService
{
	// Constructor: client is IMoyoDataGatewayClient (generated)
	public GatewayService(IMoyoDataGatewayClient client, IHttpHeaderContext headerContext)
	{
		this.client = client;
		this.headerContext = headerContext;
	}

	// Helper: Execute with additional headers
	protected async Task<T> ExecuteWithHeaders<T>(
		Dictionary<string, string>? additionalHeaders,
		Func<Task<T>> operation
	)
	{
		if (additionalHeaders == null || !additionalHeaders.Any())
			return await operation();

		try
		{
			headerContext.SetHeaders(additionalHeaders);
			return await operation();
		}
		finally
		{
			headerContext.ClearHeaders();
		}
	}

	// Helper: Handle gateway errors
	private void HandleErrors<T>(GraphQLResponse<T> result)
	{
		if (result.Errors != null && result.Errors.Any())
		{
			var errorMessage = string.Join(", ", result.Errors.Select(e => e.Message));
			throw new GatewayException($"Gateway error: {errorMessage}");
		}
	}

	// Actual methods follow IGatewayService interface
	public async Task<List<ColorGatewayDto>> Colors(CancellationToken cancellationToken) { ... }
	public async Task<List<CustomerGatewayDto>> Customers(CancellationToken cancellationToken) { ... }
	// ... more methods
}
```

---

## Dependency Injection & Registration

### How DI is Configured

#### 1. App-Level DependencyInjection.cs

```csharp
// StockChecking.Application/DependencyInjection.cs
namespace Microsoft.Extensions.DependencyInjection;

public static class DependencyInjection
{
	public static IServiceCollection AddStockCheckingApplicationServices(
		this IServiceCollection services
	)
	{
		// AutoMapper: For entity → DTO mapping
		services.AddAutoMapper(Assembly.GetExecutingAssembly());

		// FluentValidation: For command/query validation
		services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

		// Custom filters
		services.AddSingleton<ISensitiveClaimsFilter, SensitiveClaimsFilter>();

		return services;
	}
}
```

#### 2. Infrastructure DependencyInjection.cs

```csharp
// StockChecking.Infrastructure/DependencyInjection.cs
namespace Microsoft.Extensions.DependencyInjection;

public static class DependencyInjection
{
	public static IServiceCollection AddStockCheckingInfrastructureServices(
		this IServiceCollection services,
		IConfiguration configuration
	)
	{
		// Gateway Service: Implement the interface
		services.AddScoped<IGatewayService, GatewayService>();

		// Database: DbContext
		services.AddDbContext<ApplicationDbContext>(options =>
			options.UseSqlServer(configuration.GetConnectionString("DefaultConnection"))
		);

		// Repositories
		services.AddScoped<IStockQuoteRepository, StockQuoteRepository>();

		return services;
	}
}
```

#### 3. Program.cs Orchestration

```csharp
// StockChecking.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

// JSON Serialization
builder.Services.Configure<JsonSerializerOptions>(options =>
{
	if (options.Converters.All(c => c is not UriJsonConverter))
	{
		options.Converters.Add(new UriJsonConverter());
	}
});

// Wolverine Messaging
builder.Host.UseWolverine(opts =>
{
	opts.Policies.AddMiddleware(typeof(AuthorizationMiddleware));
	WolverineExtensions.UseMoyoMessaging(opts, serviceBusOptions);  // RabbitMQ
});

// Register all services
builder.Services.AddCommonApplicationServices();        // Common layer
builder.Services.AddStockCheckingApplicationServices(); // App layer
builder.Services.AddStockCheckingInfrastructureServices(builder.Configuration);  // Infra

// Vault/Configuration
var vaultClient = VaultExtensions.ConfigureVault();
builder.Services.AddSingleton(vaultClient);

// Cache
var cachingOptions = VaultExtensions.GetCachingOptions(...);
CachingExtensions.ConfigureFusionCacheWithBackplaneAndRedis(builder.Services, builder.Configuration, cachingOptions);

// Authentication (Azure AD)
builder.Services.AddAuthentication(options =>
{
	options.DefaultAuthenticateScheme = "UIBearer";
	options.DefaultChallengeScheme = "UIBearer";
})
.AddMicrosoftIdentityWebApi(...);

var app = builder.Build();

app.MapWolverineEndpoints();
app.Run();
```

---

## Caching Strategy

### Cache Tiers

Moyo uses **multi-tier caching**:

1. **L1 Cache**: In-process memory (FusionCache)
2. **L2 Cache**: Distributed Redis
3. **Source**: Gateway/Database

### CacheDefaults Configuration

```csharp
// StockChecking.Application/CacheDefaults.cs
public static class CacheDefaults
{
	public const string Tag = "stock-checking";

	// Standard TTL
	public static readonly TimeSpan Duration = TimeSpan.FromMinutes(10);

	// Failsafe TTL (if Redis down)
	public static readonly TimeSpan FailSafeMaxDuration = TimeSpan.FromMinutes(30);
}
```

### Using Cache in Handlers

```csharp
public static async Task<List<ColorDto>> Handle(
	GetColorsQuery _,
	IGatewayService gatewayService,
	IFusionCacheWithKeyTracking cache,  // ← Injected cache
	CancellationToken cancellationToken
)
{
	const string cacheKey = "stock-checking:colors";

	return await cache.GetOrSetAsync(
		cacheKey,                                      // Key
		async ct => await gatewayService.Colors(ct),  // Factory (if miss)
		TimeSpan.FromHours(1),                         // Duration
		cancellationToken
	);
}
```

### Cache Invalidation

**When to clear cache**:
- Admin updates configuration
- Matrix changes in real-time
- Explicit invalidation (e.g., on command)

```csharp
// In command handler after update
public static async Task Handle(
	UpdateColorCommand command,
	IGatewayService gateway,
	IFusionCacheWithKeyTracking cache,
	CancellationToken cancellationToken
)
{
	// ... update logic ...

	// Invalidate related cache entries
	await cache.RemoveAsync("stock-checking:colors", cancellationToken);
	await cache.RemoveByTagAsync("stock-checking", cancellationToken);  // Clear all by tag
}
```

---

## Authorization & Middleware

### Authorization Levels

| Decorator | Effect |
|-----------|--------|
| `[Authorize]` | Any authenticated user |
| `[Authorize(Roles = "Admin")]` | Only Admin role |
| `[Authorize(Roles = "Admin,Manager")]` | Admin OR Manager |
| `[AllowAnonymous]` | No auth required |

**Example**:
```csharp
[Authorize]  // ← Requires authentication
[WolverineGet($"{Routes.Base}/Colors")]
public static Task<List<ColorDto>> GetColors(...) { ... }

[Authorize(Roles = "Admin")]  // ← Requires Admin role
[WolverinePut($"{Routes.Base}/Colors/{id}")]
public static Task UpdateColor(...) { ... }
```

### Authentication Flow

1. **Request** comes in with JWT Bearer token (Azure AD)
2. **AuthenticationMiddleware** validates token
3. **AuthorizationMiddleware** checks [Authorize] attribute
4. **Handler** executes with authenticated user context

### Middleware

**Available Middleware**:
```csharp
// In Program.cs Wolverine config
opts.Policies.AddMiddleware(typeof(Moyo.Common.Application.Middleware.AuthorizationMiddleware));
```

---

## Full Request Flow Example

### Scenario: User calls `GET /api/StockChecking/Colors`

```
1. HTTP Request
   └─> GET /api/StockChecking/Colors
	   Header: Authorization: Bearer [JWT_TOKEN]

2. Authentication Middleware
   └─> Validates JWT token against Azure AD
	   └─> If valid, extracts claims (UserId, Roles, etc.)

3. Routing
   └─> Matches to endpoint in StockChecking.Api
	   └─> GetColors(GetColorsQuery, IMessageBus, CancellationToken)

4. Deserialization
   └─> Query string → GetColorsQuery object

5. Validation (via Wolverine)
   └─> Finds GetColorsQueryValidator
   └─> Runs validation rules
   └─> If invalid: returns 400 Bad Request

6. Authorization (via Wolverine)
   └─> Checks [Authorize] attribute
   └─> If user not authenticated: returns 401 Unauthorized

7. Message Bus Invocation
   └─> messageBus.InvokeAsync<List<ColorDto>>(query, cancellationToken)

8. Handler Discovery
   └─> Wolverine finds GetColorsQueryHandler.Handle()

9. Dependency Injection
   └─> Injects:
	   - IGatewayService (from DI container)
	   - IFusionCacheWithKeyTracking (from DI container)
	   - CancellationToken (from context)

10. Handler Execution
	└─> GetColorsQueryHandler.Handle(...)

	a) Check Cache
	   └─> cache.GetOrSetAsync("stock-checking:colors", ...)

	b) Cache Miss?
	   └─> Call gateway.Colors(cancellationToken)

	c) Gateway Service
	   └─> GatewayService.Colors(...)
	   └─> Calls IMoyoDataGatewayClient.Colors (GraphQL client)
	   └─> HTTP → Amtrack Gateway API

	d) Transform Data
	   └─> ColorGatewayDto → ColorDto (mapping)
	   └─> Order by name

	e) Store in Cache
	   └─> L1: In-process memory (FusionCache)
	   └─> L2: Redis (backplane)
	   └─> TTL: 1 hour

11. Response
	└─> Returns List<ColorDto> to handler
	└─> Wolverine serializes to JSON
	└─> HTTP 200 OK + JSON body
		{
		  "data": [
			{ "code": "RED", "name": "Red", "id": "1", "internalId": "RED" },
			{ "code": "BLUE", "name": "Blue", "id": "2", "internalId": "BLUE" }
		  ]
		}

12. Client receives response
	└─> Parses JSON
	└─> Displays colors in UI
```

---

## Key Patterns & Best Practices

### 1. **Layering Pattern**

✅ **DO**: Keep dependencies flowing downward (API → Application → Infrastructure → Domain)

```
API Layer (Controllers/Endpoints)
	↓
Application Layer (Commands/Queries/Handlers)
	↓
Infrastructure Layer (Services/Repositories)
	↓
Domain Layer (Entities/ValueObjects)
```

❌ **DON'T**: Have Infrastructure depend on API, or Domain depend on anything

### 2. **Gateway Pattern**

✅ **DO**: All external data access through gateway service

```csharp
// Good: Gateway in the middle
Handler → IGatewayService → IMoyoDataGatewayClient → Amtrack API

// Bad: Direct access
Handler → IMoyoDataGatewayClient → Amtrack API
```

### 3. **CQRS Separation**

✅ **DO**: Separate Queries (read) from Commands (write)

```csharp
// Good: Query for reading
public record GetOrdersQuery { }
public record GetOrderByIdQuery(int OrderId) { }

// Good: Command for writing
public record CreateOrderCommand(int CustomerId, List<OrderItemInput> Items) { }
public record UpdateOrderCommand(int OrderId, OrderStatusEnum NewStatus) { }
```

❌ **DON'T**: Mix reads and writes in same handler

### 4. **DTO Mapping**

✅ **DO**: Use AutoMapper or manual mapping to transform DTOs

```csharp
// Good: Gateway DTO → Application DTO
var colorDto = new ColorDto(
	gatewayDto.Code,
	gatewayDto.Name,
	gatewayDto.Id,
	gatewayDto.InternalId
);
```

❌ **DON'T**: Return Gateway DTOs directly to API

### 5. **Caching Strategy**

✅ **DO**: Cache read-heavy queries, clear on updates

```csharp
// Query: Cache result
const string cacheKey = "stock-checking:colors";
return await cache.GetOrSetAsync(cacheKey, async ct => ..., TimeSpan.FromHours(1), ct);

// Command: Invalidate cache
await cache.RemoveByTagAsync("stock-checking", cancellationToken);
```

❌ **DON'T**: Cache mutable data without invalidation

### 6. **Validation at All Layers**

✅ **DO**: Validate in multiple places

```csharp
// API Layer: Input binding
[FromBody] CreateOrderCommand command

// Application Layer: FluentValidator
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand> { ... }

// Domain Layer: Business rules
public Order Create(CreateOrderCommand cmd) 
{
	if (cmd.Items.Count == 0)
		throw new DomainException("Orders must contain items");
}
```

### 7. **Error Handling**

✅ **DO**: Use typed exceptions

```csharp
public class GatewayException : Exception { }
public class EntityNotFoundException : Exception { }
public class DomainException : Exception { }

// Use in handlers
throw new EntityNotFoundException(nameof(Customer), customerId);
```

### 8. **Async/Await Consistency**

✅ **DO**: Use async all the way

```csharp
public static async Task<List<ColorDto>> Handle(
	GetColorsQuery _,
	IGatewayService gatewayService,
	CancellationToken cancellationToken
)
{
	var colors = await gatewayService.Colors(cancellationToken);
	return colors.OrderBy(x => x.Name).ToList();
}
```

---

## Creating a New Endpoint

### Step-by-Step Checklist

#### **Step 1: Define the Query/Command** (Application Layer)

```csharp
// Application/Endpoints/Products/Queries/GetProductDetails/GetProductDetailsQuery.cs
namespace Moyo.Apps.StockChecking.Application.Endpoints.Products.Queries.GetProductDetails;

public record GetProductDetailsQuery(int ProductId);
```

#### **Step 2: Create Validator** (Application Layer)

```csharp
// Application/Endpoints/Products/Queries/GetProductDetails/GetProductDetailsQueryValidator.cs
public class GetProductDetailsQueryValidator : AbstractValidator<GetProductDetailsQuery>
{
	public GetProductDetailsQueryValidator()
	{
		RuleFor(x => x.ProductId).GreaterThan(0).WithMessage("Product ID must be positive");
	}
}
```

#### **Step 3: Create Output DTO** (Application Layer)

```csharp
// Application/Endpoints/Products/Queries/GetProductDetails/ProductDetailsDto.cs
public record ProductDetailsDto(
	int ProductId,
	string Name,
	string Description,
	decimal Price
);
```

#### **Step 4: Create Handler** (Application Layer)

```csharp
// Application/Endpoints/Products/Queries/GetProductDetails/GetProductDetailsQueryHandler.cs
public static class GetProductDetailsQueryHandler
{
	private const string CacheKey = "stock-checking:product-details:{0}";

	public static async Task<ProductDetailsDto> Handle(
		GetProductDetailsQuery query,
		IGatewayService gatewayService,
		IFusionCacheWithKeyTracking cache,
		CancellationToken cancellationToken
	)
	{
		var key = string.Format(CacheKey, query.ProductId);

		return await cache.GetOrSetAsync(
			key,
			async ct =>
			{
				var product = await gatewayService.GetProduct(query.ProductId, ct);
				return new ProductDetailsDto(
					product.Id,
					product.Name,
					product.Description,
					product.Price
				);
			},
			TimeSpan.FromHours(1),
			cancellationToken
		);
	}
}
```

#### **Step 5: Add Gateway Interface Method** (Application Layer)

```csharp
// Application/Interfaces/Gateway/GatewayInterface/Products.cs
public partial interface IGatewayService
{
	Task<ProductGatewayDto> GetProduct(int productId, CancellationToken cancellationToken);
}

// Application/Interfaces/Gateway/Dtos/ProductGatewayDto.cs
public record ProductGatewayDto(
	int Id,
	string Name,
	string Description,
	decimal Price
);
```

#### **Step 6: Implement Gateway Method** (Infrastructure Layer)

```csharp
// Infrastructure/Services/Gateway/Products.cs
public partial class GatewayService : IGatewayService
{
	public async Task<ProductGatewayDto> GetProduct(
		int productId,
		CancellationToken cancellationToken
	)
	{
		var result = await ExecuteWithHeaders(
			DefaultHeaders,
			async () => await client.Product.ExecuteAsync(new { id = productId }, cancellationToken)
		);

		HandleErrors(result);

		var data = result.Data?.Product;
		if (data == null)
			throw new EntityNotFoundException(nameof(Product), productId);

		return new ProductGatewayDto(data.Id, data.Name, data.Description, data.Price);
	}
}
```

#### **Step 7: Create API Endpoint** (API Layer)

```csharp
// StockChecking.Api/Endpoints/Products/GetProductDetails.cs
namespace StockChecking.Api.Endpoints.Products;

public static partial class StockCheckingEndpoints
{
	/// <summary>
	/// Retrieves detailed information about a specific product.
	/// </summary>
	[Authorize]
	[WolverineGet($"{Routes.Base}/Products/{{id}}")]
	public static Task<ProductDetailsDto> GetProductDetails(
		[FromRoute] int id,
		IMessageBus messageBus,
		CancellationToken cancellationToken
	) => messageBus.InvokeAsync<ProductDetailsDto>(
		new GetProductDetailsQuery(id),
		cancellationToken
	);
}
```

#### **Summary of Files Created**

```
Application/
├── Endpoints/Products/Queries/GetProductDetails/
│   ├── GetProductDetailsQuery.cs
│   ├── GetProductDetailsQueryValidator.cs
│   ├── GetProductDetailsQueryHandler.cs
│   └── ProductDetailsDto.cs
├── Interfaces/Gateway/GatewayInterface/
│   └── Products.cs (added method)
└── Interfaces/Gateway/Dtos/
	└── ProductGatewayDto.cs

Infrastructure/
└── Services/Gateway/
	└── Products.cs (added method)

Api/
└── Endpoints/Products/
	└── GetProductDetails.cs
```

---

## Quick Reference: Annotation Guide

### Wolverine Endpoint Decorators

```csharp
// GET: Retrieve data
[WolverineGet($"{Routes.Base}/Products/{id}")]
public static Task<ProductDto> GetProduct(int id, IMessageBus bus, CancellationToken ct)
	=> bus.InvokeAsync<ProductDto>(new GetProductQuery(id), ct);

// POST: Create resource
[WolverinePost($"{Routes.Base}/Products")]
public static async Task<CreatedDto> CreateProduct(
	[FromBody] CreateProductCommand cmd,
	IMessageBus bus,
	CancellationToken ct
) => await bus.InvokeAsync<CreatedDto>(cmd, ct);

// PUT: Replace resource
[WolverinePut($"{Routes.Base}/Products/{id}")]
public static async Task UpdateProduct(
	[FromRoute] int id,
	[FromBody] UpdateProductCommand cmd,
	IMessageBus bus,
	CancellationToken ct
) => await bus.InvokeAsync(cmd, ct);

// DELETE: Remove resource
[WolverineDelete($"{Routes.Base}/Products/{id}")]
public static async Task DeleteProduct(
	[FromRoute] int id,
	IMessageBus bus,
	CancellationToken ct
) => await bus.InvokeAsync(new DeleteProductCommand(id), ct);

// Authorization
[Authorize]                              // Any authenticated user
[Authorize(Roles = "Admin")]             // Specific role
[AllowAnonymous]                         // No auth required
```

### Handler Pattern Recognition

Wolverine finds these patterns automatically:

```csharp
// Pattern 1: Static method in *Handler class
public static class GetColorsQueryHandler
{
	public static async Task<List<ColorDto>> Handle(
		GetColorsQuery query,
		IGatewayService gateway,
		CancellationToken ct
	) { ... }
}

// Pattern 2: Async method
public static async Task Handle(CreateOrderCommand cmd, IRepository repo, CancellationToken ct)
{ ... }

// Pattern 3: IMessageHandler interface
public class SpecialHandler : IMessageHandler<SpecialQuery>
{
	public async Task Handle(SpecialQuery query, ...) { ... }
}

// Pattern 4: MediatR-style
public class MyHandler : IRequestHandler<MyRequest, MyResponse>
{
	public async Task<MyResponse> Handle(MyRequest req, CancellationToken ct) { ... }
}
```

---

## Summary

### App Structure at a Glance

| Layer | Purpose | Key Classes |
|-------|---------|-------------|
| **API** | HTTP Endpoints | Controllers/Endpoints + Decorators |
| **Application** | Business Logic | Commands/Queries + Handlers + Validators |
| **Infrastructure** | Data & Integration | Gateway Services + Repositories + DbContext |
| **Domain** | Business Entities | Entities + ValueObjects + Exceptions |

### Request Flow Simplified

```
HTTP Request
  ↓
Wolverine Routes to Handler
  ↓
Dependencies Injected
  ↓
Validation Runs
  ↓
Authorization Checked
  ↓
Handler Logic (using Gateway, Cache, Repo)
  ↓
Response Returned as JSON
  ↓
HTTP Response
```

### Key Technologies

- **Wolverine**: Messaging & endpoint routing
- **FusionCache**: Multi-tier caching
- **FluentValidation**: Validation rules
- **AutoMapper**: DTO mapping
- **Entity Framework Core**: Database access
- **Azure AD**: Authentication

---

## Next Steps for LeadTimeMatrix Implementation

Using this guide, you're now ready to:

1. ✅ Create **Domain entities** for lead time calculations (BrandingDepartment, QuantityBreak, etc.)
2. ✅ Create **Application layer** with Commands/Queries for lead time operations
3. ✅ Create **Gateway interface** for Amtrack lead time data
4. ✅ Create **Infrastructure** with gateway implementation + database
5. ✅ Create **API endpoints** following Wolverine patterns
6. ✅ Add **caching** for calculation results
7. ✅ Add **authorization** for admin functions

See the **Developer Overview** and **Architect Feedback** documents for LeadTimeMatrix-specific context.

---

**Document Version**: 1.0  
**Based On**: StockChecking & SupplyChain Apps (Real Production Code)  
**Last Reviewed**: 2026  
**Owner**: Development Team
