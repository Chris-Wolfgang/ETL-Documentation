# Reading and Writing Compressed Files and Streams

Because every extractor and loader in this framework accepts a `Stream`, compression is applied at the call site by wrapping the underlying stream -- the extractor and loader themselves stay format-specific and never need to know the bytes are compressed.

This page shows how to read and write the three compression formats built into .NET (`GZipStream`, `BrotliStream`, `DeflateStream`) with any extractor or loader.

## Why This Works Without Changing the Extractor or Loader

An extractor reads bytes from a `Stream` and deserializes them into items. A loader serializes items into bytes and writes them to a `Stream`. Neither cares where the bytes on the other end came from.

`GZipStream`, `BrotliStream`, and `DeflateStream` are each a `Stream` themselves -- they read from or write to another stream underneath, transforming bytes as they go. Pass one of these wrappers to the extractor or loader instead of the raw file/memory stream, and compression happens transparently.

```
Caller       Wrapper stream          Underlying stream
 |                                         |
 v                                         v
 Extractor -> new GZipStream(fileStream, Decompress) -> File.OpenRead("data.jsonl.gz")
 Loader    -> new GZipStream(fileStream, Compress)   -> File.Create("out.jsonl.gz")
```

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
| `NoCompression` | No compression -- useful for testing the wrapper plumbing |
| `Fastest` | Lowest CPU cost, larger output |
| `Optimal` | Balanced; the default for most scenarios |
| `SmallestSize` | Lowest output size, highest CPU cost (not all formats support this level) |

For batch ETLs where throughput matters, `Fastest` is often enough -- the bottleneck is usually the extractor's source or the loader's destination, not the compressor.

For archival writes where the output is read rarely, `SmallestSize` (where supported) is worth the extra CPU.


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

Compression streams buffer output internally. Disposing the wrapper stream flushes its buffer to the underlying stream -- **before** the underlying stream is disposed. The `using` ordering in the examples above is correct; reversing it will truncate the output.

```csharp
// WRONG -- file stream disposes first, gzip buffer is lost
using var gzip = new GZipStream(File.Create("out.gz"), CompressionLevel.Optimal);

// RIGHT -- file stream disposes last, after gzip has flushed
using var fileStream = File.Create("out.gz");
using var gzip = new GZipStream(fileStream, CompressionLevel.Optimal);
```


## See Also

- [Building an Extractor](Building-an-Extractor) -- step-by-step extractor guide
- [Building a Loader](Building-a-Loader) -- step-by-step loader guide, including the "Why Streams, Not Files" rationale
