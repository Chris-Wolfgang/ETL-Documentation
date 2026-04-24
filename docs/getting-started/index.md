# Getting Started

Step-by-step guides for building your first extractor, loader, and end-to-end pipeline with the Wolfgang.Etl framework.

Start here if you are new to the framework.

## Pages in this section

- **[Installation](installation.md)** — NuGet packages, version pinning, test-package setup
- **[Your First ETL](your-first-etl.md)** — build an end-to-end pipeline using pre-built libraries (`DbExtractor` + `JsonSingleStreamLoader`) — see the whole shape before diving into custom components
- **[Your First Extractor](your-first-extractor.md)** — build a custom extractor from scratch using `JsonLineExtractor` as a worked example
- **[Your First Loader](your-first-loader.md)** — build a custom loader from scratch using `JsonLineLoader` as a worked example

## What comes after

Once you have built your first ETL and know how to extend it, see:

- [Pipeline Composition](../guides/pipeline-composition.md) — variations on the basic pattern: chaining transformers, cancellation, progress reporting, error handling
- [Guides](../guides/index.md) — topic-oriented walkthroughs (error handling, progress reporting, performance, etc.)
- [Reference](../reference/index.md) — formal API reference for the Abstractions interfaces and base classes
