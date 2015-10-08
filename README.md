## WampSharp Documentation

This repository contains documentation for the [WampSharp project](http://github.com/Code-Sharp/WampSharp).

### Introduction

[WAMP (The Web Application Messaging Protocol)](http://wamp.ws) is an open standard WebSocket subprotocol that provides two application messaging patterns in one unified protocol: Remote Procedure Calls + Publish & Subscribe.   

[WampSharp](http://github.com/Code-Sharp/WampSharp) is a .NET open source implementation of WAMP which allows you to write RPC services and Pub/Sub based applications in a comfortable way.

### Roadmap

* [Roadmap](Roadmap.md)

### WAMPv2 Documentation

#### Tutorials
* [Getting started with WAMPv2](WAMP2/Getting-Started-with-WAMPv2.md)
* [Getting started with Callee](WAMP2/Roles/Callee/Getting-Started-with-Callee.md)
* [Getting started with Caller](WAMP2/Roles/Caller/Getting-Started-with-Caller.md)
* [Getting started with Publisher](WAMP2/Roles/Publisher/Getting-Started-with-Publisher.md)
* [Getting started with Subscriber](WAMP2/Roles/Publisher/Getting-Started-with-Subscriber.md)

#### Roles

##### Callee
* [Reflection based Callee](WAMP2/Roles/Callee/Reflection-based-Callee.md)
* [Raw Callee](WAMP2/Roles/Callee/Raw-Callee.md)

##### Caller
* [Reflection based Caller](WAMP2/Roles/Caller/Reflection-based-Caller.md)
* [Raw Caller](WAMP2/Roles/Caller/Raw-Caller.md)

##### Publisher
* [Rx based Publisher](WAMP2/Roles/Publisher/Rx-based-Publisher.md)
* [Reflection based Publisher](WAMP2/Roles/Publisher/Reflection-based-Publisher.md)
* [Raw Publisher](WAMP2/Roles/Publisher/Raw-Publisher.md)

##### Subscriber
* [Rx based Subscriber](WAMP2/Roles/Subscriber/Rx-based-Subscriber.md)
* [Reflection based Subscriber](WAMP2/Roles/Subscriber/Reflection-based-Subscriber.md)
* [Raw Subscriber](WAMP2/Roles/Subscriber/Raw-Subscriber.md)

#### Advanced profile features

The following [Advanced profile features](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced.md) are supported

* [Progressive call results](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/progressive-call-results.md): [caller tutorial](WAMP2/Roles/Caller/Reflection-based-Caller.md#progressive-call-results), [callee tutorial](WAMP2/Roles/Callee/Reflection-based-Callee.md#progressive-call-results)
* [Caller identification](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/caller-identification.md): [caller tutorial](WAMP2/Roles/Caller/Reflection-based-Caller.md#caller-identification), [callee tutorial](WAMP2/Roles/Callee/Reflection-based-Callee.md)
* [Session meta api](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/session-meta-api.md), [Registration meta api](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/registration-meta-api.md), [Subscription meta api](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/subscription-meta-api.md) - see [release notes](Release-notes/WampSharp-v1.2.3.12-beta-release-notes.md#meta-api-descriptor-service)
* [Shared registrations](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/shared-registration.md), see also [here](http://crossbar.io/docs/Shared-Registrations/) - see [callee tutorial](WAMP2/Roles/Callee/Reflection-based-Callee.md#shared-registrations)
* [Subscriber black and whitelisting](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/subscriber-blackwhite-listing.md)
* [Publisher exclusion](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/publisher-exclusion.md)
* [Publisher identification](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/publisher-identification.md)
* [Pattern-based subscriptions](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/pattern-based-subscription.md) - see also [here](http://crossbar.io/docs/Pattern-Based-Subscriptions/) - see [subscriber tutorial](WAMP2/Roles/Subscriber/Reflection-based-Subscriber.md#pattern-based-subscriptions)
* [Pattern-based registrations](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/pattern-based-registration.md) - see also [here](http://crossbar.io/docs/Pattern-Based-Registrations/) - see [callee tutorial](WAMP2/Roles/Callee/Reflection-based-Callee.md#pattern-based-registrations)
* [RawSocket transport](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/rawsocket-transport.md) - see [release notes](Release-notes/WampSharp-v1.2.3.12-beta-release-notes.md#rawsocket-rewrite)
* [Authentication](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/authentication.md) - see [Router side authentication](WAMP2/Router/Router-side-authentication.md), [Client side authentication](WAMP2/Client/Client-side-authentication.md).
* [WAMP-CRA](https://github.com/wamp-proto/wamp-proto/blob/master/spec/advanced/challenge-response-authentication.md) - see [WAMP-CRA router side authentication](WAMP2/Router/WAMP-CRA-router-side-authentication.md), [WAMP-CRA client side authentication](WAMP2/Client/WAMP-CRA-client-side-authentication.md)

#### Router

* [WampHost](WAMP2/Router/WampHost.md)
* Authentication
  * [Router side authentication](WAMP2/Router/Router-side-authentication.md)
  * [WAMP-CRA router side authentication](WAMP2/Router/WAMP-CRA-router-side-authentication.md)
* Transports
  * WebSockets
    * Fleck
    * [Vtortola](Release-notes/WampSharp-v1.2.1.6-beta-release-notes.md#vtortolawebsocketlistener-support)
  * [RawSocket](Release-notes/WampSharp-v1.2.3.12-beta-release-notes.md#rawsocket-rewrite)
  * [SignalR](https://github.com/Code-Sharp/AutobahnJS.SignalR)

#### Client

* [WampChannel](WAMP2/Client/WampChannel.md)
* [WampChannelReconnector](WAMP2/Client/WampChannelReconnector.md)
* Authentication
  * [Client side authentication](WAMP2/Client/Client-side-authentication.md)
  * [WAMP-CRA client side authentication](WAMP2/Client/WAMP-CRA-client-side-authentication.md)

### WAMPv1

* [Get Started!](WAMP1/Getting-started-with-WAMPv1.md)
* [Getting Started with WAMP client](WAMP1/Getting-started-with-WAMPv1-client.md)
* [Server RPC Hosting](WAMP1/Server RPC hosting (WAMPv1).md)
* [Server PubSub Hosting](WAMP1/Server PubSub Hosting (WAMPv1).md)
* [Notes for WAMPv1 users](WAMP1/Notes-for-WAMPv1-users.md)

### Release Notes

* [WampSharp v1.2.1.6-beta release notes](Release-notes/WampSharp-v1.2.1.6-beta-release-notes.md)
* [WampSharp v1.2.2.8-beta release notes](Release-notes/WampSharp-v1.2.2.8-beta-release-notes.md)
* [WampSharp v1.2.3.12-beta release notes](Release-notes/WampSharp-v1.2.3.12-beta-release-notes.md)
