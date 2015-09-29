A WampChannel is an object that represents a WAMP client session to a remote router.

In order to obtain a WampChannel, instantiate an instance of the WampChannelFactory class, and call CreateChannel with the desired realm to connect to, the desired WampConnection and the desired binding.

For example: connecting to a remote router with Msgpack support and WebSocket4NetConnection:

```csharp
string serverAddress = "ws://127.0.0.1:8080/ws";

JTokenMsgpackBinding msgpackBinding = new JTokenMsgpackBinding();

IWampChannel channel =
    factory.CreateChannel("realm1",
                          new WebSocket4NetBinaryConnection<JToken>(serverAddress, msgpackBinding),
                          msgpackBinding);

Task openTask = channel.Open();
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

Task openTask = channel.Open();            
```

##DefaultWampChannelFactory

Since the common case is to use a WebSocket4Net connection with MessagePack or Json binding, there exists the DefaultWampChannelFactory class.

It has overloads that allow you obtain a WampChannel in a easier way:

Obtaining a Json channel:
```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

IWampChannel channel = 
    factory.CreateJsonChannel(serverAddress, "realm1");

Task openTask = channel.Open();            
```

Obtaining a Msgpack channel:
```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

IWampChannel channel = 
    factory.CreateMsgpackChannel(serverAddress, "realm1");

Task openTask = channel.Open();            
```