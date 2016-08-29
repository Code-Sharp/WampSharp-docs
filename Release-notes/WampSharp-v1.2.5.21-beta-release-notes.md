## WampSharp v1.2.5.21-beta release notes

**Contents**

1. [C# 7.0 tuples support](#c-70-tuples-support)
    * [Reflection-based callee tuples support](#reflection-based-callee-tuples-support)
    * [Reflection-based caller tuples support](#reflection-based-caller-tuples-support)
    * [Rx-based publish/subscribe tuples support](#rx-based-publishsubscribe-tuples-support)
        * [Rx-based publisher tuples support](#rx-based-publisher-tuples-support-sample)
        * [Rx-based subscriber tuples support](#rx-based-publisher-tuples-support-sample)
2. [Other features]
    * [Instance providers for reflection-based callee](#instance-providers-for-reflection-based-callee)
    * [.NET Standard WebSocket4Net support](#net-standard-websocket4net-support)
    * [Formatted logging](#formatted-logging)

### C# 7.0 tuples support

This verion mainly focuses on [C# 7.0 tuples](https://blogs.msdn.microsoft.com/dotnet/2016/08/24/whats-new-in-csharp-7-0/) support.
> Note: C# 7.0 tuples syntax is currently supported only by [Visual Studio "15" Preview 4](https://www.visualstudio.com/en-us/news/releasenotes/vs15-relnotes). You can download it [here](https://www.visualstudio.com/en-us/downloads/visual-studio-next-downloads-vs.aspx).

#### Reflection-based callee tuples support

From this version, you can return a C# 7.0 tuple from a reflection-based callee method. The tuple will be serialized to either the arguments keywords or to the arguments array of the YIELD message, depending on whether the returned tuple has named elements or positional elements. (tuples having elements which are partially named are not supported)

For example: A reflection-based callee that returns a tuple:

```csharp
public interface IComplexResultService
{
    [WampProcedure("com.myapp.add_complex")]
    (int c, int ci) AddComplex(int a, int ai, int b, int bi);

    [WampProcedure("com.myapp.split_name")]
    (string, string) SplitName(string fullname);
}

public class ComplexResultService : IComplexResultService
{
    public (int c, int ci) AddComplex(int a, int ai, int b, int bi)
    {
       return (a + b, ai + bi);
    }

    public (string, string) SplitName(string fullname)
    {
        string[] splitted = fullname.Split(' ');

        string forename = splitted[0];
        string surname = splitted[1];

        return (forename, surname);
    }
}
```
> Note: as usual, you can put the WampProcedureAttributes on the methods themselves instead of implementing an interface, i.e:
>
>```csharp
>[WampProcedure("com.myapp.add_complex")]
>public (int c, int ci) AddComplex(int a, int ai, int b, int bi)
>{
>   // ...
>}
>
>[WampProcedure("com.myapp.split_name")]
>public (string, string) SplitName(string fullname)
>{
>   // ...
>}
>```


Which can be consumed from [AutobahnJS](https://github.com/crossbario/autobahn-js):

```javascript
session.call('com.myapp.add_complex', [2, 3, 4, 5]).then(
    function (res) {
        console.log("Result: " + res.kwargs.c + " + " + res.kwargs.ci + "i");
    }
);

session.call('com.myapp.split_name', ['Homer Simpson']).then(
    function (res) {
        console.log("Forename: " + res.args[0] + ", Surname: " + res.args[1]);
    }
;)
```

>Note:  The samples are based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS/AutobahnPython sample

#### Reflection-based caller tuples support

Reflection-based callers also support C# 7.0 tuple return values from this version. You can simply declare a method returning a C# 7.0 tuple in your callee proxy interface.

For example, declare the following callee proxy interface:

```csharp
public interface IComplexResultServiceProxy
{
	[WampProcedure("com.myapp.add_complex")]
	Task<(int c, int ci)> AddComplexAsync(int a, int ai, int b, int bi);

	[WampProcedure("com.myapp.split_name")]
    Task<(string, string)> SplitNameAsync(string fullname);

    [WampProcedure("com.myapp.add_complex")]
    (int c, int ci) AddComplex(int a, int ai, int b, int bi);

    [WampProcedure("com.myapp.split_name")]
    (string, string) SplitName(string fullname);
}
```

And then obtain a callee proxy and simply call its methods:

```csharp
public async Task Run()
{
    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

    IWampChannel channel =
        factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

    await channel.Open();

    IComplexResultServiceProxy proxy =
        channel.RealmProxy.Services.GetCalleeProxy<IComplexResultServiceProxy>();

    (string forename, string surname) = await proxy.SplitNameAsync("Homer Simpson");
    // Synchronous version: 
    // (string forename, string surname) = proxy.SplitName("Homer Simpson");

    Console.WriteLine($"Forename: {forename}, Surname: {surname}");

    (int c, int ci) = await proxy.AddComplexAsync(2, 3, 4, 5);
    // Synchronous version: 
    // (int c, int ci) = proxy.AddComplex(2, 3, 4, 5);

    Console.WriteLine($"Result: {c} + {ci}i");
}
```

This code can consume the following code written in Javascript:

```javascript
function add_complex(args, kwargs) {
    return new autobahn.Result([], {c: args[0] + args[2], ci: args[1] + args[3]});
}

function split_name(args) {
    var splitted = args[0].split(" ");
    var forename = splitted[0];
    var surname = splitted[1];
    return new autobahn.Result([forename, surname]);
}

session.register('com.myapp.add_complex', add_complex).then(
    function (registration) {
        console.log("Procedure registered:", registration.id);
    },
    function (error) {
        console.log("Registration failed:", error);
    }
);

session.register('com.myapp.split_name', split_name).then(
    function (registration) {
        console.log("Procedure registered:", registration.id);
    },
    function (error) {
        console.log("Registration failed:", error);
    }
);
```
>Note:  The samples are based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/rpc/complex) AutobahnJS/AutobahnPython sample

#### Rx-based publish/subscribe tuples support

This version also introduces Rx-based publish/subscribe C# 7.0 tuple support. This allows to handle topics that have complex arguments in a strongly typed manner using ISubject<> api.

In order to use this feature, a mechanism called IWampEventValueTupleConverter is introduced.

This interface is responsible for converting a IWampSerializedEvent instance to a specified tuple and a specified tuple instance to a IWampEvent.

Luckily enough, in order to implement this interface it is sufficient to derive from WampEventValueTupleConverter<> and specify the desired tuple type. Nothing else is needed to be done.

(This might seem a bit odd, but that's the best way I'm aware of for preserving ValueTuple element names after compilation)

Then, just pass an instance of your IWampEventValueTupleConverter to the new overload of IWampRealmServiceProvider's GetSubject method, which receives the topic's uri and an instance of IWampEventValueTupleConverter, in order to receive a ISubject<> instance of your desired tuple type.

##### Rx-based subscriber tuples support sample

```csharp
public async Task Run()
{
	DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

	IWampChannel channel =
		factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

	await channel.Open();

	ISubject<(int, int)> topic1Subject =
		channel.RealmProxy.Services.GetSubject
				("com.myapp.topic1",
				new MyPositionalTupleEventConverter());

	topic1Subject.Subscribe(value =>
	{
		(int number1, int number2) = value;
		Console.WriteLine($">com.myapp.topic1: Got event: number1:{number1}, number2:{number2}");
	});

	ISubject<(int number1, int number2, string c, MyClass d)> topic2Subject =
		channel.RealmProxy.Services.GetSubject
				("com.myapp.topic2",
				new MyKeywordTupleEventConverter());

	topic2Subject.Subscribe(value =>
	{
		(int number1, int number2, string c, MyClass d) = value;
		Console.WriteLine($">com.myapp.topic2: Got event: number1:{number1}, number2:{number2}, c:{c}, d:{d}");
	});
}

public class MyPositionalTupleEventConverter : WampEventValueTupleConverter<(int, int)>
{
}

public class MyKeywordTupleEventConverter : WampEventValueTupleConverter<(int number1, int number2, string c, MyClass d)>
{
}

public class MyClass
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }

    public override string ToString()
    {
        return string.Format("counter: {0}, foo: [{1}]",
            Counter,
            string.Join(", ", Foo));
    }
}
```

This code can consume events published by the following Javascript code:

```javascript
var counter = 0;

setInterval(function () {
    var obj = {'counter': counter, 'foo': [1, 2, 3]};

    session.publish('com.myapp.topic1', [randint(0, 100), 23], {});
    session.publish('com.myapp.topic2', [], {number1: randint(0, 100), number2: 23, c: "Hello", d: obj});

    counter += 1;

    console.log("events published");
}, 1000);
```
> This example is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample.

##### Rx-based publisher tuples support sample

```csharp
public async Task Run()
{
	DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

	IWampChannel channel =
		factory.CreateJsonChannel("ws://localhost:8080/ws", "realm1");

	await channel.Open();

	ISubject<(int, int)> topic1Subject =
		channel.RealmProxy.Services.GetSubject
				("com.myapp.topic1",
				new MyPositionalTupleEventConverter());

	ISubject<(int number1, int number2, string c, MyClass d)> topic2Subject 
		channel.RealmProxy.Services.GetSubject
				("com.myapp.topic2",
				new MyKeywordTupleEventConverter());

	IObservable<int> timer =
		Observable.Timer(TimeSpan.FromMilliseconds(0),
						 TimeSpan.FromMilliseconds(1000))
						 .Select((x, i) => i);

	Random random = new Random();

	IDisposable disposable =
		timer.Subscribe(value =>
		{
			topic1Subject.OnNext((random.Next(0, 100), 23));
			topic2Subject.OnNext((random.Next(0, 100), 23, "Hello",
								  new MyClass()
								  {
									  Counter = value,
									  Foo = new int[] { 1, 2, 3 }
								  }));
		});
}

public class MyPositionalTupleEventConverter : WampEventValueTupleConverter<(int, int)>
{
}

public class MyKeywordTupleEventConverter : WampEventValueTupleConverter<(int number1, int number2, string c, MyClass d)>
{
}

public class MyClass
{
    [JsonProperty("counter")]
    public int Counter { get; set; }

    [JsonProperty("foo")]
    public int[] Foo { get; set; }

    public override string ToString()
    {
        return string.Format("counter: {0}, foo: [{1}]",
            Counter,
            string.Join(", ", Foo));
    }
}
```

The following Javascript code can consume events published by the code above:

```javascript
function on_topic1(args, kwargs) {
    console.log("com.myapp.topic1: Got event:", args, kwargs);
}

function on_topic2(args, kwargs) {
    console.log("com.myapp.topic2: Got event:", args, kwargs);
}

session.subscribe('com.myapp.topic1', on_topic1);
session.subscribe('com.myapp.topic2', on_topic2);
```

> This example is based on [this](https://github.com/tavendo/AutobahnPython/tree/master/examples/twisted/wamp/pubsub/complex) AutobahnJS sample.

### Other features

#### Instance providers for reflection-based callee

From this version, new overloads for IWampRealmServiceProvider's RegisterCallee method were added. These overloads allow you to specify an instance provider for your callee service instance, that is a Func<> delegate that returns an instance of your callee service provider.

This allows you to control yourself the lifecycle of the callee service instance. You can of course use an dependency injection framework to help you manage that.

For instance, using [Ninject](http://www.ninject.org/):

```
public static async Task Run()
{
    IKernel kernel = new StandardKernel();

    kernel.Bind<IAddCalculator>().To<Calculator>();

    DefaultWampChannelFactory factory = new DefaultWampChannelFactory();

	IWampChannel channel =
		factory.CreateJsonChannel("ws://127.0.0.1:8080/ws", "realm1");

	await channel.Open();

    await channel.RealmProxy.Services.RegisterCallee(() => kernel.Get<IAddCalculator>());
}

public interface IAddCalculator
{
    [WampProcedure("com.arguments.add2")]
    int Add2(int x, int y);
}

public class Calculator : IAddCalculator
{
    public int Add2(int x, int y)
    {
        return (x + y);
    }
}
```

> Note: if you're using Ninject, you're recomended to use [Ninject.Extensions.Factory](https://github.com/ninject/Ninject.Extensions.Factory) which allows you to get typed factories instances or Func<> instances without being aware about the dependency injection container.

#### .NET Standard WebSocket4Net support

[WebSocket4Net](https://github.com/kerryjiang/WebSocket4Net) has been ported to .NET Standard 1.3 recently. From this version WampSharp.WebSocket4Net is supported for .NET Standard 1.3 as well.

This could be an alternative for running WampSharp over WebSockets on non-Windows platforms for now, until an update for .NET Core which supports WebSockets on non-Windows platforms is released.

#### Formatted logging

From this version, some formatted logs are written using [structured logging](https://github.com/serilog/serilog/wiki/Structured-Data). These include properties such as session ids, Incoming WAMP Json/MsgPack messages and more. This feature can be leveraged by [Serilog](https://serilog.net/).