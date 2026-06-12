# Architecture of Contoso University – Modernized .NET 8 App

## Overview

Contoso University is a university management web application that has been migrated from ASP.NET MVC 5 on .NET Framework 4.8 to **ASP.NET Core MVC on .NET 8**. The modernized application removes Windows-only infrastructure dependencies (MSMQ, IIS Express, local file system) and is designed to run on cloud-native hosting such as **Azure Container Apps**, using managed Azure services for data, messaging, and file storage.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Browser / Client                          │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼───────────────────────────────────────┐
│                  ASP.NET Core MVC (.NET 8)                       │
│                                                                  │
│  ┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │ Controllers │──▶│    Services      │──▶│   Data / EF Core │  │
│  │             │   │  (Notifications) │   │   (SchoolContext) │  │
│  └──────┬──────┘   └──────────────────┘   └────────┬─────────┘  │
│         │                                           │            │
│  ┌──────▼──────┐                          ┌────────▼─────────┐  │
│  │    Views    │                          │    SQL Server /   │  │
│  │  (Razor)    │                          │  Azure SQL DB     │  │
│  └─────────────┘                          └──────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    wwwroot (static assets)               │   │
│  │         Bootstrap · site.css · notifications.js          │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
ContosoUniversity/
├── Controllers/                # MVC Controllers (one per aggregate)
│   ├── HomeController.cs       # Landing page & enrollment statistics
│   ├── StudentsController.cs   # Student CRUD + pagination & search
│   ├── CoursesController.cs    # Course CRUD + teaching material upload
│   ├── InstructorsController.cs# Instructor CRUD + office assignments
│   ├── DepartmentsController.cs# Department CRUD
│   └── NotificationsController.cs # In-app notification feed API
│
├── Data/
│   ├── SchoolContext.cs        # EF Core DbContext (all entity sets)
│   └── DbInitializer.cs        # Database seeder (dev / test data)
│
├── Models/
│   ├── Person.cs               # Base class (TPH discriminator)
│   ├── Student.cs              # Inherits Person
│   ├── Instructor.cs           # Inherits Person
│   ├── Course.cs               # Course with teaching material path
│   ├── Department.cs
│   ├── Enrollment.cs
│   ├── CourseAssignment.cs     # Instructor ↔ Course join table
│   ├── OfficeAssignment.cs     # 1-to-1 with Instructor
│   ├── Notification.cs         # In-app notification record
│   ├── ErrorViewModel.cs
│   └── SchoolViewModels/       # View-specific DTOs
│       ├── AssignedCourseData.cs
│       ├── EnrollmentDateGroup.cs
│       └── InstructorIndexData.cs
│
├── Services/
│   ├── INotificationService.cs      # Notification abstraction
│   └── InMemoryNotificationService.cs # Default: System.Threading.Channels
│
├── Views/                      # Razor views (per controller + Shared)
├── wwwroot/                    # Static files (CSS, JS, lib)
│   ├── css/
│   ├── js/
│   └── lib/                    # Bootstrap (via libman)
│
├── Uploads/
│   └── TeachingMaterials/      # Uploaded course images (gitignored)
│
├── Program.cs                  # App bootstrap & DI registration
├── appsettings.json            # Production configuration
├── appsettings.Development.json# Development overrides
└── ContosoUniversity.csproj    # SDK-style project file (net8.0)
```

---

## Key Architectural Components

### 1. Web Framework – ASP.NET Core MVC (.NET 8)

The application uses the **ASP.NET Core MVC** pattern with:

- **Minimal hosting model** (`WebApplication.CreateBuilder`) in `Program.cs`.
- **Razor views** for server-side HTML rendering.
- **Dependency injection** wired at startup for `DbContext`, `INotificationService`, and other services.
- **Middleware pipeline**: HTTPS redirection → static files → routing → authorization → MVC routes.

### 2. Data Access – Entity Framework Core 8

`SchoolContext` (inherits `DbContext`) manages all database operations:

| Entity | Table | Notes |
|---|---|---|
| `Person` | `Person` | Table-per-hierarchy (TPH); discriminator column selects `Student` or `Instructor` |
| `Student` | `Person` | Derived from `Person` |
| `Instructor` | `Person` | Derived from `Person` |
| `Course` | `Course` | Holds `TeachingMaterialImagePath` |
| `Department` | `Department` | |
| `Enrollment` | `Enrollment` | Student ↔ Course many-to-many |
| `CourseAssignment` | `CourseAssignment` | Instructor ↔ Course many-to-many (composite PK) |
| `OfficeAssignment` | `OfficeAssignment` | 1-to-1 with `Instructor` |
| `Notification` | `Notification` | Persisted notification log |

Database is initialized in development via `DbInitializer.Initialize()` which seeds sample students, instructors, courses, and departments.

**Connection string** (configured in `appsettings.json`):
```json
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ContosoUniversity;..."
}
```
For production, this is replaced with an **Azure SQL Database** connection string.

### 3. Notification System

The notification system replaces the original MSMQ dependency with a cloud-portable abstraction:

| Component | Role |
|---|---|
| `INotificationService` | Interface: `SendNotificationAsync`, `GetRecentNotificationsAsync` |
| `InMemoryNotificationService` | Default implementation using `System.Threading.Channels` (bounded, drop-oldest) |
| `NotificationsController` | REST endpoint polled by the browser every 5 seconds |
| `notifications.js` | Front-end polling & toast display |
| `notifications.css` | Toast styling |

The provider is selected via `appsettings.json`:
```json
"Notifications": { "Provider": "InMemory" }
```

For production, this abstraction can be swapped for an **Azure Service Bus** implementation without changing any controller or view code.

### 4. Teaching Material File Upload

Courses can have an associated teaching material image:

- **Endpoint**: `CoursesController` (Create / Edit actions with `IFormFile`).
- **Storage**: Files are saved to `Uploads/TeachingMaterials/` with a unique GUID filename (`course_{id}_{guid}.{ext}`).
- **Cloud migration path**: Replace local storage with **Azure Blob Storage** by changing only the upload/download logic in the controller.

### 5. Configuration & Environments

| File | Purpose |
|---|---|
| `appsettings.json` | Base configuration (connection string, notification provider, log levels) |
| `appsettings.Development.json` | Verbose EF Core SQL logging for development |

`IConfiguration` and environment variables follow ASP.NET Core's layered configuration model, enabling 12-factor app compatibility for container deployments.

---

## Dependency Diagram

```
Program.cs
  └── builder.Services
        ├── AddControllersWithViews()
        ├── AddDbContext<SchoolContext>          ← SQL Server / Azure SQL
        └── AddSingleton<INotificationService>  ← InMemoryNotificationService
                                                   (swappable for Azure Service Bus)
```

---

## Migration: .NET Framework 4.8 → .NET 8

| Aspect | Before (.NET Framework 4.8) | After (.NET 8) |
|---|---|---|
| Framework | ASP.NET MVC 5 | ASP.NET Core MVC |
| Hosting | IIS Express (Windows-only) | Kestrel / Azure Container Apps |
| Database | EF Core 3.1 + SQL LocalDB | EF Core 8 + Azure SQL Database |
| Messaging | MSMQ (Windows-only) | `System.Threading.Channels` → Azure Service Bus |
| File storage | Local file system (`/Uploads/`) | Local → Azure Blob Storage |
| Config | `Web.config` | `appsettings.json` + environment variables |
| Project file | `packages.config` + `.csproj` | SDK-style `.csproj` |
| Entry point | `Global.asax` | `Program.cs` (minimal hosting) |
| Target OS | Windows only | Cross-platform (Linux containers) |

---

## Running Locally

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- SQL Server LocalDB (included with Visual Studio) **or** any SQL Server instance

### Steps

```bash
cd ContosoUniversity
dotnet restore
dotnet run
```

The application will be available at `https://localhost:5001` (or the port shown in the console). The database is created and seeded automatically on first run in Development mode.

---

## Cloud Deployment Target

The application is designed to be deployed as a **containerized workload** on **Azure Container Apps** with the following managed services:

| Service | Role |
|---|---|
| Azure Container Apps | Application hosting (auto-scaling, managed TLS) |
| Azure SQL Database | Relational data persistence |
| Azure Service Bus | Reliable async notifications (replaces InMemory provider) |
| Azure Blob Storage | Teaching material file storage (replaces local `/Uploads/`) |
| Azure Container Registry | Container image registry |
