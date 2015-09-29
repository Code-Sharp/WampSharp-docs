## Getting started with Caller

#### Before you start

See [Getting started with WAMPv2](../../Getting-started-with-WAMPv2.md) and create a WampChannel/WampHost your caller will use.

### About Caller role

WAMPv2 defines a Caller role, that is a role that can call a remote procedure call registered by a [Callee](../Callee/Getting-started-with-Callee.md) (using the CALL messages). The caller can receive a response with a result or an error from the router  (using the RESULT/ERROR message).

#### Callee proxy

The Callee proxy is the easiest way to call rpc methods of a WAMP router.

In order to use it, create an interface having methods decorated with [WampProcedure] attribute.
Then create a proxy to a callee using the GetCalleeProxy method of the Services property  
of the IWampRealm/IWampRealmProxy instance.

##### Client sample

```csharp
using System;
using WampSharp.V2;
using WampSharp.V2.Rpc;

namespace MyNamespace
{
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
    }

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

            IArgumentsService proxy =
                channel.RealmProxy.Services.GetCalleeProxy<IArgumentsService>();

            proxy.Ping();
            Console.WriteLine("Pinged!");

            int result = proxy.Add2(2, 3);
            Console.WriteLine("Add2: {0}", result);

            var starred = proxy.Stars();
            Console.WriteLine("Starred 1: {0}", starred);

            starred = proxy.Stars(nick: "Homer");
            Console.WriteLine("Starred 2: {0}", starred);

            starred = proxy.Stars(stars: 5);
            Console.WriteLine("Starred 3: {0}", starred);

            starred = proxy.Stars(nick: "Homer", stars: 5);
            Console.WriteLine("Starred 4: {0}", starred);

            string[] orders = proxy.Orders("coffee");
            Console.WriteLine("Orders 1: {0}", string.Join(", ", orders));

            orders = proxy.Orders("coffee", limit: 10);
            Console.WriteLine("Orders 2: {0}", string.Join(", ", orders));

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
using WampSharp.V2.Rpc;

namespace MyNamespace
{
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
    }

    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampHost host = new DefaultWampHost(serverAddress);

            host.Open();

            Console.WriteLine("Press enter when a client finishes registering methods");
            Console.ReadLine();

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            IArgumentsService proxy =
                realm.Services.GetCalleeProxy<IArgumentsService>();

            proxy.Ping();
            Console.WriteLine("Pinged!");

            int result = proxy.Add2(2, 3);
            Console.WriteLine("Add2: {0}", result);

            var starred = proxy.Stars();
            Console.WriteLine("Starred 1: {0}", starred);

            starred = proxy.Stars(nick: "Homer");
            Console.WriteLine("Starred 2: {0}", starred);

            starred = proxy.Stars(stars: 5);
            Console.WriteLine("Starred 3: {0}", starred);

            starred = proxy.Stars(nick: "Homer", stars: 5);
            Console.WriteLine("Starred 4: {0}", starred);

            string[] orders = proxy.Orders("coffee");
            Console.WriteLine("Orders 1: {0}", string.Join(", ", orders));

            orders = proxy.Orders("coffee", limit: 10);
            Console.WriteLine("Orders 2: {0}", string.Join(", ", orders));

            Console.ReadLine();
        }
    }
}
```

#### See also

* [Reflection-based Caller](Reflection-based-Caller.md)
* [Raw Caller](Raw-Caller.md)
