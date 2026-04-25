# Libraries

Catalog of the libraries and packages shipped with the Wolfgang.Etl framework. Most are format-specific extractor and loader implementations; the TestKit packages are testing infrastructure for verifying components built on the Abstractions.

Each page is a short overview — what the package does, when to use it, and when to pick something else — with links to its source repo for detailed API reference and recipes.

## Libraries in this section

### Files

Text-based file formats. Stream rows through any of these libraries without buffering the whole file in memory.

- **[FixedWidth](fixedwidth.md)** — attribute-based fixed-width file extractor and loader. Mainframe exports, legacy banking formats, columnar ASCII output.
- **[XML](xml.md)** — XML extractor and loader. Single-document-with-many-records or one-file-per-record streams.
- **[JSON](json.md)** — JSON extractor and loader. JSONL / NDJSON, JSON arrays, or per-file multi-stream.

### Databases

Relational and bulk-import paths.

- **[DbClient](dbclient.md)** — ADO.NET database extractor and loader. Any relational DB with a `DbConnection` provider (SQL Server, PostgreSQL, SQLite, MySQL, Oracle, etc.).

### Testing

Tooling for verifying that your custom extractors, transformers, and loaders honor the framework's contract.

- **[TestKit](testkit.md)** — `Wolfgang.Etl.TestKit` and `Wolfgang.Etl.TestKit.Xunit`. Test doubles (`TestExtractor<T>`, `TestLoader<T>`, `TestTransformer<T>`), deterministic test helpers (`ManualProgressTimer`, `SynchronousProgress<T>`), and contract test base classes that verify your custom components honor the framework's contract.

## Related

- [Architecture](../architecture.md) — how these libraries implement the Abstractions contracts
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractors and loaders from different libraries into a single pipeline
- [Reference](../reference/index.md) — the Abstractions interfaces and base classes every library implements
