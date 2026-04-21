# Description
The loader is the final step in the ETL process, writing 
data to the final destination. 


## Requirements 
1. A loader should be implemented in a generic way, allowing 
   it to be reused for different types of ETL processes. For example,
   a loader that writes data to a JSON file could be used to write 
   any type of data to JSON regardless of the type. 
   ```csharp
   // JsonLoader should be able to write Foo and Bar
   var loader1 = new JsonLoader<Foo>();`
   var loader2 = new JsonLoader<Bar>();`
    ```
1. Each `Loader<TDestination>` should be interchangeable 
   for any other `Loader<TDestination>`, where TDestination are 
   of the same type. This allows your application to swap 
   loaders at compile time or runtime making your ETL more flexible.
   ```csharp
    // jsonLoader and csvLoader should be interchangble
    var jsonLoader = new JsonLoader<MyDataType>();
    var csvLoader = new CsvLoader<MyDataType>();
    ```
1. A loader is responsible for handling any exceptions that 
   may occur during the loading process, including but not limited 
   to network issues, permissions issues, space full issues,
   and destination unavailability. 
1. The loader should be resilient and capable of retrying operations
   in case of transient failures.
1. The data is passed to the loader as an `asynchronous` stream of type 
   `TDestination`, allowing for efficient processing by the loader.
   Whenever possible, the loader should write 1 item at a time.

   >For performance reasons it may be necessary to buffer items and
   write them in small batches. However, the loader should not
   accumulate large amounts of data in memory before writing.
   For example, a loader that writes to a CSV file may find it more
   efficient to flush a few rows at a time, but should not hold
   the entire dataset in memory before writing, as some pipelines
   may contain gigabytes of data.
1. The loader should **not** do any transformation of the data.
   Transforming the data is the responsibility of the transformer.
   Its sole responsibility is to write data to the destination.
   >Occasionally, some minimal transformation may be necessary to
   ensure the data is in a suitable format for the destination.
   For example, a CSV loader might serialize each record to a
   delimited string before writing. This kind of format-specific
   serialization is acceptable; business-logic transformation
   belongs in a transformer.
  

# Interfaces

| Interface | Description |
|-----------|-------------|
|[ILoadAsync](#ILoadAsync) | Provides a method to load data asynchronously from the source to destination. |
|[ILoadWithCancellationAsync](#ILoadWithCancellationAsync) | Provides a method to load data asynchronously to destination with the ability to cancel the operation. |
|[ILoadWithProgressAsync](#ILoadWithProgressAsync) | Provides a method to load data asynchronously to destination while reporting progress. |
|[ILoadWithProgressAndCancellationAsync](#ILoadWithProgressAndCancellationAsync) | Provides a method to load data asynchronously to destination while reporting progress and allowing cancellation of the operation. |


## ILoadAsync

### Type Parameters
`TDestination` The type of elements being passed in. This represents your destination or target data.

### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items)` | Writes data to the destination |


### LoadAsync\<TDestination\>
Asynchronously enumerates the items passed in and writes them to the destination.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be written.

#### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 

**Introduced in:** 0.4.0

    
## ILoadWithCancellationAsync

### Type Parameters
`TDestination` The type of elements being passed in. This represents your destination or target data.

### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, CancellationToken cancellationToken)` | Writes data to the destination with the ability to cancel|


### LoadAsync\<TDestination\> (CancellationToken token)
Asynchronously enumerates the items passed in and writes them to the destination.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be written.

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 

**Introduced in:** 0.4.0

    

## ILoadWithProgressAsync\<TDestination, TProgress\>

### Type Parameters
`TDestination` The type of elements being passed in. This represents your destination or target data.

`TProgress` The type of object being used to report progress

### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress)` | Writes data to the destination reporting progress|


### LoadAsync\<TDestination\> (CancellationToken token)
Asynchronously enumerates the items passed in and writes them to the destination.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be written.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the loader

#### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 

**Introduced in:** 0.4.0




## ILoadWithProgressAsync\<TDestination, TProgress\>

### Type Parameters
`TDestination` The type of elements being passed in. This represents your destination or target data.

`TProgress` The type of object being used to report progress

### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress)` | Writes data to the destination reporting progress|


### LoadAsync\<TDestination\> (CancellationToken token)
Asynchronously enumerates the items passed in and writes them to the destination.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be written.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the loader

#### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 

**Introduced in:** 0.4.0


    
## ILoadWithProgressAndCancellationAsync\<TDestination, TProgress\>

### Type Parameters
`TDestination` The type of elements being passed in. This represents your destination or target data.

`TProgress` The type of object being used to report progress

### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Writes data to the destination reporting progress with the ability to cancel|


### LoadAsync\<TDestination\> (CancellationToken token)
Asynchronously enumerates the items passed in and writes them to the destination.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be written.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) A object that will be used to report the current progress of the loader

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 

**Introduced in:** 0.4.0



# Base Classes

## LoaderBasee\<TDestination, TProgress\> 

### Definition

Provides an abstract base class that implements 
[`ILoadAsync`](#ILoadAsync), 
[`ILoadWithCancellationAsync`](#ILoadWithCancellationAsync), 
[`ILoadWithProgressAsync`](#ILoadWithProgressAsync), and
[`ILoadWithProgressAndCancellationAsync`](#ILoadWithProgressAndCancellationAsync) 
with basic functionality allowing for quickly building user defined 
loader. This abstract class only requires that you implment two methods 
[`CreateProgressReport`](#CreateProgressReport) and 
[`LoadWorkerAsync`](#LoadWorkerAsync)  


### Type Parameters

`TDestination` The type of elements being loaded. This represents your 
destination or target data.
 
`TProgress` The type of object being used to report progress


### Constructors
| Method | Description |
|--------|-------------|
|`LoaderBase()`|Initializes a new instance of the class


### Properties

| Property | Description |
|----------|-------------|
|`CurrentItemCount`|The current number of items that have been loaded|
|`MaximumItemCount`|The maximum number of items to loader before exiting and reporting complete|
|`ReportingInterval`|The number of milliseconds between progress reports|
|`SkipItemCount`|The number of items to skip before before writing items|


### CurrentItemCount
The current number of items that have been loaded. This property is updated as items are loaded and can be used to report progress.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The current number of items that have been loaded.

#### Default Value
0 

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0




### MaximumItemCount
The maximum number of items to be loaded before exiting and reporting complete. This property can be used to limit the number of items loaded, which is useful for testing or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The maximum number of items to been loader.

#### Default Value
[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) This means that by default there is no limit on the number of items loaded.

>If you are trying to loader a large number of items, you are limited to 
the maximum value of a 32-bit integer, which is 2,147,483,647. If you need 
to loader more than this number of items, you will need to implement your own logic 
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
The number of items to skip before writing items.
This property can be used to ignore the first N items, which is useful
for resuming partway through a pipeline or when only a subset of data is needed.

#### Property Value
[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) The number of items to skip.

#### Default Value
0 -- meaning no items are skipped. All items received from the pipeline are written to the destination.

#### Exceptions
[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) Thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0




### Methods

| Method | Description |
|--------|-------------|
| `CreateProgressReport()` | (abstract) Creates a progress report object of type TProgress to be used for reporting progress |
| `LoadAsync(IAsyncEnumerable<TDestination> items)` | Writes data to the destination asynchronously |
| `LoadAsync(IAsyncEnumerable<TDestination> items, CancellationToken cancellationToken)` | Writes data to the destination asynchronously with the ability to cancel |
| `LoadAsync(IAsyncEnumerable<TDestination> items , IProgress<TProgress> progress)` | Writes data to the destination asynchronously reporting progress |
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Writes data to the destination asynchronously reporting progress with the ability to cancel |
| `LoadWorkerAsync(IAsyncEnumerable<TDestination> items, CancellationToken token)`| (abstract) The worker method that performs the actual writing of data asynchronously. This method should be implemented by derived classes to provide the specific trnsformation logic.



### LoadAsync(IAsyncEnumerable\<TDestination\> items)
Writes data to the destination asynchronously by enumerating it asynchronously.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be loaded.



#### Returns
[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 



### LoadAsync(IAsyncEnumerable\<TDestination\> items, CancellationToken cancellationToken)

Writes data to the destination asynchronously by enumerating it asynchronously with the ability to cancel.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be loaded.

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 


### LoadAsync(IAsyncEnumerable\<TDestination\> items, IProgress\<TProgress\> progress)

Writes data to the destination asynchronously by enumerating it asynchronously periodically reporting progress.

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be loaded.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the loader

#### Returns
[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 


#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.


### LoadAsync(IAsyncEnumerable\<TDestination\> items, IProgress\<TProgress\> progress, CancellationToken cancellationToken)

#### Parameters
`items` [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) An asynchronous stream of items of type TDestination, representing the data to be loaded.

`progress` [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1) An object that will be used to report the current progress of the loader

`cancellationToken` [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)  The token to monitor for cancellation requests.

#### Returns
[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task)
A task that represents the asynchronous operation. The task will complete when all items have been written to the destination. 


#### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) Thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0
