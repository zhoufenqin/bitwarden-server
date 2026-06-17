# Assessment Overview

This document provides navigation links to all supplementary fact documents generated as part of the application assessment for Bitwarden Server.

## Supplementary Documents

| Document | Description |
|----------|-------------|
| [Architecture Diagram](./architecture-diagram.md) | High-level application architecture diagram and component relationship diagram for the Bitwarden Server multi-service .NET 10 backend, including technology stack summary and key architectural decisions. |
| [Dependency Map](./dependency-map.md) | Visual map of all external NuGet package dependencies grouped by functional category (Web Frameworks, Identity & Security, Database/ORM, Messaging, Caching, Cloud Storage, Email, Payments, Scheduling, Utilities), with version and compatibility risk analysis. |
| [API & Service Communication Contracts](./api-service-contracts.md) | Inventory of all REST API endpoints across the eight independently deployable services (Api, Identity, Admin, Billing, Notifications, Events, EventsProcessor, Icons), communication patterns, DTOs, and a service communication sequence diagram. |
| [Data Architecture & Persistence Layer](./data-architecture.md) | Documentation of the EF Core + Dapper dual data access layer, entity model ER diagram, repository interfaces, caching strategy, and data classification and sensitivity analysis for PII/PCI data. |
| [Configuration & Externalized Settings Inventory](./configuration-inventory.md) | Comprehensive inventory of all configuration sources, runtime profiles (Development, Production, QA, SelfHosted), secrets management workflow, feature flags (LaunchDarkly), and framework/runtime version catalog. |
| [Core Business Workflows](./business-workflows.md) | End-to-end documentation of the primary business workflows (user registration, authentication with MFA, vault sync, cipher save and share, organization onboarding, passwordless login, policy enforcement) with a Mermaid sequence diagram. |
