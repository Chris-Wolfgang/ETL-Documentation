# Description
The transformer is the second step in the ETL process, converting 
data from the source format, `TSource`, to the destination format, `TDestination`. 


## Requirements 
1. A transformer is usually **application-specific**, not generic.
   The mapping from `ErpCustomer` to `CrmAccount` is inherently
   particular to your ETL -- which fields map where, how values are
   translated between systems, what defaults apply when a field is
   missing. Trying to force a single generic transformer to cover
   these variations usually produces an abstraction that fits no
   real use case well.

   The pattern for reuse is not one giant generic transformer but
   many small focused ones that compose through shared intermediate
   types. For example, address validation is genuinely generic:

   ```
   ErpAddress --(ErpAddressToAddress)--> Address --(AddressValidator)--> Address --(AddressToErpAddress)--> ErpAddress
   CrmAddress --(CrmAddressToAddress)--> Address --(AddressValidator)--> Address --(AddressToCrmAddress)--> CrmAddress
   ```

   The `AddressValidator` in the middle is reusable across every
   ETL that works with addresses. The shape-conversion transformers
   on either side are application-specific and small enough to be
   easy to write and test.

   Use `MultistepTransformer` to compose a chain like this into a
   single transformer you can hand to the loader -- see
   [Building a Complete ETL](Building-a-Complete-ETL#chaining-transformers).

   > Generic transformers that are broadly reusable (validators,
   > format converters, pass-through / no-op) live in
   > `Wolfgang.Etl.Transformers`. This package is under active
   > development -- see [Chris-Wolfgang/ETL-Transformers](https://github.com/Chris-Wolfgang/ETL-Transformers).
1. Each `transformer<TSource, TDestination>` should be interchangeable 
   for any other `transformer<TSource, TDestination>`, where TSource and 
   TDestination are of the same type. This allows your application to 
   swap transformors at compile time or runtime making your ETL more flexible.
1. A transformer is responsible for handling any exceptions that 
   may occur during the transformation process, including but not limited 
   to invalid values and missing values.
1. The data to transform is received as an `asynchronous` stream of type 
   `TSource`, transformed, and returned as an `asynchronous` stream of type 
   TDestination, allowing for efficient processing by any additional 
   transformer(s) and the loader.
1. The transformer is decoupled from the source and destination data, 
   meaning it does not need to know where the data is coming from or 
   where it is going. It only needs to know how to transform the data 
   from one type to another. This means the the transformer can be easily 
   tested using unit tests and does not need any integration tests to 
   verify its functionality.
  

# Interfaces

| Interface | Description |
|-----------|-------------|
|[ITransformAsync](#ITransformAsync) | Provides a method to transform data asynchronously from the source to destination. |
|[ITransformWithCancellationAsync](#ITransformWithCancellationAsync) | Provides a method to transform data asynchronously from the source to destination with the ability to cancel the operation. |
|[ITransformWithProgressAsync](#ITransformWithProgressAsync) | Provides a method to transform data asynchronously from the source to destination while reporting progress. |
|[ITransformWithProgressAndCancellationAsync](#ITransformWithProgressAndCancellationAsync) | Provides a method to transform data asynchronously from the source to destination while reporting progress and allowing cancellation of the operation. |


## ITransformAsync

### Type Parameters
`TSource` The type of elements being passed into the transformer. This represents your source data.

`TDestination` The type of elements being returned. This represents your destination or target data.

### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>)` | Transforms data from the source format to the destination format asynchronously |


### TransformAsync\<TSource,TDestination\>
Transforms data from the source format to the destination format by enumerating it asynchronously.

#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

#### Returns
[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)
 An asynchronous stream of items of type TDestination, representing the transformed data.

**Introduced in:** 0.4.0

    
## ITransformWithCancellationAsync

### Type Parameters
`TSource` The type of elements being passed in. This represents your source data.

`TDestination` The type of elements being returned. This represents your destination or target data.


### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, CancellationToken cancellationToken)` | Transforms data from the source format to the destination format asynchronously with the ability to cancel |


### TransformAsync\<TSource, TDestination\>
Transforms data from the source format to the destination format by enumerating it asynchronously.


#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the transformed data.

**Introduced in:** 0.4.0

    

## ITransformWithProgressAsync\<TSource, TDestination, TProgress\>

### Type Parameters
`TSource` The type of elements being passed in. This represents your source data.

`TDestination` The type of elements being returned. This represents your destination or target data.

`TProgress` The type of object being used to report progress

### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, IProgress<TProgress> progress)` | Transforms data from the source format to the destination format asynchronously reporting progress |


#### TransformAsync\<TSource, TDestination\>
Transforms data from the source format to the destination format by enumerating it asynchronously.

#### Parameters

`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the transformer

#### Returns
[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the transformed data.

**Introduced in:** 0.4.0

    
## ITransformWithProgressAndCancellationAsync\<TSource, TDestination, TProgress\>

### Type Parameters
`TSource` The type of elements being passed in. This represents your source data.

`TDestination` The type of elements being returned. This represents your destination or target data.

`TProgress` The type of object being used to report progress

#### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Transforms data from the source format to the destination format asynchronously reporting progress with the ability to cancel |


#### TransformAsync\<TSource, TDestination, TProgress\>
Transforms data from the source format to the destination format by enumerating it asynchronously.

#### Parameters

`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the transformer

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.


#### Returns
[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the transformed data.

**Introduced in:** 0.4.0


# Base Classes

## TransformorBasee\<TSource, TDestination, TProgress\> 

### Definition

Provides an abstract base class that implements 
[`ITransformAsync`](#ITransformAsync), 
[`ITransformWithCancellationAsync`](#ITransformWithCancellationAsync), 
[`ITransformWithProgressAsync`](#ITransformWithProgressAsync), and
[`ITransformWithProgressAndCancellationAsync`](#ITransformWithProgressAndCancellationAsync) 
with basic functionality allowing for quickly building user defined 
transformers. This abstract class only requires that you implment two methods 
[`CreateProgressReport`](#CreateProgressReport) and 
[`TransformWorkerAsync`](#TransformWorkerAsync)  


### Type Parameters
`TSource` The type of elements being passed in. This represents your source data.

`TDestination` The type of elements being returned. This represents your destination or target data.
 
`TProgress` The type of object being used to report progress


### Constructors
| Method | Description |
|--------|-------------|
|`TransformorBase()`|Initializes a new instance of the class


### Properties

| Property | Description |
|----------|-------------|
|`CurrentItemCount`|The current number of items that have been transformed|
|`MaximumItemCount`|The maximum number of items to transform before exiting and reporting complete|
|`ReportingInterval`|The number of milliseconds between progress reports|
|`SkipItemCount`|The number of items to skip before before returning items|


### CurrentItemCount
The current number of items that have been transformed. This property is updated as items are transformed and can be used to report progress.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The current number of items that have been transformed.

#### Default Value
0 

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### MaximumItemCount
The maximum number of items to be transformed before exiting and reporting complete. This property can be used to limit the number of items transformed, which is useful for testing or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The maximum number of items to been transform.

#### Default Value
[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) This means that by default there is no limit on the number of items transformed.

>If you are trying to transform a large number of items, you are limited to 
the maximum value of a 32-bit integer, which is 2,147,483,647. If you need 
to transform more than this number of items, you will need to implement your own logic 
to handle this.

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### ReportingInterval

The number of milliseconds between progress reports. 
This property can be used to control how often progress is reported, 
which can be useful for performance tuning or to reduce the 
frequency of updates in a UI.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The number of milliseconds between progress reports.

#### Default Value
1000 (1 second)

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 1.

**Introduced in:** 0.4.0


### SkipItemCount
The number of items to skip before returning items.
This property can be used to limit the number of items transformed, 
which is useful for testing or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The number of items to skip.

#### Default Value
0 

[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) This means that by default there is no limit on the number of items transformed.

>If you are trying to transform a large number of items, you are limited to 
the maximum value of a 32-bit integer, which is 2,147,483,647. If you need 
to transform more than this number of items, you will need to implement your own logic 
to handle this.

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0




### Methods

| Method | Description |
|--------|-------------|
| `CreateProgressReport()` | (abstract) Creates a progress report object of type TProgress to be used for reporting progress |
| `TransformAsync(IAsyncEnumerable<TSource> items)` | Transforms data from the source format to the destination format asynchronously |
| `TransformAsync(IAsyncEnumerable<TSource> items, CancellationToken cancellationToken)` | Transforms data from the source format to the destination format asynchronously with the ability to cancel |
| `TransformAsync(IAsyncEnumerable<TSource> items , IProgress<TProgress> progress)` | Transforms data from the source format to the destination format asynchronously reporting progress |
| `TransformAsync(IAsyncEnumerable<TSource> items, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Transforms data from the source format to the destination format asynchronously reporting progress with the ability to cancel |
| `TransformWorkerAsync(IAsyncEnumerable<TSource> items, CancellationToken token)`| (abstract) The worker method that performs the actual transformation of data asynchronously. This method should be implemented by derived classes to provide the specific transformation logic.



### TransformAsync(IAsyncEnumerable\<TSource\> items)
Transforms data from the source format to the destination format by enumerating it asynchronously.

#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.



#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the transformed data.


### TransformAsync(IAsyncEnumerable\<TSource\> items, CancellationToken cancellationToken)

Transforms data from the source format to the destination format by enumerating it asynchronously with the ability to cancel.

#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the transformed data.


### TransformAsync(IAsyncEnumerable\<TSource\> items, IProgress\<TProgress\> progress)

Transforms data from the source format to the destination format by enumerating it asynchronously periodically reporting progress.

#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the transformer

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the transformed data.

#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.


### TransformAsync(IAsyncEnumerable\<TSource\> items, IProgress\<TProgress\> progress, CancellationToken cancellationToken)

#### Parameters
`items` [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the data to be transformed.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the transformer

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the transformed data.

#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0
