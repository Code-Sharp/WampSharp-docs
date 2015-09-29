_**Note**: WampSharp is still in development, not stable yet_

WampSharp contains a Pub/Sub mechanism.

### Declaring topics
All WampSharp server-side topics are contained in a WampTopicContainer. This is a property of the DefaultWampHost (called TopicContainer).

There are two types of topics:
* Temporary topics - created automatically when a subscriber subscribes to them, deleted when no one is subscribed.
* Persistent topics - created once by the application, never deleted.

In order to subscribe to new created topics, the WampTopicContainer has a TopicCreated event. It also has a TopicRemoved event that allows to subscribe to removed topics.

In order to create a persistent topic, call CreateTopicByUri, and pass true for the persistent parameter.

### About topics
Topics are rx Subjects, that means that they are both observable and observers. That means you can both subscribe to them and send them messages.

In order to subscribe to a topic, Use reactive extensions subscribe extension method. This will allow you to be notified about events published to topic.

In order to send messages to a topic, use OnNext method which sends an object to all topic subscribers.

A topic also contains events for notifying when a new subscriber subscribes to a topic and when it leaves.