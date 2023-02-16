---
layout: post
title: "Azure AD B2C session management with MSAL and React.js - Part 1."
categories: misc
tags:
- Azure-AD-B2C
- OAuth
- Authentication
- React
---

In [previous article](https://melmanm.github.io/misc/2023/01/30/article3-openid-connect-session-management.html) I described session management possibilities provided by OpenId Connect. OpenId-Connect **session** represents the authenticated user context, maintained between Applications, running on users device, and Identity Provider Server. Vast majority of modern web application works in the context of logged-in user. OAuth-based solutions delegate identity management to external Identity Provider Server. From application perspective it is important to be synchronized with Identity Provider and be able to react on user session state changes. It becomes even more relevant, if multiple applications operate in the context of the same user session (SSO solutions).  

In this article I will describe Azure AD B2C-specific approach to session management in SSO environment. As the article goal is to inspect session management from developer perspective, I will refer to the code samples. Code samples originate from React.js SPA application, supported by MSAL.js library.

---
*Full application code is available on my [GitHub](https://github.com/melmanm/react-js-azure-b2c-session-management-sample).*  

---

## Table of contents
- [MSAL](#msal)
- [SSO- Single sign-on and Single sign-out](#sso--single-sign-on-and-single-sign-out)
- [Azure AD B2C supported session management methods](#azure-ad-b2c-supported-session-management-methods)
- [/authorize endpoint polling](#authorize-endpoint-polling)
  - [User Experience](#user-experience-)
  - [React to session expiration](#react-to-session-expiration)
  - [Third party cookies](#third-party-cookies)
  - [Azure AD B2C API rate limits](#azure-ad-b2c-api-rate-limits)
- [To be continued](#to-be-continued)

## MSAL
MSAL (Microsoft Authentication Library) is an open-source library, which enables developers to utilize OAuth 2.0 based services like Azure AD or Azure AD B2C. MSAL provides number of functions designed to execute OAuth 2.0 flows and acquire tokens. MSAL helps application developers with user management, by supporting login, logout, edit profile or change password user flows. MASL introduces the user context to the application. User context is stored in browser storage. It enables applications to maintain their own user-related cookies layer. 

MSAL is available on many platforms (MSAL.js, MSAL for Java, MSAL.NET and more) 

## SSO- Single sign-on and Single sign-out 
Single sign-on and single sign-out functionalities are important in the context of user session management. They allow to synchronize user session status across many applications launched in the context of the same browser. SSO improves user experience by bypassing frequent re-authentications. 

### Single sign-on
Azure AD B2C single sign-on (SSO) mechanism allows the same user session to be used by multiple applications. It provides great user experience, since user can be automatically signed in to multiple applications, after single authentication. It is possible, thanks to user session cookie, which is stored in user browser after successful login. Session cookie is sent to Azure AD B2C with subsequent requests, where can be recognized and validated. Once it is valid, Azure AD B2C assumes user session is active. In result login form is not displayed again.

### SSO scope
Azure AD B2C allows to configure applications that are allowed to share the same user session. More information on how to configure Azure AD B2C can be found in [Microsoft docs](https://learn.microsoft.com/en-us/azure/active-directory-b2c/session-behavior?pivots=b2c-custom-policy#configure-azure-ad-b2c-session-behavior). 

### Single sign-out
Single sign-out mechanism ensures that after user logs out from single application, other applications in SSO scope, which use the same user session, are notified. In response to notification, applications can drop the context of current user and redirect to login page, indicating current user is logged out and authentication is required. 

## Azure AD B2C supported session management methods 
Below table shows which OpenId Session management solutions, described in [previous article](/_posts/2023-01-30-article3-openid-connect-session-management.md), are supported by Azure AD B2C: 


| Solution Type | Solution                          | Azure AD B2C support  |
| :---          | :----                             |:----|
| polling based | check_session_iframe endpoint     | No  |
| polling based | /authorize endpoint polling       | Yes |
| Logout based  | Front-channel logout              | Yes |
| Logout based  | Back-channel logout               | No  |

## /authorize endpoint polling
In order to determine user session status, application can cyclically call Azure AD B2C */authorize* endpoint. */authorize* endpoint is used to initialize OAuth flow. In the context authentication it is called with *scope=openid* parameter. As the result of the flow *id_token* is passed to the application. 

---
*MSAL.js always initializes PKCE grant flow to obtain id_token.* 
---

Once user authenticates with Azure AD B2C, session cookie is created in user’s browser. Azure AD B2C creates ***x-ms-cpim-sso:{tenantId}*** cookie with the value of user’s session id. Cookie is encrypted and can be interpreted solely by Azure AD B2C.

If Azure AD B2C user session cookie exists in user’s browser, it is automatically attached to every call to its origin, including subsequent */authorize* endpoint calls. In case attached session cookie is valid (not expired, not malformed, not revoked) PKCE flow is completed, without user being asked to enter credentials.  

*Note that each flow, results in fresh id_token being passed to the application. Application is always updated with latest user data.

To take advantage from */authorize* endpoint pooling, *prompt=none* parameter should be included in */authorize* request. Otherwise, login form will always be displayed, regardless the active session. 

Following diagram present initial authentication flow. During initial authentication user is asked to enter credentials. Next, session cookie is settled in user's browser.
![authorize pooling first call](/assets/img/article4/azure-b2c-authorize-pooling.png)

After initial authentication, application can start polling */authorize* endpoint (diagram presents behavior, when there is a valid session cookie available in the user browser) 

![authorize pooling subsequent call](/assets/img/article4/azure-b2c-authorize-pooling-subsequent-request.png)

### User Experience 
The common practice to improve user experience, is to embed */authorize* request in the iframe. Instead of redirecting the user to IdP site and back, iframe gives a feeling of a background call, which doesn’t interrupt user interaction with application. MSAL.js provides ***ssoSilent*** function to acquire *id_token* using iframe. 

Below there is a fragment of MSAL.js core implementation related with *ssoSilent* function 

```js
async ssoSilent(request: SsoSilentRequest): Promise<AuthenticationResult> { 
 const correlationId = this.getRequestCorrelationId(request); 
   //... 
 const silentIframeClient = this.createSilentIframeClient(validRequest.correlationId); 
 result = silentIframeClient.acquireToken(validRequest); 
 //... 
}
```

### React to session expiration
So far, we focused on happy path, assuming there is a valid user session cookie available in the browser. Once there is no active user session provided to Azure AD B2C, *ssoSilent* function call fails. 

In case session cookie is missing, expired or not valid, Azure AD B2C displays login form. In order to prevent clickjacking attack, Azure AD B2C doesn’t allow to display login form in iframe. It is indicated by X-Frame-Options: DENY http header, added by Azure AD B2C to login page. Clickjacking is very well described in [OWASP](https://owasp.org/www-community/attacks/Clickjacking) and [Auth0](https://auth0.com/blog/preventing-clickjacking-attacks/) articles.

Attempt to display login form in iframe using *ssoSilent* method ends with: 
```
InteractionRequiredAuthError: interaction_required: AADB2C90077: 
User does not have an existing session and request prompt parameter has a value of 'None'. 
```
This exception can be used as an indication of user session expiration. Exception handling block can redirect user to custom logout page or force to reauthenticate by redirecting to */authorize* endpoint (without iframe). Following example shows how to logout user from application when session no longer valid:

```js
const callSsoSilent = async () => { 
  try { 
    await instance.ssoSilent(silentRequest);
  } catch (e) { 
    if (e instanceof InteractionRequiredAuthError) { 
      instance.logoutRedirect({ 
        account: instance.getActiveAccount(), 
        //prevents from notifying a server about logout, logout is performed only locally - in applicaiton scpoe 
        onRedirectNavigate: false 
      }); 
    } 
    else{ 
      //handling of other error 
    } 
  } 
} 
```

### Third party cookies
*/autorize* endpoint polling solution is dependent on third-party cookie access. Azure AD B2C, requested in iframe from application domain, needs to access session cookie. In this context, it is considered as third party cookies. 

For security reasons, some browsers implement strict policies related with third-party cookies access. To ensure access to session cookie is possible, Azure AD B2C tenant can be configure to be available in the same domain as application. For example `https://login.example.com` while app is hosted on `https://example.com`. Now, session cookie is not considered as third party, since it originates from the same domain. It can be achieved using [Azure Front Door](https://learn.microsoft.com/en-us/azure/active-directory-b2c/custom-domain?pivots=b2c-custom-policy). 

### Azure AD B2C API rate limits
It is important to consider increased network traffic and possible latency in polling implementation. Every polling-based strategy should consider API rate limits, to ensure they are not exceeded. Azure AD B2C limits requests in contexts of tenant and IP.

|Measure | Limit |
|:--- | :---|
|Maximum requests per IP per Azure AD B2C tenant | 6,000/5min |
| Maximum requests per Azure AD B2C tenant | 200/sec |


## To be continued
In next article I will focus on **front-channel logout**.

