# Wolfgang.Etl.FixedWidth

Attribute-based fixed-width file extractor and loader for the ETL Framework.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.FixedWidth` |
| **Version** | 0.1.0 |
| **Dependencies** | Wolfgang.Etl.Abstractions 0.10.2, System.Memory, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-FixedWidth](https://github.com/Chris-Wolfgang/ETL-FixedWidth) |

## Overview

Wolfgang.Etl.FixedWidth reads and writes fixed-width text files using attribute-decorated POCOs. Fields are defined with `[FixedWidthField]` attributes that specify column index and width. The library supports headers, separators, delimiters, custom value parsing, blank-line handling, malformed-line policies, and line filtering.

## Attributes

### FixedWidthFieldAttribute

Marks a property as a fixed-width field. Applied to properties on the record POCO.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `index` | `int` | Yes | -- | Zero-based column index. Must be unique across all field and skip attributes. |
| `length` | `int` | Yes | -- | Number of characters this field occupies. |
| `Header` | `string?` | No | Property name | Column header label for output. |
| `Alignment` | `FieldAlignment` | No | `Left` | Padding alignment during writes. |
| `Pad` | `char` | No | `' '` | Padding character. |
| `Format` | `string?` | No | `null` | Format string for DateTime, numeric, or custom conversions. |
| `TrimValue` | `bool` | No | `true` | Whether to trim whitespace from extracted values. |

### FixedWidthSkipAttribute

Declares a column in the file to skip during extraction. Does not map to a property. Multiple instances can be stacked on the same property to skip several consecutive columns.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `index` | `int` | Yes | -- | Zero-based column index. |
| `length` | `int` | Yes | -- | Number of characters occupied by the skipped column. |
| `Message` | `string?` | No | `null` | Documentation-only description of the skipped column. |

## Enums

### FieldAlignment

| Value | Description |
|---|---|
| `Left` | Value is left-aligned, padded on the right. Default for string fields. |
| `Right` | Value is right-aligned, padded on the left. Typical for numeric fields. |

### BlankLineHandling

| Value | Description |
|---|---|
| `ThrowException` | (Default) Throws `LineTooShortException`. |
| `Skip` | Blank lines are invisible to all counting logic. |
| `ReturnDefault` | Yields a default instance of the record type. |

### MalformedLineHandling

| Value | Description |
|---|---|
| `ThrowException` | (Default) Throws a `MalformedLineException`. |
| `Skip` | Skips the line, increments `CurrentSkippedItemCount`. |
| `ReturnDefault` | Yields a default instance; discards partial parse state. |

### LineAction

Returned by the `LineFilter` delegate to control line processing.

| Value | Description |
|---|---|
| `Process` | Parse the line normally. |
| `Skip` | Skip the line without parsing. Invisible to counting logic. |
| `Stop` | Stop reading immediately; end the async stream. |

## FixedWidthExtractor

`FixedWidthExtractor<TRecord, TProgress>` inherits `ExtractorBase<TRecord, TProgress>` and implements `IDisposable`.

**Type constraints:** `TRecord : notnull, new()` and `TProgress : notnull`.

### Constructors

```csharp
// TextReader-based (caller owns the reader lifetime):
public FixedWidthExtractor
(
    TextReader reader,
    ILogger<FixedWidthExtractor<TRecord, TProgress>>? logger = null
);

// Stream-based (64 KB internal buffer for throughput):
public FixedWidthExtractor
(
    Stream stream,
    ILogger<FixedWidthExtractor<TRecord, TProgress>>? logger = null
);
```

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `HasHeader` | `bool` | `false` | When `true`, sets `HeaderLineCount` to 1. |
| `HeaderLineCount` | `int` | `0` | Number of header lines to skip before data. |
| `FieldSeparator` | `char?` | `null` | When set with headers, skips one separator line after header lines. |
| `FieldDelimiter` | `string?` | `null` | Delimiter between fields (e.g. `" \| "`). Must match the loader. |
| `BlankLineHandling` | `BlankLineHandling` | `ThrowException` | Policy for blank lines. |
| `MalformedLineHandling` | `MalformedLineHandling` | `ThrowException` | Policy for parse failures. |
| `ValueParser` | `FixedWidthValueParser` | `FixedWidthConverter.DefaultParser` | Custom value parsing delegate. |
| `LineFilter` | `Func<string, LineAction>` | Always returns `Process` | Pre-parse line filter. |
| `CurrentLineNumber` | `long` | -- | 1-based physical line number (thread-safe via `Interlocked`). |

### Basic extraction example

```csharp
public class CustomerRecord
{
    [FixedWidthField(0, 20, Header = "First Name")]
    public string FirstName { get; set; }

    [FixedWidthField(1, 20, Header = "Last Name")]
    public string LastName { get; set; }

    [FixedWidthField(2, 5, Alignment = FieldAlignment.Right, Pad = '0')]
    public int Age { get; set; }
}

await using var stream = File.OpenRead("customers.txt");
using var extractor = new FixedWidthExtractor<CustomerRecord, FixedWidthReport>(stream);
extractor.HasHeader = true;
extractor.FieldSeparator = '-';

await foreach (var customer in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine($"{customer.FirstName} {customer.LastName}, age {customer.Age}");
}
```

## FixedWidthLoader

`FixedWidthLoader<TRecord, TProgress>` inherits `LoaderBase<TRecord, TProgress>` and implements `IDisposable`.

**Type constraints:** `TRecord : notnull` and `TProgress : notnull`.

### Constructors

```csharp
// TextWriter-based (caller owns the writer):
public FixedWidthLoader
(
    TextWriter writer,
    ILogger<FixedWidthLoader<TRecord, TProgress>>? logger = null
);

// Stream-based (64 KB internal buffer):
public FixedWidthLoader
(
    Stream stream,
    ILogger<FixedWidthLoader<TRecord, TProgress>>? logger = null
);
```

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `WriteHeader` | `bool` | `false` | Write a header line before data records. |
| `FieldSeparator` | `char?` | `null` | Character for the separator line after the header. |
| `FieldDelimiter` | `string?` | `null` | String inserted between fields on every line. |
| `ValueConverter` | `Func<object, FieldContext, string>` | `FixedWidthConverter.Strict` | Converts field values to strings. |
| `HeaderConverter` | `Func<string, FieldContext, string>` | `FixedWidthConverter.StrictHeader` | Converts header labels to strings. |
| `CurrentLineNumber` | `long` | -- | 1-based physical line number of the last line written (thread-safe). |

### Basic loading example

```csharp
await using var stream = File.Create("output.txt");
using var loader = new FixedWidthLoader<CustomerRecord, FixedWidthReport>(stream);
loader.WriteHeader = true;
loader.FieldSeparator = '-';
loader.FieldDelimiter = " | ";

await loader.LoadAsync(customerStream, cancellationToken);
```

### Console table output

```csharp
var loader = new FixedWidthLoader<CustomerRecord, Report>(Console.Out);
loader.WriteHeader = true;
loader.FieldSeparator = '-';
loader.FieldDelimiter = " | ";

await loader.LoadAsync(customerStream, cancellationToken);
// Output:
// First Name           | Last Name            |   Age
// -------------------- | -------------------- | -----
// John                 | Smith                |    42
```

## FixedWidthReport

Extends `Report` with fixed-width-specific progress data.

| Property | Type | Description |
|---|---|---|
| `CurrentCount` | `int` | Number of records processed (inherited). |
| `CurrentSkippedItemCount` | `int` | Number of records skipped. |
| `CurrentLineNumber` | `long` | 1-based physical line number being processed. |

## FixedWidthConverter

Static class providing built-in converter and parser functions.

### Value converters (loader)

| Function | Behavior |
|---|---|
| `Strict` | (Default) Throws `FieldOverflowException` if the converted string exceeds field width. |
| `Truncate` | Silently truncates values that exceed field width. |

### Header converters (loader)

| Function | Behavior |
|---|---|
| `StrictHeader` | (Default) Throws `FieldOverflowException` if the header label exceeds field width. |
| `TruncateHeader` | Silently truncates headers that exceed field width. |

### Value parser (extractor)

| Function | Behavior |
|---|---|
| `DefaultParser` | Parses strings, numerics, DateTime (requires Format), booleans, and nullable types. Uses `TypeDescriptor` on netstandard2.0 and span-based parsing on net8.0+. |

### Custom parser example

```csharp
extractor.ValueParser = (text, ctx) =>
    ctx.PropertyType == typeof(bool)
        ? (object)(text.Span.SequenceEqual("Y".AsSpan()))
        : FixedWidthConverter.DefaultParser(text, ctx);
```

## FixedWidthValueParser delegate

```csharp
public delegate object FixedWidthValueParser(ReadOnlyMemory<char> text, FieldContext context);
```

The `text` parameter is a zero-copy slice of the source line. Access characters via `text.Span` for zero-allocation processing. Call `text.ToString()` only when a `string` allocation is needed.

## FieldContext

Immutable metadata passed to value parsers and converters.

| Property | Type | Description |
|---|---|---|
| `PropertyName` | `string` | Name of the property being converted/parsed. |
| `PropertyType` | `Type` | CLR type of the target property. |
| `FieldLength` | `int` | Maximum character width from the attribute. |
| `Pad` | `char` | Padding character. |
| `Alignment` | `FieldAlignment` | Padding alignment. |
| `Format` | `string?` | Format string from the attribute, or null. |
| `HeaderLabel` | `string` | Header label (from `Header` property or property name). |

## Exceptions

All parsing exceptions derive from `MalformedLineException`, which provides `LineNumber` and `LineContent` properties.

| Exception | Description | Extra Properties |
|---|---|---|
| `MalformedLineException` | Abstract base for parse failures. | `LineNumber`, `LineContent` |
| `LineTooShortException` | Line is shorter than the total field width. | `ExpectedWidth`, `ActualWidth` |
| `FieldConversionException` | A field value cannot be converted to the property type. | `FieldName`, `ExpectedType`, `RawValue` |
| `FieldOverflowException` | A converter returned a string wider than field width (loader). | `PropertyName`, `FieldLength`, `ActualLength` |

## Skipping columns

Use `FixedWidthSkipAttribute` to declare columns in the file that have no corresponding property:

```csharp
public class EmployeeRecord
{
    [FixedWidthField(0, 10)]
    public string FirstName { get; set; }

    // Skip DOB (8 chars) and HireDate (8 chars):
    [FixedWidthSkip(1, 8, Message = "DOB")]
    [FixedWidthSkip(2, 8, Message = "HireDate")]
    [FixedWidthField(3, 5)]
    public string EmployeeNumber { get; set; }
}
```

## Performance notes

- **Compiled delegates**: field accessors are compiled once per record type and cached.
- **ReadOnlyMemory zero-copy parsing**: field slicing avoids string allocations during extraction.
- **Span-based numerics on net8.0+**: numeric types are parsed directly from `ReadOnlySpan<char>`, bypassing `TypeDescriptor` and the `ToString()` allocation.
- **64 KB I/O buffer**: Stream-based constructors use a 64 KB buffer (vs. the default 1 KB) to reduce syscall overhead on large files.
- **Synchronous ReadLine**: extraction uses synchronous `ReadLine` to avoid async state machine overhead per line while still exposing `IAsyncEnumerable`.

## Examples

See the [examples directory](https://github.com/Chris-Wolfgang/ETL-FixedWidth/tree/main/examples/) in the ETL-FixedWidth repository for runnable samples.
