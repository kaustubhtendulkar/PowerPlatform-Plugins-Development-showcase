<h1 align="center">PowerPlatform Plugins Development Framework</h1>

<p align="center">
  <b>Enterprise-grade Dynamics 365 CE plugin development framework with 4-layer architecture, BDD testing, DAXIF deployment, and GitHub Copilot Skills integration</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Dynamics%20365%20CE-0B53CE?style=flat&logo=microsoft&logoColor=white" alt="Platform" />
  <img src="https://img.shields.io/badge/Language-C%23-239120?style=flat&logo=csharp&logoColor=white" alt="C#" />
  <img src="https://img.shields.io/badge/.NET-Framework%204.6.2-512BD4?style=flat&logo=dotnet&logoColor=white" alt=".NET" />
  <img src="https://img.shields.io/badge/SDK-Dataverse%209.0.2.51-0078D4?style=flat&logo=microsoft&logoColor=white" alt="SDK" />
  <img src="https://img.shields.io/badge/Testing-BDD%20%7C%20Reqnroll%20%7C%20FakeXrmEasy-6DB33F?style=flat" alt="Testing" />
  <img src="https://img.shields.io/badge/Deployment-DAXIF-FF6F00?style=flat" alt="DAXIF" />
  <img src="https://img.shields.io/badge/CI%2FCD-Azure%20Pipelines-2560E0?style=flat&logo=azurepipelines&logoColor=white" alt="CI/CD" />
  <img src="https://img.shields.io/badge/AI-GitHub%20Copilot%20Skills-8957e5?style=flat&logo=githubcopilot&logoColor=white" alt="Copilot" />
</p>

---

## Overview

A production-ready, opinionated framework for building Dynamics 365 CE plugins at scale. It enforces a strict **4-layer architecture** with clear separation of concerns, provides **BDD testing** that runs entirely in-memory (no live CRM needed), automates deployment via **DAXIF plugin sync**, and includes a **GitHub Copilot Skill** that can scaffold complete plugin solutions from a single prompt.

Built to solve real-world problems encountered in enterprise Dynamics 365 projects — sandbox isolation constraints, multi-team consistency, testability, and deployment automation.

---

## Solution Structure

```
Microsoft.D365.Plugins.sln
│
├── Microsoft.D365.Plugins.Shared          (Shared Project - .shproj)
│   ├── Controllers/                       Base controller classes
│   ├── Constants/                         Error codes, pipeline constants
│   ├── Defaults/                          DefaultValueProvider base
│   ├── Helpers/                           Entity extensions, context extensions
│   ├── Logging/                           ILogger → ITracingService adapter
│   ├── Models/                            XrmContext (auto-generated early-bound entities)
│   ├── Repositories/                      IDataverseService, DataverseService
│   ├── Utilities/                         QueryExpressionBuilder, FilterExpressionBuilder
│   ├── Validation/                        Validator, StronglyTypedValidator<T>, ValidationResult
│   ├── Plugin.cs                          Abstract base plugin with fluent registration
│   ├── CustomApi.cs                       Abstract base for Custom API plugins
│   ├── IDataverseContext.cs               Central dependency interface
│   └── LocalPluginContext.cs              IServiceProvider → IDataverseContext wrapper
│
├── Microsoft.D365.Sales.Plugins          (Plugin Assembly - .csproj)
│   └── Controllers/                       Entity-specific controllers + ControllerFactory
│
├── Microsoft.D365.Plugins.Tests          (Shared Test Infrastructure - .csproj)
│   ├── IPowerPlatformAssemblyProvider.cs  Entity metadata resolution interface
│   └── AppSettingsHelper.cs               Test configuration reader
│
├── Microsoft.D365.Sales.Plugins.Tests    (BDD Feature Tests - .csproj)
│   ├── Features/                          Gherkin .feature files
│   ├── StepDefinitions/                   Reqnroll step bindings
│   ├── AssemblyProvider.cs                Early-bound assembly resolver
│   └── Hooks.cs                           Reqnroll DI registration
│
└── Microsoft.D365.Tools                  (Console App - .csproj)
    ├── Daxif/                             Plugin sync & context generation scripts
    └── _Config.ps1                        Centralized environment config
```

### Why a Shared Project?

The `Microsoft.D365.Plugins.Shared` project uses Visual Studio's **Shared Project** format (`.shproj`/`.projitems`). Source files are compiled directly into the consuming plugin assembly — no separate DLL is produced. This is critical because the **Dynamics 365 sandbox only allows a single assembly** to be registered; external DLL references are blocked at runtime.

This also means the 26 shared source files (base classes, helpers, utilities) become part of the plugin DLL without any assembly loading issues.

---

## Architecture

### The 4-Layer Plugin Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PLUGIN LAYER                                 │
│  Plugin.Execute(IServiceProvider)                                   │
│    ├── Creates LocalPluginContext (wraps IServiceProvider)           │
│    ├── Matches RegisteredEvents by Stage + Message + Entity         │
│    ├── Opens structured logging scope (PluginName, EntityId, Op)    │
│    └── Invokes ExecuteLogic(IDataverseContext)                      │
│         ├── FaultException<OrganizationServiceFault> → log + rethrow│
│         ├── InvalidPluginExecutionException → rethrow (user-facing) │
│         └── Exception → wrap in InvalidPluginExecutionException     │
├─────────────────────────────────────────────────────────────────────┤
│                     CONTROLLER LAYER                                │
│  ControllerFactory.GetDataverseController(pluginContext)            │
│    ├── Switch on PrimaryEntityName → return entity controller       │
│    └── Controller receives IDataverseContext + repositories via ctor│
│                                                                     │
│  EntityController : DataverseController                             │
│    ├── GetValidators() → Validator[]                                │
│    ├── GetDefaultValueProviders() → DefaultValueProvider[]          │
│    └── Business methods (Create, Update, Delete handlers)           │
├─────────────────────────────────────────────────────────────────────┤
│                     VALIDATION LAYER                                │
│  Validator.Validate(Entity preImage, Entity postImage)              │
│    ├── StronglyTypedValidator<T> auto-converts Entity → T           │
│    ├── ValidationResult accumulates errors via AddError()           │
│    ├── Multiple validators per entity, merged via Merge()           │
│    └── CheckRequired() helper for mandatory field checks            │
│                                                                     │
│  DefaultValueProvider.SetDefaults(Entity target, Entity preImage)   │
│    ├── GetNewValue<T>() / GetOldValue<T>() for field access         │
│    └── SetValue<T>() / SetValueIfDifferent<T>() for writes          │
├─────────────────────────────────────────────────────────────────────┤
│                     REPOSITORY LAYER                                │
│  IDataverseService : IOrganizationService, IDisposable              │
│    ├── CreateQuery<TEntity>() → LINQ queryable                      │
│    ├── GetById<T>(Guid, selector) → typed retrieval                 │
│    ├── Execute<T>(OrganizationRequest) → typed response             │
│    └── Delete(EntityReference) / Delete(Entity) overloads           │
│                                                                     │
│  DataverseService wraps OrganizationServiceContext                   │
│    └── MergeOption.NoTracking (no change tracking overhead)         │
└─────────────────────────────────────────────────────────────────────┘
```

### Request Flow

```
IServiceProvider (CRM Pipeline)
        │
        ▼
   Plugin.Execute()
        │
        ├── LocalPluginContext created
        │     ├── Lazy IOrganizationService (caller context)
        │     ├── Lazy IOrganizationService (admin/SYSTEM context)
        │     ├── IPluginExecutionContext
        │     ├── ITracingService
        │     └── ILogger<T> (via XrmLogger adapter)
        │
        ├── Match RegisteredEvent (stage + message + entity)
        │
        ▼
   ExecuteLogic(IDataverseContext)
        │
        ├── ControllerFactory.GetDataverseController()
        │     └── Returns entity-specific controller
        │
        ├── controller.GetValidators()
        │     ├── validator1.Validate(preImage, postImage)
        │     ├── validator2.Validate(preImage, postImage)
        │     └── Merge all → ValidationResult
        │             └── If invalid → throw InvalidPluginExecutionException
        │
        ├── controller.GetDefaultValueProviders()
        │     └── provider.SetDefaults(target, preImage)
        │
        └── controller.ExecuteBusinessLogic()
              └── Uses repositories for data access
```

---

## Key Components

### Fluent Plugin Registration API

Plugins register their pipeline steps declaratively in the constructor. **DAXIF reads these registrations from the compiled DLL** and auto-registers them in CRM — no manual Plugin Registration Tool required.

**Supported fluent methods:**

| Method | Purpose |
|--------|---------|
| `RegisterPluginStep<T>(operation, stage, handler)` | Register a pipeline step |
| `.SetExecutionOrder(int)` | Control execution sequence |
| `.SetExecutionMode(ExecutionMode)` | Synchronous or Asynchronous |
| `.AddFilteredAttributes(x => x.Prop, ...)` | Only trigger on specific field changes |
| `.AddImage(name, alias, imageType, x => x.Prop, ...)` | Register pre/post images with specific columns |
| `.SetName(string)` | Custom step name |

### IDataverseContext — Central Dependency Interface

The core abstraction that replaces traditional DI containers (which are blocked in the CRM sandbox):

| Member | Purpose |
|--------|---------|
| `Service` | `IDataverseService` — caller-context org service with LINQ support |
| `AdminService` | `IDataverseService` — SYSTEM-context org service for elevated operations |
| `PluginExecutionContext` | `IPluginExecutionContext` — pipeline metadata, Target, images |
| `TracingService` | `ITracingService` — plugin trace log |
| `GetLogger<T>()` | `ILogger<T>` — structured logging via `Microsoft.Extensions.Logging` patterns |

Both `Service` and `AdminService` are **lazy-initialized** — they're only created when first accessed, avoiding unnecessary service factory calls.

### Structured Logging (Sandbox-Safe)

The framework implements `Microsoft.Extensions.Logging` patterns without shipping the runtime DLL:

```
┌────────────────────────────┐     ┌────────────────────────────┐
│ Microsoft.Extensions       │     │ Inlined Logging Abstractions│
│ .Logging.Abstractions      │     │ (5 source files in Shared)  │
│ NuGet Package              │     │                              │
│                            │     │ • ILogger / ILogger<T>       │
│ Used at compile-time ONLY: │     │ • LogLevel enum              │
│ • Source generators         │────▶│ • EventId struct             │
│ • Analyzers                │     │ • LoggerMessage factory      │
│ • [LoggerMessage] support  │     │                              │
│                            │     │ Provide runtime types in     │
│ ExcludeAssets=runtime      │     │ sandbox (no DLL dependency)  │
└────────────────────────────┘     └────────────────────────────┘
                                              │
                                              ▼
                                   ┌────────────────────────┐
                                   │ XrmLogger<T>            │
                                   │ Adapts ILogger → Trace()│
                                   │ ITracingService.Trace() │
                                   └────────────────────────┘
```

Typed log messages with `EventId` and scoped context:

| Extension Method | When Used |
|-----------------|-----------|
| `PluginExecutionScope()` | Opens scope with PluginName, EntityId, EntityName, Operation |
| `StartingPluginExecution()` | Entry point logging |
| `FinishedPluginExecution()` | Exit point logging |
| `FaultExceptionCaught()` | CRM fault exception handling |
| `UnhandledExceptionCaught()` | Unexpected exception handling |

### Validation Pipeline

```
Validator (abstract)
   │
   ├── Validates against raw Entity objects
   ├── CheckRequired() — mandatory field helper
   └── Returns ValidationResult
         │
         ├── AddError(string message)
         ├── Merge(ValidationResult other)
         ├── IsValid → bool (no errors)
         └── Errors → IReadOnlyCollection<string>

StronglyTypedValidator<T> : Validator
   │
   └── Auto-converts Entity → early-bound T
       before calling Validate(T preImage, T postImage)
```

Multiple validators per entity are composed and merged:

```
var validationResult = new ValidationResult();
foreach (var validator in controller.GetValidators())
    validationResult.Merge(validator.Validate(preImage, postImage));
if (!validationResult.IsValid)
    throw new InvalidPluginExecutionException(string.Join(Environment.NewLine, validationResult.Errors));
```

### Fluent Query Building

Type-safe QueryExpression construction with lambda-based column selection:

| Builder | Methods |
|---------|---------|
| **`QueryExpressionBuilder<T>`** | `WithDistinct()`, `WithTop(n)`, `WithOrderAscending(x => x.Prop)`, `WithOrderDescending(x => x.Prop)`, `WithColumns(x => new { x.Col1, x.Col2 })`, `WithFilter(filter => ...)`, `Build()` |
| **`FilterExpressionBuilder<T>`** | `WithEqual(x => x.Prop, value)`, `WithNotEqual()`, `WithNull()`, `WithNotNull()`, `WithIn(x => x.Prop, values)` |

Column names are resolved via the `[Column("logical_name")]` attribute on early-bound entity properties — ensuring compile-time safety.

### Entity Extension Helpers

| Helper | What It Does |
|--------|-------------|
| `TryGetAttributeValue<T>()` | Auto-converts `OptionSetValue` → enum, `EntityReference` → `Guid` |
| `SetAttribute()` | Write with `OverwriteBehavior` (DoNotOverwrite / OverwriteIfNull / Overwrite) |
| `GetEntityBeforeDatabaseOperation()` | Returns PreImage entity |
| `GetEntityAfterDatabaseOperation()` | Merges Target over PreImage (simulates post-state) |
| `GetTargetEntity<T>()` | Typed Target extraction |
| `GetInputParameter<T>()` / `GetOutputParameter<T>()` | Custom API parameter access |
| `GetColumnName<T>(expression)` | Resolves `[Column]` attribute or lowercased member name |

---

## Testing Framework

### BDD with Reqnroll + FakeXrmEasy

All tests run **in-memory** using FakeXrmEasy's simulated CRM context. No live Dynamics 365 environment is needed — not even for CI pipelines.

```
┌──────────────────────────┐     ┌──────────────────────────────┐
│  Feature Files (.feature) │     │  FakeXrmEasy (In-Memory CRM) │
│                           │     │                                │
│  Given/When/Then steps    │────▶│  Simulated plugin pipeline     │
│  in Gherkin syntax        │     │  Entity metadata from assembly │
│                           │     │  No live CRM connection        │
└──────────────────────────┘     └──────────────────────────────┘
         │                                    │
         ▼                                    ▼
┌──────────────────────────┐     ┌──────────────────────────────┐
│  Step Definitions         │     │  Assertions (FluentAssertions) │
│  (Shared + Feature-area)  │     │  + NUnit                       │
└──────────────────────────┘     └──────────────────────────────┘
```

### Test Patterns

**Plugin Execution Test:**

```gherkin
Feature: Account Validation

Scenario: Reject accounts without a name
    Given the following plugin registration
        | Stage          | Message Name | Execution Mode | PrimaryEntityName |
        | PreValidation  | Create       | Synchronous    | account           |
    And an account as Target with the following values
        | Property | Value |
        | name     |       |
    When the plugin AccountValidationPlugin is executed
    Then the plugin throws an error
```

**Pre-existing Data Test:**

```gherkin
Scenario: Validate against existing records
    Given the following set of contact records
        | Alias   | First Name | Last Name |
        | contact1| John       | Doe       |
    And a contact as PreImage with the following values
        | Property   | Value  |
        | First Name | Jane   |
    When the plugin ContactPlugin is executed
    Then the target entity has the following values
        | Property   | Value  |
        | Full Name  | Jane Doe |
```

**Custom API Test:**

```gherkin
Scenario: Execute custom API with parameters
    Given the following input parameters
        | Name        | Type   | Value    |
        | AccountName | String | Contoso  |
    When the plugin CreateAccountApi is executed
    Then the following output parameters are returned
        | Name      | Type | Value |
        | AccountId | Guid | *     |
```

### Test Infrastructure

| Component | Purpose |
|-----------|---------|
| **`IPowerPlatformAssemblyProvider`** | Interface for resolving the assembly containing early-bound entity types |
| **`AssemblyProvider`** | Returns the plugin assembly for FakeXrmEasy entity metadata resolution |
| **`Hooks`** | Reqnroll `[BeforeScenario]` — registers `AssemblyProvider` in the DI container |
| **`AppSettingsHelper`** | Reads `app.config` for test settings (`LanguageCode=1033`, date/time formats) |

### Shared Step Bindings (Reusable Across Feature Areas)

| Step Pattern | Action |
|-------------|--------|
| `Given the following set of {entity} records` | Creates multiple in-memory records from DataTable |
| `Given a/an {entity} as Target/PreImage/PostImage` | Sets pipeline entity context |
| `Given the following plugin registration` | Configures stage, message, execution mode |
| `When the plugin {ClassName} is executed` | Invokes plugin via reflection against FakeXrmEasy |
| `Then the target entity has the following values` | Asserts Target entity properties |
| `Then an InvalidPluginExecutionException is thrown` | Asserts expected CRM error |

> **Convention:** Property names in DataTables use **Dataverse display names** (not logical names). The framework resolves them via entity metadata from the early-bound assembly.

---

## Deployment

### DAXIF Plugin Sync

DAXIF reads plugin registrations directly from the **compiled DLL** (via the fluent registration API) and synchronizes them to Dynamics 365 — no Plugin Registration Tool required.

```
┌─────────────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  Plugin Source Code  │     │  Compiled Plugin DLL     │     │  Dynamics 365 CE │
│                      │     │                          │     │                  │
│  RegisterPluginStep  │────▶│  DAXIF reads registered  │────▶│  Plugin steps    │
│  in constructor      │     │  steps from reflection   │     │  auto-created    │
│                      │     │                          │     │  in target env   │
└─────────────────────┘     └─────────────────────────┘     └──────────────────┘
```

### Deployment Scripts

| Script | What It Does |
|--------|-------------|
| **`_Config.ps1`** | Centralized environment config — CRM URLs, OAuth credentials, solution name, entity list, paths |
| **`_InitDaxif.ps1`** | Initializes DAXIF runtime — loads DLLs, creates environment connection via OAuth |
| **`PluginSyncDevSalesPlugins.ps1`** | Iterates plugin projects → locates DLL → calls `[DG.Daxif.Plugin]::Sync()` in sandbox mode |
| **`GenerateCSharpContext.ps1`** | Runs XrmContext to generate early-bound C# entity classes from live Dataverse metadata |

### XrmContext Generation

Generates strongly-typed C# entity classes from Dataverse:

```
XrmContext.exe
  ├── Connects to CRM via OAuth
  ├── Reads entity metadata for configured entities
  ├── Generates C# classes in Microsoft.D365.Plugins.Shared.Models namespace
  └── Output: XrmServiceContext + entity classes (Account, Contact, etc.)
```

### Azure Pipelines CI/CD

```yaml
Trigger:     master, develop
Pool:        windows-latest
Steps:
  1. NuGet restore (from private + public feeds via NuGet.config)
  2. VS Build (Release | Any CPU)
  3. VS Test
     ├── Code coverage enabled
     ├── Filter: TestCategory=Unit
     ├── Framework: .NETFramework 4.6.2
     └── RunSettings: CodeCoverage.runsettings
```

---

## GitHub Copilot Skill: `PowerPlatform-Plugin-Daxif`

An AI workflow package that can scaffold entire plugin solutions from a single prompt.

### What It Scaffolds

| Component | Generated Assets |
|-----------|-----------------|
| **Solution** | `.sln` file with fresh GUIDs, correct project references |
| **Shared Project** | All 26 base class files, inlined logging abstractions, helpers, utilities |
| **Plugin Project** | `.csproj` with NuGet refs, SNK key, CodeCoverage settings, controller factory, folder structure |
| **Test Projects** | Shared infrastructure + feature-area tests with Reqnroll config and hooks |
| **Tools Project** | DAXIF scripts, XrmContext generation, centralized config |
| **DevOps** | `azure-pipelines.yml`, `NuGet.config`, `.gitignore`, `RenameSolution.ps1` |

### Code Generation Procedures

| Ask Copilot to... | It generates... |
|---|---|
| Add a new Plugin | Plugin class with fluent registration API + `ExecuteLogic` handler |
| Add a new Validator | `StronglyTypedValidator<T>` with typed validation logic |
| Add a new Controller | `DataverseController` subclass + `ControllerFactory` registration |
| Add a new Repository | `I{Entity}Repository` interface + implementation with LINQ/QueryExpression |
| Add a new DefaultValueProvider | Provider with `GetNewValue`/`SetValue` pattern |
| Add a new Custom API | `CustomAPI` subclass with input/output parameter handling |
| Add a new BDD Test | `.feature` file + step definitions following shared patterns |

### Skill Synchronization

```
┌──────────────────────────┐         ┌──────────────────────────────┐
│  Any Workspace Repo      │         │  CentralSkillsRepository     │
│  .github/skills/         │ ──push──│  skills/                     │
│    Sync-Skills.ps1       │         │    PowerPlatform-Plugin-...  │
│    PowerPlatform-Plugin/ │ ◄─pull──│    dataverse-migration-...   │
└──────────────────────────┘         └──────────────────────────────┘
```

Skills are version-controlled and synchronized across repos via a `Sync-Skills.ps1` push/pull mechanism.

---

## Tech Stack

| Domain | Technologies |
|---|---|
| **Platform** | Dynamics 365 CE, Dataverse |
| **Language** | C# (.NET Framework 4.6.2) |
| **SDK** | Microsoft.CrmSdk.CoreAssemblies 9.0.2.51 |
| **Logging** | Microsoft.Extensions.Logging.Abstractions 6.0.1 (compile-time) + inlined runtime types |
| **Annotations** | System.ComponentModel.Annotations 5.0.0 |
| **Testing** | Reqnroll 2.2.1 (BDD), FakeXrmEasy 1.58.1, FluentAssertions 6.12.0, NUnit 3.13.3 |
| **Deployment** | DAXIF, XrmContext, PowerShell |
| **CI/CD** | Azure Pipelines (YAML), NuGet |
| **AI** | GitHub Copilot Skills (parameterized scaffolding) |

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Shared Project** instead of Class Library | CRM sandbox only allows a single registered assembly — no external DLL loading |
| **Inlined logging abstractions** | Use `[LoggerMessage]` source generators without shipping a runtime dependency into sandbox |
| **Manual DI** via `IDataverseContext` | Dynamics 365 sandbox blocks DI containers; `IDataverseContext` serves as a focused service locator |
| **Lazy service creation** | `Service` and `AdminService` are only instantiated when accessed — avoids unnecessary `IOrganizationServiceFactory` calls |
| **`MergeOption.NoTracking`** in `DataverseService` | Eliminates change tracking overhead for read operations |
| **DAXIF** over Plugin Registration Tool | Enables declarative, code-first plugin registration that can be automated in CI/CD |
| **XrmContext** over CrmSvcUtil | Generates cleaner early-bound classes with proper `[Column]` attributes |
| **Reqnroll** over SpecFlow | Open-source successor with active maintenance and .NET compatibility |
| **Display names** in test DataTables | More readable Gherkin — framework resolves to logical names via metadata |

---

## Author

**Kaustubh Tendulkar**
Senior Dynamics 365 CE & Power Platform Consultant | 14+ Years

<p>
  <a href="https://www.linkedin.com/in/kaustubhtendulkar/" target="_blank">
    <img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat&logo=linkedin&logoColor=white" alt="LinkedIn" />
  </a>
  <a href="https://github.com/kaustubhtendulkar" target="_blank">
    <img src="https://img.shields.io/badge/GitHub-Profile-181717?style=flat&logo=github&logoColor=white" alt="GitHub" />
  </a>
</p>

---

> *This is a showcase repository. Source code is available upon request for authorized collaborators.*
