# Guides

Topic-oriented walkthroughs for building ETL pipelines with the Wolfgang.Etl framework. Each guide answers a specific "how do I…?" question that spans the framework rather than living inside a single library.

If you are new to the framework, start with [Getting Started](../getting-started/index.md) first.

## Pages in this section

- **[Pipeline Composition](pipeline-composition.md)** — wire an extractor, transformer, and loader into a complete pipeline. Pull-based model, data type flow, inline transformations, `PassThroughTransformer`, chaining with `MultistepTransformer`, interchangeability, and a complete worked example.
- **[Error Handling](error-handling.md)** — per-component responsibility, pipeline-level try/catch patterns, cancellation, custom-transformer errors, and pointers to each library's exception hierarchy.
- **[Progress Reporting](progress-reporting.md)** — the `IProgress<T>` pattern, tuning `ReportingInterval`, custom progress types, monitoring all pipeline stages, console progress bars.
- **[Custom Parsers & Converters](custom-parsers.md)** — the customization hooks each library exposes, with one example per library and a composition pattern for stacking multiple rules inside a delegate.
- **[Performance](performance.md)** — framework-level performance characteristics, streaming vs. materializing, tuning `SkipItemCount` / `MaximumItemCount` / `ReportingInterval`, profiling recipes.
- **[Reading and Writing Compressed Files and Streams](compressed-streams.md)** — wrap any extractor or loader with `GZipStream`, `BrotliStream`, or `DeflateStream` for transparent compression.

## Related

- [Getting Started](../getting-started/index.md) — step-by-step tutorials for building your first extractor and loader
- [Reference](../reference/index.md) — formal API reference
- [Libraries](../libraries/index.md) — per-library catalog entries
