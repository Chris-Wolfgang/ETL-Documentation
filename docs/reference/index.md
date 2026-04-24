# Reference

Formal API reference for the interfaces, base classes, and test infrastructure in `Wolfgang.Etl.Abstractions`, `Wolfgang.Etl.TestKit`, and `Wolfgang.Etl.TestKit.Xunit`.

If you are building a pipeline for the first time, see [Getting Started](../getting-started/index.md) — the reference pages assume you already know the shape of the framework.

## Pages in this section

- **[Extractor](extractor.md)** — the four `IExtract*Async` interfaces (diamond inheritance) and the `ExtractorBase<TSource, TProgress>` abstract class. Constructors, properties, methods, and the `ExtractWorkerAsync` / `CreateProgressReport` extension points.
- **[Transformer](transformer.md)** — the four `ITransform*Async` interfaces and `TransformerBase<TSource, TDestination, TProgress>`. Same structure as the extractor, with notes on when transformers are application-specific and when generic reuse is possible.
- **[Loader](loader.md)** — the four `ILoad*Async` interfaces and `LoaderBase<TDestination, TProgress>`.
- **[TestKit](testkit.md)** — test doubles (`TestExtractor<T>`, `TestLoader<T>`, `TestTransformer<T>`), contract test base classes, the `ManualProgressTimer`, and the timer-injection pattern for writing deterministic progress tests.

## Related

- [Architecture](../architecture.md) — the layered overview and the four-interface diamond shape
- [Getting Started](../getting-started/index.md) — tutorial-level walk-throughs
- [Guides](../guides/index.md) — topic-oriented techniques
