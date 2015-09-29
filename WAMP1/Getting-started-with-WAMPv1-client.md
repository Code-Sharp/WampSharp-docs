## Getting started with WAMPv1 client

###Before you begin
Raise up a WampHost such as in the [[Getting started tutorial|Getting started with WAMPv1]].

This tutorial will work against it.

###Create a console application
Create a new Console Application in Visual Studio.

Open Package Manager Console (Tools -> Library Package Manager -> Package Manager Console) and enter the command
<code>
Install-Package WampSharp.Default -Pre
</code>

This will install the pre-release version of WampSharp.

###Creating a channel factory

In order to create a channel factory use DefaultWampChannelFactory's default constructor.

```csharp
DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();
```

###Obtaining a channel

A channel is an object that represents a WAMP session, and provides services for server communication.
In order to obtain a channel, use the channel factory you've created earlier and call CreateChannel extension method:

```csharp
IWampChannel<JToken> channel =
    channelFactory.CreateChannel("ws://127.0.0.1:9000/");
```

After obtaining a channel, we open it. This establishes a connection to the server and waits for the WELCOME message:

```csharp
channel.Open();
// NOTE: async version is also available;
// Task channelOpen = channel.OpenAsync();
```

### Consuming RPC methods
In order to consume a WAMP RPC method, we first need to declare an interface representing our service contract:
For instance

```csharp
public interface ICalculator
{
    [WampRpcMethod("http://example.com/simple/calc#add")]
    int Add(int x, int y);
}

public interface IAsyncCalculator
{
    [WampRpcMethod("http://example.com/simple/calc#add")]
    Task<int> Add(int x, int y);
}
```

After that, we can use the channel to get a proxy to this interface, and call its method:
```csharp
ICalculator proxy = channel.GetRpcProxy<ICalculator>();
int five = proxy.Add(2, 3);
Console.WriteLine("2 + 3 = " + five);

// Async version
IAsyncCalculator asyncProxy = channel.GetRpcProxy<IAsyncCalculator>();
Task<int> asyncFive = asyncProxy.Add(2, 3);
Console.WriteLine("2 + 3 = " + asyncFive.Result);
```

### Consuming PubSub topics capabilities
In order to use pub/sub capabilities, we call channel's GetSubject method: this method has a generic type parameter representing the type of the event published to the Subject. In addition, this method receives the uri of the topic we want to subscribe/publish to.

We first create the event type:
```csharp
public class TopicEvent
{
    [JsonProperty("a")]
    public string A { get; set; }

    [JsonProperty("b")]
    public string B { get; set; }

    [JsonProperty("c")]
    public int C { get; set; }

    public override string ToString()
    {
        return string.Format("A: {0}, B: {1}, C: {2}", A, B, C);
    }
}
```

After that, we can get a proxy to the server's topic:
```csharp
ISubject<TopicEvent> subject =
    channel.GetSubject<TopicEvent>(@"http://example.com/simple");
```

Subscribe to it:
```csharp
IDisposable subscription =
    subject.Subscribe(x => Console.WriteLine(x));
```

Publish events to it:
```csharp
subject.OnNext(new TopicEvent()
                   {
                       A = "Yo",
                       B = "Cool",
                       C = 3
                   });
```

And unsubscribe from it:
```csharp
subscription.Dispose();
```

### A complete sample
```csharp
using System;
using System.Reactive.Subjects;
using System.Threading.Tasks;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using WampSharp.V1;
using WampSharp.V1.Rpc;

namespace ClientSample
{
    public class Program
    {
        public class TopicEvent
        {
            [JsonProperty("a")]
            public string A { get; set; }

            [JsonProperty("b")]
            public string B { get; set; }

            [JsonProperty("c")]
            public int C { get; set; }

            public override string ToString()
            {
                return string.Format("A: {0}, B: {1}, C: {2}", A, B, C);
            }
        }

        public interface ICalculator
        {
            [WampRpcMethod("http://example.com/simple/calc#add")]
            int Add(int x, int y);
        }

        public interface IAsyncCalculator
        {
            [WampRpcMethod("http://example.com/simple/calc#add")]
            Task<int> Add(int x, int y);
        }

        static void Main(string[] args)
        {
            DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

            IWampChannel<JToken> channel =
                channelFactory.CreateChannel("ws://127.0.0.1:9000/");

            channel.Open();

            // RPC Method Call
            ICalculator proxy = channel.GetRpcProxy<ICalculator>();

            int five = proxy.Add(2, 3);

            Console.WriteLine("2 + 3 = " + five);

            // Async version
            IAsyncCalculator asyncProxy =
                channel.GetRpcProxy<IAsyncCalculator>();

            Task<int> asyncFive =
                asyncProxy.Add(2, 3);

            Console.WriteLine("2 + 3 = " + asyncFive.Result);


            // PubSub subscription:
            ISubject<TopicEvent> subject =
                channel.GetSubject<TopicEvent>(@"http://example.com/simple");

            IDisposable subscription =
                subject.Subscribe(x => Console.WriteLine("Received " + x));

            while (true)
            {
                Console.WriteLine("Enter publish or unsubscribe");
                string line = Console.ReadLine();

                switch (line)
                {
                    case "publish":
                        subject.OnNext(new TopicEvent()
                        {
                            A = "Yo",
                            B = "Cool",
                            C = 3
                        });
                        break;
                    case "unsubscribe":
                        subscription.Dispose();
                        break;
                }
            }
        }
    }
}
```
