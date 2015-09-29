### Reflection base callee

As demonstrated in the [Getting Started with Callee](Getting-Started-with-Callee.md) page, reflection base callee allows you to register classes instances with method decorated with a [WampProcedure] attribute as remote procedure operation to a WAMP realm.

Both placing attributes on a class method and placing attributes on an interface implemented by the class are supported.

The following features are supported:
* Default parameter values: a method can have default value parameters. These will be used in case the user sends only part of the method's parameters. Example:
```csharp
public interface IArgumentsService
{
    [WampProcedure("com.arguments.stars")]
    string Stars(string nick = "somebody", int stars = 0);
}
```
* Async method support. A method returning a Task<> will be awaited. Example:
```csharp
public class SlowSquareService
{
    [WampProcedure("com.math.slowsquare")]
    public async Task<int> SlowSquare(int x)
    {
        await Task.Delay(1000);
        return x * x;
    }

    [WampProcedure("com.math.square")]
    public int Square(int x)
    {
        return x * x;
    }
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/slowsquare) AutobahnJS sample

* Out/ref parameters: For synchronous methods, out/ref parameters are supported. Note: this is not supported for asynchronous methods. Example:
```csharp
public class ComplexResultService
{
    [WampProcedure("com.myapp.add_complex")]
    public void AddComplex(int a, int ai, int b, int bi, out int c, out int ci)
    {
        c = a + b;
        ci = ai + bi;
    }
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS sample

* Multi-valued results: in order to return an multivalued array in the RESULT/YIELD WAMPv2 message, return an array from a rpc method and place above it a [return: WampResult(CollectionResultTreatment.Multivalued)] attribute. Example:
```csharp
public class MultivaluedResultService
{
    [WampProcedure("com.myapp.split_name")]
    [return: WampResult(CollectionResultTreatment.Multivalued)]
    public string[] SplitName(string fullname)
    {
        string[] splitted = fullname.Split(' ');
        return splitted;
    }
}
```
* Exception support: throw a WampException/WampRpcRuntimeException in order to send a ERROR message.

### Progressive callee

In order to use progressive calls as a Callee, declare in your callee service a [WampProcedure] method having a [WampProgressiveCall] attribute and a IProgress&lt;T&gt; as the last parameter.
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

create a service with that implements that interface.
In order to report progress, call the progress Report method. Example:

```csharp
public class LongOpService : ILongOpService
{
    public async Task<int> LongOp(int n, IProgress<int> progress)
    {
        for (int i = 0; i < n; i++)
        {
            progress.Report(i);
            await Task.Delay(100);
        }

        return n;
    }
}
```

> Note: you can put the attributes on the method itself instead of implementing an interface, i.e:
> 
>```csharp
> public class LongOpService
>{
>    [WampProcedure("com.myapp.longop")]
>    [WampProgressiveResultProcedure]
>    public async Task<int> LongOp(int n, IProgress<int> progress)
>    {
>	    // ...
>    }
> }
>```

Then register it to the realm regularly:
```csharp
public async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel = factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    ILongOpService service = new LongOpService();

    IAsyncDisposable disposable = 
        await channel.RealmProxy.Services.RegisterCallee(service);

    Console.WriteLine("Registered LongOpService");
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/progress) AutobahnJS sample

## Caller identification

It is possible to get caller identification details. According to WAMP2 specification, a Callee can request to get caller identification details (by specifying disclose_caller = true on registration), and a Caller can request to disclose its identification (by specifying disclose_me = true on call request).

Specifying these is possible on callee registration and when obtaining callee proxy:

Callee registration example:

```csharp
public async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel = 
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    SquareService service = new SquareService();

    var registerOptions =
        new RegisterOptions
        {
            DiscloseCaller = true
        };

    IAsyncDisposable disposable =
        await channel.RealmProxy.Services.RegisterCallee(service,
            new CalleeRegistrationInterceptor(registerOptions));
}
```

In order to obtain these details as a Callee (using reflection api), access WampInvocationContext.Current.

Sample:

```csharp
public class SquareService
{
    [WampProcedure("com.myapp.square")]
    public int Square(int n)
    {
        InvocationDetails details = 
            WampInvocationContext.Current.InvocationDetails;

        Console.WriteLine("Someone is calling me: " + details.Caller);

        return n*n;
    }
}

```

> Note: The samples are based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/options) AutobahnJS sample

### WampInvocationContext

WampInvocationContext allows you to get the invocation details provided with the current invocation. It currently contains the caller identification (if present) and whether the caller requested a progressive call. 
Example:

```csharp
public class LongOpService : ILongOpService
{
    public async Task<int> LongOp(int n, IProgress<int> progress)
    {
        InvocationDetails details = 
            WampInvocationContext.Current.InvocationDetails;

        for (int i = 0; i < n; i++)
        {
            if (details.ReceiveProgress == true)
            {
                progress.Report(i);                    
            }

            await Task.Delay(100);
        }

        return n;
    }
}
```

### Raw callee

Raw callee gives lower-level api for callee.

In order to use it, derive from IWampRpcOperation. A IWampFormatter is passed to your methods in order to allow you to deserialize the method parameters. Example:

```csharp
using System.Collections.Generic;
using WampSharp.Core.Serialization;
using WampSharp.V2.Core;
using WampSharp.V2.Core.Contracts;
using WampSharp.V2.Rpc;

public class Add2Operation : IWampRpcOperation
{
    public string Procedure
    {
        get
        {
            return "com.arguments.add2";
        }
    }

    public void Invoke<TMessage>(IWampRawRpcOperationRouterCallback caller, IWampFormatter<TMessage> formatter, InvocationDetails details)
    {
        Dictionary<string, object> dummyDetails = new Dictionary<string, object>();

        caller.Error(WampObjectFormatter.Value, dummyDetails, "wamp.error.runtime_error",
                     new object[] { "Expected parameters" });
    }

    public void Invoke<TMessage>(IWampRawRpcOperationRouterCallback caller, IWampFormatter<TMessage> formatter, InvocationDetails details,
                                 TMessage[] arguments)
    {
        InnerInvoke(caller, formatter, arguments);
    }

    public void Invoke<TMessage>(IWampRawRpcOperationRouterCallback caller, IWampFormatter<TMessage> formatter, InvocationDetails details,
                                 TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords)
    {
        InnerInvoke(caller, formatter, arguments);
    }

    private static void InnerInvoke<TMessage>(IWampRawRpcOperationRouterCallback caller, IWampFormatter<TMessage> formatter,
                                              TMessage[] arguments)
    {
        int x = formatter.Deserialize<int>(arguments[0]);
        int y = formatter.Deserialize<int>(arguments[1]);
        int result = x + y;

        YieldOptions dummyDetails = new YieldOptions();

        caller.Result(WampObjectFormatter.Value, dummyDetails, new object[] { result });
    }
}
```

Since this is kind of hard, WampSharp defines some base-classes that make this easier, called SyncLocalRpcOperation and AsyncLocalRpcOperation.

In these classes, we define the parameters we expect and let WampSharp deserialize them for us.
Example:
```csharp
public class Add2Operation : SyncLocalRpcOperation
{
    private readonly RpcParameter[] mParameters = new RpcParameter[]
        {
            new RpcParameter(name: "x", type: typeof (int), position: 0),
            new RpcParameter(name: "y", type: typeof (int), position: 1)
        };

    public Add2Operation()
        : base("com.arguments.add2")
    {
    }

    public override RpcParameter[] Parameters
    {
        get
        {
            return mParameters;
        }
    }

    public override bool HasResult
    {
        get
        {
            return true;
        }
    }

    public override CollectionResultTreatment CollectionResultTreatment
    {
        get
        {
            return CollectionResultTreatment.SingleValue;
        }
    }

        protected override object InvokeSync<TMessage>(IWampRawRpcOperationRouterCallback caller,
                                                       IWampFormatter<TMessage> formatter,
                                                       InvocationDetails details,
                                                       TMessage[] arguments,
                                                       IDictionary<string, TMessage> argumentsKeywords,
                                                       out IDictionary<string, object> outputs)
    {
        object[] parameters = UnpackParameters(formatter, arguments, argumentsKeywords);
        
        int x = (int) parameters[0];
        int y = (int) parameters[1];

        outputs = null;

        return (x + y);
    }
}
```

This handles lookups for parameters named x and y (or positioned at 0, 1), and handles exception throw if not both present.
Note that you can return an array and return CollectionResultTreatment.Multivalued from CollectionResultTreatment if you want to return an array with more than one item as the ARGUMENTS of the YIELD/RESULT message.
Note that you can also set outputs to a dictionary with keywords arguments parameters that will be sent as ARGUMENTSKWS of the YIELD/RETURN message.