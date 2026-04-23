# Installation

## Prerequisites

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download) or later

## Installing a Library

Each ETL library is a standalone NuGet package. Install the one that matches your data format:

```bash
# Fixed-width files
dotnet add package Wolfgang.Etl.FixedWidth

# Database (ADO.NET / Dapper)
dotnet add package Wolfgang.Etl.DbClient

# JSON / JSONL
dotnet add package Wolfgang.Etl.Json

# XML
dotnet add package Wolfgang.Etl.Xml
```

Each library depends on `Wolfgang.Etl.Abstractions` — it is pulled in automatically as a transitive dependency.

## Pinning vs. Floating Versions

`dotnet add package` records an exact version in your `.csproj`:

```xml
<PackageReference Include="Wolfgang.Etl.Json" Version="0.1.0" />
```

Pinning an exact version gives you reproducible builds — the same build inputs today and next year produce the same output. This is the recommended default for libraries and applications that value stability over always-latest.

If you want to always pull the latest minor/patch release — useful for internal applications that regularly absorb upstream fixes — replace the `Version` value with a floating reference:

```xml
<!-- Any version >= 0.10.2 -->
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="[0.10.2,)" />

<!-- Any version — always latest -->
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="*" />
```

Pin when shipping a library that others will depend on. Float when you control both ends of the dependency chain and want to stay on the latest release without manual bumps. Either way, rebuild your lock file (`packages.lock.json`) after a floating resolve to capture the version actually used.

## Installing Test Packages

To write contract tests for your custom extractors and loaders, add the TestKit packages to your test project:

```bash
dotnet add package Wolfgang.Etl.TestKit
dotnet add package Wolfgang.Etl.TestKit.Xunit
```

!!! note
    Use xUnit 2.9.3 with `xunit.runner.visualstudio` 2.8.2. xUnit 3.x is **not compatible** with `TestKit.Xunit`.

## Package Summary

| Package | Version | Purpose |
|---------|---------|---------|
| `Wolfgang.Etl.Abstractions` | 0.10.2 | Base classes and interfaces (transitive dependency) |
| `Wolfgang.Etl.FixedWidth` | 0.1.0 | Fixed-width file extractor and loader |
| `Wolfgang.Etl.DbClient` | 0.1.0 | ADO.NET database extractor and loader |
| `Wolfgang.Etl.Json` | 0.1.0 | JSON, JSONL, and multi-stream JSON |
| `Wolfgang.Etl.Xml` | 0.1.0 | XML single-stream and multi-stream |
| `Wolfgang.Etl.TestKit` | 0.5.0 | Test doubles for integration tests |
| `Wolfgang.Etl.TestKit.Xunit` | 0.5.0 | Contract test base classes for xUnit |

## Next Steps

- [Your First Extractor](your-first-extractor.md) — build a custom extractor from scratch
- [Your First Loader](your-first-loader.md) — build a custom loader from scratch
- [Architecture](../architecture.md) — understand the base classes and design patterns
