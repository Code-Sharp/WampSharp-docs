_**Note**: WampSharp WAMP v2 support is still in development, not stable yet_

_**Note**: You need [NuGet](http://www.nuget.org) for this tutorial_

Create a new Console Application in Visual Studio.

Install WampSharp: 

Go to Tools -> NuGet Package Manager -> Package Manager Console. 

Enter in the Package Manager Console
<code>
Install-Package WampSharp.Default -Pre
</code>

Now WampSharp is installed on your project.

### About realms

WAMPv2 protocol consists of the idea of realms. A WAMP realm, can be thought as a domain, where uris are mapped to procedures/topics.

WampSharp WAMPv2 api is based on the realm idea. In the following sections we describe how to access realms from router/client code.

Other tutorials describe how to consume WAMP roles from realm api.

#### Getting started with a WAMPv2 Router

In order to Add WAMPv2 router capabilities to your application, create a WampHost:

```csharp
const string location = "ws://127.0.0.1:8080/";
using (IWampHost host = new DefaultWampHost(location))
{
    IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

    // Host WAMP application components

    host.Open();

    Console.WriteLine("Server is running on " + location);
    Console.ReadLine();
}
```

A WampHost hosts your WAMP application components. It consists of IWampRealms.

IWampRealm is an interface that represents a WAMPv2 realm.

The realms are accessible from the WampHost's RealmContainer property. 

#### Getting started with a WAMPv2 Client

In order to connect to a router's realm, create a WampChannel that connects to the realm.

```csharp
const string location = "ws://127.0.0.1:8080/";

DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

IWampChannel channel = channelFactory.CreateJsonChannel(location, "realm1");

IWampRealmProxy realmProxy = channel.RealmProxy;

await channel.Open();

// Host WAMP application components

```

A WampChannel represents a session to a WAMPv2 router. It contains a property named RealmProxy which is a IWampRealmProxy, that is a proxy to the router's remote realm.

### More tutorials

See the following tutorials for getting started with a WAMPv2 role:

* [Getting Started with Callee](Roles\Callee\Getting-Started-with-Callee.md)
* [Getting Started with Caller](Roles\Caller\Getting-Started-with-Caller.md)
* [Getting Started with Subscriber](Roles\Subscriber\Getting-Started-with-Subscriber.md)
* [Getting Started with Publisher](Roles\Publisher\Getting-Started-with-Publisher.md)