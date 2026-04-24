# Cookbook

End-to-end worked examples for common ETL scenarios. Each page is a complete, runnable recipe: the source data shape, the destination shape, the transformer, the pipeline wiring, and notes on error handling and progress.

If you are new to the framework, start with [Getting Started](../getting-started/index.md) — the cookbook assumes you are comfortable wiring a basic extractor + transformer + loader.

## Pages in this section

- **[File to Database](file-to-database.md)** — read a file (JSONL, XML, fixed-width) and load it into a relational database. Common batch-import pattern.
- **[Database to File](database-to-file.md)** — read from a database and write to a file. Common batch-export pattern.
- **[Transform Chains](transform-chains.md)** — pipelines that compose multiple transformers in sequence, including shared-intermediate-type patterns (e.g. validating `Address` across ERP / CRM / billing records).

## Related

- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractors, transformers, and loaders together
- [Libraries](../libraries/index.md) — which library to pick for each source / destination format
