## WampSharp v1.2.3.12-beta release notes

**Contents**

1. [New features](#new-features)
    * [Router side authentication](#router-side-authentication)
    * [WampChannelFactory fluent syntax](#wampchannelfactory-fluent-syntax)
    * [RawSocket rewrite](#rawsocket-rewrite)
    * [Meta-api descriptor service](#meta-api-descriptor-service)
2. [Bug fixes](#bug-fixes)
	* [Uri verification (Issue #84)](#uri-verification-issue-84)
	* [StackOverflowException (Issue #92)](#stackoverflowexception-issue-92)
	* [Stress issues](#stress-issues-91-93-and-others)

### New features

####Router side authentication

From this version, [router-side authentication](../WAMP2/Router/Router-side-authentication.md) is supported. Also [WAMP-CRA is supported](../WAMP2/Router/WAMP-CRA-router-side-authentication.md).

Also, [Cookie based authenticators are supported](../WAMP2/Router/Cookie-based-router-side-authentication.md).

Currently authentication details (authid, authmethod and authrole) are forwarded to callees if the caller is disclosed. These are accessible via WampInvocationContext. This might change according to the [WAMP spec decision](https://github.com/wamp-proto/wamp-proto/issues/57).

####WampChannelFactory fluent syntax

This version introduces new api to obtain a IWampChannel. The api is fluent and allows customization.

In order to use it, you need to add an using directive. Note that the available api depends on which WampSharp packages you installed. (For instance, if you haven't installed WampSharp.NewtonsoftMsgpack, then msgpack extension methods won't be present)

```csharp
using WampSharp.V2.Fluent;
```


Examples:

```csharp
IWampChannelFactory factory = new WampChannelFactory();

IWampChannel channel =
    factory.ConnectToRealm("realm1")
           .WebSocketTransport("ws://127.0.0.1:8080/ws")
           .MsgpackSerialization()
           .Build();

await channel.Open();
```

This feels more modular in some sense:

```csharp
IWampChannelFactory factory = new WampChannelFactory();

IWampChannel channel =
    factory.ConnectToRealm("realm1")
           .WebSocketTransport("ws://127.0.0.1:8080/ws")
           .JsonSerialization(new JsonSerializer
           {
               ContractResolver = new CamelCasePropertyNamesContractResolver()
           })
           .CraAuthentication(authenticationId: "peter", secret: "secret1")
           .Build();

await channel.Open();
```

The actual reason this api is introduced is for RawSocket channels, see RawSocket section.

####RawSocket rewrite

This version includes a total rewrite of [RawSocket transport](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_transport_rawsocket.md). The current rewrite implements the [revised RawSocket transport spec](https://github.com/tavendo/AutobahnPython/issues/291) and is implemented using the framework's TcpListener class, instead of [SuperSocket](https://github.com/kerryjiang/SuperSocket).

In order to use the router-side RawSocket transport, install the WampSharp.RawSocket package. Then create a WampHost (or an WampAuthenticationHost) and register the RawSocketTransport. Example:
```csharp
IWampHost host = new WampHost();

RawSocketTransport transport =
    new RawSocketTransport(TcpListener.Create(8080));

host.RegisterTransport(transport,
                       new JTokenJsonBinding());

```


#####RawSocket client transport

The RawSocket rewrite made it easier to use the same codebase in order to implement a RawSocket client transport. In order to obtain a RawSocket IWampChannel, you can use the fluent syntax api:

```csharp
IWampChannelFactory factory = new WampChannelFactory();

IWampChannel channel =
    factory.ConnectToRealm("realm1")
           .RawSocketTransport("127.0.0.1", 8080)
           .MsgpackSerialization()
           .Build();

```
This fluent-api allows some RawSocket customization features:
```csharp
IWampChannelFactory factory = new WampChannelFactory();

IWampChannel channel =
    factory.ConnectToRealm("realm1")
           .RawSocketTransport("127.0.0.1", 8080)
           .ConnectFrom(new IPEndPoint(IPAddress.Loopback, 1345))
           // Chooses the port to connect from
           .AutoPing(TimeSpan.FromSeconds(45))
           // Enables auto-ping
           .MsgpackSerialization()
           .Build();
```

> Note: currently [crossbar](crossbar.io) and Autobahn variants [don't implement RawSocket's ping/pongs](https://github.com/crossbario/crossbar/issues/381). Therefore, auto-ping is disabled by default. You can enable it manually, both for router-side transport (by passing to the RawSocketTransport constructor an non-null auto-ping interval) and for client-side transport (as in the sample).

####Meta-api descriptor service

From this version WAMP meta-api is implemented (i.e. [session meta api](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_pubsub_session_meta_api.md), [registration meta api](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_rpc_registration_meta_api.md) and [subscription meta api](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/text/advanced/ap_pubsub_subscription_meta_api.md)). It is possible both to consume WAMP meta-api from a WampSharp client, and to expose it from a WampSharp router.

##### Exposing meta-api

In order to expose meta-api, you can call an extension method of IWampHostedRealm, named "HostMetaApiService". This method returns an IDisposable which you can dispose in order to unregister the meta-api service.
> Note: it is important to call HostMetaApiService before hosting any other components (callees/subscribers), since otherwise the meta-api service isn't be able to track components registered before it.

```csharp
DefaultWampHost host = new DefaultWampHost("ws://127.0.0.1:8080/ws");

IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

IDisposable disposable = realm.HostMetaApiService();

host.Open();
```

##### Consuming meta-api

In order to consume meta-api, you can use declare the WAMP meta-api contracts yourself and consume it with plain WampSharp client-side code. In order to save that amount of work, a client-side api is provided which allows you to consume the meta-api without having to write any contracts yourself. This is api is available via an extension method of IWampRealmProxy which is named "GetMetaApiServiceProxy".

```csharp
private static async Task Run()
{
    WampChannelFactory channelFactory = new WampChannelFactory();

    IWampChannel channel =
        channelFactory.ConnectToRealm("realm1")
                      .WebSocketTransport("ws://127.0.0.1:8080/ws")
                      .JsonSerialization()
                      .Build();

    await channel.Open().ConfigureAwait(false);

    WampMetaApiServiceProxy proxy = channel.RealmProxy.GetMetaApiServiceProxy();

    long sessionCount = await proxy.CountSessionsAsync();

    IAsyncDisposable onCreateDisposable =
        await proxy.SubscribeTo.Subscription.OnCreate(
            (id, details) =>
            {
                Console.WriteLine($"Subscription with id {id} created, topic uri {details.Uri}");
            })
            .ConfigureAwait(false);

    IAsyncDisposable onJoinDisposable =
        await proxy.SubscribeTo.Session.OnJoin(
            details =>
            {
                Console.WriteLine($"Session with id {details.Session} joined");
            })
            .ConfigureAwait(false);
}
```

### Bug fixes

#### Uri verification ([Issue #84](https://github.com/Code-Sharp/WampSharp/issues/84))

This is related to [the corresponding part of the spec](https://github.com/wamp-proto/wamp-proto/blob/master/rfc/draft-oberstet-hybi-tavendo-wamp.md#uris-uris) which discusses valid uris. From this version, WampSharp router verifies that uris are valid as in the spec. The default behavior is loose/relaxed uri verification. You can change that behavior by passing a IWampUriValidator to the WampHost constructor. For example:

```csharp
WampHost host =
    new DefaultWampHost("ws://127.0.0.1:8080/ws",
                        uriValidator: new StrictUriValidator());

host.Open();
```

You can also implement yourself IWampUriValidator in order to define yourself what uris are valid. (Your implementation should be a subset of the loose/relaxed uri definition, but this isn't forced)

#### StackOverflowException ([Issue #92](https://github.com/Code-Sharp/WampSharp/issues/92))

It turns out that using the WampSharp router resulted sometimes in a StackOverflowException. The reason for this is related to Reactive Extensions implementation of the Merge operator which uses recursion.. WampSharp used the Merge operator as a queue mechanism for sending messages serially per client. When a large number of messages was gathered for a client and then a client suddenly disconnected, this resulted in a large recursion stack which resulted in a StackOverflowException. It turns out that this is a [known rx issue](https://github.com/Reactive-Extensions/Rx.NET/issues/19).

In order to solve this issue, WampSharp's queue mechanism was replaced with a [different implementation (Ix-Async based)](https://github.com/Code-Sharp/WampSharp/commit/e476bb9a63c4198bbf763ab9ecd0e55593901a7b).

#### Stress issues ([#91](https://github.com/Code-Sharp/WampSharp/issues/91) [#93](https://github.com/Code-Sharp/WampSharp/issues/93) and [others](https://github.com/nj4x/WampSharpTests))

Thanks to [@nj4x](https://github.com/nj4x/) another couple of multi-threaded issues related to high stress issues were detected and fixed.

> Written with [StackEdit](https://stackedit.io/).
