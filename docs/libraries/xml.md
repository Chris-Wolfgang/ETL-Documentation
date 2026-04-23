# Wolfgang.Etl.Xml

XML extractor and loader using XmlSerializer for the ETL Framework.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.Xml` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.Xml) |
| **Dependencies** | Wolfgang.Etl.Abstractions, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-Xml](https://github.com/Chris-Wolfgang/ETL-Xml) |

## Overview

Wolfgang.Etl.Xml provides XML extraction and loading using `XmlSerializer`. Two modes are supported:

- **SingleStream**: a single XML document with a root element wrapping child elements (e.g. `<ArrayOfPerson><Person/>...</ArrayOfPerson>`). Uses streaming `XmlReader`/`XmlWriter` to avoid buffering the entire document in memory.
- **MultiStream**: one XML document per stream, with a factory function that creates/provides streams on demand.

All record types must have a public parameterless constructor (`new()` constraint) because `XmlSerializer` requires it.

## XmlSingleStreamExtractor

`XmlSingleStreamExtractor<TRecord>` inherits `ExtractorBase<TRecord, XmlReport>`.

**Type constraints:** `TRecord : notnull, new()`.

### Constructors

```csharp
// Basic:
public XmlSingleStreamExtractor
(
    Stream stream,
    ILogger<XmlSingleStreamExtractor<TRecord>> logger
);

// With custom XmlReaderSettings:
public XmlSingleStreamExtractor
(
    Stream stream,
    XmlReaderSettings readerSettings,
    ILogger<XmlSingleStreamExtractor<TRecord>> logger
);
```

### Behavior

Reads an XML document from the stream, advances past the root element, then deserializes each depth-1 child element as a `TRecord` using `XmlSerializer`. The root element name is expected to follow the `XmlSerializer` convention (e.g. `ArrayOfPerson` for `Person` records), but only the child element names are validated against the serializer.

The stream is not closed by the extractor (`CloseInput = false`). Reading is asynchronous (`Async = true`).

### Example

```csharp
// Input XML:
// <ArrayOfPerson>
//   <Person><Name>Alice</Name><Age>30</Age></Person>
//   <Person><Name>Bob</Name><Age>25</Age></Person>
// </ArrayOfPerson>

using var stream = File.OpenRead("people.xml");
var extractor = new XmlSingleStreamExtractor<Person>(stream, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine($"{person.Name}, age {person.Age}");
}
```

## XmlSingleStreamLoader

`XmlSingleStreamLoader<TRecord>` inherits `LoaderBase<TRecord, XmlReport>`.

**Type constraints:** `TRecord : notnull, new()`.

### Constructors

```csharp
// Basic:
public XmlSingleStreamLoader
(
    Stream stream,
    ILogger<XmlSingleStreamLoader<TRecord>> logger
);

// With custom XmlWriterSettings:
public XmlSingleStreamLoader
(
    Stream stream,
    XmlWriterSettings writerSettings,
    ILogger<XmlSingleStreamLoader<TRecord>> logger
);
```

### Behavior

Writes an XML document to the stream with:
1. An XML declaration (`<?xml ...?>`)
2. A root element named `"ArrayOf" + typeof(TRecord).Name` (e.g. `ArrayOfPerson`)
3. Each item serialized as a child element using `XmlSerializer`
4. Empty namespace declarations to produce clean output

The stream is not closed by the loader (`CloseOutput = false`). Writing defaults to `Indent = true` if no custom settings are provided.

### Example

```csharp
using var stream = File.Create("output.xml");
var loader = new XmlSingleStreamLoader<Person>(stream, logger);

await loader.LoadAsync(personStream, cancellationToken);

// Output:
// <?xml version="1.0" encoding="utf-8"?>
// <ArrayOfPerson>
//   <Person>
//     <Name>Alice</Name>
//     <Age>30</Age>
//   </Person>
//   ...
// </ArrayOfPerson>
```

## XmlMultiStreamExtractor

`XmlMultiStreamExtractor<TRecord>` inherits `ExtractorBase<TRecord, XmlReport>`.

**Type constraints:** `TRecord : notnull, new()`.

### Constructors

```csharp
public XmlMultiStreamExtractor
(
    IEnumerable<Stream> streams,
    ILogger<XmlMultiStreamExtractor<TRecord>> logger
);
```

### Behavior

Iterates over the provided `IEnumerable<Stream>`, deserializing one `TRecord` from each stream using `XmlSerializer`. Each stream is disposed after reading. Streams that deserialize to `null` are skipped with a log message.

### Example

```csharp
var streams = Directory.GetFiles("data/", "*.xml")
    .Select(path => (Stream)File.OpenRead(path));

var extractor = new XmlMultiStreamExtractor<Person>(streams, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine(person.Name);
}
```

## XmlMultiStreamLoader

`XmlMultiStreamLoader<TRecord>` inherits `LoaderBase<TRecord, XmlReport>`.

**Type constraints:** `TRecord : notnull, new()`.

### Constructors

```csharp
// Basic:
public XmlMultiStreamLoader
(
    Func<TRecord, Stream> streamFactory,
    ILogger<XmlMultiStreamLoader<TRecord>> logger
);

// With custom XmlWriterSettings:
public XmlMultiStreamLoader
(
    Func<TRecord, Stream> streamFactory,
    XmlWriterSettings writerSettings,
    ILogger<XmlMultiStreamLoader<TRecord>> logger
);
```

### Behavior

For each item in the input sequence, calls the stream factory to obtain a `Stream`, serializes the item as a complete XML document, flushes, and disposes the stream. The factory receives the item being written, so stream creation can depend on item properties (e.g. file names based on record fields).

The factory must not return `null`; doing so throws `InvalidOperationException`.

### Example

```csharp
var loader = new XmlMultiStreamLoader<Person>
(
    person => File.Create($"output/{person.Id}.xml"),
    logger
);

await loader.LoadAsync(personStream, cancellationToken);
// Creates one XML file per person: output/1.xml, output/2.xml, etc.
```

## XmlReport

Extends `Report` with XML-specific progress data.

| Property | Type | Description |
|---|---|---|
| `CurrentCount` | `int` | Number of records processed (inherited). |
| `CurrentSkippedItemCount` | `int` | Number of records skipped. |

## Root element naming

The `XmlSingleStreamLoader` names the root element as `"ArrayOf" + typeof(TRecord).Name`. For a record type named `Person`, the root element is `ArrayOfPerson`. This matches the convention used by `XmlSerializer` when serializing arrays and lists.

The `XmlSingleStreamExtractor` reads any root element name -- it only validates child element names against the serializer via `CanDeserialize`.

## XmlReaderSettings / XmlWriterSettings customization

Both single-stream classes accept optional settings objects. The extractor and loader override two properties regardless of what the caller provides:

| Setting | Forced Value | Reason |
|---|---|---|
| `CloseInput` / `CloseOutput` | `false` | The caller owns the stream lifetime. |
| `Async` | `true` | Enables async read/write operations. |

If no custom settings are provided:
- The extractor uses default `XmlReaderSettings`.
- The loader uses `new XmlWriterSettings { Indent = true }`.

## Common patterns

### SkipItemCount and MaximumItemCount

All four classes (single and multi, extractor and loader) support `SkipItemCount` and `MaximumItemCount` from the base class:

```csharp
var extractor = new XmlSingleStreamExtractor<Person>(stream, logger);
extractor.SkipItemCount = 10;      // Skip first 10 records
extractor.MaximumItemCount = 100;   // Extract at most 100 records
```

### Progress reporting

All classes use `XmlReport` as their progress type. Override `CreateProgressReport` in a subclass for custom progress types:

```csharp
public class MyXmlExtractor : XmlSingleStreamExtractor<Person>
{
    public MyXmlExtractor(Stream stream, ILogger<MyXmlExtractor> logger)
        : base(stream, logger) { }
}
```

### new() constraint

All XML classes require `TRecord : new()` because `XmlSerializer` needs a public parameterless constructor. This is enforced at compile time. If your record type does not have a parameterless constructor, you will get a compiler error.
