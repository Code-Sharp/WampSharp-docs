### Callee proxy

As demonstrated in the [[Getting Started with Caller]] page, callee proxy allows you to call callee methods of a WAMP realm, by declaring an interface with methods decorated with a [WampProcedure] attribute.

The interface must be public.

The following features are supported:
* Async method support. A method returning a Task<> can be awaited. Example:
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

* Out/ref parameters: For synchronous methods, out/ref parameters are supported. Note: this is not supported for asynchronous methods. Example:
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

* Multi-valued results: in order to get an multivalued array from the RESULT/YIELD WAMPv2 message, set the return value of the rpc method to an array and place above it a [return: WampResult(CollectionResultTreatment.Multivalued)] attribute. Example:
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

* Exception support: catch a WampException in order to treat a ERROR message.

### Progressive calls

In order to use progressive calls as a Caller, declare in your callee service a [WampProcedure] method having a [WampProgressiveCall] attribute and a IProgress&lt;T&gt; as the last parameter.
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

### Caller identification

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

### Raw api

The callee proxy is the easiest way to consume WAMP caller capabilities, but it is limited to C# features. In some cases you might want to handle a RESULT/YIELD message differently. For these cases, the Raw callback api exists.

In order to use raw callback api from a WampSharp client, create a class implementing IWampRawRpcOperationClientCallback. This class will be notified when a result arrives.
Then create a new instance of your class, and access RpcCatalog property of WampChannel's RealmProxy, then call Invoke of your desired method with desired parameters. 

The IWampRawRpcOperationClientCallback methods receive a IWampFormatter so you can deserialize the message parameters yourself.

Client sample code:
```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using WampSharp.Core.Serialization;
using WampSharp.V2;
using WampSharp.V2.Client;
using WampSharp.V2.Core.Contracts;
using WampSharp.V2.Rpc;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampChannelFactory factory = new DefaultWampChannelFactory();
            IWampChannel channel = factory.CreateJsonChannel(serverAddress, "realm1");

            Task openTask = channel.Open();

            openTask.Wait(5000);

            IWampRealmProxy realmProxy = channel.RealmProxy;

            realmProxy.RpcCatalog.Invoke
                (new MyCallback(),
                 new CallOptions(),
                 "com.myapp.add_complex",
                 new object[] {2, 3, 4, 5});

            Console.ReadLine();
        }
    }

    public class MyCallback : IWampRawRpcOperationClientCallback
    {
        public void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details)
        {
            throw new NotImplementedException();
        }

        public void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments)
        {
            throw new NotImplementedException();
        }

        public void Result<TMessage>(IWampFormatter<TMessage> formatter,
                                     ResultDetails details,
                                     TMessage[] arguments,
                                     IDictionary<string, TMessage> argumentsKeywords)
        {
            int c = formatter.Deserialize<int>(argumentsKeywords["c"]);
            int ci = formatter.Deserialize<int>(argumentsKeywords["ci"]);

            Console.WriteLine("Got result: " + new {c, ci});
        }

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error)
        {
            throw new NotImplementedException();
        }

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error, TMessage[] arguments)
        {
            throw new NotImplementedException();
        }

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error, TMessage[] arguments,
                                    TMessage argumentsKeywords)
        {
            throw new NotImplementedException();
        }
    }
}
```