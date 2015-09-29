## Publisher role

### WampSubject

As demonstrated in the [Getting Started with Publisher](Getting-Started-with-Publisher.md) page, WampSubject allows you to publish to a topic of a WAMP realm, with the type of the sent events.

If specifying a generic type, the ARGUMENTS parameter of the EVENT message will contain the single parameter OnNext receives.

Specifying no generic type will a return a IWampSubject, that is a IObserver of a IWampEvent. IWampEvent is an interface representing an outgoing WAMP PUBLISH message. It has properties that represent PUBLISH message arguments.

For example:

```csharp
const string serverAddress = "ws://127.0.0.1:8080/ws";

DefaultWampHost host = new DefaultWampHost(serverAddress);

host.Open();

Console.WriteLine("Press enter after a subscriber subscribes to com.myapp.topic1");

Console.ReadLine();

IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

IWampSubject subject =
    realm.Services.GetSubject("com.myapp.topic2");

int counter = 0;
Random random = new Random();

IObservable<long> timer =
    Observable.Timer(TimeSpan.FromMilliseconds(0),
                     TimeSpan.FromMilliseconds(1000));

IDisposable disposable =
    timer.Subscribe(x =>
        {
            var obj = new
                {
                    counter = counter,
                    foo = new int[] {1, 2, 3}
                };

            WampEvent @event = new WampEvent()
                {
                    Options = new PublishOptions(),
                    Arguments = new object[] {random.Next(0, 100), 23},
                    ArgumentsKeywords = new Dictionary<string, object> {{"c", "Hello"}, {"d", obj}}
                };

            try
            {
                subject.OnNext(@event);
                Console.WriteLine("events published");
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }

            counter += 1;
        });

Console.ReadLine();
```
