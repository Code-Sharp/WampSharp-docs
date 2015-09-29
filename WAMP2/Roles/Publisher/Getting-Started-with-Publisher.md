## Getting started with Publisher

#### Before you start

See [Getting started with WAMPv2](../../Getting-started-with-WAMPv2.md) and create a WampChannel/WampHost your publisher will use.

### About Publisher role

WAMPv2 defines a Publisher role, that is a role that can publish events to a WAMP realm's topic.

WampSharp supports two methods for consuming the publisher role.

#### WampSubject

The WampSubject is the easiest way to publish to a topic of a WAMP router.

In order to use it, call the GetSubject method of the Services property  
of the IWampRealm/IWampRealmProxy instance with a generic type representing the type you want to send to the router.

After that, call OnNext of the Subject with the event you are willing to publish.

##### Client sample

```csharp
using System;
using System.Reactive.Linq;
using System.Reactive.Subjects;
using System.Threading.Tasks;
using WampSharp.V2;
using WampSharp.V2.Client;
using WampSharp.V2.Realm;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

            IWampChannel channel = channelFactory.CreateJsonChannel(serverAddress, "realm1");

            Task openTask = channel.Open();

            openTask.Wait(5000);

            Console.WriteLine("Press enter after a subscriber subscribes to com.myapp.topic1");

            Console.ReadLine();

            IWampRealmProxy realm = channel.RealmProxy;

            ISubject<int> subject =
                realm.Services.GetSubject<int>("com.myapp.topic1");

            int counter = 0;

            IObservable<long> timer =
                Observable.Timer(TimeSpan.FromMilliseconds(0),
                                 TimeSpan.FromMilliseconds(1000));

            IDisposable disposable =
                timer.Subscribe(x =>
                {
                    counter++;

                    Console.WriteLine("Publishing to topic 'com.myapp.topic1': " + counter);
                    try
                    {
                        subject.OnNext(counter);
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine(ex);
                    }
                });

            Console.ReadLine();
        }
    }
}
```

##### Router sample

```csharp
using System;
using System.Reactive.Linq;
using System.Reactive.Subjects;
using WampSharp.V2;
using WampSharp.V2.Realm;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampHost host = new DefaultWampHost(serverAddress);

            host.Open();

            Console.WriteLine("Press enter after a subscriber subscribes to com.myapp.topic1");

            Console.ReadLine();

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            ISubject<int> subject =
                realm.Services.GetSubject<int>("com.myapp.topic1");

            int counter = 0;

            IObservable<long> timer =
                Observable.Timer(TimeSpan.FromMilliseconds(0),
                                 TimeSpan.FromMilliseconds(1000));

            IDisposable disposable =
                timer.Subscribe(x =>
                {
                    counter++;

                    Console.WriteLine("Publishing to topic 'com.myapp.topic1': " + counter);
                    try
                    {
                        subject.OnNext(counter);
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine(ex);
                    }
                });

            Console.ReadLine();
        }
    }
}
```

#### See also

* [Reflection-based Publisher](Reflection-based-Publisher.md)
* [Rx based Publisher](Rx-based-Publisher.md)
* [Raw Publisher](Raw-Publisher.md)
