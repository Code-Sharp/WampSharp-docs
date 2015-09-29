## Getting started with Subscriber

#### Before you start

See [Getting started with WAMPv2](..\..\Getting-started-with-WAMPv2.md) and create a WampChannel/WampHost your subscriber will use.

### About Subscriber role

WAMPv2 defines a Subscriber role, that is a role that can subscribe to a WAMP realm's topic. The subscriber will be notified about events published to the topic by publishers.

WampSharp supports two methods for consuming the subscriber role.

#### WampSubject

The WampSubject is the easiest way to subscribe to a topic of a WAMP router.

In order to use it, call the GetSubject method of the Services property  
of the IWampRealmProxy/IWampRealm instance with a generic type representing the type expected to be received from the router.


##### Client sample

```csharp
using System;
using WampSharp.V2;
using WampSharp.V2.Client;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            DefaultWampChannelFactory factory =
                new DefaultWampChannelFactory();

            const string serverAddress = "ws://127.0.0.1:8080/ws";

            IWampChannel channel =
                factory.CreateJsonChannel(serverAddress, "realm1");

            channel.Open().Wait(5000);

            IWampRealmProxy realmProxy = channel.RealmProxy;

            int received = 0;
            IDisposable subscription = null;

            subscription =
                realmProxy.Services.GetSubject<int>("com.myapp.topic1")
                     .Subscribe(x =>
                         {
                             Console.WriteLine("Got Event: " + x);

                             received++;

                             if (received > 5)
                             {
                                 Console.WriteLine("Closing ..");
                                 subscription.Dispose();
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

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            int received = 0;
            IDisposable subscription = null;

            subscription =
                realm.Services.GetSubject<int>("com.myapp.topic1")
                     .Subscribe(x =>
                         {
                             Console.WriteLine("Got Event: " + x);

                             received++;

                             if (received > 5)
                             {
                                 Console.WriteLine("Closing ..");
                                 subscription.Dispose();
                             }
                         });

            Console.ReadLine();
        }
    }
}
```

#### See also

* [Reflection-based Subscriber](Reflection-based-Subscriber.md)
* [Rx based Subscriber](Rx-based-Subscriber.md)
* [Raw Subscriber](Raw-Subscriber.md)
