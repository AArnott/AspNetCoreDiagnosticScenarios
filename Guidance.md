# Common Pitfalls writing scalable services in ASP.NET Core

This document serves as a guide for writing scalable services in ASP.NET Core. Some of the guidance is general purpose but will be explained through the lens of writing 
web services. The examples shown here are based on experiences with customer applications and issues found on Github and Stack Overflow. Besides general guidance,
this repository will have guides on how to diagnose common issue in the various tools available (WinDbg, lldb, sos, Visual Studio, PerfView etc).

# Asynchronous Programming

Asynchronous programming has been around for several years on the .NET platform but has historically been very difficult to do well. Since the introduction of async/await
in C# 5 asynchronous programming has become mainstream. Modern frameworks (like ASP.NET Core) are fully asynchronous and it's very hard to avoid the async keyword when writing
web services. As a result, there's been lots of confusion on the best practices for async and how to use it properly. This section will try to lay out some guidance with examples of bad and good patterns of how to write asynchronous code.

## Asynchrony is viral 

Once you go async, all of your callers **MUST** be async, there's no good way gradually migrate callers to be async. It's all or nothing.

❌ **BAD** This example uses the `Task.Result` and as a result blocks the current thread to wait for the result. This is an example of [sync over async](#avoid-using-taskresult-and-taskwait).

```C#
public async int DoSomethingAsync()
{
    var result = CallDependencyAsync().Result
    return result + 1;
}
```

✔️**GOOD** This example uses the await keyword to get the result from `CallDependencyAsync`.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Async void

Use of async void in ASP.NET Core applications is *ALWAYS* bad. Avoid it, never do it. Typically, it's used when developers are trying to implement fire and forget patterns triggered by a controller action. Async void methods will crash the process if an exception is thrown. We'll look at more of the patterns that cause developers to do this in ASP.NET Core applications but here's a simple example:

❌ **BAD** Async void methods can't be tracked and therefore unhandled exceptions can result in application crashes.

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        BackgroundOperationAsync();
        return Accepted();
    }
    
    public async void BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        return result + 1;
    }
}
```

✔️**GOOD** Task returning methods are better since unhandled exceptions trigger the [TaskScheduler.UnobservedTaskException](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception?view=netframework-4.7.2).

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        Task.Run(BackgroundOperationAsync);
        return Accepted();
    }
    
    public async Task BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        return result + 1;
    }
}
```

## Avoid using Task.Run for long running work that blocks the thread

`Task.Run` will queue a work item to thread pool. The assumption is that that work will finish quickly (or quickly enough to allow reusing that thread within some rasonable timeframe). Stealing a thread pool thread for long running work is bad since it takes that thread away from other work that could be done (timer callbacks, task continuations etc). Instead, spawn a new thread manually to do long running blocking work.

❌ **BAD** This example steals a thread pool thread forever, to execute queued work on a `BlockingCollection<T>`.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        Task.Run(ProcessQueue);
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

✔️**GOOD** This example uses a dedicated thread to process the message queue instead of a thread pool thread.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        var thread = new Thread(ProcessQueue) 
        {
            // This is important as it allows the process to exit without
            IsBackground = true
        };
        thread.Start();
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

## Avoid using Task.Result and Task.Wait

There are very few ways to use Task.Result and Task.Wait correctly so the general advice is to completely avoid using them in your code. 

### :warning: Sync over async

Using Task.Result or Task.Wait to block wait on an asynchronous operation to complete is *MUCH* worse than calling a truly synchronous API to block. This phenomenon is dubbed "Sync over async". Here is what happens at a very high level:

- An asynchronous operation is kicked off.
- The calling thread is blocked waiting for that operation to complete.
- When the asynchronous operation completes, it schedules a continuation to the thread pool to resume the code waiting on that operation.

The result is that we need to use 2 threads instead of 1 to complete synchronous operations. This usually leads to [thread pool starvation](https://blogs.msdn.microsoft.com/vancem/2018/10/16/diagnosing-net-core-threadpool-starvation-with-perfview-why-my-service-is-not-saturating-all-cores-or-seems-to-stall/) and results in service outages.

### :warning: Deadlocks

The `SynchronizationContext` is an abstraction that gives application models a chance to control where asynchronous continuations run. ASP.NET (non-core), WPF and Windows Forms each have an implementation that will result in a deadlock if Task.Wait or Task.Result is used on the main thread. This behavior has lead to a bunch of "clever" code snippets that show the "right" way to block waiting for a Task. The truth is, there's is no good way to block waiting for a Task to complete.

❌ **BAD** The below are all examples that are trying to avoid the dead lock situation but still succumb to "sync over async" problems.

```C#
public string DoOperationBlocking()
{
    return Task.Run(() => DoAsyncOperation()).Result;
}

public string DoOperationBlocking2()
{
    return Task.Run(() => DoAsyncOperation()).GetAwaiter().GetResult();
}

public string DoOperationBlocking3()
{
    return Task.Run(() => DoAsyncOperation().Result).Result;
}

public string DoOperationBlocking4()
{
    return Task.Run(() => DoAsyncOperation().GetAwaiter().GetResult()).GetAwaiter().GetResult();
}

public string DoOperationBlocking5()
{
    return DoAsyncOperation().Result;
}

public string DoOperationBlocking6()
{
    return DoAsyncOperation().GetAwaiter().GetResult();
}

public string DoOperationBlocking7()
{
    var task = DoAsyncOperation();
    task.Wait();
    return task.GetAwaiter().GetResult();
}
```

## Prefer await over ContinueWith

`Task` existed before the async/await keywords were introduced and as such provided ways to execute continuations without a reliance the language. Although these
methods are still valid to use, we generally recommend that you prefer async/await to using ContinueWith. ContinueWith also does not capture the `SynchronizationContext` and as a result is actually semantically different to async/await.

❌ **BAD** The example uses ContinueWith instead of async

```C#
public async Task<int> DoSomethingAsync()
{
    return CallDependencyAsync().ContinueWith(task =>
    {
        return task.Result + 1;
    });
}
```

✔️**GOOD** This example uses the await keyword to get the result from `CallDependencyAsync`.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Always create TaskCompletionSource\<T\> with TaskCreationOptions.RunContinuationsAsynchronously

`TaskCompletionSource<T>` is an important building block for libraries trying to adapt things that are not inherently awaitable to be awaitable via a `Task`. It is also commonly used to build higher level operations (such as batching and other combinatiors) on top of existing asynchronous APIs. By default, `Task` continuations will run *inline* on the same thread that calls Try/Set(Result/Exception/Canceled). As a library author, this means having to understand that calling code can resume directly on your thread. This is extremely dangerous and can result in deadlocks, thread pool starvation, corruption of state (if code runs unexpectedly) and more. 

Always use `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`. This will dispatch the continuation onto the thread pool instead of executing it inline.

❌ **BAD** This example does not use TaskCreationOptions.RunContinuationsAsynchronously when creating the `TaskCompletionSource<T>`.

```C#
public async Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>();
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on this thread!
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

✔️**GOOD** This example uses TaskCreationOptions.RunContinuationsAsynchronously when creating the `TaskCompletionSource<T>`.

```C#
public async Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on a different thread pool thread
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

:warning: **NOTE: There are 2 enums that look alike. [TaskCreationOptions.RunContinuationsAsynchronously](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcreationoptions?view=netcore-2.0#System_Threading_Tasks_TaskCreationOptions_RunContinuationsAsynchronously) and [TaskContinuationOptions.RunContinuationsAsynchronously](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcontinuationoptions?view=netcore-2.0). Be careful not to confuse their usage.** 

## Always dispose CancellationTokenSource(s) used for timeouts

CancellationTokenSources that are used for timeouts (are created with timers or uses the CancelAfter method), can put pressure on the timer queue if not disposed. 

❌ **BAD** This example does not dispose the `CancellationTokenSource` and as a result the timer stays in the queue for 10 seconds after each request is made.

```C#
public async Task<Stream> HttpClientAsyncWithCancellationBad()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    using (var client = _httpClientFactory.CreateClient())
    {
        var response = await client.GetAsync("http://backend/api/1", cts.Token);
        return await response.Content.ReadAsStreamAsync();
    }
}
```

✔️**GOOD** This example disposes the `CancellationTokenSource` and properly removes the timer from the queue.

```C#
public async Task<Stream> HttpClientAsyncWithCancellationGood()
{
    using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)))
    {
        using (var client = _httpClientFactory.CreateClient())
        {
            var response = await client.GetAsync("http://backend/api/1", cts.Token);
            return await response.Content.ReadAsStreamAsync();
        }
    }
}
```

## Always flow CancellationToken(s) to APIs that take a CancellationToken

Cancellation is coorperative in .NET. Everything in the call chain has to be explicitly passed the `CancellationToken` in order for it to work well. This means you need to explicitly pass the token into other APIs that take a token if you want cancellation to be most effective.

❌ **BAD** This example neglects to pass the CancellationToken to `Stream.ReadAsync` making the operation effectively not cancellable.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

✔️**GOOD** This example passes the CancellationToken into `Stream.ReadAsync`.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

## Cancelling uncancellable operations

One of the coding patterns that appears when doing asynchronous programming is cancelling an uncancellable operation. This usually means creating another task
that completes when a timeout or `CancellationToken` fires, and then using `Task.WhenAny` to detect a complete or cancelled operation.

### Using CancellationTokens

❌ **BAD** This example uses `Task.Delay(-1, token)` to create the `Task` fires when the `CancellationToken` fires, but if it doesn't fire, there's no way to dispose the `CancellationTokenRegistration`. This can lead to a memory leak.

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    // There's no way to dispose the registration
    var delayTask = Task.Delay(-1, cancellationToken);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

✔️**GOOD** This example disposes the `CancellationTokenRegistration` when one of the `Task(s)` complete.

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    var tcs = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);

    // This disposes the registration as soon as one of the tasks trigger
    using (cancellationToken.Register(state =>
    {
        ((TaskCompletionSource<object>)state).TrySetResult(null);
    },
    tcs))
    {
        var resultTask = await Task.WhenAny(task, tcs.Task);
        if (resultTask == tcs.Task)
        {
            // Operation cancelled
            throw new OperationCanceledException(cancellationToken);
        }

        return await task;
    }
}
```

### Using a timeout

❌ **BAD** This example does not cancel the timer even if the operation successfuly completes. This means you could end up with lots of timers that can flood the timer queue. 

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    var delayTask = Task.Delay(timeout);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

✔️**GOOD** This example cancels the timer if the operation succesfully completes.

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    using (var cts = new CancellationTokenSource())
    {
        var delayTask = Task.Delay(timeout, cts.Token);

        var resultTask = await Task.WhenAny(task, delayTask);
        if (resultTask == delayTask)
        {
            // Operation cancelled
            throw new OperationCanceledException();
        }
        else
        {
            // Cancel the timer task so that it does not fire
            cts.Cancel();
        }

        return await task;
    }
}
```

## Always call FlushAsync on StreamWriter(s) or Stream(s) before calling Dispose

When writing to a `Stream` or `StreamWriter` even if the asynchronous overloads are used for writing, the underlying data might be buffered. When data is buffered, disposing the `Stream` or `StreamWriter` will synchronously write/flush which results in blocking the thread and could lead to thread pool starvation.

:warning: **NOTE: This is only problematic if the underlying subsystem does IO.**

❌ **BAD** This example ends up blocking the request by writing synchronously to the http response body.

```C#
app.Run(async context =>
{
    // The implicit Dispose call will synchronously write to the response body
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        streamWriter.WriteAsync("Hello World");
    }
});
```

✔️**GOOD** This example asynchronously flushes any buffered data before disposing the `StreamWriter`.

```C#
app.Run(async context =>
{
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        streamWriter.WriteAsync("Hello World");
        
        // Force an asynchronous flush
        await streamWriter.FlushAsync();
    }
});
```

## Prefer async/await over directly returning Task

There are benefits to using the async/await keyword instead of directly returning the Task:
- Asynchronous and synchronous exceptions are normalized to always be asynchronous.
- The code is easier to modify (consider adding a using for example).
- The compiler generated code is constantly being improved.
- The performance of async/await is constantly being improved.
- Diagnostics of asynchronous methods are easier (debugging hangs etc).

❌ **BAD** This example directly returns the `Task` to the caller.

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync();
}
```

✔️**GOOD** This examples uses async/await instead of directly returning the Task.

```C#
public async Task<int> DoSomethingAsync()
{
    return await CallDependencyAsync();
}
```

:warning: **NOTE: There are performance considerations when using an async state machine over directly returning the Task. It's always faster to directly return the Task since it does less work but you end up changing the behavior and potentially losing some of the benefits of the async state machine.**

# Scenarios

The above tries to distill general guidance but doesn't do justice to the kinds of real world situation that cause code like this to be written in the first place (bad code). This section will try to take concrete examples from real applications and distill them into something simple to understand to help you relate these problems to existing code bases.

## Timer callbacks

❌ **BAD** The timer callback is void returning and we have asynchronous work to execute. This example uses async void to accomplish it and as a result
can crash the process if there's an exception thrown.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = new HttpClient();
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public async void Heartbeat(object state)
    {
        await httpClient.GetAsync("http://mybackend/api/ping");
    }
}
```

❌ **BAD** This attempts to block in the timer callback. This may result in thread pool starvation and is an example of [sync over async](#warning-sync-over-async)

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = new HttpClient();
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        httpClient.GetAsync("http://mybackend/api/ping").GetAwaiter().GetResult();
    }
}
```

✔️**GOOD** This example uses an async Task based method and discards the Task in the Timer callback. If this method fails, it will not crash the process.
Instead, it will fire the [TaskScheduler.UnobservedTaskException](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception?view=netframework-4.7.2) event.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = new HttpClient();
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        // Discard the result
        _ = DoAsyncPing();
    }

    private static async Task DoAsyncPing()
    {
        await httpClient.GetAsync("http://mybackend/api/ping");
    }
}
```

## Implicit async void delegates

Imagine a `BackgroundQueue` with a `FireAndForget` that takes a callback. This method will execute the callback at some time in the future.

❌ **BAD** This will force callers to either block in the callback or use an async void delegate.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
}
```

❌ **BAD** This calling code is creating an async void method implicitly. The compiler fully supports this today.

```C#
public class Program
{
    public void Main(string[] args)
    {
        var httpClient = new HttpClient();
        BackgroundQueue.FireAndForget(async () =>
        {
            await httpClient.GetAsync("http://pinger/api/1");
        });
        
        Console.ReadLine();
    }
}
```

✔️**GOOD** This BackgroundQueue implementation offers both sync and async callback overloads.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
    public static void FireAndForget(Func<Task> action) { }
}
```

## ConcurrentDictionary.GetOrAdd

It's pretty common to cache the result of an asynchronous operation and ConcurrentDictionary is a good data structure for doing that. GetOrAdd is a convenience API for trying to get an item if it's already there or adding it if it isn't. The callback is synchronous so it's tempting to write code that uses Task.Result to produce the value of an asynchronous process but that can lead to thread pool starvation.

❌ **BAD** This may result in thread pool starvation since we're blocking the request thread if the person data is not cached.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Person> _cache = new ConcurrentDictionary<int, Person>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public IActionResult Get(int id)
   {
       var person = _cache.GetOrAdd(id, (key) => db.People.FindAsync(key).Result);
       return Ok(person);
   }
}
```

❌ **BAD** This won't result in thread pool starvation but will potentially run the cache callback multiple times in parallel.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Task<Person>> _cache = new ConcurrentDictionary<int, Task<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => db.People.FindAsync(key));
       return Ok(person);
   }
}
```

✔️**GOOD** This implementation fixes the multiple executing callback issue by using the async lazy pattern.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, AsyncLazy<Person>> _cache = new ConcurrentDictionary<int, AsyncLazy<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => new AsyncLazy<Person>(() => db.People.FindAsync(key)));
       return Ok(person);
   }
   
   private class AsyncLazy<T> : Lazy<Task<T>>
   {
      public AsyncLazy(Func<Task<T>> valueFactory) : base(valueFactory)
      {
      }
   }
}
```

## Constructors

Constructors are synchronous. If you need to initialize some logic that may be asynchronous, there are a couple patterns for dealing with this.

Here's an example of using a client API that needs to connect asynchronously before use.

```C#
public interface IRemoteConnectionFactory
{
   Task<IRemoteConnection> ConnectAsync();
}

public interface IRemoteConnection
{
    Task PublishAsync(string channel, string message);
    Task DisposeAsync();
}
```


❌ **BAD** This example uses Task.Result to get the connection in the constructor. This could lead to thread pool starvation and dead locks.

```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;
    
    public Service(IRemoteConnectionFactory connectionFactory)
    {
        _connection = connectionFactory.ConnectAsync().Result;
    }
}
```
