####Client-side authentication

Client-side authentication is supported. In order to use client authentication, you need to implement an interface named IWampClientAuthenticator. Then, pass it to CreateChannel/CreateJsonChannel/CreateMsgpackChannel overloads of DefaultChannelFactory.
In IWampClientAuthenticator we supply the supported authentication methods and the authenticationid, these are passed in the HELLO message to the router (as details.authmethods, details.authid). We also implement Authenticate method, which sends an AUTHENTICATE message to the router upon CHALLENGE.

Example:
```csharp
public class TicketAuthenticator : IWampClientAuthenticator
{
    private static readonly string[] mAuthenticationMethods = { "ticket" };

    private readonly IDictionary<string, string> mTickets =
        new Dictionary<string, string>()
        {
            {"peter", "magic_secret_1"},
            {"joe", "magic_secret_2"}
        };

    private const string User = "peter";

    public AuthenticationResponse Authenticate(string authmethod, ChallengeDetails extra)
    {
        if (authmethod == "ticket")
        {
            Console.WriteLine("authenticating via '" + authmethod + "'");

            AuthenticationResponse result =
                new AuthenticationResponse {Signature = mTickets[User]};

            return result;
        }
        else
        {
            throw new WampAuthenticationException("don't know how to authenticate using '" + authmethod + "'");
        }
    }

    public string[] AuthenticationMethods
    {
        get
        {
            return mAuthenticationMethods;
        }
    }

    public string AuthenticationId
    {
        get
        {
            return User;
        }
    }
}
```

Then we pass an instance of our authenticator to the ChannelFactory:

```csharp
public async Task Run()
{
    DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

    IWampClientAuthenticator authenticator = new TicketAuthenticator();

    IWampChannel channel =
        channelFactory.CreateJsonChannel("ws://127.0.0.1:8080/ws",
            "realm1",
            authenticator);

    IWampRealmProxy realmProxy = channel.RealmProxy;

    await channel.Open();

    // Call a rpc for example
    ITimeService proxy = realmProxy.Services.GetCalleeProxy<ITimeService>();

    try
    {
        string now = await proxy.Now();
        Console.WriteLine("call result {0}", now);
    }
    catch (Exception e)
    {
        Console.WriteLine("call error {0}", e);
    }
}
```

>Note:  The sample is based on [this](https://github.com/tavendo/AutobahnPython/tree/v0.10.3/examples/twisted/wamp/authentication/ticket) AutobahnJS sample
