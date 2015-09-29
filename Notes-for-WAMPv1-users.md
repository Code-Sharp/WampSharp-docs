If you are using WampSharp with WAMPv1 support and want to update to WAMPv2 version (without updating the protocol to WAMPv2) please read the following notes:

* All classes of WampSharp that are specific for WAMPv1 implementation have moved to namespace WampSharp.V1
* Since Fleck has been updated to a newer version, specifying a host name in the serverAddress of a DefaultWampHost isn't supported anymore. You need to specify an IP address.