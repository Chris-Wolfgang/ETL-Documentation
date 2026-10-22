# Reading and Writing Compressed Files and Streams

Because every extractor and loader in this framework accepts a `Stream`, compression is applied at the call site by wrapping the underlying stream — the extractor and loader themselves stay format-specific and never need to know the bytes are compressed.

This page shows how to read and write the three compression formats built into .NET (`GZipStream`, `BrotliStream`, `DeflateStream`) with any extractor or loader.

## Why This Works Without Changing the Extractor or Loader

An extractor reads bytes from a `Stream` and deserializes them into items. A loader serializes items into bytes and writes them to a `Stream`. Neither cares where the bytes on the other end came from.

`GZipStream`, `BrotliStream`, and `DeflateStream` are each a `Stream` themselves — they read from or write to another stream underneath, transforming bytes as they go. Pass one of these wrappers to the extractor or loader instead of the raw file/memory stream, and compression happens transparently.

![Caller wraps an underlying stream with GZipStream and hands the wrapper to the extractor or loader](../assets/compressed-streams-flow-light.svg#only-light)
![Caller wraps an underlying stream with GZipStream and hands the wrapper to the extractor or loader](../assets/compressed-streams-flow-dark.svg#only-dark)

The extractor and loader were built to work with `Stream`. The wrapper *is* a `Stream`. No changes needed.


## Reading Compressed Files

### gzip

```csharp
using var fileStream = File.OpenRead("data.jsonl.gz");
using var gzip = new GZipStream(fileStream, CompressionMode.Decompress);

var extractor = new JsonLineExtractor<Person>
(
    gzip,
    logger
);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    // ...
}
```

### Brotli

```csharp
using var fileStream = File.OpenRead("data.jsonl.br");
using var brotli = new BrotliStream(fileStream, CompressionMode.Decompress);

var extractor = new JsonLineExtractor<Person>(brotli, logger);
```

### Deflate

```csharp
using var fileStream = File.OpenRead("data.jsonl.deflate");
using var deflate = new DeflateStream(fileStream, CompressionMode.Decompress);

var extractor = new JsonLineExtractor<Person>(deflate, logger);
```


## Other Formats

The examples above use JSON Lines, but nothing about them is JSON-specific. Every extractor and loader that takes a `Stream` works the same way.

### JSON array (single document)

```csharp
using var fileStream = File.OpenRead("people.json.gz");
using var gzip = new GZipStream(fileStream, CompressionMode.Decompress);

var extractor = new JsonSingleStreamExtractor<Person>(gzip);
```

### XML (single document with many records)

```csharp
using var fileStream = File.OpenRead("orders.xml.gz");
using var gzip = new GZipStream(fileStream, CompressionMode.Decompress);

var extractor = new XmlSingleStreamExtractor<Order>(gzip);
```

### XML (one compressed file per record)

`XmlMultiStreamExtractor<T>` reads one record from each stream in a sequence and disposes each stream after reading it, so the files can be opened lazily as the extractor reaches them:

```csharp
var streams = Directory
    .EnumerateFiles("inbox", "*.xml.gz")
    .Select(path => (Stream)new GZipStream(File.OpenRead(path), CompressionMode.Decompress));

var extractor = new XmlMultiStreamExtractor<Order>(streams);
```

### Fixed-width (including EBCDIC mainframe exports)

Mainframe-to-cloud migrations commonly ship fixed-width files that are both compressed and EBCDIC-encoded. Decompress with the wrapper stream and decode with the code page; the two concerns are independent:

```csharp
// .NET Core / .NET 5+ only resolve EBCDIC code pages once this provider is registered
// (.NET Framework has them built in)
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);

using var fileStream = File.OpenRead("accounts.dat.gz");
using var gzip = new GZipStream(fileStream, CompressionMode.Decompress);

var extractor = new FixedWidthExtractor<AccountRecord>(gzip, Encoding.GetEncoding("IBM037"));
```


## Writing Compressed Files

### gzip

```csharp
using var fileStream = File.Create("output.jsonl.gz");
using var gzip = new GZipStream(fileStream, CompressionLevel.Optimal);

var loader = new JsonLineLoader<Contact>(gzip, logger);

await loader.LoadAsync
(
    transformer.TransformAsync(extractor.ExtractAsync(ct), ct),
    ct
);
```

### Brotli

```csharp
using var fileStream = File.Create("output.jsonl.br");
using var brotli = new BrotliStream(fileStream, CompressionLevel.Optimal);

var loader = new JsonLineLoader<Contact>(brotli, logger);
```

### Deflate

```csharp
using var fileStream = File.Create("output.jsonl.deflate");
using var deflate = new DeflateStream(fileStream, CompressionLevel.Optimal);

var loader = new JsonLineLoader<Contact>(deflate, logger);
```


## Choosing a Compression Level

`CompressionLevel` trades off CPU time against output size:

| Level | Behavior |
|-------|----------|
| `NoCompression` | No compression — useful for testing the wrapper plumbing |
| `Fastest` | Lowest CPU cost, larger output |
| `Optimal` | Balanced; the default for most scenarios |
| `SmallestSize` | Lowest output size, highest CPU cost (not all formats support this level) |

For batch ETLs where throughput matters, `Fastest` is often enough — the bottleneck is usually the extractor's source or the loader's destination, not the compressor.

For archival writes where the output is read rarely, `SmallestSize` (where supported) is worth the extra CPU.


## Choosing a Compression Format

| Format | .NET class | Best for |
|--------|------------|----------|
| gzip | `GZipStream` | Widest compatibility: `.gz` files, HTTP `Content-Encoding: gzip`, object stores such as S3 |
| Brotli | `BrotliStream` | Better compression ratio than gzip; built in from .NET Core 2.1 (not available on .NET Framework) |
| Deflate | `DeflateStream` | Raw deflate data without the gzip header; rarely used on its own |
| Zstandard | Not in the BCL — community packages | The best speed / ratio trade-off of the four, if both ends can use it |

Any of these that exposes itself as a `Stream` plugs in exactly like the examples above.


## Reading a Compressed HTTP Response

Compression is not limited to files. If an HTTP endpoint returns a compressed body, wrap the response stream the same way:

```csharp
using var response = await httpClient.GetAsync
(
    "https://api.example.com/export.jsonl.gz",
    HttpCompletionOption.ResponseHeadersRead,
    cancellationToken
);

response.EnsureSuccessStatusCode();

await using var responseStream = await response.Content.ReadAsStreamAsync(cancellationToken);
using var gzip = new GZipStream(responseStream, CompressionMode.Decompress);

var extractor = new JsonLineExtractor<Person>(gzip, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    // Each person streamed from the server, decompressed on the fly,
    // and yielded without buffering the full response.
}
```

This is the full payoff of the stream-based design: a single extractor reads compressed data from a file *or* an HTTP response *or* an in-memory buffer, because the extractor does not know the difference.


## Important: Dispose in the Right Order

Compression streams buffer output internally, and the compressed data is only complete once the wrapper has been disposed (which writes the final block). Disposing the wrapper also disposes the underlying stream, unless you pass `leaveOpen: true`. So:

- **Dispose the wrapper before the underlying stream.** With `using` declarations that means declaring the underlying stream first, as every example on this page does; `using` disposes in reverse order.
- **Do not read the output until the wrapper is disposed.** If you wrap a `MemoryStream` and inspect it right after `LoadAsync` returns, the last block is still in the wrapper's buffer.
- **Use `leaveOpen: true` when you need the underlying stream afterwards**, for example to rewind a `MemoryStream` in a test:

```csharp
using var buffer = new MemoryStream();

using (var gzip = new GZipStream(buffer, CompressionLevel.Optimal, leaveOpen: true))
{
    var loader = new JsonLineLoader<Contact>(gzip);
    await loader.LoadAsync(contacts, cancellationToken);
}   // gzip disposed here: final block written, buffer still open

buffer.Position = 0;   // now safe to read the complete .gz bytes
```


## Tips

- **Compressed streams are not seekable.** `GZipStream`, `BrotliStream` and `DeflateStream` report `CanSeek == false`, so anything that seeks cannot work through them. For example, resuming a `JsonLineExtractor` from a checkpoint (`JsonLineExtractorOptions.StartByteOffset` greater than zero) throws `InvalidOperationException` ("Stream must be seekable ...") on a compressed stream. To resume, decompress to a seekable file first, or restart from the beginning and use `SkipItemCount`.
- **Compression costs CPU; decompression is cheap.** Decompressing is usually faster than reading the uncompressed bytes from disk or the network, so reading compressed input is often a net win. Writing compressed output costs CPU in proportion to the `CompressionLevel`; see [Choosing a Compression Level](#choosing-a-compression-level).
- **Buffer the underlying file stream, not the wrapper.** `File.OpenRead` / `File.Create` already buffer; if you build a `FileStream` yourself, give it a reasonably large buffer (64 KB is a good start for batch work).


## See Also

- [Your First Extractor](../getting-started/your-first-extractor.md) — step-by-step extractor guide
- [Your First Loader](../getting-started/your-first-loader.md) — step-by-step loader guide, including the "Why Streams, Not Files" rationale
