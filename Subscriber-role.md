### WampSubject

As demonstrated in the [[Getting Started with Subscriber]] page, WampSubject allows you to subscribe to a topic of a WAMP realm, given the type of the received events.

If specifying a generic type, only the first argument in the ARGUMENTS parameter of the EVENT message is deserialized.

Specifying no generic type will a return a IWampSubject, that is a IObservable of a IWampSerializedEvent. IWampSerializedEvent is an interface representing an incoming WAMP EVENT message. It has properties of type ISerializedValue that can be deserialized.

For example:

```csharp
using System;
using System.Linq;
using System.Reactive.Linq;
using Newtonsoft.Json;
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

            var topic =
                realmProxy.Services.GetSubject("com.myapp.topic2")
                          .Select(x =>
                              {
                                  var arguments =
                                      x.Arguments.Select(argument => argument.Deserialize<int>()).ToArray();

                                  string c =
                                      x.ArgumentsKeywords["c"].Deserialize<string>();

                                  DPropertyClass d =
                                      x.ArgumentsKeywords["d"].Deserialize<DPropertyClass>();


                                  return new
                                      {
                                          arguments,
                                          argumentsKeywords = new
                                              {
                                                  c,
                                                  d
                                              }
                                      };
                              });

            IDisposable disposable =
                topic.Subscribe(x =>
                {
                    Console.WriteLine("Got event: args: [{0}], kwargs: {{ {1} }}",
                                      string.Join(", ", x.arguments),
                                      x.argumentsKeywords);
                });

            Console.ReadLine();

            disposable.Dispose();
        }
    }

    public class DPropertyClass
    {
        [JsonProperty("counter")]
        public int Counter { get; set; }

        [JsonProperty("foo")]
        public int[] Foo { get; set; }

        public override string ToString()
        {
            return string.Format("counter: {0}, foo: [{1}]", Counter, string.Join(", ", Foo));
        }
    }
}
```
