## Cookie based router-side authentication

It is possible to build authenticators that are cookie-based. In order to that, one needs to implement an interface named ICookieAuthenticatorFactory and pass it to DefaultWampAuthenticationHost's constructor, (or to FleckAuthenticatedWebSocketTransport/VtortolaAuthenticatedWebSocketTransport constructors).

The interface consists of a single method named "CreateAuthenticator", which receives as a parameter an ICookieProvider, that is an interface that allows to access client cookies for read-only.

The authenticator created by the "CreateAuthenticator" method will be passed to the IWampSessionAuthenticatorFactory passed to the (Default)WampAuthenticationHost as the transportAuthenticator parameter. This allows to combine between authentication methods.

> Note: it is not possible to set cookies from WampSharp. At this moment [Fleck](https://github.com/statianzo/Fleck) doesn't support cookie set on Handshake, so this won't be possible until someone implements this feature.

### Example

This example demonstrates cookie-based authenticator usage. Cookies are set using an embedded [uhttpSharp server](https://github.com/Code-Sharp/uHttpSharp).

Open "http://localhost" in your browser to see this demo in action.

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;
using uhttpsharp;
using uhttpsharp.Handlers;
using uhttpsharp.Headers;
using uhttpsharp.Listeners;
using uhttpsharp.RequestProviders;
using WampSharp.V2;
using WampSharp.V2.Authentication;
using WampSharp.V2.Core.Contracts;
using WampSharp.V2.Realm;
using WampSharp.V2.Rpc;

namespace CookieDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            DefaultWampAuthenticationHost host =
                new DefaultWampAuthenticationHost("ws://0.0.0.0:8080/ws",
                                                  new MyAuthenticationFactory(),
                                                  new MyCookieAuthenticatorFactory());

            IWampHostedRealm realm = host.RealmContainer.GetRealmByName("realm1");

            realm.Services.RegisterCallee(new Calculator());

            host.Open();

            using (var httpServer = new HttpServer(new HttpRequestProvider()))
            {
                httpServer.Use(new TcpListenerAdapter(new TcpListener(IPAddress.Any, 80)));

                httpServer.Use(new HttpRouter().With("", new FrontendHandler())
                                               .With("setcookie", new CookieHandler()));

                httpServer.Use((context, next) => Task.Factory.GetCompleted());

                httpServer.Start();

                Console.ReadLine();
            }
        }
    }

    public class Calculator
    {
        [WampProcedure("com.arguments.add2")]
        public int Add2(int x, int y)
        {
            return (x + y);
        }

        [WampProcedure("com.arguments.mul2")]
        public int Mul2(int x, int y)
        {
            return (x * y);
        }

        [WampProcedure("com.arguments.sub2")]
        public int Sub2(int x, int y)
        {
            return (x - y);
        }

        [WampProcedure("com.arguments.div2")]
        public int Div2(int x, int y)
        {
            return (x / y);
        }
    }

    internal class MyCookieAuthenticatorFactory : ICookieAuthenticatorFactory
    {
        public IWampSessionAuthenticator CreateAuthenticator(ICookieProvider cookieProvider)
        {
            return new MyCookieAuthenticator(cookieProvider);
        }
    }

    internal class MyCookieAuthenticator : WampSessionAuthenticator
    {
        private readonly ICookieProvider mCookieProvider;

        private readonly IDictionary<string, IWampAuthorizer> mBeetleToAuthorizers =
            new Dictionary<string, IWampAuthorizer>()
            {
                ["John"] =
                    new WampStaticAuthorizer(new List<WampUriPermissions>()
                    {
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.add2",
                            CanCall = true
                        }
                    }),
                ["Ringo"] =
                    new WampStaticAuthorizer(new List<WampUriPermissions>()
                    {
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.add2",
                            CanCall = true
                        },
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.mul2",
                            CanCall = true
                        }
                    }),
                ["Paul"] =
                    new WampStaticAuthorizer(new List<WampUriPermissions>()
                    {
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.sub2",
                            CanCall = true
                        },
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.div2",
                            CanCall = true
                        }
                    }),
                ["George"] =
                    new WampStaticAuthorizer(new List<WampUriPermissions>()
                    {
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.add2",
                            CanCall = true
                        },
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.mul2",
                            CanCall = true
                        },
                        new WampUriPermissions()
                        {
                            Uri = "com.arguments.sub2",
                            CanCall = true
                        }
                    })
            };

        public MyCookieAuthenticator(ICookieProvider cookieProvider)
        {
            mCookieProvider = cookieProvider;

            Cookie cookie = mCookieProvider.GetCookieByName("beetle");

            if (cookie != null)
            {
                Beetle = cookie.Value;
                this.Authorizer = mBeetleToAuthorizers[Beetle];
                IsAuthenticated = true;
            }
        }

        public override void Authenticate(string signature, AuthenticateExtraData extra)
        {
            throw new WampAuthenticationException("Cookie wasn't present");
        }

        public string Beetle { get; set; }

        public override string AuthenticationId
        {
            get
            {
                return Beetle;
            }
        }

        public override string AuthenticationMethod
        {
            get
            {
                return "cookie";
            }
        }
    }

    internal class MyAuthenticationFactory : IWampSessionAuthenticatorFactory
    {
        public IWampSessionAuthenticator GetSessionAuthenticator
            (WampPendingClientDetails details,
             IWampSessionAuthenticator transportAuthenticator)
        {
            if (!transportAuthenticator.IsAuthenticated)
            {
                throw new WampAuthenticationException("Cookie wasn't present");
            }

            return transportAuthenticator;
        }
    }

    internal class HtmlHandler : IHttpRequestHandler
    {
        private readonly HttpResponse mResponse;
        private readonly HttpResponse mKeepAliveResponse;

        public HtmlHandler(string contents)
        {
            MemoryStream memoryStream = new MemoryStream(Encoding.UTF8.GetBytes(contents));

            mKeepAliveResponse = new HttpResponse(HttpResponseCode.Ok, "text/html",
                                                  memoryStream,
                                                  true);

            mResponse = new HttpResponse(HttpResponseCode.Ok, "text/html",
                                         memoryStream,
                                         false);
        }

        public virtual Task Handle(IHttpContext context, Func<Task> next)
        {
            context.Response = context.Request.Headers.KeepAliveConnection()
                ? mKeepAliveResponse
                : mResponse;

            return Task.Factory.GetCompleted();
        }
    }

    internal class FrontendHandler : HtmlHandler
    {
        private const string TemplateHtml = @"<!DOCTYPE html>
<html>
<body>
    <h1>Cookie authentication frontend</h1>
    <p>Open JavaScript console to watch output.</p>
    <p><a href=""http://{0}/setcookie"">Set cookie.</a></p>
    <script src=""https://autobahn.s3.amazonaws.com/autobahnjs/latest/autobahn.min.jgz""></script>
    <script type=""text/javascript"">
        var connection = new autobahn.Connection({{
            url: 'ws://{0}:8080/ws',
            realm: 'realm1'
        }});

        connection.onopen = function (session) {{

            session.call('com.arguments.add2', [2, 3]).then(
                function (res) {{
                    console.log(""Add2:"", res);
                }},
                function (err) {{
                    console.log(""Add2 Error:"", err.error, err.args, err.kwargs);
                }}
            );

            session.call('com.arguments.mul2', [5, 6]).then(
                function (res) {{
                    console.log(""Mul2:"", res);
                }},
                function (err) {{
                    console.log(""Mul2 Error:"", err.error, err.args, err.kwargs);
                }}
            );

            session.call('com.arguments.sub2', [1, 9]).then(
                function (res) {{
                    console.log(""Sub2:"", res);
                }},
                function (err) {{
                    console.log(""Sub2 Error:"", err.error, err.args, err.kwargs);
                }}
            );

            session.call('com.arguments.div2', [10, 4]).then(
                function (res) {{
                    console.log(""Div2:"", res);
                }},
                function (err) {{
                    console.log(""Div2 Error:"", err.error, err.args, err.kwargs);
                }}
            );
        }};

        connection.open();
    </script>
</body>
</html>";

        public FrontendHandler() : base(string.Format(TemplateHtml, Environment.MachineName))
        {
        }
    }

    internal class CookieHandler : HtmlHandler
    {
        private static readonly string[] mNames = new[] {"John", "Paul", "George", "Ringo"};

        private readonly Random mRandom = new Random();
        private readonly object mLock = new object();

        private const string HtmlTemplate = @"<!DOCTYPE html>
<html>
<body>
    <h1>Cookie authentication frontend</h1>
    <p>Cookie set! visit <a href=""/"">home</a></p>.
</body>
</html>";

        public CookieHandler() : base(HtmlTemplate)
        {
        }

        public override Task Handle(IHttpContext context, Func<Task> next)
        {
            context.Cookies.Upsert("beetle", GetBeetleName());
            return base.Handle(context, next);
        }

        private string GetBeetleName()
        {
            int random;

            lock (mLock)
            {
                random = mRandom.Next(mNames.Length);
            }

            return mNames[random];
        }
    }
}
```
