## Rx-based Subscriber

**Rx-based Subscriber** allows you to subscribe to events of a topic of a WAMP router realm, using [Reactive Extensions](http://reactivex.io/) Observable api.

### Basic usage

```csharp
private static void Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    const string serverAddress = "ws://127.0.0.1:8080/ws";

    IWampChannel channel =
        factory.CreateJsonChannel(serverAddress, "realm1");

    channel.Open().Wait(5000);

    IWampRealmProxy realmProxy = channel.RealmProxy;

    IDisposable subscription =
        realmProxy.Services.GetSubject<int>("com.myapp.topic1")
                  .Subscribe(x =>
                  {
                      Console.WriteLine("Got Event: " + x);
                  });

    // Call subscription.Dispose(); to unsubscribe.
}
```

### GetSubject overloads

GetSubject allows you to subscribe to a topic of a WAMP realm, given the type of the received events.

If a generic type is specified, only the first argument in the ARGUMENTS parameter of the EVENT message is deserialized.

Specifying no generic type will a return a IWampSubject, that is a IObservable of a IWampSerializedEvent. IWampSerializedEvent is an interface representing an incoming WAMP EVENT message. It has properties of type ISerializedValue that can be deserialized.

#### IWampSubject example

```csharp
public static void Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    const string serverAddress = "ws://127.0.0.1:8080/ws";

    IWampChannel channel =
        factory.CreateJsonChannel(serverAddress, "realm1");

    channel.Open().Wait(5000);

    IWampRealmProxy realmProxy = channel.RealmProxy;

    IWampSubject topic =
        realmProxy.Services.GetSubject("com.myapp.topic2");

    IDisposable disposable =
        topic.Subscribe(OnTopic2);

    Console.ReadLine();

    disposable.Dispose();
}

private static void OnTopic2(IWampSerializedEvent serializedEvent)
{
    int[] arguments =
        serializedEvent.Arguments.Select(argument => argument.Deserialize<int>())
         .ToArray();

    string c =
        serializedEvent.ArgumentsKeywords["c"].Deserialize<string>();

    ComplexContract d =
        serializedEvent.ArgumentsKeywords["d"].Deserialize<ComplexContract>();


    var deserializedArguments =
        new
        {
            arguments,
            argumentsKeywords = new
            {
                c,
                d
            }
        };

    Console.WriteLine("Got event: args: [{0}], kwargs: {{ {1} }}",
                      string.Join(", ", deserializedArguments.arguments),
                      deserializedArguments.argumentsKeywords);
}

public class ComplexContract
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }

    public override string ToString()
    {
        return $"counter: {Counter}, foo: [{string.Join(", ", Foo)}]";
    }
}
```

> This example is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample.
