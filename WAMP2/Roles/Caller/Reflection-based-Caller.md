## Reflection-based Caller

**Reflection-based Caller** (or **"Callee proxy"**) allows you to call callee methods of a WAMP realm, by declaring an interface with methods decorated with a [WampProcedure] attribute.

The interface must be public.

### Basic usage

```csharp
public interface IArgumentsService
{
    [WampProcedure("com.arguments.add2")]
    int Add2(int a, int b);
}

public static void Run()
{
    DefaultWampChannelFactory factory =
        new DefaultWampChannelFactory();

    const string serverAddress = "ws://127.0.0.1:8080/ws";

    IWampChannel channel =
        factory.CreateJsonChannel(serverAddress, "realm1");

    channel.Open().Wait(5000);

    IArgumentsService proxy =
        channel.RealmProxy.Services.GetCalleeProxy<IArgumentsService>();

    int five = proxy.Add2(2, 3);
}
```

### Supported features

The following features are supported:

#### Async method support

A method returning a Task&lt;&gt; can be awaited. Example:

```csharp
public interface IArgumentsService
{
    [WampProcedure("com.arguments.ping")]
    void Ping();

    [WampProcedure("com.arguments.add2")]
    int Add2(int a, int b);

    [WampProcedure("com.arguments.stars")]
    string Stars(string nick = "somebody", int stars = 0);

    [WampProcedure("com.arguments.orders")]
    string[] Orders(string product, int limit = 5);

    [WampProcedure("com.arguments.ping")]
    Task PingAsync();

    [WampProcedure("com.arguments.add2")]
    Task<int> Add2Async(int a, int b);

    [WampProcedure("com.arguments.stars")]
    Task<string> StarsAsync(string nick = "somebody", int stars = 0);

    [WampProcedure("com.arguments.orders")]
    Task<string[]> OrdersAsync(string product, int limit = 5);
}
```

Call example:

```csharp
proxy.Ping();
Console.WriteLine("Pinged!");

int result = await proxy.Add2Async(2, 3);
Console.WriteLine("Add2: {0}", result);

var starred = await proxy.StarsAsync();
Console.WriteLine("Starred 1: {0}", starred);

starred = await proxy.StarsAsync(nick: "Homer");
Console.WriteLine("Starred 2: {0}", starred);

starred = await proxy.StarsAsync(stars: 5);
Console.WriteLine("Starred 3: {0}", starred);

starred = await proxy.StarsAsync(nick: "Homer", stars: 5);
Console.WriteLine("Starred 4: {0}", starred);

string[] orders = await proxy.OrdersAsync("coffee");
Console.WriteLine("Orders 1: {0}", string.Join(", ", orders));

orders = await proxy.OrdersAsync("coffee", limit: 10);
Console.WriteLine("Orders 2: {0}", string.Join(", ", orders));
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/arguments) AutobahnJS sample

#### out/ref parameters

For synchronous methods, out/ref parameters are supported.
> Note: this is not supported for asynchronous methods.

Example:

```csharp
public interface IComplexResultService
{
    [WampProcedure("com.myapp.add_complex")]
    void AddComplex(int a, int ai, int b, int bi, out int c, out int ci);
}
```
Call example:

```csharp
int ci;
int c;
proxy.AddComplex(2, 3, 4, 5, out c, out  ci);
```
>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS sample

#### Multi-valued results

In order to get an multivalued array from the RESULT/YIELD WAMPv2 message, set the return value of the rpc method to an array and place above it a [return: WampResult(CollectionResultTreatment.Multivalued)] attribute. Example:

```csharp
public interface IMultivaluedResultService
{
    [WampProcedure("com.myapp.split_name")]
    [return: WampResult(CollectionResultTreatment.Multivalued)]
    string[] SplitName(string fullname);
}
```

Call example:

```csharp
string[] splitted = proxy.SplitName("Homer Simpson");
```
>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS sample

#### Exception support
You can catch a WampException in order to treat a ERROR message.

Example:

```csharp
try
{
    await proxy.CheckNameAsync("Moses Montefiore").ConfigureAwait(false);
}
catch (WampException ex)
{
    string errorUri = ex.ErrorUri; // "com.myapp.error.invalid_length"
    IDictionary<string, object> arguments = ex.ArgumentsKeywords; // {"min": 3, "max": 10}
}
```
>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/errors) AutobahnJS sample

#### Progressive call results

In order to use progressive call results as a Caller, declare in your callee service a [WampProcedure] method having a [WampProgressiveResultProcedure] attribute and a IProgress&lt;T&gt; as the last parameter.
> Note that the method return type should be Task&lt;T&gt; where this is the same T as in the IProgress&lt;T&gt; of the last parameter.

Example:

```csharp
public interface ILongOpService
{
    [WampProcedure("com.myapp.longop")]
    [WampProgressiveResultProcedure]
    Task<int> LongOp(int n, IProgress<int> progress);
}
```

Then obtain the proxy and call it:

```csharp
public async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel = factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    ILongOpService proxy = channel.RealmProxy.Services.GetCalleeProxy<ILongOpService>();

    Progress<int> progress =
        new Progress<int>(i => Console.WriteLine("Got progress " + i));

    int result = await proxy.LongOp(10, progress);

    Console.WriteLine("Got result " + result);
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/progress) AutobahnJS sample

#### Registration customization

The GetCalleeProxy method of IWampRealmServiceProvider now has an overload that receives an "interceptor" instance. The "interceptor" allows customizing the call being performed.

For instance, assume you want to call procedures of a contract that its procedures uris are known only on runtime.  This is possible implementing a ICalleeProxyInterceptor:

```csharp
public class MyCalleeProxyInterceptor : CalleeProxyInterceptor
{
    private readonly int mCalleeIndex;

    public MyCalleeProxyInterceptor(int calleeIndex) :
        base(new CallOptions())
    {
        mCalleeIndex = calleeIndex;
    }

    public override string GetProcedureUri(MethodInfo method)
    {
        string format = base.GetProcedureUri(method);
        string result = string.Format(format, mCalleeIndex);
        return result;
    }
}

```

This interceptor modifies the procedure uri of the procedure to call. For example, we can declare an interface with a method with this signature:

```csharp
public interface ISquareService
{
    [WampProcedure("com.myapp.square.{0}")]
    Task<int> Square(int number);
}
```

And then specify the index in runtime:

```csharp
public static async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    int index = GetRuntimeIndex();

    ISquareService proxy =
        channel.RealmProxy.Services.GetCalleeProxy<ISquareService>
        (new CachedCalleeProxyInterceptor(
            new MyCalleeProxyInterceptor(index)));

    int nine = await proxy.Square(3); // Calls ("com.myapp.square." + index)
}
```

> Note: we wrap our interceptor with the CachedCalleeProxyInterceptor in order to cache the results of our interceptor, in order to avoid calculating them each call.

In addition, the interceptor allows to modify the options sent to each call. The following sections demonstrates modifications that can be used to leverage WAMP advanced profile features.

> Note: these interceptors are still "static", i.e: they don't allow returning a value that depends on the call parameters.

#### Caller identification

Callee proxy sample:

According to WAMP2 specification, a Caller can request to disclose its identification (by specifying disclose_me = true on call request).

Specifying this is possible when obtaining callee proxy.

```csharp
public async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    var callOptions = new CallOptions()
    {
        DiscloseMe = true
    };

    ISquareService proxy =
        channel.RealmProxy.Services.GetCalleeProxy<ISquareService>
        (new CachedCalleeProxyInterceptor(new CalleeProxyInterceptor(callOptions)));

    await proxy.Square(-2);
    await proxy.Square(0);
    await proxy.Square(2);
}
```

> Note: The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/options) AutobahnJS sample
