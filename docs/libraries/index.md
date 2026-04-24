# Libraries

Catalog of the ETL libraries shipped with the Wolfgang.Etl framework. Each page is a short overview — what the library does, when to use it, and when to pick something else — with links to the library's source repo for detailed API reference and recipes.

## Libraries in this section

- **[FixedWidth](fixedwidth.md)** — attribute-based fixed-width file extractor and loader. Mainframe exports, legacy banking formats, columnar ASCII output.
- **[DbClient](dbclient.md)** — ADO.NET database extractor and loader. Any relational DB with a `DbConnection` provider (SQL Server, PostgreSQL, SQLite, MySQL, Oracle, etc.).
- **[XML](xml.md)** — XML extractor and loader. Single-document-with-many-records or one-file-per-record streams.
- **[JSON](json.md)** — JSON extractor and loader. JSONL / NDJSON, JSON arrays, or per-file multi-stream.

## Related

- [Architecture](../architecture.md) — how these libraries implement the Abstractions contracts
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractors and loaders from different libraries into a single pipeline
- [Reference](../reference/index.md) — the Abstractions interfaces and base classes every library implements
