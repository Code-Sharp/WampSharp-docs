## Getting started with Callee

#### Before you start

See [Getting started with WAMPv2](../../Getting-started-with-WAMPv2.md) and create a WampChannel/WampHost your calleee will be registered to.


### About Callee role

WAMPv2 defines a Callee role, that is a role that can register a remote procedure call to the router (using the REGISTER/UNREGISTER messages). The callee's procedure can be invoked by the router (using the INVOCATION message). The callee can respond with a result or an error to the router  (using the YIELD/ERROR message).

WampSharp supports two methods for consuming the callee role.

#### Reflection based callee

The Reflection based callee is the easiest way to register callee methods to a WAMP router.

In order to use it, create a class having methods decorated with [WampProcedure] attribute.
Then create an instance of the class and register it using the RegisterCalee method of the Services property  
of the IWampRealmProxy/IWampRealm instance.

##### Client sample

```csharp
using System;
using System.Threading.Tasks;
using SystemEx;
using WampSharp.V2;
using WampSharp.V2.Client;
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
    }

    public class ArgumentsService : IArgumentsService
    {
        public void Ping()
        {
        }

        public int Add2(int a, int b)
        {
            return a + b;
        }

        public string Stars(string nick = "somebody", int stars = 0)
        {
            return string.Format("{0} starred {1}x", nick, stars);
        }
    }

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

            IArgumentsService instance = new ArgumentsService();

            IWampRealmProxy realm = channel.RealmProxy;

            Task<IAsyncDisposable> registrationTask = realm.Services.RegisterCallee(instance);
            // await registrationTask;
            registrationTask.Wait();

            Console.ReadLine();
        }
    }
}
```

##### Router sample

```csharp
using System;
using System.Threading.Tasks;
using SystemEx;
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
    }

    public class ArgumentsService : IArgumentsService
    {
        public void Ping()
        {
        }

        public int Add2(int a, int b)
        {
            return a + b;
        }

        public string Stars(string nick = "somebody", int stars = 0)
        {
            return string.Format("{0} starred {1}x", nick, stars);
        }
    }

    internal class Program
    {
        public static void Main(string[] args)
        {
            const string location = "ws://127.0.0.1:8080/";

            using (IWampHost host = new DefaultWampHost(location))
            {
                IArgumentsService instance = new ArgumentsService();

                IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

                Task<IAsyncDisposable> registrationTask = realm.Services.RegisterCallee(instance);
                // await registrationTask;
                registrationTask.Wait();

                host.Open();

                Console.WriteLine("Server is running on " + location);
                Console.ReadLine();
            }
        }
    }
}
```

#### See also

* [Reflection-based Callee](Reflection-based-Callee.md)
* [Raw Callee](Raw-Callee.md)
