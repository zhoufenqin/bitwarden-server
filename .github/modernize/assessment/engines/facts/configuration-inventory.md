# Configuration & Externalized Settings Inventory

Bitwarden Server uses a layered ASP.NET Core configuration system with approximately 5 config files per service (base + environment overrides), secrets stored as environment variables or `secrets.json` for local dev, and LaunchDarkly for remote feature flag evaluation.

## Configuration Sources

| Source | Type | Path / Location | Notes |
|--------|------|----------------|-------|
| `appsettings.json` | JSON config (base) | `src/{Service}/appsettings.json` | Base defaults; all secret values set to `"SECRET"` placeholder |
| `appsettings.Development.json` | JSON config (env override) | `src/{Service}/appsettings.Development.json` | Local dev overrides; uses Azurite for storage |
| `appsettings.Production.json` | JSON config (env override) | `src/{Service}/appsettings.Production.json` | Production service URIs, Braintree/BitPay production flags |
| `appsettings.QA.json` | JSON config (env override) | `src/{Service}/appsettings.QA.json` | QA environment service URIs |
| `appsettings.SelfHosted.json` | JSON config (env override) | `src/{Service}/appsettings.SelfHosted.json` | Self-hosted overrides; all `baseServiceUri` set to null (resolved at runtime) |
| `secrets.json` (User Secrets) | ASP.NET User Secrets | `~/.microsoft/usersecrets/{UserSecretsId}/secrets.json` | Local secrets (DB passwords, API keys); not committed to source |
| `dev/.env` | Docker Compose env file | `dev/.env` | Database passwords, compose project name; sourced from `.env.example` |
| `dev/secrets.json.example` | Example secrets file | `dev/secrets.json.example` | Template for populating User Secrets; not committed |
| `dev/docker-compose.yml` | Docker Compose | `dev/docker-compose.yml` | Local dev infrastructure (SQL Server, PostgreSQL, MySQL, Azurite, Mailcatcher, Redis, RabbitMQ) |
| `flags.json` | LaunchDarkly file source | `flags.json` (per service, optional) | Used as fallback when `launchDarkly__sdkKey` is absent (local dev); JSON file with flag overrides |
| Environment variables | Runtime env vars | Injected at deploy time | Override any `globalSettings__*` key via double-underscore path notation |

> Note: A setup script `dev/setup_secrets.ps1` / `dev/setup_azurite.ps1` automates the initialization of local User Secrets and Azurite storage for developers.

## Build Profiles

| Profile / Configuration | Activation | Purpose | Key Changes |
|------------------------|-----------|---------|-------------|
| `Debug` | Default in VS / `dotnet build` | Local development; debugger attachable | No optimizations; PDB files generated |
| `Release` | `-c Release` / CI pipeline | Production-ready build | Optimizations enabled; no PDB emitted in output |
| `OSS` (DefineConstant) | `-p:DefineConstants=OSS` | Open-source build without commercial license features | Excludes `Commercial.Core`, `Commercial.Infrastructure.EntityFramework` project references in Api |
| Docker multi-stage | Dockerfile `AS build` stage | Build and publish inside Alpine SDK container | `dotnet publish -c Release --no-restore --runtime linux-musl-{arch}` |

The `Bitwarden.Server.Sdk` MSBuild SDK (version `1.5.2`, defined in `global.json`) provides shared build targets, including `BitIncludeAuthentication` and `BitIncludeFeatures` properties used to opt services into shared authentication and feature-flag registration.

## Runtime Profiles

| Profile | Activation Method | Config Files Applied | Key Overrides |
|---------|------------------|---------------------|---------------|
| `Development` | `ASPNETCORE_ENVIRONMENT=Development` | `appsettings.json` + `appsettings.Development.json` + User Secrets | Local service URIs; Azurite for storage; SMTP on localhost:10250; SQLite or SQL Server |
| `Production` | `ASPNETCORE_ENVIRONMENT=Production` | `appsettings.json` + `appsettings.Production.json` | Production URIs (`*.bitwarden.com`); Braintree/BitPay production mode; reduced logging level |
| `QA` | `ASPNETCORE_ENVIRONMENT=QA` | `appsettings.json` + `appsettings.QA.json` | QA environment URIs (`*.qa.bitwarden.pw`) |
| `SelfHosted` | `globalSettings__selfHosted=true` + optional `ASPNETCORE_ENVIRONMENT=SelfHosted` | `appsettings.json` + `appsettings.SelfHosted.json` | All `baseServiceUri` entries set to `null`; resolved by Nginx reverse proxy at runtime |

Note: `globalSettings__selfHosted=true` (set via environment variable) toggles self-hosted behavior throughout the application independently of `ASPNETCORE_ENVIRONMENT`, enabling a self-hosted Production deployment.

## Properties Inventory

### Global (shared across all services via `GlobalSettings`)

| Property Key | Default | Profile Override | Source |
|-------------|---------|-----------------|--------|
| `globalSettings__selfHosted` | `false` | SelfHosted: `true` | `appsettings.json` / env var |
| `globalSettings__siteName` | `"Bitwarden"` | — | `appsettings.json` |
| `globalSettings__projectName` | `"{ServiceName}"` | — | Per-service `appsettings.json` |
| `globalSettings__databaseProvider` | `"sqlServer"` | Env var for alt DBs | Env var (`globalSettings__databaseProvider`) |
| `globalSettings__sqlServer__connectionString` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__postgreSql__connectionString` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__mySql__connectionString` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__sqlite__connectionString` | `"Data Source=:memory:"` | — | `GlobalSettings.cs` default |
| `globalSettings__storage__connectionString` | `"SECRET"` / `"UseDevelopmentStorage=true"` | Dev: Azurite | User Secrets / env var |
| `globalSettings__attachment__connectionString` | `"SECRET"` / `"UseDevelopmentStorage=true"` | Dev: Azurite | User Secrets / env var |
| `globalSettings__events__connectionString` | `"SECRET"` / `"UseDevelopmentStorage=true"` | Dev: Azurite | User Secrets / env var |
| `globalSettings__send__connectionString` | `"SECRET"` / `"UseDevelopmentStorage=true"` | Dev: Azurite | User Secrets / env var |
| `globalSettings__serviceBus__connectionString` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__serviceBus__applicationCacheTopicName` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__mail__sendGridApiKey` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__mail__replyToEmail` | `"no-reply@bitwarden.com"` | — | `appsettings.json` |
| `globalSettings__mail__smtp__host` | — | Dev: `"localhost"` | `appsettings.Development.json` |
| `globalSettings__mail__smtp__port` | — | Dev: `10250` | `appsettings.Development.json` |
| `globalSettings__identityServer__certificateThumbprint` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__dataProtection__certificateThumbprint` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__stripe__apiKey` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__braintree__merchantId` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__braintree__production` | `false` | Production: `true` | `appsettings.Production.json` |
| `globalSettings__bitPay__production` | `false` | Production: `true` | `appsettings.Production.json` |
| `globalSettings__duo__aKey` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__yubico__clientid` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__amazon__accessKeyId` | `"SECRET"` | — | User Secrets / env var |
| `globalSettings__launchDarkly__sdkKey` | `""` (empty) | — | User Secrets / env var; absent → reads `flags.json` |
| `globalSettings__launchDarkly__flagDataFilePath` | `"flags.json"` | — | `GlobalSettings.cs` default |
| `globalSettings__distributedIpRateLimiting__enabled` | `true` | — | `appsettings.json` |
| `globalSettings__distributedIpRateLimiting__maxRedisTimeoutsThreshold` | `10` | — | `appsettings.json` |
| `globalSettings__distributedIpRateLimiting__slidingWindowSeconds` | `120` | — | `appsettings.json` |
| `globalSettings__disableUserRegistration` | `false` | — | `GlobalSettings.cs` default |
| `globalSettings__enableNewDeviceVerification` | `false` | — | `GlobalSettings.cs` default |
| `globalSettings__organizationInviteExpirationHours` | `120` (5 days) | — | `GlobalSettings.cs` default |
| `globalSettings__deviceLastActivityCacheTtlHours` | `120` | — | `GlobalSettings.cs` default |
| `globalSettings__pricingUri` | — | Dev: QA pricing URL | `appsettings.Development.json` |

### Service URI Configuration

| Property Key | Development | Production |
|-------------|------------|-----------|
| `globalSettings__baseServiceUri__vault` | `https://localhost:8080` | `https://vault.bitwarden.com` |
| `globalSettings__baseServiceUri__api` | `http://localhost:4000` | `https://api.bitwarden.com` |
| `globalSettings__baseServiceUri__identity` | `http://localhost:33656` | `https://identity.bitwarden.com` |
| `globalSettings__baseServiceUri__admin` | `http://localhost:62911` | `https://admin.bitwarden.com` |
| `globalSettings__baseServiceUri__notifications` | `http://localhost:61840` | `https://notifications.bitwarden.com` |
| `globalSettings__baseServiceUri__sso` | `http://localhost:51822` | `https://sso.bitwarden.com` |

### Rate Limiting (Api + Identity)

| Property Key | Default |
|-------------|---------|
| `IpRateLimitOptions__EnableEndpointRateLimiting` | `true` |
| `IpRateLimitOptions__HttpStatusCode` | `429` |
| `IpRateLimitOptions__RealIpHeader` | `"X-Connecting-IP"` |
| `IpRateLimitOptions__GeneralRules__POST` | 60 req/min, 5 req/s |
| `IpRateLimitOptions__GeneralRules__GET` | 200 req/min |
| `IpRateLimitOptions__GeneralRules__POST:/connect/token` | 10 req/min (Identity) |

## Startup Parameters & Resource Requirements

| Service | Runtime Options | Notes |
|---------|----------------|-------|
| All services | `ASPNETCORE_ENVIRONMENT={Development\|Production\|QA}` | Controls config file layering |
| All services | `globalSettings__*` (env vars) | Any GlobalSettings property overridable via env var (double-underscore path) |
| Api / Identity | `globalSettings__databaseProvider=postgreSql\|mySql\|sqlite` | Selects EF Core database provider at startup |
| Docker build | Alpine SDK `10.0-alpine3.23` → Alpine ASPNet `10.0-alpine3.23` | Multi-stage; final image is Alpine-based for minimal footprint |

No explicit JVM-equivalent heap settings are required. Container memory/CPU limits are not defined in the source repository's Docker Compose files — they are expected to be set by the deployment orchestration (Kubernetes, Azure Container Apps, etc.).

## Startup Dependency Chain

The services have no formal `depends_on` health-check chain in Docker Compose beyond the database containers. In production:

1. **Database** (SQL Server / PostgreSQL) must be available before any service starts — connection errors on startup will cause the service to crash. The `dbup-sqlserver` migrator (`MsSqlMigratorUtility`) is run as a migration job before any application services start.
2. **Redis** must be available if `DistributedIpRateLimiting.Enabled=true` (Api) or SignalR backplane is configured (Notifications). Services fall back to in-memory rate limiting if Redis is unavailable at startup.
3. **Azure Service Bus / SQS** must be accessible before `IApplicationCacheService` can receive invalidation messages. Absence causes in-memory caching to be used without cross-instance invalidation.
4. **Identity service** must be reachable by the Api, Admin, Billing, and Notifications services at first authenticated request (JWT validation uses the Identity's OIDC discovery endpoint at startup).
5. **EventsProcessor** depends on the events queue (Azure Service Bus or SQS) being available; it will retry indefinitely if the queue is temporarily unavailable.

Local dev setup uses `dev/setup_secrets.ps1` and `dev/setup_azurite.ps1` to initialize prerequisites before running services.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage |
|-----------------|------|---------|
| `globalSettings__sqlServer__connectionString` | DB connection string (SQL Server) | User Secrets / env var [MASKED] |
| `globalSettings__postgreSql__connectionString` | DB connection string (PostgreSQL) | User Secrets / env var [MASKED] |
| `globalSettings__mySql__connectionString` | DB connection string (MySQL) | User Secrets / env var [MASKED] |
| `globalSettings__storage__connectionString` | Azure Storage connection string | User Secrets / env var [MASKED] |
| `globalSettings__serviceBus__connectionString` | Azure Service Bus connection string | User Secrets / env var [MASKED] |
| `globalSettings__stripe__apiKey` | Stripe API secret key | User Secrets / env var [MASKED] |
| `globalSettings__braintree__privateKey` | Braintree private key | User Secrets / env var [MASKED] |
| `globalSettings__braintree__merchantId` | Braintree merchant ID | User Secrets / env var [MASKED] |
| `globalSettings__amazon__accessKeyId` | AWS access key ID | User Secrets / env var [MASKED] |
| `globalSettings__amazon__accessKeySecret` | AWS secret access key | User Secrets / env var [MASKED] |
| `globalSettings__mail__sendGridApiKey` | SendGrid API key | User Secrets / env var [MASKED] |
| `globalSettings__identityServer__certificateThumbprint` | Identity signing cert thumbprint | User Secrets / env var [MASKED] |
| `globalSettings__dataProtection__certificateThumbprint` | Data Protection cert thumbprint | User Secrets / env var [MASKED] |
| `globalSettings__launchDarkly__sdkKey` | LaunchDarkly server SDK key | User Secrets / env var [MASKED] |
| `globalSettings__duo__aKey` | Duo integration secret key | User Secrets / env var [MASKED] |
| `globalSettings__yubico__clientid` / `key` | YubiKey validation API credentials | User Secrets / env var [MASKED] |
| `globalSettings__notificationHub__connectionString` | Azure Notification Hub connection string | User Secrets / env var [MASKED] |
| `globalSettings__bitPay__token` / `webhookKey` | BitPay API token and webhook key | User Secrets / env var [MASKED] |
| `globalSettings__internalIdentityKey` | Internal identity shared key | User Secrets / env var [MASKED] |
| `globalSettings__oidcIdentityClientKey` | OIDC identity client key | User Secrets / env var [MASKED] |
| User master key / encryption keys in DB | Encrypted user keys | Database column; encrypted via ASP.NET Core Data Protection (`DataProtectionConverter`) [MASKED] |

### Secrets Provisioning Workflow

**Local Development**:
1. Developer copies `dev/.env.example` → `dev/.env` and sets database passwords.
2. Runs `dev/setup_secrets.ps1` which populates `~/.microsoft/usersecrets/{UserSecretsId}/secrets.json` with all required `globalSettings__*` keys (connection strings, API keys, certificates).
3. Runs `dev/setup_azurite.ps1` to create Azurite containers and storage accounts for local blob/queue/table emulation.
4. ASP.NET Core User Secrets are loaded automatically in `Development` environment.

**Production / Cloud Deployment**:
Environment variables are injected at container startup by the deployment platform (Azure Container Apps, Kubernetes, or similar). Secrets are managed through the platform's secret store (Azure Key Vault via managed identity, or Kubernetes Secrets). No secret values are stored in the repository; all `appsettings.json` entries for secrets contain only the placeholder `"SECRET"`.

**Certificate Management**:
TLS certificates for `IdentityServer` signing and ASP.NET Core Data Protection are referenced by certificate thumbprint (`identityServer.certificateThumbprint`, `dataProtection.certificateThumbprint`). Certificates are expected to be pre-installed in the OS certificate store on each server/container at deployment time.

## Feature Flags

| Flag Name / Key | Default | Controlled By | Notes |
|----------------|---------|--------------|-------|
| LaunchDarkly remote flags | Varies | LaunchDarkly SaaS | All dynamic feature flags evaluated via `IFeatureService.IsEnabled(key)` |
| `launchDarkly__sdkKey` absent | File-based fallback | `flags.json` file per service | Used in development/test when no LD SDK key is configured; flag values read from local JSON file |
| `globalSettings__selfHosted` | `false` | Environment variable / config | Enables self-hosted mode; disables cloud-only features |
| `globalSettings__disableUserRegistration` | `false` | Environment variable / config | Prevents new user registration |
| `globalSettings__enableNewDeviceVerification` | `false` | Environment variable / config | Enables email verification for new device logins |
| `globalSettings__enableCloudCommunication` | `false` | Environment variable / config | Enables phone-home features in self-hosted deployments |
| `globalSettings__suppressOnboardingInterstitials` | `false` | Environment variable / config | Suppresses onboarding UI prompts |
| `globalSettings__testPlayIdTrackingEnabled` | `false` | Environment variable / config | Internal A/B test tracking |
| `BitIncludeAuthentication` (MSBuild) | `false` | `Bitwarden.Server.Sdk` MSBuild SDK | Build-time; opts service into shared authentication DI registration |
| `BitIncludeFeatures` (MSBuild) | `false` | `Bitwarden.Server.Sdk` MSBuild SDK | Build-time; opts service into feature-flag DI registration |
| `OSS` (DefineConstant) | Not set | Build: `-p:DefineConstants=OSS` | Excludes commercial project references at compile time |

## Framework & Runtime Versions

| Component | Version | Source |
|-----------|---------|--------|
| .NET Runtime | 10.0 | `Directory.Build.props` `<TargetFramework>net10.0</TargetFramework>` |
| .NET SDK | 10.0.103 (rollForward: latestFeature) | `global.json` |
| ASP.NET Core | 10.0 (included with .NET 10) | Runtime |
| Entity Framework Core | 8.0.8 | `Infrastructure.EntityFramework.csproj` |
| EF Core Tools (CLI) | 8.0.8 | `.config/dotnet-tools.json` |
| Duende IdentityServer | 7.4.6 | `Core.csproj` |
| SignalR | 10.0.8 | `Notifications.csproj` |
| Dapper | 2.1.66 | `Infrastructure.Dapper.csproj` |
| Quartz.NET | 3.15.1 | `Core.csproj` |
| LaunchDarkly Server SDK | 8.11.0 | `Core.csproj` |
| Swashbuckle.AspNetCore | 10.1.7 | `Api.csproj`, `Billing.csproj` |
| Swashbuckle CLI (dotnet tool) | 10.1.7 | `.config/dotnet-tools.json` |
| Bitwarden.Server.Sdk (MSBuild) | 1.5.2 | `global.json` msbuild-sdks |
| Aspire.AppHost.Sdk | 13.3.4 | `global.json` msbuild-sdks |
| .NET Aspire (AppHost) | 13.3.4 | `AppHost.csproj` |
| Docker base image (build) | `mcr.microsoft.com/dotnet/sdk:10.0-alpine3.23` | `src/*/Dockerfile` |
| Docker base image (runtime) | `mcr.microsoft.com/dotnet/aspnet:10.0-alpine3.23` | `src/*/Dockerfile` |
| xUnit | 2.6.6 | `Directory.Build.props` |
| Microsoft.NET.Test.Sdk | 18.0.1 | `Directory.Build.props` |
