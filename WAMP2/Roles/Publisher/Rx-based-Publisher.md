## Rx-based Publisher

**Rx-based Publisher** allows you to publish events to a topic of a WAMP router realm, using [Reactive Extensions](http://reactivex.io/) Observer api.

### Basic usage

```csharp
private static void Run()
{
    const string serverAddress = "ws://127.0.0.1:8080/ws";

    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel = channelFactory.CreateJsonChannel(serverAddress, "realm1");

    Task openTask = channel.Open();

    openTask.Wait(5000);

    IWampRealmProxy realm = channel.RealmProxy;

    ISubject<int> subject =
        realm.Services.GetSubject<int>("com.myapp.topic1");

    // Publishes 5 to com.myapp.topic1
    subject.OnNext(5);
}
```

### GetSubject overloads

If a generic type is specified in the GetSubject method, the ARGUMENTS parameter of the EVENT message will contain the single parameter OnNext receives.

Specifying no generic type will a return a IWampSubject, that is a IObserver of a IWampEvent. IWampEvent is an interface representing an outgoing WAMP PUBLISH message. It has properties that represent PUBLISH message arguments.

#### IWampSubject example

```csharp
private static void Run()
{
    const string serverAddress = "ws://127.0.0.1:8080/ws";

    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel = channelFactory.CreateJsonChannel(serverAddress, "realm1");

    Task openTask = channel.Open();

    openTask.Wait(5000);

    IWampRealmProxy realm = channel.RealmProxy;

    IWampSubject subject =
        realm.Services.GetSubject("com.myapp.topic2");

    int counter = 0;

    IObservable<long> timer =
        Observable.Timer(TimeSpan.FromMilliseconds(0),
                         TimeSpan.FromMilliseconds(1000));

    Random random = new Random();

    IDisposable disposable =
        timer.Subscribe(x =>
        {
            var obj =
                new
                {
                    counter = counter,
                    foo = new int[] {1, 2, 3}
                };

            WampEvent @event = new WampEvent()
            {
                Options = new PublishOptions {DiscloseMe = true},
                Arguments = new object[] {random.Next(0, 100), 23},
                ArgumentsKeywords = new Dictionary<string, object>
                {
                    {"c", "Hello"},
                    {"d", obj}
                }
            };

            subject.OnNext(@event);
            Console.WriteLine("events published");

            counter += 1;
        });

    Console.ReadLine();
}
```

> This example is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample.
