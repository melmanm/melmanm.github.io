## Device Authorization Grant
Device authorization grant enables applications with limited user input or display capabilities to to authorize users. Device authorization grant is commonly used on devices not equipped with a keyboard or other input possibility. Besides that it is designed for browserless devices and for environments that there is noo guarantee that the browser is available. Authorization grant is a perfect solution to enable user's authorization in applications running on:
* IoT devices
* printers
* Smart TV's - even if some smart tvs have a browser installed, input capabilities are very limited, and usually user needs to enter text using remote control.
* console applications - since console applications can be run in text-based used interface, there is usually no guarantee that it will be possible to launch browser and perform authorization with authorization code grant or implicit flow .

Device Authorization Grant flow requires a secondary device, like smartphone or pc, which user can use to perform authorization. Below diagram presents device authorization grant flow:

## Non-textual verification uri QR
QR code can 



--copy

## Implementing Authentication and Authorization

### 1. Register application
After logging in to auth0.com, navigate to `applications` and click `create application`. Enter application name and select 'Native' application type.
![auth0-select-application-type](/assets/img/article9/auth0-application-selection.jpg)

Next, set the callback url in `Allowed callback url` in application `Settings` tab.
![auth0-select-application-type](/assets/img/article9/auth0-callback-url.jpg)

### 2. Install nuget packages
Auth0 provides `Auth0.OidcClient.WPF` and `Auth0.OidcClient.WinForms` nuget packages, which simplifies authentication implementation in WPF and Winforms- based applications. 

Create a .NET desktop project. Depending on project type, following PM commands can be used to install packages
```Install-Package Auth0.OidcClient.WPF```
```Install-Package Auth0.OidcClient.WinForms```

### 3. Application implementation

#### Authentication
Installed nuget package can be now used to authenticate user. Following code can be used. 

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

As the result of `LoginAsync` function, login prompt will be displayed to the user.

![auth0-application-basic-settings](/assets/img/article7/auth0-inapp-login.png).

Application domain and client id can be found in application registration `Settings` tab
![auth0-application-basic-settings](/assets/img/article7/auth0-callback-url.jpg).

#### Authorization
In order to obtain JWT access_token, which can be used to access external API, it is required to specify 'scope' and 'audience' parameters. Scope defines the access level to an API. Audience identifies an API itself. Once API receives access_token it checks its audience claim to ensure the token is intended for this specific API.

More information on how to register api and validate access_token on API side can be found in auth0 documentation
* Registering API: https://auth0.com/docs/get-started/auth0-overview/set-up-apis
* Validating access_token in ASP.NET core https://auth0.com/docs/quickstart/backend/aspnet-core-webapi

Following code presents how to obtain an access_token with specific scope.

```csharp
client = new Auth0Client(new Auth0ClientOptions
{
    Domain = domain,
    ClientId = clientId,
    Scope = ""; //specify a scope
}
var extraParameters = new Dictionary<string, string>()
{
    ["audience"]= "" //specify API identifier
});

var loginResult = await client.LoginAsync(extraParameters);
var accessToken = loginResult.AccessToken;
```