---
layout: post
title: "Auth0: using system browser to authenticate in .NET desktop application"
categories: misc
tags:
- OAuth2.0
- Authentication
- Auth0
cover-img: /assets/img/article7/cover.jpg
---

In [previous article](https://melmanm.github.io/misc/2023/02/13/article6-oauth20-authorization-in-desktop-applicaions.html) I described general ideas on how to integrate OAuth 2.0 authorization and authentication with desktop applications. In this article I will describe how to implement authentication in .NET desktop application with Auth0, using default system browser to perform user's login and logout.


- [Auth0 nuget packages](#auth0-nuget-packages)
  - [Default implementation.](#default-implementation)
- [Display auth0 login form in system browser](#display-auth0-login-form-in-system-browser)
  - [Desired architecture](#desired-architecture)
  - [Custom IBrowser implementation](#custom-ibrowser-implementation)
  - [Implementation considerations](#implementation-considerations)
    - [Redirect URI](#redirect-uri)
    - [HttpListener instance](#httplistener-instance)
    - [Timeout](#timeout)



## Auth0 nuget packages
Auth0 provides `Auth0.OidcClient.WPF` and `Auth0.OidcClient.WinForms` nuget packages, which simplifies authentication implementation in WPF and WinForms-based applications.
Following example shows minimal code, needed to authenticate user and gather id_token using `Auth0.OidcClient.WPF` library.

```csharp
string domain = ;//application domain
string clientId = ;//application clientId
client = new Auth0Client(new Auth0ClientOptions
{
    Domain = domain,
    ClientId = clientId
});
var loginResult = await client.LoginAsync();
var id_token = loginResult.IdentityToken;
var userClaims = loginResult.User.Claims;
```

`Auth0ClientOptions` includes `IBrowser Browser {get; set;}` property, which takes responsibility of displaying login screen to the user. Moreover, it is used to perform user logout process.

### Default implementation. 
By default Auth0 uses `WebViewBrowser` implementation of `IBrowser` interface. It uses [`WebViewCompatible`](https://learn.microsoft.com/en-us/windows/communitytoolkit/controls/wpf-winforms/webviewcompatible) component to display login window to the user.
By default it renders html login page in new application window
![auth0-default-login](/assets/img/article7/auth0-inapp-login.png)

## Display Auth0 login form in system browser
In this section I will present custom implementation of `IBrowser` interface, which displays login form using system browser.

### Desired architecture
In order to display login form, system browser should be launched by desktop application, with authorization request configured as startup URL.

After user login, application has to capture redirect URI auth, with authorization code provided as parameter. Since system web browser is running on client machine, it can successfully resolve localhost loopback address. Application can host http server on one of the localhost ports e.g. `http://localhost:8888`. Once localhost address is set as redirect URI on authorization server and specified in */authorize* request, authorization response containing code can be captured by localhost server.

![system-browser-flow](/assets/img/article6/system-browser-flow.png)

### Custom IBrowser implementation
`IBrowser` interface includes a single signature:

```csharp
public interface IBrowser
{
    Task<BrowserResult> InvokeAsync(BrowserOptions options, CancellationToken cancellationToken = default(CancellationToken));
}
```

`BrowserOptions`, passed as an argument, specifies `StartUrl` and `EndUrl`
* `StartUrl` - URL to initiate OAuth action. In case of authentication or authorization it is auth server's `/authorize` URL. In the context of logout it specifies `/logout` endpoint request.
* `EndUrl` - Redirect URL. In case of logout it represents post logout redirect url.

Following, custom, `IBrowser` implementation utilizes system browser to perform login and logout

```csharp
internal class SystemBrowser : IBrowser
{
    const string ERROR_MESSAGE = "Error ocurred.";
    const string SUCCESSFUL_AUTHENTICATION_MESSAGE = "You have been successfully authenticated. You can now continue to use desktop application.";
    const string SUCCESSFUL_LOGOUT_MESSAGE = "You have been successfully logged out.";
    private HttpListener _httpListener;
    private void StartSystemBrowser(string startUrl)
    {
        Process.Start(new ProcessStartInfo(startUrl) { UseShellExecute = true });
    }
    public async Task<BrowserResult> InvokeAsync(BrowserOptions options, CancellationToken cancellationToken = default)
    {
        StartSystemBrowser(options.StartUrl);
        var result = new BrowserResult();

        //abort _httpListener if exists
        _httpListener?.Abort();
        using (_httpListener = new HttpListener())
        {
            var listenUrl = options.EndUrl;

            //HttpListenerContext require uri ends with /
            if (!listenUrl.EndsWith("/"))
                listenUrl += "/";
            
            _httpListener.Prefixes.Add(listenUrl);
            _httpListener.Start();
            using (cancellationToken.Register(() =>
            {
                _httpListener?.Abort();
            }))
            {
                HttpListenerContext context;
                try
                {
                    context = await _httpListener.GetContextAsync();
                }
                //if _httpListener is aborted while waiting for response it throws HttpListenerException exception
                catch (HttpListenerException)
                {
                    result.ResultType = BrowserResultType.UnknownError;
                    return result;
                }

                //set result response url
                result.Response = context.Request.Url.AbsoluteUri;

                //generate message displayed in the browser, and set resultType based on request
                string displayMessage;
                if (context.Request.QueryString.Get("code") != null)
                {
                    displayMessage = SUCCESSFUL_AUTHENTICATION_MESSAGE;
                    result.ResultType = BrowserResultType.Success;
                }
                else if(options.StartUrl.Contains("/logout") && context.Request.Url.AbsoluteUri == options.EndUrl)
                {
                    displayMessage = SUCCESSFUL_LOGOUT_MESSAGE;
                    result.ResultType = BrowserResultType.Success;
                }
                else
                {
                    displayMessage = ERROR_MESSAGE;
                    result.ResultType = BrowserResultType.UnknownError;
                }

                //return message to be displayed in the browser
                Byte[] buffer = System.Text.Encoding.UTF8.GetBytes(displayMessage);
                context.Response.ContentLength64 = buffer.Length;
                context.Response.OutputStream.Write(buffer, 0, buffer.Length);
                context.Response.OutputStream.Close();
                context.Response.Close();
                _httpListener.Stop();
            }
        }
        return result;
    }
}
```

Once authorization code is captured, system browser displays

![system-browser-flow](/assets/img/article7/successful-authentication.jpg)

After successful logout user should see

![system-browser-flow](/assets/img/article7/successful-logout.jpg)

`SystemBrowser` class can be used to instantiate `Auth0ClientOptions` as below
```csharp

private Auth0Client _client =  new Auth0Client(new Auth0ClientOptions
{
    Domain = ;//application domain
    ClientId = ;//application clientId
    Browser = new SystemBrowser(),
    RedirectUri = "http://localhost:8888",
    PostLogoutRedirectUri = "http://localhost:8888/logout"
});

public void Login
{
    var loginResult = await _client.LoginAsync();
    var id_token = loginResult.IdentityToken;
    var userClaims = loginResult.User.Claims;
}
```

Note that given `RedirectUri` and `PostLogoutRedirectUri` need to be registered in Auth0 application settings.

### Implementation considerations
#### Redirect URI
Proposed implementation arbitrary specifies redirect URI as `http://localhost:8888`. It can be considered as risky approach, since port `8888` is assumed not occupied by other process on client machine. In case this specific port is used by other process, authentication could not be performed.

To mitigate this risk, implementation could verify if desired localhost port is not occupied. In case it is occupied, other port can be verified and eventually used i.e. `8889`.

Note that all possible redirect URIs needs to be registered in Auth0 application settings. As for now Auth0 does not provide an option to specify redirect URI port using wildcard (i.e. `http://localhost:*`). However, it is possible to register multiple localhost redirect URIs, with a certain port range. Application can try to find a free port within the registered range.

#### HttpListener instance
In presented example, `SystemBrowser` holds an instance of `HttpListener`. Before the instance is initialized, there is `_httpListener?.Abort();` executed to close and dispose existing `HttpListener` instance (if there is any). This mechanism is intended for scenarios, where user initiates login, but does not complete it. In subsequent login attempt existing listener will be aborted first, before initializing new instance. `Abort()` causes `HttpListenerException` in original thread, so it needs to be handled properly.

#### Timeout
Presented example, for sake of simplicity, does not consider timeout. `BrowserOptions`, passed as input parameter, contains `public TimeSpan Timeout { get; set; }` property. Further implementation can abort `HttpListener`, if no request was captured within given timeout.





