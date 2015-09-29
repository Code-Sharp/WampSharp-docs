## WAMP-CRA client-side authentication

[WAMP-CRA](https://github.com/tavendo/WAMP/blob/master/spec/advanced/challenge-response-authentication.md/) client side authentication is  supported. In order to use it, instantiate a new instance of WampCraAuthenticator and pass it to the channel factory.

### Example

```csharp
public async Task Run()
{
    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampClientAuthenticator authenticator;

    if (false)
    {
        authenticator = new WampCraClientAuthenticator(authenticationId: "joe", secret: "secret2");
    }
    else
    {
        authenticator =
            new WampCraClientAuthenticator(authenticationId: "peter", secret: "secret1", salt: "salt123",
                                           iterations: 100, keyLen: 16);
    }

    IWampChannel channel =
        channelFactory.CreateJsonChannel("ws://127.0.0.1:8080/ws",
                                         "realm1",
                                         authenticator);

    channel.RealmProxy.Monitor.ConnectionEstablished +=
        (sender, args) =>
        {
            Console.WriteLine("connected session with ID " + args.SessionId);

            dynamic details = args.WelcomeDetails.OriginalValue.Deserialize<dynamic>();

            Console.WriteLine("authenticated using method '{0}' and provider '{1}'", details.authmethod,
                              details.authprovider);

            Console.WriteLine("authenticated with authid '{0}' and authrole '{1}'", details.authid,
                              details.authrole);
        };

    channel.RealmProxy.Monitor.ConnectionBroken += (sender, args) =>
    {
        dynamic details = args.Details.OriginalValue.Deserialize<dynamic>();
        Console.WriteLine("disconnected " + args.Reason + " " + details.reason + details);
    };

    IWampRealmProxy realmProxy = channel.RealmProxy;

    await channel.Open().ConfigureAwait(false);

    // call a procedure we are allowed to call (so this should succeed)
    //
    IAdd2Service proxy = realmProxy.Services.GetCalleeProxy<IAdd2Service>();

    try
    {
        var five = await proxy.Add2Async(2, 3)
                              .ConfigureAwait(false);

        Console.WriteLine("call result {0}", five);
    }
    catch (Exception e)
    {
        Console.WriteLine("call error {0}", e);
    }

    // (try to) register a procedure where we are not allowed to (so this should fail)
    //
    Mul2Service service = new Mul2Service();

    try
    {
        await realmProxy.Services.RegisterCallee(service)
                        .ConfigureAwait(false);

        Console.WriteLine("huh, function registered!");
    }
    catch (WampException ex)
    {
        Console.WriteLine("registration failed - this is expected: " + ex.ErrorUri);
    }

    // (try to) publish to some topics
    //
    string[] topics =
    {
        "com.example.topic1",
        "com.example.topic2",
        "com.foobar.topic1",
        "com.foobar.topic2"
    };


    foreach (string topic in topics)
    {
        IWampTopicProxy topicProxy = realmProxy.TopicContainer.GetTopicByUri(topic);

        try
        {
            await topicProxy.Publish(new PublishOptions() {Acknowledge = true})
                            .ConfigureAwait(false);

            Console.WriteLine("event published to topic " + topic);
        }
        catch (WampException ex)
        {
            Console.WriteLine("publication to topic " + topic + " failed: " + ex.ErrorUri);
        }
    }
}


public interface IAdd2Service
{
    [WampProcedure("com.example.add2")]
    Task<int> Add2Async(int x, int y);
}

public class Mul2Service
{
    [WampProcedure("com.example.mul2")]
    public int Multiply2(int x, int y)
    {
        return x*y;
    }
}
```

>Note:  The sample is based on [this](https://github.com/crossbario/crossbarexamples/tree/master/authenticate/wampcra) AutobahnJS sample
