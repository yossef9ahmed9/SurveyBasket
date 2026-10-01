# 📊 SurveyBasket — Survey & Poll Management API

> A production-grade ASP.NET Core 9 REST API for creating and managing surveys, collecting votes, and analyzing results — built with clean architecture, role-based permissions, background jobs, and real-time email notifications.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [How It Works](#-how-it-works)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Project Structure](#-project-structure)
- [Tech Stack](#-tech-stack)
- [API Endpoints](#-api-endpoints)
- [Authentication & Authorization](#-authentication--authorization)
- [API Versioning](#-api-versioning)
- [Getting Started](#-getting-started)
- [Configuration](#-configuration)

---

## 🌟 Overview

**SurveyBasket** is a fully featured survey platform API. Admins create polls with questions and multiple-choice answers, members vote on active polls, and results are aggregated into meaningful analytics. The project demonstrates a range of production patterns: JWT auth with granular permissions, API versioning, hybrid caching, rate limiting, background jobs via Hangfire, structured logging with Serilog, and HTML email notifications.

---

## ⚙️ How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  Admin                                                      │
│  Creates Polls → Adds Questions & Answers → Publishes Poll  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  ASP.NET Core 9 REST API                                    │
│  • JWT Auth + Custom Permission Claims                      │
│  • EF Core → SQL Server                                     │
│  • HybridCache (L1 + L2) for available questions           │
│  • Hangfire background jobs for email notifications         │
│  • Rate limiting per IP / per user / concurrency            │
│  • Health checks for DB, Hangfire, Mail                     │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│  Member                                                     │
│  Browses Active Polls → Votes → Results Aggregated          │
└─────────────────────────────────────────────────────────────┘
```

---

## ✨ Key Features

### 🗳️ Poll Lifecycle Management
- Create polls with a title, summary, start date, and end date
- Add multiple questions per poll, each with multiple-choice answers
- Toggle published status — triggers an instant email notification to all members if the poll starts today
- A daily Hangfire job notifies members of polls starting each day

### 🔐 Granular Permission System
- Two roles out of the box: **Admin** and **Member**
- Permissions are embedded as JWT claims (`polls:read`, `users:add`, `results:read`, etc.)
- A custom `[HasPermission]` attribute with a dynamic `IAuthorizationPolicyProvider` enforces them at the endpoint level — no hard-coded policies in `Startup`

### ⚡ Hybrid Caching
- Available questions for a poll (the voting payload) are cached using `Microsoft.Extensions.Caching.Hybrid` (combines in-memory L1 + distributed L2)
- Cache is automatically invalidated on question add, update, or status toggle

### 🚦 Rate Limiting
- **IP-based fixed window** (2 req / 20 s) on the auth endpoints to throttle brute-force attempts
- **User-based fixed window** (2 req / 20 s) on the `GET /polls/current` endpoint
- **Concurrency limiter** (1000 concurrent, queue 100) on vote submission
- Returns HTTP 429 with a structured error body

### 📬 Email Notifications
- HTML email templates for: email confirmation, password reset, and poll notifications
- Sent via MailKit (SMTP / STARTTLS)
- `{{placeholder}}` token replacement in templates via `EmailBodyBuilder`

### 📅 Background Jobs (Hangfire)
- **Recurring daily job** — sends poll notification emails to all members for polls starting today
- **Enqueued job** — fires immediately when a poll is published on its start date
- Dashboard secured with basic auth at `/jobs`

### 📊 Results & Analytics
- Raw vote data per respondent per poll
- Votes grouped by day (trend chart data)
- Votes per answer per question (bar/pie chart data)

### 🏥 Health Checks
Exposed at `GET /health` (JSON format via HealthChecks UI):
- SQL Server connectivity
- Hangfire server availability
- Mail provider (SMTP connect + auth)

### 📄 Pagination, Search & Sort
- All list endpoints support `pageNumber`, `pageSize`, `searchValue`, `sortColumn`, and `sortDirection`
- Dynamic LINQ sorting via `System.Linq.Dynamic.Core`
- `PaginatedList<T>` returns `TotalPages`, `HasPreviousPage`, `HasNextPage`

### 🛡️ Result Pattern
- Services return `Result` / `Result<T>` — no thrown exceptions for business errors
- `Error` record carries `Code`, `Description`, and `StatusCode`
- `ToProblem()` extension converts failures to RFC 7807 `ProblemDetails` responses

---

## 🏗️ System Architecture

### Two-Layer Design

| Layer | Technology | Responsibility |
|---|---|---|
| **API** | ASP.NET Core 9 | Controllers, auth, validation, rate limiting, versioning |
| **Data** | EF Core + SQL Server | Persistence, migrations, audit fields |
| **Background** | Hangfire + SQL Server | Scheduled and enqueued jobs |
| **Cache** | HybridCache | In-memory + distributed caching |
| **Email** | MailKit | Transactional HTML emails |
| **Logging** | Serilog | Structured request + app logging |

---

## 📁 Project Structure

```
SurveyBasket/
│
├── Controllers/               # 8 API controllers
│   ├── AuthController.cs      # Login, register, email confirm, password reset
│   ├── AccountController.cs   # Current user profile + password change
│   ├── PollsController.cs     # Poll CRUD + publish toggle (v1 & v2)
│   ├── QuestionsController.cs # Question CRUD + status toggle (paginated)
│   ├── VotesController.cs     # Start vote session + submit answers
│   ├── ResultsController.cs   # Raw data, votes-per-day, votes-per-question
│   ├── RolesController.cs     # Role CRUD + status toggle + permissions
│   └── UsersController.cs     # User CRUD + disable/enable + unlock
│
├── Entities/                  # 9 EF Core entities
│   ├── Poll.cs                # Title, Summary, IsPublished, StartsAt, EndsAt
│   ├── Question.cs            # Content, PollId, IsActive
│   ├── Answer.cs              # Content, QuestionId, IsActive
│   ├── Vote.cs                # PollId, UserId, SubmittedOn
│   ├── VoteAnswer.cs          # VoteId, QuestionId, AnswerId
│   ├── ApplicationUser.cs     # IdentityUser + FirstName, LastName, IsDisabled, RefreshTokens
│   ├── ApplicationRole.cs     # IdentityRole + IsDefault, IsDeleted
│   ├── RefreshToken.cs        # Owned entity — Token, ExpiresOn, RevokedOn
│   └── AuditableEntity.cs     # CreatedById, CreatedOn, UpdatedById, UpdatedOn
│
├── Services/                  # Business logic
│   ├── AuthService.cs         # JWT generation, refresh token, registration, email flows
│   ├── PollService.cs         # Poll CRUD, publish toggle, background job trigger
│   ├── QuestionService.cs     # Question CRUD, available questions with HybridCache
│   ├── VoteService.cs         # Vote validation + submission
│   ├── ResultService.cs       # Vote aggregation queries
│   ├── RoleService.cs         # Role + permission claim management
│   ├── UserService.cs         # User management, profile, password
│   ├── EmailService.cs        # MailKit SMTP sender
│   └── NotificationService.cs # Poll notification email builder + dispatcher
│
├── Authentication/
│   ├── JwtProvider.cs         # Token generation + validation
│   ├── JwtOptions.cs          # Bound config model (Key, Issuer, Audience, Expiry)
│   └── Filters/
│       ├── HasPermissionAttribute.cs              # [HasPermission("polls:read")]
│       ├── PermissionAuthorizationPolicyProvider.cs # Dynamic policy creation
│       ├── PermissionAuthorizationHandler.cs      # JWT claim validation
│       └── PermissionRequirement.cs               # IAuthorizationRequirement wrapper
│
├── Abstractions/
│   ├── Result.cs              # Result<T> railway pattern
│   ├── Error.cs               # Error record (Code, Description, StatusCode)
│   ├── PaginatedList.cs       # Generic async paginator
│   └── Consts/
│       ├── Permissions.cs     # All permission string constants
│       ├── DefaultRoles.cs    # "Admin" / "Member" + seeded GUIDs
│       ├── RateLimiters.cs    # Rate limiter policy name constants
│       └── RegexPatterns.cs   # Validation regex (password strength, etc.)
│
├── Contracts/                 # Request + Response DTOs + FluentValidation validators
│   ├── Authentication/        # Login, Register, ConfirmEmail, ResetPassword, etc.
│   ├── Polls/                 # PollRequest, PollResponse, PollResponseV2
│   ├── Questions/             # QuestionRequest, QuestionResponse
│   ├── Answers/               # AnswerResponse
│   ├── Votes/                 # VoteRequest, VoteAnswerRequest
│   ├── Results/               # PollVotesResponse, VotesPerDayResponse, etc.
│   ├── Roles/                 # RoleRequest, RoleResponse
│   ├── Users/                 # CreateUserRequest, UpdateUserRequest, UserResponse
│   └── Common/                # RequestFilters (pagination + search + sort)
│
├── Persistence/
│   ├── ApplicationDbContext.cs        # EF Core + Identity + audit auto-set
│   ├── Migrations/                    # EF Core migration history
│   └── EntitiesConfigurations/        # Fluent API entity configs
│
├── Health/
│   └── MailProviderHealthCheck.cs     # Custom SMTP health check
│
├── Templates/                         # HTML email templates
│   ├── EmailConfirmation.html
│   ├── ForgetPassword.html
│   └── PollNotification.html
│
├── Mapping/
│   └── MappingConfigurations.cs       # Mapster custom mappings
│
├── Settings/
│   └── MailSettings.cs                # SMTP config model
│
├── Extensions/                        # ClaimsPrincipal extension (GetUserId)
├── Errors/                            # Typed error constants (PollErrors, etc.)
├── Helpers/                           # EmailBodyBuilder (template token replace)
├── Program.cs                         # App startup + middleware pipeline
├── DependencyInjection.cs             # All service registrations
├── GlobalUsings.cs                    # Project-wide using directives
└── appsettings.json                   # Configuration (connection strings, JWT, mail, etc.)
```

---

## 🛠️ Tech Stack

| Package | Version | Purpose |
|---|---|---|
| ASP.NET Core | 9.0 | Web API framework |
| Entity Framework Core | 9.0 | ORM + Code First migrations |
| SQL Server | — | Primary database (app + Hangfire jobs) |
| ASP.NET Identity | 9.0 | User + role management |
| `Asp.Versioning.Http` | 8.1.0 | Header-based API versioning |
| `Asp.Versioning.Mvc.ApiExplorer` | 8.1.0 | Versioned Swagger docs |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | 9.0 | JWT Bearer authentication |
| `Microsoft.Extensions.Caching.Hybrid` | 9.0 | L1 + L2 hybrid cache |
| `Hangfire.AspNetCore` / `.SqlServer` | 1.8.14 | Background job processing |
| `Hangfire.Dashboard.Basic.Authentication` | 7.0.1 | Hangfire dashboard auth |
| `MailKit` | 4.7.1 | SMTP email sending |
| `Mapster` | 7.4.1 | Object-to-object mapping |
| `FluentValidation.AspNetCore` | 11.3.0 | Request model validation |
| `Serilog.AspNetCore` | 8.0.2 | Structured logging |
| `Swashbuckle.AspNetCore` | 6.6.2 | Swagger / OpenAPI UI |
| `System.Linq.Dynamic.Core` | 1.4.3 | Dynamic LINQ sort expressions |
| `AspNetCore.HealthChecks.SqlServer` | 8.0.2 | SQL Server health check |
| `AspNetCore.HealthChecks.Hangfire` | 8.0.1 | Hangfire health check |
| `AspNetCore.HealthChecks.UI.Client` | 8.0.1 | Health check JSON formatter |

---

## 🌐 API Endpoints

### Auth — `/auth`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/auth` | Login → returns JWT + refresh token | Public |
| POST | `/auth/refresh` | Exchange refresh token for new JWT | Public |
| POST | `/auth/revoke-refresh-token` | Invalidate a refresh token | Public |
| POST | `/auth/register` | Register a new member account | Public |
| POST | `/auth/confirm-email` | Confirm email with OTP code | Public |
| POST | `/auth/resend-confirmation-email` | Resend confirmation code | Public |
| POST | `/auth/forget-password` | Send password reset code to email | Public |
| POST | `/auth/reset-password` | Reset password with code | Public |

### Account — `/me`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/me` | Get own profile | Bearer |
| PUT | `/me/info` | Update own name/details | Bearer |
| PUT | `/me/change-password` | Change own password | Bearer |

### Polls — `/api/polls`

| Method | Endpoint | Description | Permission |
|---|---|---|---|
| GET | `/api/polls` | List all polls | `polls:read` |
| GET | `/api/polls/current` *(v1)* | Active polls — includes `IsPublished` | Member |
| GET | `/api/polls/current` *(v2)* | Active polls — without `IsPublished` | Member |
| GET | `/api/polls/{id}` | Get single poll | `polls:read` |
| POST | `/api/polls` | Create poll | `polls:add` |
| PUT | `/api/polls/{id}` | Update poll | `polls:update` |
| DELETE | `/api/polls/{id}` | Delete poll | `polls:delete` |
| PUT | `/api/polls/{id}/togglePublish` | Toggle published status | `polls:update` |

### Questions — `/api/polls/{pollId}/questions`

| Method | Endpoint | Description | Permission |
|---|---|---|---|
| GET | `/api/polls/{pollId}/questions` | Paginated, searchable, sortable list | `questions:read` |
| GET | `/api/polls/{pollId}/questions/{id}` | Single question with answers | `questions:read` |
| POST | `/api/polls/{pollId}/questions` | Add question + answers | `questions:add` |
| PUT | `/api/polls/{pollId}/questions/{id}` | Update question + answers | `questions:update` |
| PUT | `/api/polls/{pollId}/questions/{id}/toggleStatus` | Enable/disable question | `questions:update` |

### Votes — `/api/polls/{pollId}/vote`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/polls/{pollId}/vote` | Get available questions (cached) | Member |
| POST | `/api/polls/{pollId}/vote` | Submit vote answers | Member |

### Results — `/api/polls/{pollId}/results`

| Method | Endpoint | Description | Permission |
|---|---|---|---|
| GET | `/api/polls/{pollId}/results/row-data` | Raw votes per respondent | `results:read` |
| GET | `/api/polls/{pollId}/results/votes-per-day` | Daily vote trend | `results:read` |
| GET | `/api/polls/{pollId}/results/votes-per-question` | Answer distribution | `results:read` |

### Roles — `/api/roles`

| Method | Endpoint | Description | Permission |
|---|---|---|---|
| GET | `/api/roles` | List roles (optionally include disabled) | `roles:read` |
| GET | `/api/roles/{id}` | Get role with permissions | `roles:read` |
| POST | `/api/roles` | Create role + assign permissions | `roles:add` |
| PUT | `/api/roles/{id}` | Update role + permissions | `roles:update` |
| PUT | `/api/roles/{id}/toggle-status` | Enable/disable role | `roles:update` |

### Users — `/api/users`

| Method | Endpoint | Description | Permission |
|---|---|---|---|
| GET | `/api/users` | List all users | `users:read` |
| GET | `/api/users/{id}` | Get user with roles | `users:read` |
| POST | `/api/users` | Create user + assign roles | `users:add` |
| PUT | `/api/users/{id}` | Update user info + roles | `users:update` |
| PUT | `/api/users/{id}/toggle-status` | Disable / enable user account | `users:update` |
| PUT | `/api/users/{id}/unlock` | Unlock a locked-out user | `users:update` |

---

## 🔐 Authentication & Authorization

### JWT Flow

```
POST /auth  { email, password }
→ 200 { token, expiresIn, refreshToken, refreshTokenExpiration }

Headers for protected endpoints:
Authorization: Bearer <token>
```

The token payload includes:
- `sub` — user ID
- `email`, `given_name`, `family_name`
- `jti` — unique token ID
- `roles` — JSON array e.g. `["Admin"]`
- `permissions` — JSON array e.g. `["polls:read","polls:add"]`

### Refresh Token Flow

```
POST /auth/refresh  { token, refreshToken }
→ 200 { token, expiresIn, refreshToken, refreshTokenExpiration }

POST /auth/revoke-refresh-token  { token, refreshToken }
→ 200 (token can no longer be refreshed)
```

### Permissions

| Resource | Permissions |
|---|---|
| Polls | `polls:read` · `polls:add` · `polls:update` · `polls:delete` |
| Questions | `questions:read` · `questions:add` · `questions:update` |
| Users | `users:read` · `users:add` · `users:update` |
| Roles | `roles:read` · `roles:add` · `roles:update` |
| Results | `results:read` |

Permissions are assigned to roles and stored as claims. The `[HasPermission("polls:read")]` attribute dynamically creates an authorization policy at runtime — no static policy registration needed.

---

## 🔢 API Versioning

Versioning is header-based. Include `x-api-version` in your request:

```
x-api-version: 1    ← default (deprecated)
x-api-version: 2    ← current
```

Currently versioned endpoint:

| Version | `GET /api/polls/current` response |
|---|---|
| v1 | `{ id, title, summary, isPublished, startsAt, endsAt }` |
| v2 | `{ id, title, summary, startsAt, endsAt }` — `isPublished` removed |

The API reports available versions in response headers (`api-supported-versions`, `api-deprecated-versions`).

---

## 🚀 Getting Started

### Prerequisites

- [.NET 9 SDK](https://dotnet.microsoft.com/download)
- SQL Server (LocalDB works for development)
- An SMTP account for emails (the project is pre-configured for [Ethereal](https://ethereal.email) in dev)

### 1. Clone and restore

```bash
git clone <repo-url>
cd SurveyBasket
dotnet restore
```

### 2. Configure secrets

Set your JWT signing key and mail password using User Secrets (keeps secrets out of source control):

```bash
cd SurveyBasket
dotnet user-secrets set "Jwt:Key" "your-super-secret-key-at-least-32-chars"
dotnet user-secrets set "MailSettings:Password" "your-smtp-password"
dotnet user-secrets set "HangfireSettings:Username" "admin"
dotnet user-secrets set "HangfireSettings:Password" "your-dashboard-password"
```

Or update `appsettings.Development.json` directly for a quick start.

### 3. Apply migrations

```bash
dotnet ef database update
```

This creates both the `SurveyBasket` app database and seeds the default Admin/Member roles and a default admin user.

### 4. Run

```bash
dotnet run
```

- Swagger UI: `https://localhost:{port}/swagger`
- Hangfire dashboard: `https://localhost:{port}/jobs`
- Health check: `https://localhost:{port}/health`

---

## ⚙️ Configuration

All settings live in `appsettings.json`. Secrets should be overridden via User Secrets or environment variables in production.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=SurveyBasket;...",
    "HangfireConnection": "Server=...;Database=SurveyBasketJobs;..."
  },
  "Jwt": {
    "Key": "<signing-key>",
    "Issuer": "SurveyBasketApp",
    "Audience": "SurveyBasketApp users",
    "ExpiryMinutes": 30
  },
  "MailSettings": {
    "Mail": "sender@example.com",
    "DisplayName": "Survey Basket",
    "Password": "<smtp-password>",
    "Host": "smtp.ethereal.email",
    "Port": 587
  },
  "HangfireSettings": {
    "Username": "<dashboard-user>",
    "Password": "<dashboard-password>"
  },
  "AllowedOrigins": [
    "https://www.survey-basket.com"
  ]
}
```
