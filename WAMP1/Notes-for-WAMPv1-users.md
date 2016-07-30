## Notes for WAMP1 users

If you are using WampSharp with WAMPv1 support and want to update to WAMPv2 version (without updating the protocol to WAMPv2) please read the following notes:

* All classes of WampSharp that are specific for WAMPv1 implementation have moved to namespace WampSharp.V1
* Since Fleck has been updated to a newer version, specifying a host name in the serverAddress of a DefaultWampHost isn't supported anymore. You need to specify an IP address.
* From this version v1.2.2.8, WAMPv1 support has been moved to a dedicated dll name WampSharp.WAMP1.dll. The types DefaultWampHost, DefaultWampCraHost and DefaultWampChannelFactory (and some extension methods of IWampChannelFactory) are located in WampSharp.WAMP1.Default.dll.
In order to update a WAMP1 application to this version, please uninstall WampSharp.Default and install WampSharp.WAMP1.Default instead.
* Please also note that [WAMPv1 is deprecated](https://groups.google.com/forum/#!msg/autobahnws/k-Jo8NnFtjA/qxnmFp2qGkMJ), and you're encouraged to upgrade your application to WAMPv2.
* .NET Standard support isn't available at this time for WAMPv1.