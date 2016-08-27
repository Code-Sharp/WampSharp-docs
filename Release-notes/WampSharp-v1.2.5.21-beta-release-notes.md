## WampSharp v1.2.4.18-beta release notes

**Contents**

1. [New features](#new-features)
    * [C# 7.0 tuples support]

### New features

#### C# 7.0 tuples support

This verion mainly focuses on C# 7.0 tuples support.

##### Reflection-based callee tuples support

From this version, you can return a C# 7.0 ValueTuple from a reflection-based callee method. The ValueTuple will be serialized to either the arguments keywords or to the arguments array of the YIELD message, depending on whether the returned ValueTuple has named elements or positional elements. (ValueTuples having elements which are partially named are not supported)

For example: A reflection-based callee that returns ValueTuples:

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
    [WampProcedure("com.myapp.add_complex")]
    public (int c, int ci) AddComplex(int a, int ai, int b, int bi)
    {
       return (a + b, ai + bi);
    }

    [WampProcedure("com.myapp.split_name")]
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

##### Reflection-based caller tuples support

Reflection-based callers also support ValueTuple return values from this version. Simply declare a method returning a C# 7.0 ValueTuple in your callee proxy interface.

For example, declare the following callee proxy interface:

```csharp
public interface IComplexResultServiceProxy
{
    [WampProcedure("com.myapp.add_complex")]
    Task<(int c, int ci)> AddComplexAsync(int a, int ai, int b, int bi);

    [WampProcedure("com.myapp.split_name")]
    Task<(string, string)> SplitNameAsync(string fullname);
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

This code can consume the following code written in javascript:

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