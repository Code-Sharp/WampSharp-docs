A WampHost is an object that hosts a WampSharp router.

In order to use WampHost, instantiate a new instance of WampHost class.
Then register the host with the transport and bindings your are interested in: 

For example: Initialization of a WampHost with FleckWebSocketTransport and Json and Msgpack binding support:
```csharp
const string serverAddress = "ws://127.0.0.1:8080/ws";

WampHost host = new WampHost();

host.RegisterTransport(new FleckWebSocketTransport(serverAddress),
                       new JTokenJsonBinding(),
                       new JTokenMsgpackBinding());

host.Open();
```

For example: Initialization of a WampHost with SignalRTransport and Json binding support:
```csharp
const string serverAddress = "http://127.0.0.1:8080/wampsharp";

WampHost host = new WampHost();

host.RegisterTransport(new SignalRTransport(serverAddress),
                       new JTokenJsonBinding());

host.Open();
```

Combining multiple transports:

```csharp
const string fleckAddress = "ws://127.0.0.1:8080/ws";
const string signalRAddress = "http://127.0.0.1:9090/signalR";

WampHost host = new WampHost();

JTokenBinding jsonBinding = new JTokenJsonBinding();
JTokenMsgpackBinding msgpackBinding = new JTokenMsgpackBinding();

host.RegisterTransport(new FleckWebSocketTransport(fleckAddress),
                       jsonBinding,
                       msgpackBinding);

host.RegisterTransport(new SignalRTransport(signalRAddress),
                       jsonBinding);

host.Open();
```

After that, you consume the roles you are interested in as described in the getting started tutorials:
* [[Getting Started with Callee]]
* [[Getting Started with Caller]]
* [[Getting Started with Subscriber]]
* [[Getting Started with Publisher]]

### DefaultWampHost

Since the most common usage of WampSharp is with Fleck and Json/Msgpack support, there exists a class named DefaultWampHost which instantiates a WampHost with FleckWebSocketTransport and JTokenJsonBinding, JTokenMsgpackBinding.

In order to use it, provide only the server address in the class's constructor.

Example:

```csharp
const string serverAddress = "http://127.0.0.1:8080/ws";

WampHost host = new DefaultWampHost(serverAddress);

host.Open();
```