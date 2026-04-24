# Custom Parsers & Converters

Each ETL library exposes one or more extension points that let you customize how values are read from the source or written to the destination. This page shows the shape of each library's customization hooks — enough for you to know they exist and what they can do. For concrete recipes (currency handling, enum mappings, custom date formats, and so on), see each library's own README.


## FixedWidth customization

The FixedWidth library takes user-supplied delegates in three places:

| Hook | Runs when | Typical use |
|------|-----------|-------------|
| **Value parser** (extractor) | Converting a field's raw text into its CLR value | Y/N booleans, enum codes, currency symbols, per-field date formats |
| **Value converter** (loader) | Converting a CLR value into its field text | Y/N booleans, currency formatting, enum-to-code |
| **Header converter** (loader) | Formatting the header label written above each column | Upper-case, centred, padded labels |

Each hook is a delegate assigned to a property on the extractor or loader. The library calls your delegate once per field with context (property type, field width, alignment, etc.) and falls back to the built-in parser/converter when your delegate returns `null`.

Quick example — a value parser that handles `Y`/`N` booleans and defers to the default parser for every other type:

```csharp
extractor.ValueParser = (text, ctx) =>
    ctx.PropertyType == typeof(bool)
        ? (object)(text.Span.SequenceEqual("Y".AsSpan()))
        : FixedWidthConverter.DefaultParser(text, ctx);
```

For recipes (enum mapping, currency handling, per-field date formats, custom header layouts), see the [ETL-FixedWidth README](https://github.com/Chris-Wolfgang/ETL-FixedWidth#readme).


## JSON customization

The JSON library extractors and loaders accept an optional serializer-options object that controls property naming, null handling, indentation, custom converters, and so on. Pass it to the constructor or assign it via the `SerializerOptions` property.

Quick example — camel-case JSON keys with indentation when writing:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
};

var loader = new JsonLineLoader<MyRecord>(stream, logger)
{
    SerializerOptions = options,
};
```

For the full set of customizations (custom converters, enum string handling, source-generated serialization, etc.), see the [ETL-Json README](https://github.com/Chris-Wolfgang/ETL-Json#readme) and the standard `System.Text.Json` documentation.


## XML customization

The XML library components accept optional reader-settings and writer-settings objects for controlling whitespace handling, indentation, encoding, and DTD processing. Pass them to the constructor.

Quick example — pretty-printed output with two-space indentation:

```csharp
var writerSettings = new XmlWriterSettings
{
    Indent = true,
    IndentChars = "  ",
};

var loader = new XmlSingleStreamLoader<MyRecord>(stream, writerSettings, logger);
```

For the full set of settings, see the [ETL-Xml README](https://github.com/Chris-Wolfgang/ETL-Xml#readme) and the standard `System.Xml` documentation.


## Composing multiple custom rules

When you need more than one custom rule inside a single hook — for example, a FixedWidth value parser that handles Y/N booleans, a custom date format on one field, and an enum lookup — use a chain of conditions inside the delegate:

```csharp
extractor.ValueParser = (text, ctx) =>
{
    // Rule 1: Y/N booleans
    if (ctx.PropertyType == typeof(bool))
    {
        return (object)(text.Span.SequenceEqual("Y".AsSpan()));
    }

    // Rule 2: Custom date format for a specific field
    if (ctx.PropertyName == "BirthDate" && ctx.PropertyType == typeof(DateTime))
    {
        return DateTime.ParseExact
        (
            text.ToString(),
            "dd/MM/yyyy",
            CultureInfo.InvariantCulture
        );
    }

    // Rule 3: Enum code mapping
    if (ctx.PropertyType == typeof(AccountStatus))
    {
        var trimmed = text.Span.Trim();
        if (trimmed.SequenceEqual("A".AsSpan())) return AccountStatus.Active;
        if (trimmed.SequenceEqual("I".AsSpan())) return AccountStatus.Inactive;
        return AccountStatus.Closed;
    }

    // Fallback: default parser
    return FixedWidthConverter.DefaultParser(text, ctx);
};
```

The same shape works wherever a library takes a user-supplied delegate (FixedWidth value parsers and converters, custom `JsonConverter<T>` instances, and so on): match on the context, apply your rule, fall through to the library's default.


## See Also

- [Pipeline Composition](pipeline-composition.md) — wiring extractors, transformers, and loaders into complete pipelines
- [Error Handling](error-handling.md) — what happens when a parser or converter throws
- [Wolfgang.Etl.FixedWidth](../libraries/fixedwidth.md), [Wolfgang.Etl.Json](../libraries/json.md), [Wolfgang.Etl.Xml](../libraries/xml.md) — library catalog entries
