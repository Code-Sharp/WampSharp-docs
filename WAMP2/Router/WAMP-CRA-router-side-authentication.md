## WAMP-CRA router-side authentication

[WAMP-CRA](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_authentication_cra.md) router side authentication is supported. It is available as an abstract class named WampCraSessionAuthenticator which inherits from the [WampSessionAuthenticator class](Router-side-authentication.md). The class has two abstract properties needed to be implemented: AuthenticationChallenge and Secret - the AuthenticationChallenge is the challenge to be sent upon CHALLENGE message, the Secret is the secret used to compute the signature

It is also possible to add additional data (sent upon CHALLENGE message in extra parameter), such as salt, iterations and keylen by setting the CraChallengeDetails property.

### Sample
>Note: this sample is based on [this crossbar.io sample](https://github.com/crossbario/crossbarexamples/tree/master/authenticate/wampcra).

```csharp
public class MyAuthenticatorFactory : IWampSessionAuthenticatorFactory
{
    private readonly IDictionary<string, Func<IWampSessionAuthenticator>> mUserToAuthenticatorFactory =
        new Dictionary<string, Func<IWampSessionAuthenticator>>
        {
            ["peter"] = () => new PeterWampCraAuthenticator(),
            ["joe"] = () => new JoeWampCraAuthenticator()
        };

    public IWampSessionAuthenticator GetSessionAuthenticator
        (WampPendingClientDetails details,
         IWampSessionAuthenticator transportAuthenticator)
    {
        HelloDetails helloDetails = details.HelloDetails;

        if (helloDetails.AuthenticationMethods?.Contains("wampcra") != true)
        {
            throw new WampAuthenticationException("supports only 'wampcra' authentication");
        }

        string user = helloDetails.AuthenticationId;

        Func<IWampSessionAuthenticator> authenticatorFactory;

        if (user == null ||
            !mUserToAuthenticatorFactory.TryGetValue(user, out authenticatorFactory))
        {
            throw new WampAuthenticationException
                ($"no user with authid '{user}' in user database");
        }

        return authenticatorFactory();
    }
}

public class PeterWampCraAuthenticator : WampCraSessionAuthenticator
{
    public PeterWampCraAuthenticator() : base("peter")
    {
        this.CraChallengeDetails =
            new WampCraChallengeDetails(salt: "salt123",
                                        iterations: 100,
                                        keyLen: 16);

        this.Authorizer = new FrontendStaticAuthorizer();

        this.WelcomeDetails = new WelcomeDetails()
        {
            AuthenticationRole = "frontend",
            AuthenticationProvider = "hardcoded"
        };

        AuthenticationChallenge = Guid.NewGuid().ToString();
    }

    public override string AuthenticationChallenge { get; }

    public override string Secret => "prq7+YkJ1/KlW1X0YczMHw==";
}

public class JoeWampCraAuthenticator : WampCraSessionAuthenticator
{
    public JoeWampCraAuthenticator() : base("joe")
    {
        this.Authorizer = new FrontendStaticAuthorizer();

        this.WelcomeDetails = new WelcomeDetails()
        {
            AuthenticationRole = "frontend",
            AuthenticationProvider = "hardcoded"
        };

        AuthenticationChallenge = Guid.NewGuid().ToString();
    }

    public override string AuthenticationChallenge { get; }

    public override string Secret => "secret2";
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

internal class Program
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

### WampCraUserDbAuthenticationFactory

WampCraUserDbAuthenticationFactory is a predefined implementation of IWampSessionAuthenticatorFactory which is role based and reminds [Crossbar](http://crossbar.io)'s authentication.

In order to instantiate WampCraUserDbAuthenticationFactory, two mechanisms need to be provided:

 * IWampAuthenticationProvider - an interface that provides roles - these are objects that have a role name and a IWampAuthorizer.
 * IWampCraUserDb - an interface that retrieves a WampCraUser by its authenticationId - this is an object that contains WampCra user details (role, secret and other WAMP CRA parameters).

#### Example
>Note: this sample is based on [this crossbar.io sample](https://github.com/crossbario/crossbarexamples/tree/master/authenticate/wampcra).

```csharp
public class MyHardcodedCraAuthenticationProvider : IWampAuthenticationProvider
{
    public WampAuthenticationRole GetRoleByName(string realm, string role)
    {
        if (role == "frontend")
        {
            return new WampAuthenticationRole()
            {
                Authorizer = new FrontendStaticAuthorizer()
            };
        }

        return null;
    }

    public string ProviderName => "hardcoded";
}

public class MyUserDb : IWampCraUserDb
{
    private readonly IDictionary<string, WampCraUser> mUserIdToDetails =
        new Dictionary<string, WampCraUser>()
        {
            ["peter"] = new WampCraUser()
            {
                AuthenticationRole = "frontend",
                Secret = "prq7+YkJ1/KlW1X0YczMHw==",
                Salt = "salt123",
                Iterations = 100,
                KeyLength = 16
            },
            ["joe"] = new WampCraUser()
            {
                Secret = "secret2"
            }
        };

    public WampCraUser GetUserById(string authenticationId)
    {
        WampCraUser user;

        if (mUserIdToDetails.TryGetValue(authenticationId, out user))
        {
            return user;
        }

        return null;
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

internal class Program
{
    private static void Main(string[] args)
    {
        DefaultWampAuthenticationHost host =
            new DefaultWampAuthenticationHost("ws://127.0.0.1:8080/ws",
                                              new WampCraUserDbAuthenticationFactory
                                                  (new MyHardcodedCraAuthenticationProvider(),
                                                   new MyUserDb()));


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

### WampCraStaticUserDb and WampStaticAuthenticationProvider

There exists predefined implementations for  IWampCraUserDb, IWampAuthenticationProvider and IWampAuthorizer named WampCraStaticUserDb, WampStaticAuthenticationProvider and WampStaticAuthorizer that use predefined static data.

#### Example


>Note: this sample is based on [this crossbar.io sample](https://github.com/crossbario/crossbarexamples/tree/master/authenticate/wampcra).

```csharp
public class MyAuthenticationProvider : WampStaticAuthenticationProvider
{
    public MyAuthenticationProvider() :
        base(new Dictionary<string, IDictionary<string, WampAuthenticationRole>>()
    {
        ["realm1"] = new Dictionary<string, WampAuthenticationRole>()
        {
            ["frontend"] = new WampAuthenticationRole
            {
                Authorizer = new WampStaticAuthorizer(new List<WampUriPermissions>
                {
                    new WampUriPermissions
                    {
                        Uri = "com.example.add2",
                        CanCall = true
                    },
                    new WampUriPermissions
                    {
                        Uri = "com.example.",
                        Prefixed = true,
                        CanPublish = true
                    },
                    new WampUriPermissions
                    {
                        Uri = "com.example.topic2",
                        CanPublish = false
                    },
                    new WampUriPermissions
                    {
                        Uri = "com.foobar.topic1",
                        CanPublish = true
                    },
                })
            }
        }
    })
    {
    }
}

public class MyUserDb : WampCraStaticUserDb
{
    public MyUserDb() : base(new Dictionary<string, WampCraUser>()
    {
        ["joe"] = new WampCraUser()
        {
            Secret = "secret2",
            AuthenticationRole = "frontend"
        },
        ["peter"] = new WampCraUser()
        {
            Secret = "prq7+YkJ1/KlW1X0YczMHw==",
            AuthenticationRole = "frontend",
            Salt = "salt123",
            Iterations = 100,
            KeyLength = 16
        }
    })
    {
    }
}

public class Program
{
    private static void Main(string[] args)
    {
        HostCode();
        Console.ReadLine();
    }

    private static void HostCode()
    {
        DefaultWampAuthenticationHost host =
            new DefaultWampAuthenticationHost
                ("ws://127.0.0.1:8080/ws",
                 new WampCraUserDbAuthenticationFactory(new MyAuthenticationProvider(),
                                                        new MyUserDb()));

        IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

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
