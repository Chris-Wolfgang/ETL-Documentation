# Installation

## Prerequisites

`Wolfgang.Etl.Abstractions` targets a broad matrix and runs on:

- .NET Framework 4.6.2 or later
- .NET Core 3.1
- .NET 5 or later (.NET 5, 6, 7, 8, 9, 10, ...)
- Any runtime that supports `netstandard2.0` or `netstandard2.1`

You can install a [.NET SDK](https://dotnet.microsoft.com/download) for any of those.

!!! note "Per-library requirements may differ"
    The list above is for `Wolfgang.Etl.Abstractions`. Each extractor, transformer, and loader package has its own target framework matrix and runtime dependencies — check the relevant page under [Libraries](../libraries/index.md) before installing. For example, a library that depends on a third-party client may drop the oldest TFMs.

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

## Available Packages

For the full catalog — what each package does, when to use it, and the current version — see [Libraries](../libraries/index.md).

## Next Steps

- [Your First ETL](your-first-etl.md) — build an end-to-end pipeline using pre-built libraries
- [Your First Extractor](your-first-extractor.md) — build a custom extractor from scratch
- [Your First Transformer](your-first-transformer.md) — build a custom transformer from scratch
- [Your First Loader](your-first-loader.md) — build a custom loader from scratch
- [Architecture](../architecture.md) — understand the base classes and design patterns
