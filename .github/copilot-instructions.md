# GitHub Copilot Agent Instructions for eShopOnWeb

## Repository Overview

This is the **Microsoft eShopOnWeb ASP.NET Core Reference Application** - a sample e-commerce application demonstrating monolithic architecture and deployment patterns. The application showcases modern .NET development practices, Entity Framework Core, Blazor WebAssembly, and Azure integration capabilities.

**Key Technologies:**
- ASP.NET Core 6.0 (target framework)
- Entity Framework Core with in-memory database (default) or SQL Server
- Blazor WebAssembly for admin interface
- Docker and Docker Compose support
- Azure integration (App Configuration, Identity, etc.)
- MediatR pattern implementation
- AutoMapper for object mapping
- xUnit for testing

**Repository Size:** ~40 projects including source code, tests, and deployment configurations
**Primary Languages:** C# (.NET 6.0), HTML/CSS/JavaScript, YAML (pipelines), Dockerfile

## Essential Build Instructions

### Prerequisites
**CRITICAL:** Always ensure these tools are available before building:
- .NET 6.0 SDK (required - project targets net6.0 specifically)
- LibMan CLI tool: `dotnet tool install --global Microsoft.Web.LibraryManager.Cli`
- Docker and Docker Compose (for containerized builds)

### Build Process

**1. Restore Dependencies (REQUIRED FIRST STEP)**
```bash
# Always specify the solution file - multiple project files exist in root
dotnet restore eShopOnWeb.sln
```

**2. Handle LibMan Library Dependencies**
```bash
# LibMan often fails due to network restrictions accessing cdnjs
cd src/Web
libman restore

# If libman fails with LIB002 errors, build will fail
# Workaround: Temporarily rename libman.json to bypass during builds
# mv libman.json libman.json.bak  # To disable
# mv libman.json.bak libman.json  # To re-enable
```

**3. Build Solution**
```bash
# Build from solution root with explicit solution file
dotnet build eShopOnWeb.sln

# For quieter output with security warnings:
dotnet build eShopOnWeb.sln --verbosity minimal
```

**Expected Build Warnings (NORMAL):**
- NU1902/NU1903: Security vulnerabilities in Azure.Identity, System.IdentityModel.Tokens.Jwt, System.Text.Json
- CS8625/CS8603: Nullable reference type warnings
- CS0612: Obsolete method warnings for IReadRepositoryBase methods
- LibMan warnings about missing libman.json (if temporarily disabled)

**Build Time:** Typical clean build takes 10-15 seconds, initial restore may take 60+ seconds.

**Important:** Build produces ~144 warnings which are NORMAL and expected. These include:
- Security vulnerability warnings in NuGet packages (acceptable for reference application)
- Nullable reference type warnings (CS8602, CS8603, CS8618, CS8625)
- Obsolete API warnings (CS0612 for IReadRepositoryBase methods)
- LibMan configuration warnings

### Testing

**CRITICAL .NET Version Issue:**
Tests require .NET 6.0 runtime but may fail if only .NET 8.0+ is available:

```bash
# Run unit tests (may fail with .NET version mismatch)
dotnet test tests/UnitTests/UnitTests.csproj

# Run all tests
dotnet test eShopOnWeb.sln
```

If tests fail with ".NET 6.0.0 not found" errors, this is a runtime compatibility issue, not a code problem.

### Docker Build (Recommended Alternative)

**Docker builds are more reliable** when .NET version issues occur, but may be slow due to network dependencies:

```bash
# Use 'docker compose' (not 'docker-compose')
docker compose build
docker compose up

# Access endpoints:
# Web app: http://localhost:5106
# Public API: http://localhost:5200
```

**Note:** Docker builds can take 10+ minutes on first run due to downloading .NET 6.0 base images and NuGet package restoration. Network issues may cause timeouts.

### Database Configuration

**Default:** Uses in-memory database (no setup required)

**To use SQL Server:**
1. Modify `src/Infrastructure/Dependencies.cs` line 13: `var useOnlyInMemoryDatabase = false;`
2. Update connection strings in `src/Web/appsettings.json`
3. Run Entity Framework migrations:
```bash
cd src/Web
dotnet ef database update -c catalogcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
dotnet ef database update -c appidentitydbcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
```

## Project Architecture

### Source Structure (`/src`)
- **ApplicationCore/** - Domain entities, interfaces, services, specifications
- **Infrastructure/** - Data access, EF contexts, repositories, external services
- **Web/** - Main MVC application, Razor pages, controllers, views
- **PublicApi/** - Web API for external consumption and Blazor admin
- **BlazorAdmin/** - Blazor WebAssembly admin interface  
- **BlazorShared/** - Shared Blazor components and models

### Test Structure (`/tests`)
- **UnitTests/** - Domain and service layer unit tests
- **IntegrationTests/** - Database and repository integration tests
- **FunctionalTests/** - End-to-end web application tests
- **PublicApiIntegrationTests/** - API endpoint tests

### Key Configuration Files
- **eShopOnWeb.sln** - Main solution file (ALWAYS specify this for builds)
- **global.json** - .NET SDK version requirements (6.0.x)
- **docker-compose.yml** - Multi-container orchestration
- **src/Web/libman.json** - Client-side library dependencies (often fails)
- **src/Web/appsettings.json** - Application configuration, connection strings
- **src/Infrastructure/Dependencies.cs** - Database configuration (in-memory vs SQL)

## Deployment and CI/CD

### GitHub Actions (`.github/workflows/`)
- **eshoponweb-cicd.yml** - Main CI/CD pipeline with Azure deployment
- Includes build, test, and Azure Web App deployment steps

### Azure DevOps Pipelines (`.ado/`)
- **eshoponweb-ci.yml** - Primary build pipeline
- **eshoponweb-cd-webapp-code.yml** - Web app deployment
- **eshoponweb-ci-docker.yml** - Docker container builds
- Multiple deployment variants (ACI, App Service, etc.)

## Common Issues and Solutions

### 1. LibMan Build Failures
**Symptoms:** `LIB002: library could not be resolved by "cdnjs" provider`
**Solution:** Temporarily disable libman by renaming `src/Web/libman.json` to `libman.json.bak`

### 2. .NET Version Mismatch
**Symptoms:** Test failures with "You must install .NET 6.0" errors
**Solution:** Use Docker builds or install .NET 6.0 SDK alongside newer versions

### 3. Multiple Project Files in Root
**Symptoms:** "Specify which project or solution file to use"
**Solution:** Always use `eShopOnWeb.sln` explicitly in dotnet commands

### 4. Missing Entity Framework Tools
**Solution:** Install globally: `dotnet tool install --global dotnet-ef`

### 5. Database Connection Issues
**Check:** `UseOnlyInMemoryDatabase` setting in `Dependencies.cs` - should be `true` for development

### 6. Network Connectivity Issues
**Symptoms:** Docker builds timeout, libman restore fails, NuGet restore errors
**Solution:** Network restrictions may prevent package downloads; use offline approaches or retry builds

## Validation Pipeline Replication

To replicate CI/CD checks locally:

```bash
# 1. Restore and build (matches pipeline)
dotnet restore eShopOnWeb.sln
dotnet build eShopOnWeb.sln --configuration Release

# 2. Run tests (may require .NET 6.0)
dotnet test tests/UnitTests/UnitTests.csproj --configuration Release

# 3. Publish web application
dotnet publish src/Web/Web.csproj -c Release -o ./publish

# 4. Docker validation
docker compose build
docker compose up -d
# Test endpoints: curl http://localhost:5106/health
docker compose down
```

## Trust These Instructions

**These instructions are based on comprehensive repository analysis and testing.** Only search for additional information if:
- You encounter errors not documented here
- You need to understand business logic or specific feature implementation  
- The documented workarounds fail to resolve build issues
- You're implementing new features requiring architectural understanding

**Remember:** This is a reference application - focus on maintaining architectural patterns and compatibility rather than fixing all security warnings or deprecated method usage.