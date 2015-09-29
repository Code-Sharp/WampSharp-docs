## Reflection-based Callee

**Reflection-based Callee** allows you to register classes instances with method decorated with a [WampProcedure] attribute as remote procedure operation to a WAMP realm.

### Basic usage

```csharp
public interface IArgumentsService
{
    [WampProcedure("com.arguments.ping")]
    void Ping();

    [WampProcedure("com.arguments.add2")]
    int Add2(int a, int b);

    [WampProcedure("com.arguments.stars")]
    string Stars(string nick = "somebody", int stars = 0);
}

public class ArgumentsService : IArgumentsService
{
    public void Ping()
    {
    }

    public int Add2(int a, int b)
    {
        return a + b;
    }

    public string Stars(string nick = "somebody", int stars = 0)
    {
        return string.Format("{0} starred {1}x", nick, stars);
    }
}


public static async Task Run()
{
    const string location = "ws://127.0.0.1:8080/";

    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel = channelFactory.CreateJsonChannel(location, "realm1");

    await channel.Open().ConfigureAwait(false);

    IArgumentsService instance = new ArgumentsService();

    IWampRealmProxy realm = channel.RealmProxy;

    IAsyncDisposable disposable =
        await realm.Services.RegisterCallee(instance).ConfigureAwait(false);

    // Call await disposable.DisposeAsync().ConfigureAwait(false);
    // to unregister
}

```

### Supprted features

Both placing attributes on a class method and placing attributes on an interface implemented by the class are supported.

#### Default parameter values

A method can have default value parameters. These will be used in case the user sends only part of the method's parameters. Example:
```csharp
public interface IArgumentsService
{
    [WampProcedure("com.arguments.stars")]
    string Stars(string nick = "somebody", int stars = 0);
}
```

#### Async method support

A method returning a Task&lt;&gt; will be awaited. Example:
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

#### out/ref parameters

 For synchronous methods, out/ref parameters are supported. Note: this is not supported for asynchronous methods. Example:
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

#### Multivalued results

Multi-valued results: in order to return an multivalued array in the RESULT/YIELD WAMPv2 message, return an array from a rpc method and place above it a [return: WampResult(CollectionResultTreatment.Multivalued)] attribute. Example:
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
>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS sample

#### Exception support

throw a WampException/WampRpcRuntimeException in order to send a ERROR message.

Example:

```csharp
[WampProcedure("com.myapp.checkname")]
public void CheckName(string name)
{
    if (new[] {"foo", "bar"}.Contains(name))
    {
        throw new WampException("com.myapp.error.reserved");
    }

    if (name.ToLower() != name.ToUpper())
    {
        throw new WampException("com.myapp.error.mixed_case", name.ToLower(), name.ToUpper());
    }

    if ((name.Length < 3) || (name.Length > 10))
    {
        object[] arguments = new object[] {};

        IDictionary<string, object> argumentKeywords =
            new Dictionary<string, object>()
                {
                    {"min", 3},
                    {"max", 10}
                };

        throw new WampException("com.myapp.error.invalid_length", arguments, argumentKeywords);
    }
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/errors) AutobahnJS sample


#### Progressive call results

In order to use progressive call result as a Callee, declare in your callee service a [WampProcedure] method having a [WampProgressiveResultProcedure] attribute and a IProgress&lt;T&gt; as the last parameter.
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

#### WampInvocationContext

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

#### Registration customization

The RegisterCallee method of IWampRealmServiceProvider has an overload that receives an "interceptor" instance. The "interceptor" allows customizing the registration being performed.

This allows customizing the register options and the procedure uri sent upon REGISTER message.

The following sections discuss register options modifications that can be used in order to leverage WAMP advanced profile features.

#### Caller identification

It is possible to get caller identification details. According to WAMP2 specification, a Callee can request to get caller identification details (by specifying disclose_caller = true on registration).

Specifying this is possible on callee registration.

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

In order to obtain these details as a Callee, access WampInvocationContext.Current.

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

> Note: This sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/options) AutobahnJS sample

#### Shared registrations

[Shared registrations](http://crossbar.io/docs/Shared-Registrations/) are supported.

In order to use shared registrations, pass to the Register methods, RegisterOptions with a desired Invoke (the policy to be used). The possible options are: single/first/last/random/roundrobin (there exists a static class named WampInvokePolicy with these values).

Example:
```csharp
public async Task Run()
{
    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel =
        channelFactory.CreateJsonChannel
            ("ws://127.0.0.1:8080/ws",
             "realm1");

    TaskCompletionSource<string> identTask = new TaskCompletionSource<string>();

    channel.RealmProxy.Monitor.ConnectionEstablished += (sender, args) =>
    {
        string ident =
            string.Format("MyComponent (PID {0}, Session {1})",
                          Process.GetCurrentProcess().Id,
                          args.SessionId);

        identTask.SetResult(ident);
    };

    await channel.Open().ConfigureAwait(false);

    string identValue = await identTask.Task;

    await channel.RealmProxy.Services.RegisterCallee(new MyComponent(identValue),
        new CalleeRegistrationInterceptor(new RegisterOptions()
        {
            Invoke = WampInvokePolicy.Roundrobin
        }))
        .ConfigureAwait(false);
}

public class MyComponent
{
    private readonly string mIdent;

    public MyComponent(string ident)
    {
        mIdent = ident;
    }

    [WampProcedure("com.example.add2")]
    public object Add2(double x, double y)
    {
        Console.WriteLine("add2 called on {0}", mIdent);

        return new
        {
            result = x + y,
            ident = mIdent
        };
    }
}
```

> Note: this sample is based on [this](https://github.com/crossbario/crossbarexamples/tree/master/sharedregs) Autobahn sample

#### Pattern-based registrations

[Pattern-based registrations](http://crossbar.io/docs/Pattern-Based-Registrations/) are supported.

In order to use pattern-based registrations, pass to the Register methods, RegisterOptions with a desired Match (the match pattern to be used). The possible options are: exact/prefix/wildcard. (these values are also available in a static class named WampMatchPattern)

Example:
```csharp
public static async Task Run()
{
    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel =
        channelFactory.CreateJsonChannel("ws://127.0.0.1:8080/ws",
            "realm1");

    await channel.Open().ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterCallee(new Callee1())
        .ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterCallee
        (new Callee2(),
         new CalleeRegistrationInterceptor(new RegisterOptions
         {
             Match = WampMatchPattern.Prefix
         }))
         .ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterCallee
        (new Callee3(),
         new CalleeRegistrationInterceptor(new RegisterOptions
         {
             Match = WampMatchPattern.Wildcard
         }))
         .ConfigureAwait(false);
}

public class Callee1
{
    [WampProcedure("com.example.procedure1")]
    public void Endpoint1(string message)
    {
        Console.WriteLine("endpoint1: msg = '{0}', procedure = '{1}'", message,
                          WampInvocationContext.Current.InvocationDetails.Procedure);
    }
}

public class Callee2
{
    [WampProcedure("com.example")]
    public void Endpoint2(string message)
    {
        Console.WriteLine("endpoint2: msg = '{0}', procedure = '{1}'", message,
                          WampInvocationContext.Current.InvocationDetails.Procedure);
    }             
}

public class Callee3
{
    [WampProcedure("com..procedure1")]
    public void Endpoint3(string message)
    {
        Console.WriteLine("endpoint3: msg = '{0}', procedure = '{1}'", message,
                          WampInvocationContext.Current.InvocationDetails.Procedure);
    }             
}

```

> Note: this sample is based on [this](https://github.com/crossbario/crossbarexamples/tree/master/patternregs) Autobahn sample
