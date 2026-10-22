# Transform Chains

Compose several small transformers into one stage instead of writing a single transformer that does everything. The example normalises and validates postal addresses coming out of an ERP export, using a shared `Address` type in the middle so the validation step can be reused by every pipeline that handles addresses.

```
ErpAddress --(to Address)--> Address --(drop invalid)--> Address --(normalise)--> Address --(to ErpAddress)--> ErpAddress
```

All the building blocks come from one package:

```bash
dotnet add package Wolfgang.Etl.Transformers
```


## The types

```csharp
// The ERP's own shape (what the extractor yields and the loader writes)
public record ErpAddress
{
    public string CustNo { get; init; } = "";
    public string Addr1 { get; init; } = "";
    public string City { get; init; } = "";
    public string Zip { get; init; } = "";
}



// The shared, application-wide shape the reusable steps work on
public record Address
{
    public string CustomerId { get; init; } = "";
    public string Street { get; init; } = "";
    public string City { get; init; } = "";
    public string PostalCode { get; init; } = "";
}
```


## Building the chain

Each step is one transformer from `Wolfgang.Etl.Transformers`. `Then(...)` joins two transformers whose types line up into a single `ITransformAsync<TSource, TDestination>`; the compiler checks every link.

```csharp
using Wolfgang.Etl.Abstractions;
using Wolfgang.Etl.Transformers;

public static class AddressChain
{
    public static ITransformAsync<ErpAddress, ErpAddress> Create()
    {
        // ERP shape -> shared shape (application-specific, small)
        var fromErp = new SelectTransformer<ErpAddress, Address>
        (
            erp => new Address
            {
                CustomerId = erp.CustNo,
                Street = erp.Addr1,
                City = erp.City,
                PostalCode = erp.Zip,
            }
        );

        // Reusable: drop addresses that cannot be delivered to
        var dropInvalid = new WhereTransformer<Address>
        (
            address => address.Street.Length > 0 && address.PostalCode.Length > 0
        );

        // Reusable: normalise casing and whitespace
        var normalise = new SelectTransformer<Address, Address>
        (
            address => address with
            {
                Street = address.Street.Trim(),
                City = address.City.Trim().ToUpperInvariant(),
                PostalCode = address.PostalCode.Replace(" ", ""),
            }
        );

        // Shared shape -> ERP shape (application-specific, small)
        var toErp = new SelectTransformer<Address, ErpAddress>
        (
            address => new ErpAddress
            {
                CustNo = address.CustomerId,
                Addr1 = address.Street,
                City = address.City,
                Zip = address.PostalCode,
            }
        );

        return fromErp
            .Then(dropInvalid)
            .Then(normalise)
            .Then(toErp);
    }
}
```

The result is used like any other transformer:

```csharp
var transformer = AddressChain.Create();

await loader.LoadAsync
(
    transformer.TransformAsync(extractor.ExtractAsync(token)),
    token
);
```

`Then(...)` builds a `ChainTransformer<TSource, TIntermediate, TDestination>` for each link. Construct `ChainTransformer` directly when you prefer explicit types, and use `ChainTransformerWithCancellation` (or the `Then` overload for `ITransformWithCancellationAsync` pairs) when every stage in the chain accepts a `CancellationToken`.


## Why chain instead of one big transformer

- **Reuse.** `dropInvalid` and `normalise` work on `Address`, so a CRM or billing pipeline only writes its own two shape converters and reuses the middle of the chain.
- **Testing.** Each step is tested on its own with plain input and expected output; the chain needs only an end-to-end check.
- **Streaming is kept.** Every link pulls one item at a time from the previous one, so the chain buffers nothing that a single transformer would not.


## Related

- [Your First Transformer](../getting-started/your-first-transformer.md) - writing a custom transformer when no built-in one fits
- [Pipeline Composition](../guides/pipeline-composition.md) - how stages are wired together
- [Transformer reference](../reference/transformer.md) - the transformer contract
