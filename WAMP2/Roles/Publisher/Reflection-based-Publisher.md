## **Reflection-based Publisher**

**Reflection-based Publisher** allows to use publisher features in a similar fashion as [Reflection based Callee](Reflection-based-Callee.md).

In order to use it, create a class containing an event decorated with a [WampTopic] attribute. Then register an instance of the class using the RegisterPublisher method of IWampRealmServiceProvider. The arguments published to the event will be treated as the arguments keywords of the publication.

### Basic usage
```csharp
public class MyClass
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }
}

public delegate void MyPublicationDelegate(int number1, int number2, string c, MyClass d);

public interface IMyPublisher
{
    [WampTopic("com.myapp.heartbeat")]
    event Action Heartbeat;

    [WampTopic("com.myapp.topic2")]
    event MyPublicationDelegate MyEvent;
}

public class MyPublisher : IMyPublisher
{
    private readonly Random mRandom = new Random();
    private IDisposable mSubscription;

    public MyPublisher()
    {
        mSubscription = Observable.Timer(TimeSpan.FromSeconds(0),
            TimeSpan.FromSeconds(1)).Select((x, i) => i)
            .Subscribe(x => OnTimer(x));
    }

    private void OnTimer(int value)
    {
        RaiseHeartbeat();

        RaiseMyEvent(mRandom.Next(0, 100),
            23,
            "Hello",
            new MyClass()
            {
                Counter = value,
                Foo = new int[] {1, 2, 3}
            });
    }

    private void RaiseHeartbeat()
    {
        Action handler = Heartbeat;

        if (handler != null)
        {
            handler();
        }
    }

    private void RaiseMyEvent(int number1, int number2, string c, MyClass d)
    {
        MyPublicationDelegate handler = MyEvent;

        if (handler != null)
        {
            handler(number1, number2, c, d);
        }
    }

    public event Action Heartbeat;

    public event MyPublicationDelegate MyEvent;
}

public static async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    IDisposable publisherDisposable =
        channel.RealmProxy.Services.RegisterPublisher(new MyPublisher());

    // call publisherDisposable.Dispose(); to unsubscribe from the event.
}
```

>Note: if the delegate used is of any Action&lt;&gt; type, the publication will send the parameters as the positional arguments of the publication, otherwise it will use the parameters as the keyword arguments of the publication (with the delegate parameters' names as the keys).

>Note:  These samples are based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample, but are a bit different (WampSharp doesn't support publishing both positional arguments and keyword arguments with this feature)
