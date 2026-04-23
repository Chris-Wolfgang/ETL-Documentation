# Wolfgang.Etl.Xml

XML extractor and loader with single-stream and multi-stream modes.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.Xml` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.Xml) |
| **Dependencies** | Wolfgang.Etl.Abstractions, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-Xml](https://github.com/Chris-Wolfgang/ETL-Xml) |

## What it does

Reads and writes XML documents in two modes:

- **Single-Stream** — one XML document containing many records, wrapped in a root element (e.g. `<ArrayOfPerson><Person/>...</ArrayOfPerson>`). Streams record-by-record rather than buffering the whole document in memory.
- **Multi-Stream** — one XML document per stream, with a factory that creates or provides streams on demand (one file per record, e.g. `output/1.xml`, `output/2.xml`).

Records are bound to XML via standard .NET XML serialization attributes on your record type. Record types must have a public parameterless constructor.

## When to use it

- **Legacy XML interchange** — SOAP payloads, enterprise data feeds, configuration files
- **One-file-per-record storage** — dumping records to individual XML files (multi-stream mode) or reading a directory of per-record XML files
- When you need to produce XML that other tools consume

## When to use something else

- For JSON, use `Wolfgang.Etl.Json`
- For event-driven XML parsing with no deserialization (e.g. SAX-style), use the built-in `XmlReader` directly
- For XML that doesn't map cleanly to a .NET type, process the raw elements with your own reader and build a custom extractor

## Quick example

```csharp
// Input (people.xml):
// <ArrayOfPerson>
//   <Person><Name>Alice</Name><Age>30</Age></Person>
//   <Person><Name>Bob</Name><Age>25</Age></Person>
// </ArrayOfPerson>

using var stream = File.OpenRead("people.xml");
var extractor = new XmlSingleStreamExtractor<Person>(stream, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    // ...
}
```

## More detail

- **API reference** — see the [ETL-Xml README](https://github.com/Chris-Wolfgang/ETL-Xml#readme) and XML doc comments on the public types
- **Examples** — see the [`examples/` folder](https://github.com/Chris-Wolfgang/ETL-Xml/tree/main/examples) in the source repo
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-Xml/releases)
