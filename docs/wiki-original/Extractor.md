# Description
The extractor is the first step in the ETL process, responsible for pulling data from a source, such as a database, CSV or JSON file, API, or web service. 


## Requirements 
1. An extractor should be implemented in a generic way, allowing it to 
   be reused for different types of ETL processes. For example, an
   extractor for a CSV file could be used to extract data from any CSV file, 
   regardless of its content. 
   ```csharp
   // JsonExtractor should be able to retrieve Foo and Bar
   var loader1 = new JsonExtractor<Foo>();`
   var loader2 = new JsonExtractor<Bar>();`
    ```
1. Each `Extractor<T>` should be interchangeable for any other `Extractor<T>`. 
   This allows your application to swap extractors at compile time or runtime 
   making your ETL more flexible.
   ```csharp
    // jsonExtractor and csvExtractor should be interchangble
    var jsonExtractor = new JsonExtractor<MyDataType>();
    var csvExtractor = new CsvExtractor<MyDataType>();
    ```
1. An extractor is responsible for handling any exceptions that 
   may occur during the extraction process, including but not limited 
   to network issues, data format errors, such as incomplete files, 
   or source unavailability. 
1. The extractor should be resilient and capable of retrying operations
   in case of transient failures. 
1. The extracted data is returned as an `asynchronous` stream of type 
   `TSource`, allowing for efficient processing by the transformer and 
   loader components of the ETL pipeline. Whenever possible, the 
   extractor should retrieve 1 item at a time. 

   >For performance reasons it may be necessary to extract 
   more than one item at a time and buffer them until they are requested.
   However, the extractor should not retrieve large amounts of data which 
   will consume large amounts of memory and other resources and delay 
   processing of the first item. For example, an extractor that loads
   a CSV file may find it more efficient to prefetch a few rows at a time,
   but should not load the entire file at once as some files may be gigabytes in size 
1. The extractor should **not** do any transformation of the data. 
   Transforming the data is the responsibility of the transformer.  
   Its sole responsibility is to retrieve data from the source and provide 
   it in its raw form to the transformer.
   >Occasionally, some minimal transformation may be necessary to 
   ensure the data is in a suitable format for further processing. 
   For example, a JSON extractor might read a file of JSON and deserialize 
   it into a list of objects, which would then be passed to the transformer 
   for further processing. An alternative approach would be to pass the JSON
   as a string to a JSON transformer, which would then deserialize it into 
   objects and pass it to the next transformer in the pipeline.

# Interfaces

| Interface | Description |
|-----------|-------------|
|[IExtractAsync](#IExtractAsync) | Provides a method to extract data asynchronously from a source. |
|[IExtractWithCancellationAsync](#IExtractWithCancellationAsync) | Provides a method to extract data asynchronously from a source with the ability to cancel the operation. |
|[IExtractWithProgressAsync](#IExtractWithProgressAsync) | Provides a method to extract data asynchronously from a source while reporting progress. |
|[IExtractWithProgressAndCancellationAsync](#IExtractWithProgressAndCancellationAsync) | Provides a method to extract data asynchronously from a source while reporting progress and allowing cancellation of the operation. |


## IExtractAsync

### Type Parameters
`TSource` The type of elements being returned. This represents your source data.

### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync()` | Extracts data from the source asynchronously |


### ExtractAsync\<TSource\>
Extracts data from the source by enumerating it asynchronously.

#### Parameters
none

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)
 An asynchronous stream of items of type TSource, representing the extracted data.

**Introduced in:** 0.4.0


    
## IExtractWithCancellationAsync

### Type Parameters
`TSource` The type of elements being returned. This represents your source data.

### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(CancellationToken cancellationToken)` | Extracts data from the source asynchronously with the ability to cancel |


### ExtractAsync\<TSource\>
Extracts data from the source by enumerating it asynchronously.

#### Parameters
`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.

**Introduced in:** 0.4.0

    

## IExtractWithProgressAsync\<TSource, TProgress\>

### Type Parameters
`TSource` The type of elements being returned. This represents your source data.

`TProgress` The type of object being used to report progress

### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(IProgress<TProgress> progress)` | Extracts data from the source asynchronously reporting progress |


#### ExtractAsync\<TSource\>
Extracts data from the source by enumerating it asynchronously.

#### Parameters
`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the extractor

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.

**Introduced in:** 0.4.0

    
## IExtractWithProgressAndCancellationAsync\<TSource, TProgress\>

### Type Parameters
`TSource` The type of elements being returned. This represents your source data.

`TProgress` The type of object being used to report progress

#### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(IProgress<TProgress> progress, CancellationToken cancellationToken)` | Extracts data from the source asynchronously reporting progress with the ability to cancel |


#### ExtractAsync\<TSource\>
Extracts data from the source by enumerating it asynchronously.

#### Parameters
`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the extractor

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.


#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.

**Introduced in:** 0.4.0


# Base Classes

## ExtractorBasee\<TSource, TProgress\> 

### Definition
Provides an abstract base class that implements 
[`IExtractAsync`](#IExtractAsync), 
[`IExtractWithCancellationAsync`](#IExtractWithCancellationAsync), 
[`IExtractWithProgressAsync`](#IExtractWithProgressAsync), and
[`IExtractWithProgressAndCancellationAsync`](#IExtractWithProgressAndCancellationAsync) 
with basic functionality allowing for quickly building user defined extractors. This abstract class only
requires that you implment two methods 
[`CreateProgressReport`](#CreateProgressReport) and 
[`ExtractWorkerAsync`](#ExtractWorkerAsync)  


### Type Parameters
`TSource` The type of elements being returned. This represents your source data.

`TProgress` The type of object being used to report progress


### Constructors
| Method | Description |
|--------|-------------|
|`ExtractorBase()`|Initializes a new instance of the class


### Properties

| Property | Description |
|----------|-------------|
|`CurrentItemCount`|The current number of items that have been extracted|
|`MaximumItemCount`|The maximum number of items to extract before exiting and reporting complete|
|`ReportingInterval`|The number of milliseconds between progress reports|
|`SkipItemCount`|The number of items to skip before before returning items|


### CurrentItemCount
The current number of items that have been extracted. This property is updated as items are extracted and can be used to report progress.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The current number of items that have been extracted.

#### Default Value
0 

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### MaximumItemCount
The maximum number of items to be extracted before exiting and reporting complete. This property can be used to limit the number of items extracted, which is useful for testing or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The maximum number of items to been extract.

#### Default Value
[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) This means that by default there is no limit on the number of items extracted.

>If you are trying to extract a large number of items, you are limited to 
the maximum value of a 32-bit integer, which is 2,147,483,647. If you need 
to extract more than this number of items, you will need to implement your own logic 
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
This property can be used to limit the number of items extracted, 
which is useful for testing or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The number of items to skip.

#### Default Value
0

[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) This means that by default there is no limit on the number of items extracted.

>If you are trying to extract a large number of items, you are limited to 
the maximum value of a 32-bit integer, which is 2,147,483,647. If you need 
to extract more than this number of items, you will need to implement your own logic 
to handle this.

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0




### Methods

| Method | Description |
|--------|-------------|
| `CreateProgressReport()` | (abstract) Creates a progress report object of type TProgress to be used for reporting progress |
| `ExtractAsync()` | Extracts data from the source asynchronously |
| `ExtractAsync(CancellationToken cancellationToken)` | Extracts data from the source asynchronously with the ability to cancel |
| `ExtractAsync(IProgress<TProgress> progress)` | Extracts data from the source asynchronously reporting progress |
| `ExtractAsync(IProgress<TProgress> progress, CancellationToken cancellationToken)` | Extracts data from the source asynchronously reporting progress with the ability to cancel |
| `ExtractWorkerAsync(CancellationToken token)`| (abstract) The worker method that performs the actual extraction of data asynchronously. This method should be implemented by derived classes to provide the specific extraction logic.



### ExtractAsync()
Extracts data from the source by enumerating it asynchronously.

#### Parameters
none

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.


### ExtractAsync(CancellationToken cancellationToken)

Extracts data from the source by enumerating it asynchronously with the ability to cancel.

#### Parameters
`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.


### ExtractAsync(IProgress\<TProgress\> progress)

Extracts data from the source by enumerating it asynchronously periodically reporting progress.

#### Parameters
`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the extractor

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.

#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.


### ExtractAsync(IProgress\<TProgress\> progress, CancellationToken cancellationToken)

#### Parameters
`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the extractor

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TSource, representing the extracted data.

#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0
