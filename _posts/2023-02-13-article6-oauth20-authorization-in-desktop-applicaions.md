---
layout: post
title: "OAuth 2.0 authorization in desktop applications"
categories: misc
tags:
- OAuth2.0
- Authorization
cover-img: /assets/img/article6/system-browser-flow.png
---

This article describes possible solutions for integration desktop applications with OAuth 2.0 compliant authorization serves. Thanks to OAuth 2.0 flexibility, it is possible to perform authorization on many platforms, including Desktop. This article focuses on authorizing to OAuth 2.0 servers on user’s behalf. 

#Table of contents
- [Desktop application](#desktop-application)
  - [Challenges](#challenges)
- [Solution 1: Native web browser component](#solution-1-native-web-browser-component)
  - [Redirect URI](#redirect-uri)
  - [Security Consideration](#security-consideration)
- [Solution 2. System Web Browser](#solution-2-system-web-browser)
  - [Http or Https](#http-or-https)
  - [Redirect URI](#redirect-uri--1)
  - [User experience](#user-experience)
  - [SSO](#sso)
  - [Security Consideration](#security-consideration--1)
- [Dedicated libraries](#dedicated-libraries)
- [Summary](#summary)

# Desktop application
Desktop application should be considered as public (non-confidential) client. During installation, all application’s binaries and files are copied into local file system. Since they can be easily decompiled and inspected by anyone having an access to file system, application should not contain any secrets. In this context, desktop application is similar to SPA web application, where JavaScript code can be entirely inspected.  

It is highly [recommended](https://www.rfc-editor.org/rfc/rfc8252#section-6) to use Authorization Code grant flow with PKCE extension, to authorize user with Desktop application. PKCE is designed for public clients and doesn’t require storing any secrets on user’s device. 

While performing on user’s behalf authorization, it is necessary to display a form, where users can enter their credentials. Authorization servers return html-based form in response to */authorize* request, which initiated OAuth grant flow.

Desktop application has to display html-based form to the user. Once user is authorized, application gathers authorization code. Authorization code is needed to obtain access/id token from /token endpoint.

Authorization server sends authorization code to the redirect URI via front channel. Redirect URI is specified in initial /authorize request as parameter. To prevent [open redirector attack](https://oauth.net/advisories/2014-1-covert-redirect/), redirect URI needs to be first configured on authorization server in application registration settings.  

Gathering authorization code from redirect URI, becomes a challenge for desktop application, since normally it doesn’t operate in web domain and can’t be reached by URI.

## Challenges

There are two main challenges that needs to be addressed to integrate desktop application with OAuth 2.0 authorization server.  
1. **How to display html-based authorize form**
2. **How to gather authorization code, sent to redirect URI**

# Solution 1: Native web browser component
Many desktop frameworks provide a dedicated web browser component. It can be embedded in another control or displayed in separate application widow. Application code can control browser component behavior and react to its events.

Application can subscribe to web browser component event, which is raised after address in navigation bar is changed. For instance, .NET *WebView2* control provides [*NavigationStarting*](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/navigation-events) event, triggered when

*WebView2 starts to navigate and the navigation results in a network request. The host may disallow the request during the event.* 

By subscribing to this event, application can intercept each change of browser navigation URI. Once URI matches redirect URI, application can capture authorization code provided as parameter.

![native-browser-component](/assets/img/article6/native-browser-flow.png)

Below C# demo code presents how WebView2 control can be used to handle OAuth 2.0 authorization, using Authorization code flow with PKCE.
```csharp
void Authorize() 
{ 
  var authorizeRequest = CreateAuthorizeRequest(); 
  webView2.CoreWebView2.Navigate(authorizeRequest); 

  //subscribe NavigationStarting event 
  webView2.NavigationStarting += (sender, e) => 
  { 
    if (!e.Uri.StartsWith(_redirectUri)) 
      return; 

    //get code parameter from redirect uri  
     var uri = new Uri(e.Uri); 
    var code = HttpUtility.ParseQueryString(uri.Query).Get("code"); 

    //request token using authorization code 
    var token = GetToken(code);
  }; 
} 
```
*CreateAuthorizeRequest() and GetToken(string token) methods implementation is not shown for simplicity.* 

## Redirect URI
Application doesn't require redirect URI to be reached. It takes advantage from browser just trying to navigate to it. Since it is not required to be reachable, it can be any, even not existing, URI. It only needs to be configured as allowed redirect URI on authorization server. 

Authorization server providers, usually add default redirect URI, when developer registers desktop application. It is usually a neutral domain, which belongs to provider. It most likely returns empty html page with 200 status code.

## Security Consideration
Native browser component, usually doesn't display address bar. Even it is displayed, its content is entirely controlled by desktop application. It makes users unable to verify if they are signing-in to legitimate authorization server. Authorization server address and its https certificate, should be always verified by the user to mitigate phishing attacks. Even if your application is trusted, it is not good practice to make users used to such design.

Embedding sing-in form in desktop application, violates the principle of least privilege. Since application entirely controls browser component, it can get an access to user credentials. For instance, it can be achieved, by capturing keystrokes while sign-in form is displayed. Username and password can be used by application to obtain more privileged accesses to resources, without user consent. Again, even if your application is trusted, it is not good practice to make users used to design, which can be harmful in case of dealing with malicious application.

# Solution 2. System Web Browser
Alternatively, default system browser (like chrome, firefox, edge) can be used. 

In order to display login form, system browser can be launched by desktop application, with */authorize* request configured as startup URI. Unfortunately, application logic is not able to react on system browser's navigation events, like in Solution 1.

Though application can host its own http server endpoint available at redirect URI address. Since system web browser is running on client machine it can successfully resolve localhost loopback address. Considering that, application can host http server on one of the localhost ports e.g. `http://localhost:8888`. Once localhost address is set as redirect URI on authorization server and specified in */authorize* request, authorization response containing code can be redirected to it.

![system-browser-component](/assets/img/article6/system-browser-flow.png)

Below C# demo code shows how authorization code can be captured by local server.
```csharp
string Authorize() 
{ 
   var redirectUri = "http://localhost:8888/"; 
   var authorizeRequest = CreateAuthorizeRequest(redirectUri); 
  
   //start system browser 
   Process.Start(new ProcessStartInfo(authorizeRequest) { UseShellExecute = true }); 
  
   using var listener = new HttpListener(); 
   listener.Prefixes.Add(redirectUri); 
   listener.Start(); 
    
   //wait for server captures redirect_uri  
   HttpListenerContext context = listener.GetContext(); 
   HttpListenerRequest request = context.Request; 
  
   var code = request.QueryString.Get("code"); 
  
   context.Response.Close(); 
   listener.Stop(); 
  
   var token = GetToken(code); 
   return token; 
}
```
*CreateAuthorizeRequest() and GetToken(string token) methods are not shown for simplicity.*

## Http or Https
Since the redirect never leaves the user’s PC it is acceptable to use `http://localhost:8888` instead of `https://localhost:8888`.

## Redirect URI
Some authorization servers allow to register localhost redirect URIs with the port wildcard (like `http://localhost:*`) or accepts all different ports if only `http://localhost` is register. It is very useful for desktop applications running inside user’s operating systems. Some ports can be already taken by other processes. If intended port is taken, application is able to host http server on different port.

## User experience
It is recommended to avoid closing web browser after token is obtained. If system browser is already opened on user’s device, an attempt to launch specific URL, results in opening new tab in existing window. Termination of entire browser process, closes all tabs, which leads to rather poor user experience. Instead, localhost server can return html page, with a text confirming successful authorization. For instance:

```
You have been successfully authorized. You can now continue to use desktop application.
```
## SSO
System browser usage can leverage SSO experience. Single sign-on (SSO) mechanism allows the same user session to be used by multiple applications. It provides great user experience, since user can be automatically signed-in to multiple applications, registered in authorization server, after single authentication. It is possible, thanks to user session cookie, which is stored in user browser after successful login. Session cookie is then sent to authorization server with subsequent requests. It can be then recognized and validated. Once it is valid, authorization server assumes user session is active. In result login form is not displayed again.

System browser enables desktop applications to share user session with web applications, registered under the same authorization server. (Authorization servers often allow to configure group of applications allowed to share the same user session).

## Security Consideration
Using system browser, to authorize desktop application’s user, involves a localhost server endpoint. In case loopback interface is accessible by other application, authorization code can be intercepted. To prevent it, applications should implement PKCE. It protects the intercepted authorization code from being used to obtain a token.

**System browser, is OAuth 2.0 recommended solution for authorizing users in desktop applications.**

# Dedicated libraries
Authorization server providers often distribute code libraries dedicated to their products. If dedicated library is available, it is highly recommended to use it. It makes implementation easier and ensures it is compatible with specific authorization server. 

# Summary
Table below compares key features of presented solution 

| | Native browser component | System browser 
| :---  | :--- |:--- |
| **User experience**  | Good. Authorization is entirely performed in desktop application.   | Medium. User is navigated to system browser to authorize. After process in finished user needs to navigate back to desktop application. |
| **Security** | User is not able to verify authorization server address and certificate. | OAuth 2.0 recommendation |
| **SSO** | No, partially depends on framework and component  | Yes |
| **Support in popular Authorization server providers libraries** | Yes  | Yes |
