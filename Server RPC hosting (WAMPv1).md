_**Note**: WampSharp is still in development, not stable yet_

As you've seen in [[The getting started tutorial|Getting started with WAMPv1]], WampSharp allows you to host RPC services and consume them from a WAMP client.

###Declaring RPC services

In order to declare a RPC service, just place a [WampRpcMethod] attribute above a method you want to expose. You can place the attribute either on the implemented method itself, or on a method belonging to an interface the RPC service implements.

The WampRpcMethod receives a ProcUri in its constructor - that is used as an identifier for the WAMP method (used by a WAMP CALL request).

The ProcUri can be relative to base uri or absolute.

###Hosting RPC services

Hosting RPC services is done using the DefaultWampHost's Host method. The method receives an instance of the service to host and a baseUri that methods ProcUri will be relative to.

_Note_: the same instance of the service is used for all RPC clients.

###Asynchronous services

WampSharp supports both synchronous and asynchronous RPC services.

In order to declare an asynchronous service, just declare a method that returns a Task<>. When the task is complete, its result will be sent to the client.

###Error calls

WampSharp allows to send CALLERROR to clients using the exception mechanism, just throw an exception (or set a Task's exception in asynchronous version) and a CALLERROR will be sent.

In order to specify more details about the call error, throw a WampRpcCallException which allows you to specify more details, such as errorUri, error description an error details.