##Raw api

If you want to get feedbacks for publications, you can use publisher raw api.
In order to use it in client side, access the RealmProxy of your WampChannel and then access its TopicContainer property in order to obtain a WampTopicProxy to your topic. Then call Publish with PUBLISH parameters.

Client side sample:

```csharp
const string serverAddress = "ws://127.0.0.1:8080/ws";

DefaultWampChannelFactory channelFactory = new DefaultWampChannelFactory();

IWampChannel channel = channelFactory.CreateJsonChannel(serverAddress, "realm1");

await channel.Open();

int counter = 0;

IWampTopicProxy topicProxy =
    channel.RealmProxy.TopicContainer.GetTopicByUri("com.myapp.topic1");

IObservable<long> timer =
    Observable.Timer(TimeSpan.FromMilliseconds(0),
                     TimeSpan.FromMilliseconds(1000));

timer.Subscribe(async x =>
    {
        Task<long?> publishTask =
            topicProxy.Publish
                (new PublishOptions
                    {
                        Acknowledge = true,
                        DiscloseMe = true
                    },
                 new object[] {counter});

        try
        {
            long? publicationId = await publishTask;

            if (publicationId != null)
            {
                Console.WriteLine("Event published with publication ID " + publicationId);
            }
        }
        catch (WampException ex)
        {
            Console.WriteLine("An error occured while publishing event: " +
                              ex);
        }
    });
```

In order to use it server side:
Obtain the realm by accessing the RealmContainer property of your WampHost, then access its TopicContainer property in order to access the topic by its uri, then call publish with desired parameters.

```csharp
const string serverAddress = "ws://127.0.0.1:8080/ws";

DefaultWampHost host = new DefaultWampHost(serverAddress);

host.Open();

IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

IObservable<long> timer =
    Observable.Timer(TimeSpan.FromMilliseconds(0),
                     TimeSpan.FromMilliseconds(1000));

int counter = 0;

timer.Subscribe(x =>
    {
        try
        {
            long publicationId =
                realm.TopicContainer.Publish
                    (WampObjectFormatter.Value,
                     new PublishOptions(),
                     "com.myapp.topic1",
                     new object[] {counter});

            Console.WriteLine("Event published with publication ID " + publicationId);
        }
        catch (WampException ex)
        {
            Console.WriteLine("An error occured while publishing event: " +
                              ex);
        }
    });
```
