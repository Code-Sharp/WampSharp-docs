## WampChannel

A WampChannel is an object that represents a WAMP client session to a remote router.

In order to obtain a WampChannel, instantiate an instance of the WampChannelFactory class, and call CreateChannel with the desired realm to connect to, the desired WampConnection and the desired binding.

For example: connecting to a remote router with Msgpack support and WebSocket4NetConnection:

```csharp
const string serverAddress = "ws://127.0.0.1:8080/ws";

WampChannelFactory factory = new WampChannelFactory();

JTokenMsgpackBinding msgpackBinding = new JTokenMsgpackBinding();

IWampChannel channel =
    factory.CreateChannel("realm1",
                          new WebSocket4NetBinaryConnection<JToken>(serverAddress, msgpackBinding),
                          msgpackBinding);

await channel.Open().ConfigureAwait(false);
```

For example: connecting to a remote router with Json support and SignalRConnection (with long polling transport):

```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

WampChannelFactory factory = new WampChannelFactory();

JTokenJsonBinding jsonBinding = new JTokenJsonBinding();

SignalRTextConnection<JToken> signalRConnection =
    new SignalRTextConnection<JToken>
        (serverAddress, jsonBinding, new LongPollingTransport());

IWampChannel channel =
    factory.CreateChannel("realm1",
                          signalRConnection,
                          jsonBinding);

await channel.Open().ConfigureAwait(false);
```

### DefaultWampChannelFactory

Since the common case is to use a WebSocket4Net connection with MessagePack or Json binding, there exists the DefaultWampChannelFactory class.

It has overloads that allow you obtain a WampChannel in a easier way:

Obtaining a Json channel:
```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

IWampChannel channel =
    factory.CreateJsonChannel(serverAddress, "realm1");

await channel.Open().ConfigureAwait(false);
```

Obtaining a Msgpack channel:
```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

IWampChannel channel =
    factory.CreateMsgpackChannel(serverAddress, "realm1");

await channel.Open().ConfigureAwait(false);
```

### Fluent syntax

Fluent syntax is an alternative api for the above. The api is fluent and allows customization.

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
