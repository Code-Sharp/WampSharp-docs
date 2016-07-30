## WampSharp v1.2.4.18-beta release notes

**Contents**

1. [New features](#new-features)
    * [.NET Standard support](#net-standard-support)
    * [ASP.NET Core support](#aspnet-core-support)
2. [Other changes](#other-changes)
	* [Internal changes](#internal-changes)
    * [Windows Phone issues](#windows-phone-issues)
    * [Dependencies update](#dependencies-update)

### New features

#### .NET Standard support

This verion mainly focuses on .NET Standard support. WampSharp now supports [.NET Standard 1.3](https://github.com/dotnet/corefx/blob/v1.0.0/Documentation/architecture/net-platform-standard.md) which means that it is compatible with .NET Framework 4.6, [NET Core](http://dot.net), Universal Windows Platform 10 and Mono/Xamarin platforms.

Only WAMPv2 is currenly supported (WAMPv1 hasn't been ported yet). Both Json and MsgPack support is available. RawSocket support has also been ported.

Regarding WebSockets: 
* Client-side support is available: it works via a new implementation of WampConnection which is based on [System.Net.WebSockets](https://www.nuget.org/packages/system.net.websockets/). Note that this currently doesn't work on .NET Core on Unix, due to the [following (resolved) issue](https://github.com/dotnet/corefx/issues/2486). The issue is resolved, and this should work fine on Unix environments after [.NET Core 1.1](https://github.com/dotnet/core/blob/master/roadmap.md) is released.
* Router-side support is available via ASP.NET Core [WebSockets.Server](https://www.nuget.org/packages/Microsoft.AspNetCore.WebSockets.Server/). See the following section for an usage example.

''Note'': WampSharp.Default.Router isn't present yet since I'm not aware of any standalone .NET Standard compatible WebSocket server (such as Fleck or Vtortola/WebSocketListener).

#### ASP.NET Core support

This version introduces ASP.NET core support. In order to use it, you need to create [a new empty ASP.NET Core project](https://docs.asp.net/en/latest/getting-started.html).

Install the following packages: WampSharp.AspNetCore.WebSockets.Server, WampSharp.NewtonsoftMsgpack (you can also install only WampSharp.NewtonsoftJson if you're not interested in MsgPack support).

Change your current Statup class to the following class

```csharp
public class Startup
{
    // This method gets called by the runtime. Use this method to add services to the container.
    // For more information on how to configure your application, visit http://go.microsoft.com/fwlink/?LinkID=398940
    public void ConfigureServices(IServiceCollection services)
    {
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        loggerFactory.AddConsole();

        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        WampHost host = new WampHost();

        app.Map("/ws", builder =>
        {
            // Comment this out to test native server implementations
            builder.UseWebSockets(new WebSocketOptions
            {
                ReplaceFeature = true
            });

            host.RegisterTransport(new AspNetCoreWebSocketTransport(builder),
                                   new JTokenJsonBinding(),
                                   new JTokenMsgpackBinding());
        });

        host.Open();
    }
}
```

This code hosts a WampSharp Router in a ASP.NET Core application under the "/ws" path. The WampSharp relevant code begins in the line declaring the WampHost variable. ASP.NET Core runs by default from port 5000, so you'll need to access "ws://localhost:5000/ws". You can change the default port in various ways, such as calling ".UseUrls("http://127.0.0.1:8080")" on the WebHostBuilder in your Main.

### Other changes

#### Internal changes

RawSocket implementation now uses [System.Buffers](https://www.nuget.org/packages/System.Buffers/) instead of [Microsoft.IO.RecyclableMemoryStream](https://www.nuget.org/packages/Microsoft.IO.RecyclableMemoryStream/), since the [latter is not .NET Standard compatible](https://github.com/Microsoft/Microsoft.IO.RecyclableMemoryStream/issues/11).

Message queue is now powered by [System.Threading.Tasks.Dataflow](https://www.nuget.org/packages/System.Threading.Tasks.Dataflow) for all supported platforms, except for .NET Framework 4.0, which still uses an [Ix-Async](https://www.nuget.org/packages/Ix-Async) based message queue.

#### Windows Phone issues

Windows Phone issues ([#122](https://github.com/Code-Sharp/WampSharp/issues/122), [#109](https://github.com/Code-Sharp/WampSharp/issues/109)) should now be resolved. The latter by replacing the message queue with Dataflow.

#### Dependencies update

The following dependencies have been updated: System.Collections.Immutable, vtortola.WebSocketListener, WebSocket4Net, Newtonsoft.Msgpack. In addition, [Reactive Extensions](https://github.com/Reactive-Extensions/Rx.NET) has been updated to version 3.0, which means that WampSharp now refers System.Reactive.* packages instead of Rx-* packages.

The Newtonsoft.Msgpack update is critical in order to communicate with other non-WampSharp peers via MsgPack.
