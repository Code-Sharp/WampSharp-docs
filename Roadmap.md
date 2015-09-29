## Roadmap

This is some proposal for WampSharp's future.

### Short term roadmap
* Support for .NET Core/ASP.NET 5
* Bug fixes
* WAMP Advanced profile modifications (meta-api, disclose_caller, shared registrations, etc)

### Long term roadmap
* Support for Tuples (C# 7.0) - see [here](https://github.com/dotnet/roslyn/issues/347) and [here](https://github.com/dotnet/roslyn/issues/3913)
* Replacing IAsyncDisposable with framework interface - see [here](https://github.com/dotnet/roslyn/issues/114)
* Reactive Extensions 3.0 Support - see [#1](https://vimeo.com/132192255) [#2](http://www.infoq.com/presentations/reactive-cloud-scale) [#3](http://www.infoq.com/presentations/cloud-rx) 
 * ISubscriable, ISubscription, IMultiSubject
 * remote LINQ: IAsyncReactiveQbservable, IAsyncReactiveQbserver, IAsyncReactiveQubscription
* WAMP schema router support See [#1](https://groups.google.com/forum/#!searchin/wampws/swagger/wampws/5Z2o25vx9nQ/1CR1LaSmKawJ) [#2](https://groups.google.com/forum/#!msg/wampws/jW_6UZYBhpQ/u8MQ70NMzrYJ) and [#3](https://github.com/tavendo/WAMP/issues/61) and Crossbar [roadmap (0.13.0)](https://github.com/crossbario/crossbar/blob/master/DEVELOPERS.md) and [issue](https://github.com/crossbario/crossbar/issues/301)
* WAMP schema code generators (for C#/Typescript too?)
