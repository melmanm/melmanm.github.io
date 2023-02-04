---
layout: post
title: "OpenId Connect Session Management"
categories: misc
tags:
- OpenId-Connect
- OAuth2.0
---

Session, in the context of OpenId Connect, is a way to maintain the context of logged in user across Applications and Identity Provider Server. Maintaining user session can improve user experience by 

* **Avoiding frequent authentication** – As long as there is active session available, application can acquire fresh access/id tokens from Identity Provider Server, without user being ask to re-enter credentials. 

* **Enabling Single sign-on (SSO)** - SSO enables reusing the same user session across multiple applications launched in the same web browser and working under the same Identity Provider Server. Once user logs-in into single application, other applications, from the same suite or family, can be used without user being ask to re-authenticate. 

* **Synchronizing user context across applications** - From application perspective it is important to know if it works in the context of logged-in user or not. Since identity management is delegated to Identity Provider Server, it is important for the application to be synchronized with Identity Provider and be able to react on user session events. It becomes challenging, in SSO scenarios, where multiple applications, use the same user session. Some actions like user logout, password reset or users account deletion requires all other applications using current session to be informed.  
Additionally, Identity Provider servers usually provides administrator option to revoke all user 	sessions. It can be used if user device was lost or stolen to protect against unauthorized access 	and malicious usage. 

In this article I will describe how applications determine user session status, considering SSO scenarios. I will focus on methods provided by OpenId Connect. For applications requiring custom solutions, described methods can be used as a foundation. 


## Table of contents
- [Terminology](#terminology)
- [Determining session status](#determining-session-status)
  - [Polling Based Solutions](#polling-based-solutions)
    - [check_session_iframe endpoint](#check_session_iframe-endpoint)
    - [/authorize endpoint polling](#authorize-endpoint-polling)
  - [Logout Based Solutions](#logout-based-solutions)
    - [Front-Channel Logout](#front-channel-logout)
    - [Back-Channel Logout](#back-channel-logout)

<!-- /code_chunk_output -->

## Terminology
To keep this article consistent, I will use following naming for OpenId Connect parties 

***RP*** – Relaying Party, Client Application 

***IdP*** - Authorization Server, Identity Provider Server

additionally 

***SSO scope*** – all RP’s which utilize the same user session. 

## Determining session status
Knowledge of session status is critical for RPs. Applications usually provide different functionalities depending if running in the context of logged-in user or not. Below solutions enables RP to determine current session status. 

### Polling Based Solutions 
Polling, in general, is a way to determine a status of an asset, by actively, cyclically, checking its value. In the context of OpenId Connect and session management, polling solutions are based on cyclic requests to IdP to gather latest user session status.

### check_session_iframe endpoint 
*check_session_iframe* endpoint is described in [OpenId Connect Session-related specs](https://openid.net/specs/openid-connect-session-1_0.html). It requires RP to load IdP code, which validates current session state.  

IdP, which supports this solution, returns a *session_state* parameter in response for valid request to IdP's */authorize* endpoint. Additionally, IdP stores a session cookie in user’s browser.

RP application launched in the browser, should include an iframe element that loads content from IdP’s */check_session_iframe* endpoint. Loaded content is a JavaScript function that validates if user session is active or not. Validation is based on *session_state* and *clientId* provided by RP as arguments. Function calculates expected session cookie value and compares it with actual session cookie stored in the browser. Once expected and actual session cookies are equal, function returns unchanged (which indicates active session), or changed.

IdP validation function can be executed periodically by RP, to achieve polling mechanism. OpenId Connect documentation recommends to load anther iframe directly from RP, that will be solely responsible for  triggering validation cyclically. 

In this solution Network traffic is very limited. Once IdP iframe is loaded, session can be validated using loaded JavaScript code only - with no IdP interaction. 

![check_session_iframe](/assets/img/article3/session_with_iframe.png)

Iframe based has a serious limitation in modern, security-oriented browsers. 

#### Limitation: Iframe and 3rd Party Cookies
OpenId Connect documentation stands the following: 

---
*Note that at the time of this writing, some User Agents (browsers) are starting to block access to third-party content by default to block some mechanisms used to track the End-User's activity across sites. Specifically, the third-party content being blocked is website content with an origin different that the origin of the focused User Agent window. Site data includes cookies and any web storage APIs (sessionStorage, localStorage, etc.).* 

---
For security reasons more and more browsers don’t allow cross domain cookies access. Since RP loads IdP iframe, it requires an access to the cookie that belongs to IdP.  

Following list shows some examples, how browsers approach third party cookies. 

* Safari – not supported 
* Google Chrome– planning to finish support in 2024 
* Firefox – third party cookies are blocked by default 

Some IdPs allows to configure a custom domain, common for RP and IdP. Having IdP and RP in the same domain makes cookie access possible, regardless 3-rd party cookies restrictions. For instance, IdP can be accessed at https://login.example.com while app is hosted on https://example.com. 

### /authorize endpoint polling 
[OpenId Connect specification](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest) describes */authorize* endpoint behavior which is used to initiate authentication flow (or authorization flow in the context of OAuth 2.0 in general). Authentication with OpenId Connect begins with an */authorize* endpoint call. Authorize endpoint accepts *prompt* parameter. In case *prompt* parameter set to *none*, IdP first checks for session cookie, provided within the request. If attached cookie is validated as active session, user will not be asked to enter credentials. If provided session cookie is invalid or associated user session is no longer active user will be asked to enter login credentials. 

In the context of authentication */authorize* endpoint call initiates grant flow which eventually returns *id_token* to application. By calling */authorize* endpoint cyclicaly, application is updated with latest user-related claims, stored in id_token. It can be an advantage, when other application modifies user claims in the meantime. 

Some IdPs providers, supports *rolling session*. Session expiration time can be extended every time, when related session cookie is used to acquire new id_token. It allows to keep the session alive longer, when it is actively utilized by RP. 

#### Limitation 1: Limitation: Iframe and 3rd Party Cookies 
In order to validate user session IdP requires an access to session cookie stored in browser. Considering strict 3-rd party policy, it might not be possible, when IdP was called from other domain iframe. 

Some IdPs allows to configure a custom domain, common for RP and IdP. Having IdP and RP in the same domain makes cookie access possible, regardless 3-rd party cookies restrictions. For instance, IdP can be accessed at https://login.example.com while app is hosted on https://example.com. 

#### Limitation 2: API Rate Limits 
Presented approach can be problematic in some polling scenarios, considering IdP API rate limits. If polling interval is too short API call limit can be reached. 

#### Limitation 3: Iframe and Clickjacking protection 
To provide seamless user experience */authorize* endpoint can be called using iframe. Thanks to iframe, user is not redirected to /authorize page and back on every call. Instead, call can be done "in the background"- without users notice. 

Iframe usage leads to some more security concerns than just 3rd party cookies, especially in the context of authentication and authorization. Iframe element can be used to perform clickjacking attack. Clickjacking is very well described in [OWASP](https://owasp.org/www-community/attacks/Clickjacking) and [Auth0](https://auth0.com/blog/preventing-clickjacking-attacks/) articles. To prevent the attack, a ***X-Frame-Options: DENY*** header can be added to http header, to make sure, browser will not allow to load the page inside iframe. IdPs usually protect pages which allows user interaction, by adding *X-Frame-Options: DENY* header. For instance, it is usually added to login form.  

As long as there is active user session, */authorize* endpoint can be successfully called from iframe. Once user session expires, IdP tries to redirect next */authorize* call to the login page. Since login page likely includes *X-Frame-Options: DENY* header and it is called from iframe, redirect will fail. 

As a workaround some IdP providers recommend following solution expressed in pseudo-code 
```csharp
try 
{ 
    CallAutorizeUsingIFrame() 
} 
catch 
{ 
    CallAuthorizeUsingRedirect() 
} 
```
In example above */authorize* endpoint is called using redirect, only when iframe call fails. Assuming failure is caused by expired session and attempt to display login page in iframe, user will be redirected to login page outside the iframe in catch block. After successful login new session is established and */authorize* can be called from iframe again. 

Besides that, some IdPs allows to configure *X-Frame-Options* header value if needed, however it should be considered as "unsafe mode". 

### Logout Based Solutions 
RPs use logout-based solutions to track active user session by listening on logout event.  RP provides an endpoint that can be called by IdP in case of logout event occurs in other application from SSO scope, which uses the same user session.  

RP logout is initiated by calling [*/logout* endpoint](https://openid.net/specs/openid-connect-rpinitiated-1_0.html#rfc.section.2) of IdP.  

IdP clears session cookie on logout request. Moreover, IdP can spread logout information to all other RPs using the same session. RP can be notified using 
* Front-Channel Logout 
* Back-Channel Logout 

Once logout procedure is finished user can be redirected to *post_logout_redirect_uri* endpoint of RP which triggered the logout. It can be set up in IdP configuration, per client registration. 

### Front-Channel Logout 
Front-Channel Logout is described in [OpenId Connect documentation](https://openid.net/specs/openid-connect-frontchannel-1_0.html). IdP tracks RPs which use current browser session. Once IdP receives a logout request from any of RPs in SSO scope, logout event is spread to other RPs (which uses the same session). It is done by calling RP’s *frontchannel_logout_uri*  endpoint via front channel. (RPs need to be configured with *frontchannel_logout_uri*  in IdP client registration properties). 

Other active RPs in SSO scope are recognized by IdP. IdP tracks applications session usage.  Based on session cookie, provided within /logout request it finds all applications related with current session and returns list of *frontchannel_logout_uris*. 

RPs can adjust their internal state and clean information associated with logged-in user, after receiving front-channel logout request. 

Front-channel logout mechanism utilizes iframes. IdP loads and calls a website containing iframes in response to */logout* request. Each iframe *src* corresponds with *frontchannel_logout_uri* of one RP. URIs are called by iframes in parallel. Once all RPs are notified IdP redirects user to *post_logout_redirect_uri*  specified in initial */logout* request parameters. 

![front-channel logout](/assets/img/article3/azure-b2c-front-channel-logout.png)

#### Limitation 1: Iframe and 3rd Party Cookies 
Main limitation of frontend logout solutions is, again, related with third-party cookies. Since *frontchannel_logout_uri* is iframed and called on IdP, there might be some limitations with access to RP’s cookies. 

#### Limitation 2: URI fragment component 
Additionally, *frontchannel_logout_uri* must not contains fragment component (like example.com#logout), which is natural for some Single Page Applications. 

### Back-Channel Logout
Back-Channel Logout is described in [OpenId Connect documentation](https://openid.net/specs/openid-connect-backchannel-1_0.html). Logout information is sent to all RPs using current browser session via back-channel. In response to */logout* request IdP creates a [*logout_token*](https://openid.net/specs/openid-connect-backchannel-1_0.html#LogoutToken) and sends it via POST call to RP’s */backchannel_logout_uri*. URI is associated with RP during client registration on IdP. 

*logout_token* should be validated by RP before its internal logout logic is executed. It is critical to compare *logout_token* properties (like *sub*) with current *id_token* and verify if logout request corresponds with currently logged in user. 

Once all RPs are notified IdP redirects user to *post_logout_redirect_uri*. 

Not all IdP server providers supports Back-Channel logout. If Back-Channel is supported, *backchannel_logout_supported* property is visible in IdP discovery endpoint.

