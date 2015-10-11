## Router-side authentication

 This page describes router-side authentication.

In order to use router-side authentication, you'll need to create an instance of a (Default)WampAuthenticationHost and initialize it with a IWampSessionAuthenticatorFactory - that is an interface which will create a IWampSessionAuthenticator - a mechanism which is responsible of the authentication process of an individual client.

### IWampSessionAuthenticator

IWampSessionAuthenticator is an interface which represents a mechanism which is responsible of the authentication process of an individual client. Whenever a client requests to join the router (i.e. sends a HELLO message), an IWampSessionAuthenticator is created by the IWampSessionAuthenticatorFactory passed to the WampHost.

This instance handles the client's authentication process: after the IWampSessionAuthenticator is created, the router checks if the client is authenticated (by checking the IsAuthenticated property). If the client isn't authenticated, a CHALLENGE message is sent to the client, with authmethod and extra parameters which are specified by the IWampSessionAuthenticator instance's AuthenticationMethod and ChallengeDetails properties. When a client sends an AUTHENTICATE message, the IWampSessionAuthenticator's Authenticate method is called with the received signature and extra parameters. At this point, the IWampSessionAuthenticator's IsAuthenticated property should be set to true if the client is authenticated.

In any point of this process, a WampAuthenticationException can be thrown. If such an exception is thrown, an ABORT message will be sent to the client with exception specified parameters.

Once the client is authenticated, a WELCOME message will be sent to the client with details and authid, authmethod parameters as specified in WelcomeDetails, AuthenticationId and AuthenticationMethod properties. After that, the Authorizer property of the IWampSessionAuthenticator's instance will be used determine if the client is allowed to perform certain operations. See IWampAuthorizer section for more info.

### WampSessionAuthenticator

In order to make it easier to implement the IWampSessionAuthenticator interface, the abstract class WampSessionAuthenticator is provided. In this class, the only members which are needed to be implemented are Authenticate, AuthenticationId and AuthenticationMethod. Other properties are implemented as auto-properties, which you can set their values. All members are virtual, so you can also override them.

In addition, a WampSessionAuthenticator&lt;TExtra&gt; class is provided, which is the same as WampSessionAuthenticator, but deserializes the given extra dictionary passed in AUTHENTICATE message to the given TExtra type.

### IWampAuthorizer

IWampAuthorizer is an interface which checks whether an individual client is allowed to perform certain WAMP operations.

```csharp
public interface IWampAuthorizer
{
    bool CanRegister(RegisterOptions options, string procedure);
    bool CanCall(CallOptions options, string procedure);
    bool CanPublish(PublishOptions options, string topicUri);
    bool CanSubscribe(SubscribeOptions options, string topicUri);
}

```

Each method can return a false value in order to prevent access of the client to do certain actions. In addition, a WampException can be thrown by any method, which will result in sending an ERROR message to the client with the exception details (except in Publish case, where this will happen only if the client requested acknowledge).

### Ticket-based authentication example

This is an example implementation of a very simplified version of [Ticket-based authentication](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_authentication_ticket.md). It shows off a bit C# 6.0 features too.

```csharp
public class MyAuthenticatorFactory : IWampSessionAuthenticatorFactory
{
    private readonly IDictionary<string, string> mUserToTicket =
        new Dictionary<string, string>
        {
            ["peter"] = "magic_secret_1",
            ["joe"] = "magic_secret_2"
        };

    private readonly IDictionary<string, IWampAuthorizer> mUserToAuthorizer =
        new Dictionary<string, IWampAuthorizer>
        {
            ["peter"] = new FrontendStaticAuthorizer(),
            ["joe"] = new FrontendStaticAuthorizer()
        };

    public IWampSessionAuthenticator GetSessionAuthenticator
        (WampPendingClientDetails details,
         IWampSessionAuthenticator transportAuthenticator)
    {
        HelloDetails helloDetails = details.HelloDetails;

        if (helloDetails.AuthenticationMethods?.Contains("ticket") != true)
        {
            throw new WampAuthenticationException("supports only 'ticket' authentication");
        }

        string user = helloDetails.AuthenticationId;

        string ticket;

        if (user == null ||
            !mUserToTicket.TryGetValue(user, out ticket))
        {
            throw new WampAuthenticationException
                ($"no user with authid '{user}' in user database");
        }

        return new TicketSessionAuthenticator(user, ticket, mUserToAuthorizer[user]);
    }
}

public class TicketSessionAuthenticator : WampSessionAuthenticator
{
    private readonly string mTicket;
    private readonly IWampAuthorizer mAuthorizer;

    public TicketSessionAuthenticator(string authenticationId, string ticket, IWampAuthorizer authorizer)
    {
        AuthenticationId = authenticationId;
        mTicket = ticket;
        mAuthorizer = authorizer;
    }

    public override void Authenticate(string signature, AuthenticateExtraData extra)
    {
        if (signature == mTicket)
        {
            IsAuthenticated = true;
            Authorizer = mAuthorizer;

            WelcomeDetails = new WelcomeDetails()
            {
                AuthenticationProvider = "hardcoded",
                AuthenticationRole = "frontend"
            };
        }
    }

    public override string AuthenticationId { get; }

    public override string AuthenticationMethod => "ticket";
}

public class BackendStaticAuthorizer : IWampAuthorizer
{
    public bool CanRegister(RegisterOptions options, string procedure) => true;

    public bool CanCall(CallOptions options, string procedure) => true;

    public bool CanPublish(PublishOptions options, string topicUri) => true;

    public bool CanSubscribe(SubscribeOptions options, string topicUri)
    {
        return topicUri != "com.example.topic2";
    }
}

public class FrontendStaticAuthorizer : IWampAuthorizer
{
    public bool CanRegister(RegisterOptions options, string procedure) => false;

    public bool CanCall(CallOptions options, string procedure) => procedure == "com.example.add2";

    public bool CanPublish(PublishOptions options, string topicUri)
    {
        return (topicUri.StartsWith("com.example.") && topicUri != "com.example.topic2") ||
               topicUri == "com.foobar.topic1";
    }

    public bool CanSubscribe(SubscribeOptions options, string topicUri) => false;
}
```

After that we create a WampAuthenticationHost and pass to it an instance of our MyAuthenticatorFactory:

```csharp
public class Program
{
    private static void Main(string[] args)
    {
        DefaultWampAuthenticationHost host =
            new DefaultWampAuthenticationHost("ws://127.0.0.1:8080/ws",
                                              new MyAuthenticatorFactory());


        IWampHostedRealm realm =
            host.RealmContainer.GetRealmByName("realm1");

        string[] topics = new[]
        {
            "com.example.topic1",
            "com.foobar.topic1",
            "com.foobar.topic2"
        };

        foreach (string topic in topics)
        {
            string currentTopic = topic;

            realm.Services.GetSubject<string>(topic).Subscribe
                (x => Console.WriteLine("event received on {0}: {1}", currentTopic, x));
        }

        realm.Services.RegisterCallee(new Add2Service()).Wait();

        host.Open();

        Console.ReadLine();
    }

    public class Add2Service : IAdd2Service
    {
        public int Add2(int x, int y)
        {
            return (x + y);
        }
    }

    public interface IAdd2Service
    {
        [WampProcedure("com.example.add2")]
        int Add2(int a, int b);
    }
}
```

### IWampSessionAuthenticatorFactory

You might have been asking yourself what is the transportAuthenticator parameter in IWampSessionAuthenticatorFactory's GetSessionAuthenticator method. Well glad you've asked! This is an authenticator that can be provided by the transport of the session. Some transports have built-in metadata that can be used in order to authenticate a client. The common example is [cookies](Cookie-based-router-side-authentication.md).
