# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Solutions, SDK, and Packages

- Two solution files: `eShopOnWeb.sln` (7 core projects — default) and `Everything.sln` (adds the Aspire host/service-defaults projects).
- `global.json` pins the .NET SDK to `10.0.0` with `rollForward: latestFeature`. All projects target `net10.0`. The repo README's "ASP.NET Core 9.0" note is stale.
- NuGet versions are centrally managed in `Directory.Packages.props`. Add/upgrade packages there; individual `.csproj` files declare `<PackageReference>` without `Version=`.

## Build and Test Commands

```bash
# Build entire solution
dotnet build ./eShopOnWeb.sln

# Run all tests with coverage (repo includes a runsettings file)
dotnet test ./eShopOnWeb.sln --settings CodeCoverage.runsettings

# Run specific test project
dotnet test tests/UnitTests/UnitTests.csproj
dotnet test tests/IntegrationTests/IntegrationTests.csproj
dotnet test tests/FunctionalTests/FunctionalTests.csproj

# Run single test by name
dotnet test --filter "FullyQualifiedName~TestMethodName"
```

## Running Locally

Two options. Pick one — do not mix.

**Option A: Two-terminal (core solution).** The Web app requires PublicApi running simultaneously for the Blazor Admin panel:

```bash
# Terminal 1: Start the API
dotnet run --project src/PublicApi/PublicApi.csproj

# Terminal 2: Start the Web app
dotnet run --project src/Web/Web.csproj --launch-profile https
```

Browse to `https://localhost:5001/` (store) and `https://localhost:5001/admin` (Blazor admin).

**Option B: .NET Aspire host.** `src/eShopWeb.AppHost/` orchestrates Web + PublicApi in a single process and provides the Aspire dashboard. `src/eShopWeb.AspireServiceDefaults/` adds OpenTelemetry and health check defaults to participating services. Requires the Aspire workload and `Everything.sln`:

```bash
dotnet run --project src/eShopWeb.AppHost/eShopWeb.AppHost.csproj
```

## Database Setup

Uses two separate databases (CatalogContext for catalog/cart, AppIdentityDbContext for identity). From src/Web folder:

```bash
dotnet ef database update -c catalogcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
dotnet ef database update -c appidentitydbcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
```

For in-memory database, set `"UseOnlyInMemoryDatabase": true` in appsettings.json.

## Architecture

This is a Clean Architecture monolithic application with ASP.NET Core 10.

**Layer Dependencies (inward only):**
- **ApplicationCore**: Domain entities, interfaces, specifications, domain services. No external dependencies.
- **Infrastructure**: EF Core implementations, data access, identity. Depends on ApplicationCore.
- **Web/PublicApi/BlazorAdmin**: Presentation layer. Depends on Infrastructure and ApplicationCore.

**Key Patterns:**
- **Repository Pattern**: `EfRepository<T>` wraps Entity Framework
- **Specification Pattern**: Ardalis.Specification for query encapsulation (see `src/ApplicationCore/Specifications/`)
- **MediatR**: Command/query handlers in `src/Web/Features/`
- **FastEndpoints**: REST API organization in `src/PublicApi/*Endpoints/`
- **Aggregate Pattern**: Domain aggregates in `src/ApplicationCore/Entities/` (BasketAggregate, OrderAggregate, BuyerAggregate)

**Project Structure:**
- `src/ApplicationCore/` - Business logic, domain entities, interfaces
- `src/Infrastructure/` - EF Core, data access, identity implementation
- `src/Web/` - MVC + Razor Pages main application
- `src/PublicApi/` - REST API using FastEndpoints
- `src/BlazorAdmin/` - Blazor WebAssembly admin UI
- `src/BlazorShared/` - Shared Blazor components
- `tests/` - UnitTests, IntegrationTests, FunctionalTests, PublicApiIntegrationTests

**DI Configuration:**
- `src/Web/Configuration/ConfigureCoreServices.cs` - Repositories and domain services
- `src/Web/Configuration/ConfigureWebServices.cs` - MediatR and view model services

## Test Accounts

Default seeded user: `demouser@microsoft.com` (password in seed data). Admin user: `admin@microsoft.com`.

## Docker

```bash
docker-compose build
docker-compose up
```

Web: localhost:5106, API: localhost:5200
