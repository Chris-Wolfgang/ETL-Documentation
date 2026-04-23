# Wolfgang.Etl.FixedWidth

Attribute-based fixed-width file extractor and loader.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.FixedWidth` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.FixedWidth) |
| **Dependencies** | Wolfgang.Etl.Abstractions, System.Memory, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-FixedWidth](https://github.com/Chris-Wolfgang/ETL-FixedWidth) |

## What it does

Reads and writes fixed-width text files — each line has a known layout of columns by character position. Columns are declared directly on your record type with `[FixedWidthField]` attributes that specify the column index and character width. Columns with no corresponding property are declared with `[FixedWidthSkip]`.

Supports headers, separator lines, field delimiters, custom value parsing, blank-line and malformed-line policies, and pre-parse line filtering (e.g. skip comment lines, stop at a footer marker).

## When to use it

- **Fixed-width flat files** — mainframe exports, legacy banking formats, COBOL-era data files, report printouts
- **Columnar ASCII output** — formatting tabular data for human-readable reports or console output
- When you need **fine-grained parse control** — per-column alignment, padding, custom value parsers, malformed-line policies
- When you need **line filtering** — skipping comment lines or stopping at footer sentinels before parsing

## When to use something else

- For CSV / TSV / other delimited formats, use a CSV library (a `Wolfgang.Etl.Csv` package is planned)
- For truly free-form text, use a regex-based parser or a dedicated parser library
- For binary fixed-record formats, use a binary serializer — this library works at the character level

## Quick example

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

await foreach (var customer in extractor.ExtractAsync(cancellationToken))
{
    // ...
}
```

## More detail

- **API reference** — see the [ETL-FixedWidth README](https://github.com/Chris-Wolfgang/ETL-FixedWidth#readme) and XML doc comments on the public types
- **Examples** — see the [`examples/` folder](https://github.com/Chris-Wolfgang/ETL-FixedWidth/tree/main/examples) in the source repo
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-FixedWidth/releases)
