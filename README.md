# AspJs — Enterprise-Grade ASP.NET Core 9 REST API

> **GitHub Repository Description** *(copy this into the repo's "About" field)*  
> `Production-ready multi-tenant REST API — ASP.NET Core 9, FastEndpoints, EF Core (MySQL), JWT (3 token types), MessagePack, Serilog, 5 rate-limit strategies, IP firewall & a Vanilla JS SPA. No Node.js required.`

---

> **Full-stack backend system** built with C# / ASP.NET Core 9, designed for multi-tenant SaaS products.  
> This repository contains the **technical overview only**. Source code is available on request for serious employers and freelance clients.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack at a Glance](#2-tech-stack-at-a-glance)
3. [Architecture](#3-architecture)
4. [Security Layer](#4-security-layer)
5. [Performance & Scalability](#5-performance--scalability)
6. [Database Design](#6-database-design)
7. [Logging & Observability](#7-logging--observability)
8. [Cloud & File Management](#8-cloud--file-management)
9. [Frontend (wwwroot SPA)](#9-frontend-wwwroot-spa)
10. [Background Services](#10-background-services)
11. [API Documentation](#11-api-documentation)
12. [Key Design Patterns](#12-key-design-patterns)
13. [Hire Me / Freelance](#13-hire-me--freelance)

---

## 1. Project Overview

AspJs is a **production-ready, multi-tenant REST API platform** built on ASP.NET Core 9. It powers a full business workflow including:

- User & role management with fine-grained endpoint-level permissions
- Inquiry / support-ticket system with threaded replies
- Shared-link access (time-limited, role-scoped public URLs)
- Cloud file upload / download lifecycle management
- Real-time traffic monitoring and audit trail
- Automated database migrations and scheduled maintenance

The frontend is a **zero-framework Vanilla JS SPA** served from `wwwroot`, making the entire product a single self-contained binary with no Node.js runtime dependency.

---

## 2. Tech Stack at a Glance

| Layer | Technology |
|---|---|
| Runtime | .NET 9 / C# 13 |
| Web framework | ASP.NET Core 9 + **FastEndpoints** (REPR pattern) |
| ORM | Entity Framework Core 9 (MySQL, `DbContextPool`) |
| Embedded DB | **LiteDB** (Singleton, shared-connection mode) |
| Auth | JWT Bearer — access token + refresh token + shared token |
| Serialization | `System.Text.Json` + **MessagePack** (dual content-type) |
| Logging | **Serilog** → SQLite sink (batched, async) |
| API Docs | NSwag / OpenAPI 3 via FastEndpoints Swagger integration |
| Device detection | DeviceDetectorNET |
| IP geolocation | IP3Country |
| Background jobs | `IHostedService` / `BackgroundService` |
| Rate limiting | ASP.NET Core built-in `RateLimiter` (5 strategies) |
| Frontend | Vanilla JS, HTML5, CSS3 (no bundler, no framework) |

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Frontend (wwwroot SPA)              │
│            Vanilla JS · Components · Pages           │
└───────────────────────┬─────────────────────────────┘
                        │  HTTP / WebSocket
┌───────────────────────▼─────────────────────────────┐
│               Middleware Pipeline                    │
│   FirewallMiddleware → LoggingMiddleware →           │
│   Authentication → Authorization →                  │
│   RateLimiter → FastEndpoints Router                 │
└───────────────────────┬─────────────────────────────┘
                        │
        ┌───────────────┼──────────────────┐
        ▼               ▼                  ▼
   Endpoints        Services           Helpers
  (REPR pattern)  SessionService     QueryHelper
  CrudEndpoint    MaintenanceSvc     CloudManager
  UserEndpoint    CommonService      EmailManager
  SharedLink…     MigrationSvc       ConstsHelper
        │               │                  │
        └───────────────┼──────────────────┘
                        ▼
          ┌─────────────────────────┐
          │   Data Layer            │
          │  EF Core (MySQL)        │
          │  LiteDB (embedded)      │
          │  SQLite (Serilog logs)  │
          └─────────────────────────┘
```

**Two separate C# projects:**
- `AspJs/` — application layer (endpoints, middleware, services, helpers)
- `Database/` — data contract layer (entities, custom attributes, migration helpers)

---

## 4. Security Layer

### 4.1 JWT Authentication — Three Token Types

| Token | Purpose | Storage | Validation |
|---|---|---|---|
| Access Token | Short-lived API access | HttpOnly Cookie | Session cache lookup + signature |
| Refresh Token | Renew access token | DB (`Sessions` table) | DB record match |
| Shared Token | Public read-only link access | HttpOnly Cookie | Signature only (no session) |

Token selection is **context-aware**: the middleware inspects the `Referer` header to decide which cookie to read, so a `/shared/*` URL transparently uses the shared token without any client-side logic.

Custom `OnTokenValidated` event:
- Extracts `UserId`, `TokenId`, `TenantId`, `RoleId` from claims
- Cross-checks `TokenId` against an in-memory session cache (`ISessionService`)
- Fails authentication immediately if session is revoked — **stateless JWT + stateful revocation**

### 4.2 Role-Based Endpoint Permission (Custom `IAuthorizationHandler`)

All endpoints are protected by a custom `EndpointPermissionHandler`:

- Permissions are stored per-role in the DB (`RoleEndpoints` table) as `METHOD:path` strings
- A `ConcurrentDictionary` cache with a **5-minute TTL** avoids DB hits on every request
- Supports `ActionType` flag to distinguish between read (`GET`) and write (`POST/PUT/DELETE`) permissions at the same path

### 4.3 Firewall Middleware

Custom IP-layer protection implemented as ASP.NET Core middleware using `ConcurrentDictionary<string, DateTime>`:

- Manual IP blocking with configurable duration
- Automatic violation counter per IP
- Periodic cleanup of expired entries (every 5 minutes) using a thread-safe lock guard
- Returns structured JSON `403` response including `blockedUntil` and `remainingSeconds`

### 4.4 SQL Injection Prevention

A dedicated `QueryHelper` static class sanitizes all dynamic query inputs before they reach EF Core's `System.Linq.Dynamic.Core`:

- Character-level denylist (SQL metacharacters, Unicode whitespace variants)
- SQL keyword blocklist (`DROP`, `UNION`, `EXEC`, `xp_`, `sp_`, etc.)
- Per-type sanitizers: integer, double, boolean, date, string (with max indexed length guard of 4 000 chars)
- Compiled `Regex` with execution timeout to prevent ReDoS attacks

### 4.5 Sensitive Data Masking in Logs

`LoggingMiddleware` scrubs password fields from request bodies before they are written to any sink, using a compiled `Regex` with a 100 ms timeout guard.

---

## 5. Performance & Scalability

### 5.1 DbContext Pooling

Pool size is calculated at startup from hardware:

```
poolSize = Clamp(CPU_cores × 16, min=32, max=512)
```

`AddDbContextPool<Context>(poolSize: poolSize)` — reduces GC pressure under high concurrency.

### 5.2 Five Rate-Limiting Strategies (ASP.NET Core built-in)

| Policy | Algorithm | Limit |
|---|---|---|
| `fixed` | Fixed Window | 100 req / min |
| `sliding` | Sliding Window (6 segments) | 100 req / min |
| `token` | Token Bucket | 100 tokens, refill 20 / 10 s |
| `concurrent` | Concurrency | 50 simultaneous |
| `per-ip` | Sliding Window per IP | 50 req / min per client |

All strategies queue overflow requests (FIFO) and return a structured `429` body with `retryAfter` seconds.

### 5.3 Dual Serialization Format

The API supports both `application/json` and **`application/x-msgpack`** in the same endpoints. A custom `RequestDeserializer` / `ResponseSerializer` delegates to either `System.Text.Json` or `MessagePackSerializer` based on the request `Content-Type`. MessagePack reduces payload size by ~30-50% for binary-friendly clients (mobile, IoT).

### 5.4 WebSocket Support

`UseWebSockets()` is wired into the pipeline for real-time push scenarios.

### 5.5 In-Process Session Cache

`ISessionService` is registered as **Singleton**, providing type-safe `GetAsync<T>` / `SetAsync<T>` backed by LiteDB. Generic methods handle `string` as a fast-path to skip `JsonSerializer`, minimising overhead for the most common case.

---

## 6. Database Design

### 6.1 Multi-Tenant Entity Hierarchy

```
Entity (base)
└── TenantEntity       ← adds TenantId, soft-delete, audit timestamps
    ├── User
    ├── Inquiry / InquiryReply
    ├── Category
    ├── HelpPage
    ├── SharedLink
    └── Banner
```

All tenant-scoped tables use **composite unique indexes** on `(TenantId, <unique_field>)` so the same email / username can exist across different tenants without collision.

### 6.2 Custom ORM Attributes (Database project)

| Attribute | Purpose |
|---|---|
| `[TableContract]` | Marks a class as a DB table contract |
| `[TableMember]` | Column-level metadata (nullable, max length, index hint) |
| `[Index]` | Declares DB index without EF Fluent API |
| `[HideFromSummary]` | Excludes a property from OpenAPI schema |

### 6.3 Migration System

A fully custom `MigrationService` replaces EF Migrations for fine-grained production control:

- Migration records stored in a `Migrations` table
- `UseAutoMigrationAsync()` middleware runs pending migrations on startup in `Development` mode
- Idempotent — each migration script is executed at most once

### 6.4 Entities Overview

| Entity | Notes |
|---|---|
| `User` | PBKDF2 hashed password, failed-login counter, lock-until, email verification |
| `Session` | Refresh / shared tokens with expiry |
| `AuditLog` | Immutable change log for all mutations |
| `Traffic` | Per-request analytics (IP, device, country, duration) |
| `RoleEndpoint` | Defines which HTTP method + path a role may access |
| `SharedLink` | Time-limited public access URL tied to a specific resource |
| `Cache` | Persistent KV store (LiteDB-backed) for session state |
| `Tenant` | Root of the multi-tenant tree |

---

## 7. Logging & Observability

### Structured Logging with Serilog

Every HTTP request is enriched by `LoggingMiddleware` with:

| Property | Value |
|---|---|
| `CorrelationId` | Client-supplied `X-Correlation-ID` or auto-generated GUID |
| `TraceId` | `Activity.Current.Id` for distributed tracing integration |
| `UserId` | Authenticated user or `anonymous` |
| `ApiUrl` | Path + query string |
| `ApiMethod` | HTTP verb |
| `RequestContent` | Body (truncated to 4 096 chars, passwords masked) |
| `DeviceType` | Parsed from `User-Agent` via DeviceDetectorNET |
| `Country` | Resolved from IP via IP3Country |
| `ResponseTimeMs` | Measured with `Stopwatch.GetTimestamp()` (high-resolution) |

### SQLite Sink (custom batched implementation)

- `BatchSizeLimit = 45`, `BatchPeriod = 5 min`
- `QueueLimit = 50 000` — writes never block the request thread
- Automatically creates the `Logs` table on first run

---

## 8. Cloud & File Management

`CloudManager` is a **thread-safe Singleton** (double-checked `Lazy<T>` with `ExecutionAndPublication`) that tracks file lifecycle states:

```
Idle → Uploading → Uploaded
     → Editing   → Idle
```

Uses `ConcurrentDictionary<string, FileState>` to prevent race conditions when multiple requests attempt to upload or edit the same file simultaneously. Callers use `TryBeginUpload` / `TryBeginEdit` / `CompleteUpload` — pessimistic locking without explicit `lock` statements.

`CloudSession` manages per-file credentials and `CloudSettings` provides a strongly-typed configuration binding from `appsettings.json`.

---

## 9. Frontend (wwwroot SPA)

A lightweight **Single-Page Application** with zero runtime dependencies:

```
wwwroot/
├── index.html          Main shell
├── pages/              Route-level HTML fragments
├── components/         Reusable UI widgets
├── services/           JS API client wrappers
├── utils/              Helper functions
├── js/                 Core app bootstrap & router
├── css/                Scoped stylesheets
└── assets/             Static media
```

Key decisions:
- No React / Vue / Angular — keeps cold-start time under 200 ms on low-end VPS
- Component system hand-rolled using native `<template>` elements and `CustomEvent`
- `services/` mirrors the backend endpoint structure for self-documenting API consumption
- The `i.html` shell serves as an isolated iframe container for embedded widgets

---

## 10. Background Services

### MaintenanceService (`BackgroundService`)

Runs every **2 minutes** and performs:

- Flush `SentTraffics` (in-memory `ConcurrentDictionary<string, uint>`) to the DB in bulk
- Clean expired sessions and cache entries
- Deep memory cleanup (GC compact on configurable schedule)

Features a generic `ExecuteWithRetryAsync` helper with **exponential back-off**:

```
delay = baseDelayMs × 2^(attempt-1)
```

Handles both synchronous delegates and `async Task`-returning delegates uniformly via reflection on `Task.Result`.

---

## 11. API Documentation

Swagger UI is exposed in `Development` mode via FastEndpoints' NSwag integration.

Custom `IOperationProcessor` implementations:

| Processor | What it fixes |
|---|---|
| `RemoveBodyFromGetHeadProcessor` | Strips request body schema from `GET`/`HEAD` operations (OpenAPI 3 compliance) |
| `DateTimeFormatProcessor` | Annotates all `DateTime` properties with explicit ISO-8601 format string |

---

## 12. Key Design Patterns

| Pattern | Where used |
|---|---|
| **REPR (Request–Endpoint–Response)** | All API endpoints via FastEndpoints |
| **Singleton + Lazy initialization** | `CloudManager`, `SessionService`, LiteDB |
| **Repository / Unit of Work** | EF Core `DbContext` + `DbContextPool` |
| **Strategy** | Dual serializer (JSON vs MessagePack) |
| **Chain of Responsibility** | ASP.NET Core middleware pipeline |
| **Observer / Pub-Sub** | `BackgroundService` + `ConcurrentDictionary` traffic buffer |
| **Decorator** | `EndpointPermissionHandler` wrapping default `IAuthorizationHandler` |
| **Retry with exponential back-off** | `MaintenanceService.ExecuteWithRetryAsync` |
| **Cache-aside** | Role-permission cache in `EndpointPermissionHandler` |
| **Specification** | `QueryHelper` sanitization pipeline |
| **Multi-tenancy (Shared DB, Shared Schema)** | `TenantEntity` base class + composite indexes |

---

## 13. Hire Me / Freelance

I am available for **freelance projects and full-time opportunities** in:

- Backend API development (ASP.NET Core, C#)
- Multi-tenant SaaS architecture
- REST API design & documentation
- Database design & optimization (MySQL, SQLite, LiteDB)
- Security hardening (auth, rate limiting, firewalling)
- Legacy .NET migration to .NET 8/9
- Code review & architecture consulting

> **Source code, live demo, or a technical walkthrough call is available upon request.**  
> Contact me via [GitHub Issues](../../issues) or the email listed in my profile.

---

<p align="center">
  <img src="https://img.shields.io/badge/.NET-9.0-512BD4?style=flat-square&logo=dotnet" />
  <img src="https://img.shields.io/badge/C%23-13-239120?style=flat-square&logo=csharp" />
  <img src="https://img.shields.io/badge/MySQL-EF%20Core-4479A1?style=flat-square&logo=mysql" />
  <img src="https://img.shields.io/badge/JWT-Auth-000000?style=flat-square&logo=jsonwebtokens" />
  <img src="https://img.shields.io/badge/Serilog-Structured%20Logging-CC0000?style=flat-square" />
  <img src="https://img.shields.io/badge/FastEndpoints-REPR-FF6B35?style=flat-square" />
  <img src="https://img.shields.io/badge/MessagePack-Binary%20Serialization-purple?style=flat-square" />
  <img src="https://img.shields.io/badge/LiteDB-Embedded%20NoSQL-brightgreen?style=flat-square" />
</p>
