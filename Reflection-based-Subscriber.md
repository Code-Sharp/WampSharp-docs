## Reflection-based Subscriber

**Reflection-based Subscriber** allows to use WAMPv2 subscriber features in a similar fashion as [Reflection based Callee](Reflection-based-Callee.md).

In order to use it, create a class with a method having a [WampTopic] attribute, Then call RegisterSubscriber of IWampRealmServiceProvider.

Both placing attributes on a class method and placing attributes on an interface implemented by the class are supported.

### Basic usage

```csharp
public class MyClass
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }

    public override string ToString()
    {
        return string.Format("counter: {0}, foo: [{1}]",
            Counter,
            string.Join(", ", Foo));
    }
}

public interface IMySubscriber
{
    [WampTopic("com.myapp.heartbeat")]
    void OnHeartbeat();

    [WampTopic("com.myapp.topic2")]
    void OnTopic2(int number1, int number2, string c, MyClass d);
}

public class MySubscriber : IMySubscriber
{
    public void OnHeartbeat()
    {
        long publicationId = WampEventContext.Current.PublicationId;
        Console.WriteLine("Got heartbeat (publication ID " + publicationId + ")");
    }

    public void OnTopic2(int number1, int number2, string c, MyClass d)
    {
        Console.WriteLine("Got event: number1:{0}, number2:{1}, c:{2}, d:{3}",
            number1, number2, c, d);
    }
}

public static async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    Task<IAsyncDisposable> subscriptionTask =
        channel.RealmProxy.Services.RegisterSubscriber(new MySubscriber());

    IAsyncDisposable asyncDisposable = await subscriptionTask;

    // call await asyncDisposable.DisposeAsync(); to unsubscribe from the topic.
}

```

>Note:  This sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample

### Supported features

#### WampEventContext

You can use WampEventContext.Current in order to get details about the current received event:

```csharp
public class MySubscriber
{
    [WampTopic("com.myapp.topic1")]
    public void OnTopic1(int counter)
    {
        WampEventContext context = WampEventContext.Current;

        Console.WriteLine("Got event, publication ID {0}, publisher {1}: {2}",
            context.PublicationId,
            context.EventDetails.Publisher,
            counter);
    }
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/options) AutobahnJS sample.

#### Subscription customization

The RegisterSubscriber method of IWampRealmServiceProvider has an overload that receives an "interceptor" instance. The "interceptor" allow customizing the subscription being performed.

This allows customizing the subscribe options and the topic uri sent upon SUBSCRIBE message.

The following sections discuss subscribe options modifications that can be used in order to leverage WAMP advanced profile features.

#### Pattern based subscriptions

 [Pattern based subscriptions](http://crossbar.io/docs/Pattern-Based-Subscriptions/), for both router side and client side.

In order to use it, pass SubscribeOptions with Match = "exact"/"prefix"/"wildcard" depending on your criteria (these are also available in a static class called WampMatchPattern):

```csharp
public async Task Run()
{
    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampChannel channel =
        channelFactory.CreateJsonChannel("ws://127.0.0.1:8080/ws",
            "realm1");

    await channel.Open().ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterSubscriber(new Subscriber1())
        .ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterSubscriber
        (new Subscriber2(),
         new SubscriberRegistrationInterceptor(new SubscribeOptions
         {
             Match = WampMatchPattern.Prefix
         }))
         .ConfigureAwait(false);

    await channel.RealmProxy.Services.RegisterSubscriber
        (new Subscriber3(),
         new SubscriberRegistrationInterceptor(new SubscribeOptions
         {
             Match = WampMatchPattern.Wildcard
         }))
         .ConfigureAwait(false);
}

public class Subscriber1
{
    [WampTopic("com.example.topic1")]
    public void Handler1(string message)
    {
        Console.WriteLine("handler1: msg = '{0}', topic = '{1}'", message,
                          WampEventContext.Current.EventDetails.Topic);
    }
}

public class Subscriber2
{
    [WampTopic("com.example")]
    public void Handler2(string message)
    {
        Console.WriteLine("handler2: msg = '{0}', topic = '{1}'", message,
                          WampEventContext.Current.EventDetails.Topic);
    }             
}

public class Subscriber3
{
    [WampTopic("com..topic1")]
    public void Handler3(string message)
    {
        Console.WriteLine("handler3: msg = '{0}', topic = '{1}'", message,
                          WampEventContext.Current.EventDetails.Topic);
    }             
}
```
> Note: this sample is based on [this](https://github.com/crossbario/crossbarexamples/tree/master/patternsubs) Autobahn sample
