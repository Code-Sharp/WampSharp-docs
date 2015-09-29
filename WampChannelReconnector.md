The WampChannelReconnector class helps handling re-connection of WampChannel.

Usage sample:

```csharp
public static async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    string address = "ws://localhost:8080/ws";

    MySubscriber mySubscriber = new MySubscriber();

    IWampChannel channel =
        factory.CreateJsonChannel(address, "realm1");

    Func<Task> connect = async () =>
    {
        await channel.Open();

        var subscriptionTask =
            channel.RealmProxy.Services.RegisterSubscriber(mySubscriber);

        var asyncDisposable = await subscriptionTask;
    };

    WampChannelReconnector reconnector =
        new WampChannelReconnector(channel, connect);

    reconnector.Start();
}
```