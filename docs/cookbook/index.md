# Cookbook

End-to-end worked examples for common ETL scenarios. Each page is a complete recipe: the packages to add, the source and destination shapes, the transformer, the pipeline wiring, and the common variations.

If you are new to the framework, start with [Getting Started](../getting-started/index.md) — the cookbook assumes you are comfortable wiring a basic extractor + transformer + loader.

## Pages in this section

- **[File to Database](file-to-database.md)** — read a JSON Lines file and bulk-load it into SQL Server, with notes on other file formats and other databases. Common batch-import pattern.
- **[Database to File](database-to-file.md)** — stream rows from a database query into a CSV file, with notes on JSON output and compression. Common batch-export pattern.
- **[Transform Chains](transform-chains.md)** — compose several small transformers into one stage with `Then(...)`, using a shared intermediate type (`Address`) so the validation steps are reusable across ERP / CRM / billing pipelines.

## Related

- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractors, transformers, and loaders together
- [Libraries](../libraries/index.md) — which library to pick for each source / destination format
