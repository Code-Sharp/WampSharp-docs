## Raw api

The callee proxy is the easiest way to consume WAMP caller capabilities, but it is limited to C# features. In some cases you might want to handle a YIELD message differently. For these cases, the Raw callback api exists.

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

### Manual proxy samples

This sample demonstrates how a manual implementation of [[Reflection-based callee]].

```csharp
public class ArgumentsServiceProxy : IArgumentsService
{
    private readonly IWampRpcOperationCatalogProxy mCatalogProxy;
    private readonly CallOptions mDummy = new CallOptions();

    public ArgumentsServiceProxy(IWampRpcOperationCatalogProxy catalogProxy)
    {
        mCatalogProxy = catalogProxy;
    }

    public Task PingAsync()
    {
        PingCallback callback = new PingCallback();
        mCatalogProxy.Invoke(callback, mDummy, "com.arguments.ping");
        return callback.Task;
    }

    public Task<int> Add2Async(int a, int b)
    {
        AddCallback callback = new AddCallback();
        mCatalogProxy.Invoke(callback, mDummy, "com.arguments.add2", new object[] {a, b});
        return callback.Task;
    }

    public Task<string> StarsAsync(string nick = "somebody", int stars = 0)
    {
        StarsCallback callback = new StarsCallback();

        mCatalogProxy.Invoke(callback, mDummy, "com.arguments.stars", new object[] { nick, stars });
        return callback.Task;
    }

    public Task<string[]> OrdersAsync(string product, int limit = 5)
    {
        OrdersCallback callback = new OrdersCallback();
        mCatalogProxy.Invoke(callback, mDummy, "com.arguments.orders", new object[] {product, limit});
        return callback.Task;
    }

    public void Ping()
    {
        try
        {
            this.PingAsync().Wait();
        }
        catch (AggregateException ex)
        {
            throw ex.InnerException;
        }
    }

    public int Add2(int a, int b)
    {
        try
        {
            return this.Add2Async(a, b).Result;
        }
        catch (AggregateException ex)
        {
            throw ex.InnerException;
        }
    }

    public string[] Orders(string product, int limit = 5)
    {
        try
        {
            return this.OrdersAsync(product, limit).Result;
        }
        catch (AggregateException ex)
        {
            throw ex.InnerException;
        }
    }

    public string Stars(string nick = "somebody", int stars = 0)
    {
        try
        {
            return this.StarsAsync(nick, stars).Result;
        }
        catch (AggregateException ex)
        {
            throw ex.InnerException;
        }
    }

    private abstract class Callback<T> : IWampRawRpcOperationClientCallback
    {
        protected readonly TaskCompletionSource<T> mTask = new TaskCompletionSource<T>();

        public Task<T> Task
        {
            get { return mTask.Task; }
        }

        public abstract void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details);

        public abstract void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments);

        public abstract void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords);

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error)
        {
            IDictionary<string, object> deserializedDetails =
                formatter.Deserialize<IDictionary<string, object>>(details);

            WampException exception = new WampException
                (deserializedDetails,
                 error);

            mTask.SetException(exception);
        }

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error,
                                    TMessage[] arguments)
        {
            IDictionary<string, object> deserializedDetails =
                formatter.Deserialize<IDictionary<string, object>>(details);

            object[] deserializedArguments = arguments.Cast<object>().ToArray();

            WampException exception = new WampException
                (deserializedDetails,
                 error,
                 deserializedArguments);

            mTask.SetException(exception);
        }

        public void Error<TMessage>(IWampFormatter<TMessage> formatter, TMessage details, string error,
                                    TMessage[] arguments,
                                    TMessage argumentsKeywords)
        {
            IDictionary<string, object> deserializedDetails =
                formatter.Deserialize<IDictionary<string, object>>(details);

            object[] deserializedArguments = arguments.Cast<object>().ToArray();

            IDictionary<string, object> deserializedArgumentsKeywords =
                formatter.Deserialize<IDictionary<string, object>>(argumentsKeywords);

            WampException exception = new WampException
                (deserializedDetails,
                 error,
                 deserializedArguments,
                 deserializedArgumentsKeywords);

            mTask.SetException(exception);
        }
    }

    private class PingCallback : Callback<bool>
    {
        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details)
        {
            mTask.SetResult(true);
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments)
        {
            mTask.SetResult(true);
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords)
        {
            mTask.SetResult(true);
        }
    }

    private class AddCallback : Callback<int>
    {
        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details)
        {
            throw new NotImplementedException();
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments)
        {
            int result = formatter.Deserialize<int>(arguments[0]);
            mTask.SetResult(result);
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords)
        {
            int result = formatter.Deserialize<int>(arguments[0]);
            mTask.SetResult(result);
        }
    }

    private class StarsCallback : Callback<string>
    {
        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details)
        {
            throw new NotImplementedException();
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments)
        {
            string result = formatter.Deserialize<string>(arguments[0]);
            mTask.SetResult(result);
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords)
        {
            string result = formatter.Deserialize<string>(arguments[0]);
            mTask.SetResult(result);
        }
    }

    private class OrdersCallback : Callback<string[]>
    {
        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details)
        {
            throw new NotImplementedException();
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments)
        {
            string[] result =
                arguments.Select(x => formatter.Deserialize<string>(x))
                         .ToArray();

            mTask.SetResult(result);
        }

        public override void Result<TMessage>(IWampFormatter<TMessage> formatter, ResultDetails details, TMessage[] arguments, IDictionary<string, TMessage> argumentsKeywords)
        {
            string[] result =
                arguments.Select(x => formatter.Deserialize<string>(x))
                         .ToArray();

            mTask.SetResult(result);
        }
    }
}

public async static Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel(serverAddress, "realm1");

    await channel.Open();

    ArgumentsServiceProxy proxy =
        new ArgumentsServiceProxy(channel.RealmProxy.RpcCatalog);

    int nine = await proxy.Add2Async(4, 5).ConfigureAwait(false);  
}
```
