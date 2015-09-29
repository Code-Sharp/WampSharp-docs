## Raw callee

WampSharp provides a lower level api that allows to deal with rpc operations as they're sent/received, which is called Raw callee.

Actually, the reflection based callee api is built above the Raw callee api.

In order to use the raw callee api, implement IWampRpcOperation. A IWampFormatter is passed to your methods in order to allow you to deserialize the method parameters. In order to return a result or an error, call the corresponding caller method.

After that, register it using Register method of RpcOperationCatalog property of IWampRealm/IWampRealmProxy interfaces.

### Registration samples

#### Client side

```csharp
internal class Program
{
    public static void Main(string[] args)
    {
        const string location = "ws://127.0.0.1:8080/";

        DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

        IWampChannel channel = channelFactory.CreateJsonChannel(location, "realm1");

        Task openTask = channel.Open();

        // await openTask;
        openTask.Wait();

        ArgLenOperation operation = new ArgLenOperation();

        IWampRealmProxy realm = channel.RealmProxy;

        RegisterOptions registerOptions = new RegisterOptions();

        Task<IAsyncDisposable> registrationTask = realm.RpcCatalog.Register(operation, registerOptions);
        // await registrationTask;
        registrationTask.Wait();

        Console.ReadLine();
    }
}
```

#### Router side


```csharp
internal class Program
{
    public static void Main(string[] args)
    {
        const string location = "ws://127.0.0.1:8080/";

        using (IWampHost host = new DefaultWampHost(location))
        {
            ArgLenOperation operation = new ArgLenOperation();

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            Task<IAsyncDisposable> registrationTask = realm.Services.RegisterCallee(operation);
            // await registrationTask;
            registrationTask.Wait();

            host.Open();

            Console.WriteLine("Server is running on " + location);
            Console.ReadLine();
        }
    }
}
```

### IWampRpcOperation implementation sample

```csharp
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

### SyncLocalRpcOperation and AsyncLocalRpcOperation.

WampSharp defines some base-classes that make it easier to implement IWampRpcOperation, called SyncLocalRpcOperation and AsyncLocalRpcOperation.

In these classes, we define the parameters we expect and let WampSharp deserialize them for us.

#### Static parameters example

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

#### Dynamic parameters example

```csharp
public class ArgLenOperation : SyncLocalRpcOperation
{
    private readonly RpcParameter[] mParameters = new RpcParameter[0];

    public ArgLenOperation()
        : base("com.arguments.arglen")
    {
    }

    protected override object InvokeSync<TMessage>(IWampRawRpcOperationRouterCallback caller,
                                                   IWampFormatter<TMessage> formatter,
                                                   InvocationDetails details,
                                                   TMessage[] arguments,
                                                   IDictionary<string, TMessage> argumentsKeywords,
                                                   out IDictionary<string, object> outputs)
    {
        outputs = null;

        int argumentsLength = 0;

        if (arguments != null)
        {
            argumentsLength = arguments.Length;
        }

        int argumentKeyWordsLength = 0;

        if (argumentsKeywords != null)
        {
            argumentKeyWordsLength = argumentsKeywords.Count;
        }

        return new int[] { argumentsLength, argumentKeyWordsLength };
    }

    public override RpcParameter[] Parameters
    {
        get { return mParameters; }
    }

    public override bool HasResult
    {
        get { return true; }
    }

    public override CollectionResultTreatment CollectionResultTreatment
    {
        get { return CollectionResultTreatment.Multivalued; }
    }
}
```
