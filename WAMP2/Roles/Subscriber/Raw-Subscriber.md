## Raw Subscriber

Like other roles, subscriber also has a raw version, which allows you treat EVENT messages as you like. In order to use it, you need to implement the IWampRawTopicRouterSubscriber interface.

### Usage:

#### Client sample:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using SystemEx;
using WampSharp.Core.Serialization;
using WampSharp.V2;
using WampSharp.V2.Client;
using WampSharp.V2.Core.Contracts;
using WampSharp.V2.PubSub;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

            IWampChannel channel =
                factory.CreateJsonChannel(serverAddress, "realm1");

            Task openTask = channel.Open();

            openTask.Wait(TimeSpan.FromSeconds(5));

            IWampTopicProxy topicProxy =
                channel.RealmProxy.TopicContainer.GetTopicByUri("com.myapp.topic1");

            Task<IAsyncDisposable> subscribeTask =
                topicProxy.Subscribe(new MySubscriber(), new SubscribeOptions());

            Console.ReadLine();
        }
    }

    internal class MySubscriber : IWampRawTopicClientSubscriber
    {
        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, EventDetails details)
        {
            Console.WriteLine("Got event with publication id: " + publicationId);
        }

        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, EventDetails details, TMessage[] arguments)
        {
            int number = formatter.Deserialize<int>(arguments[0]);

            Console.WriteLine("Got event " + number + " with publication id: " + publicationId);
        }

        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, EventDetails details, TMessage[] arguments,
                                    IDictionary<string, TMessage> argumentsKeywords)
        {
            int number = formatter.Deserialize<int>(arguments[0]);

            Console.WriteLine("Got event " + number + " with publication id: " + publicationId);
        }
    }
}
```

#### Router sample:

```csharp
using System;
using System.Collections.Generic;
using WampSharp.Core.Serialization;
using WampSharp.V2;
using WampSharp.V2.Core.Contracts;
using WampSharp.V2.PubSub;
using WampSharp.V2.Realm;

namespace MyNamespace
{
    internal class Program
    {
        public static void Main(string[] args)
        {
            const string serverAddress = "ws://127.0.0.1:8080/ws";

            DefaultWampHost host = new DefaultWampHost(serverAddress);

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            IWampTopic topic =
                realm.TopicContainer.GetOrCreateTopicByUri("com.myapp.topic1");

            MySubscriber subscriber = new MySubscriber();

            IDisposable subscription = topic.Subscribe(subscriber);

            host.Open();

            Console.ReadLine();
        }
    }

    internal class MySubscriber : IWampRawTopicRouterSubscriber
    {
        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, PublishOptions options)
        {
            Console.WriteLine("Got event with publication id: " + publicationId);
        }

        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, PublishOptions options, TMessage[] arguments)
        {
            int number = formatter.Deserialize<int>(arguments[0]);

            Console.WriteLine("Got event " + number + " with publication id: " + publicationId);
        }

        public void Event<TMessage>(IWampFormatter<TMessage> formatter, long publicationId, PublishOptions options, TMessage[] arguments,
                                    IDictionary<string, TMessage> argumentsKeywords)
        {
            int number = formatter.Deserialize<int>(arguments[0]);

            Console.WriteLine("Got event " + number + " with publication id: " + publicationId);
        }
    }
}
```
