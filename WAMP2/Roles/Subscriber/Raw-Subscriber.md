## Raw Subscriber

Like other roles, subscriber also has a raw version, which allows you treat EVENT messages as you like. In order to use it, you need to implement the IWampRawTopicClientSubscriber interface.

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

### Local Subscriber

WampSharp defines a base-class named LocalSubscriber that makes it easier to implement IWampRawTopicClientSubscriber.

In this class, we define the parameters we expect to receive and let WampSharp deserialize them for us.

#### Example

```csharp
public class MySubscriber : LocalSubscriber
{
    private readonly LocalParameter[] mEventParameters = new[]
    {
        new LocalParameter("number1", typeof (int), 0),
        new LocalParameter("number2", typeof (int), 1),
        new LocalParameter("c", typeof (string), 2),
        new LocalParameter("d", typeof (MyClass), 3),
    };

    protected override void InnerEvent<TMessage>
        (IWampFormatter<TMessage> formatter,
         long publicationId,
         EventDetails details,
         TMessage[] arguments,
         IDictionary<string, TMessage> argumentsKeywords)
    {
        object[] unpacked =
            base.UnpackParameters(formatter, arguments, argumentsKeywords);

        int number1 = (int) unpacked[0];
        int number2 = (int) unpacked[1];
        string c = (string) unpacked[2];
        MyClass d = (MyClass) unpacked[3];

        Console.WriteLine("Got event: number1:{0}, number2:{1}, c:{2}, d:{3}",
                          number1, number2, c, d);
    }

    public override LocalParameter[] Parameters
    {
        get { return mEventParameters; }
    }
}

private static void Run()
{
    const string serverAddress = "ws://127.0.0.1:8080/ws";

    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel(serverAddress, "realm1");

    Task openTask = channel.Open();

    openTask.Wait(TimeSpan.FromSeconds(5));

    IWampTopicProxy topicProxy =
        channel.RealmProxy.TopicContainer.GetTopicByUri("com.myapp.topic2");

    Task<IAsyncDisposable> subscribeTask =
        topicProxy.Subscribe(new MySubscriber(), new SubscribeOptions());

    Console.ReadLine();
}

public class MyClass
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }
}
```

This handles lookups for parameters named number1, number2, c and d (or positioned at 0, 1, 2, 3), and throws an exception if not all are present.
