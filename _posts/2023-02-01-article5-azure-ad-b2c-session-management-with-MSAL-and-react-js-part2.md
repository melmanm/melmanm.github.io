---
layout: post
title: "Azure AD B2C session management with MSAL and React.js - Part 2."
categories: misc
tags:
- Azure-AD-B2C
- OAuth
- Authentication
- React
---

This article continues the series, related with session management solutions in Azure AD B2C. [Previous post](https://melmanm.github.io/misc/2023/01/31/article4-azure-ad-b2c-session-management-with-MSAL-and-react-js-part1.html) outlined polling-based approach, which can be used to determine session status in SSO scope. Today I will focus on **front-channel logout**.  

As the article goal is to inspect session management from application perspective, I will refer to the code samples. Code samples originate from to React.js SPA application, supported by MSAL.js library.

---
*Full application code is available on my [GitHub](https://github.com/melmanm/react-js-azure-b2c-session-management-sample).*  

---

## Table of contents
- [Front-channel logout](#front-channel-logout)
- [Configuration of Azure AD B2C](#configuration-of-azure-ad-b2c)
- [MSAL and React.js configuration](#msal-and-reactjs-configuration)
- [Third party LocalStorage access](#third-party-localstorage-access)


## Front-channel logout
Front-channel logout idea is based on user logout notification, broadcasted by Azure AD B2C to all applications sharing the same user session. Usually, in response to the notification, application drops current user context and redirect user to login page (to indicate re-authentication is required).

Front-channel logout procedure starts after single application in SSO scope initiates [OpenId Connect logout procedure](https://openid.net/specs/openid-connect-rpinitiated-1_0.html), by sending a request to Azure AD B2C */logout* endpoint.

```
GET <tenantname.b2clogin.co/<tenantname>.onmicrosoft.com/<policy_name>/oauth2/v2.0/logout 
?post_logout_redirect_uri=<redirect uri, application will be redirected to this uri after logout once finishes> 
&id_token_hint=<id_token_hint> 
```
Below diagram presents the procedure of front-channel logout in SSO environment.

![front channel logout](/assets/img/article5/azure-b2c-front-channel-logout.png)

After */logout* request, Azure AD B2C recognizes all applications using user session, that corresponds with provided session cookie. Next, Azure AD B2C creates a list of their front-channel logout URLs.

In response to */logout* request, B2C returns a page, which loads an iframe. Iframe includes JavaScript logic calling all front-channel URLs, listed by Azure AD B2C in previous step. 

Finally, user is redirected to  *post_logout_redirect_uri* specified in the original request. 

## Configuration of Azure AD B2C 
To enable front-channel logout in Azure AD B2C, following steps needs to be done 

1. **Front-channel logout URL** has to be configured for applications registrations. Front-channel logout setting is available under *App registrations->application name->authentication*. URL needs to be HTTPS for security reasons.  Additionally, URL cannot contain fragment component (like `https://example.com#logout`).
![front-channel URL](/assets/img/article5/azure-b2c-front-channel-logout-configuration.jpg)

2. **Custom Policy Configuration**
    `Note that front-channel logout is supported with Custom Policies only.`

    Custom policy *UserJourney* needs to be decorated with *UserJourneyBehaviors*.
    *  Scope of *SingleSignOn* can be set to **Tenant**, **Application** or **Policy**. 
    * *EnforceIdTokenHint* option is required, so the current user context is always passed within /logout request to Azure AD B2C. Note that front-channel logout is supported only with Custom Policies. 
    ```xml
    <UserJourneyBehaviors> 
        <SingleSignOn Scope="Tenant" EnforceIdTokenHintOnLogout="true"/> 
    </UserJourneyBehaviors> 
    ```

## MSAL and React.js configuration
React.js & MASL application requires following configuration

1. **MSAL configuration** – MSAL configuration is expressed in JSON format. It is used to instantiate MASL application instance.
    * *allowRedirectInIframe: true* – application needs to be reached from iframe, rendered in Azure AD B2C tenant domain. This property ensures *X-Frame-Options: DENY* is not added in http header. 
    * *cacheLocation: "localStorage"* - MSAL stores user related information in browser storage. 
    ```js
    const msalInstance = new PublicClientApplication({  
    auth: {  
         //auth config  
    }, cache: {  
         cacheLocation: "localStorage",   
    }, system: {  
         allowRedirectInIframe: true  
    }})
    ```
2. **Logout endpoint** – application has to provide a logout endpoint, at the URL corresponding with Azure AD B2C registration. React.js framework is not natively prepared for handling routing, however by using React-router, it is possible to serve specific content on */logout* endpoint. Using *BrowserRouter* component, */logout* request can be routed to specific application component.
    ```xml
    <BrowserRouter>
        <Routes>
            <Route path="/logout" element={<Logout />} />
            <Route path="/" element={<Home />} />
        </Routes>
    </BrowserRouter>
    ```
    Logout component should not contain much logic. It needs to perform well. Note that once user logs out from other application in SSO domain, component is loaded using front-channel request. The longer it takes to load, the longer user waits for Azure AD B2C to complete original logout request. Logout component can be designed as follows:
    ```jsx
    import { React } from "react"; 
    import { useMsal } from "@azure/msal-react"; 
    
    export function Logout() { 
        const { instance } = useMsal(); 
    
        instance.logoutRedirect({ 
            account: instance.getActiveAccount(), 
            onRedirectNavigate: false 
        }); 
    
        return ( 
            <div>Logout</div> 
        ) 
    }    
    ```
    Setting *onRedirectNavigate: false* ensures that only local logout will be performed. Once onRedirectNavigate is set to true, local logout is followed by calling */logout* endpoint of Azure AD B2C.

3. **Handle account storage events** – MSAL stores its internal status, including current user related data, in browser storage (localStorage – according to configuration). Since the same application can be opened in many browser tabs/windows, MSAL has to react on changes in localStorage entries. In result all tabs/windows with the same application opened, can synchronize current user context. This mechanism can by following function:  
    ```js
    msalInstance.enableAccountStorageEvents(); 
    ```
    **Handling storage events is also critical in the context of front-channel logout**. After application’s */logout* endpoint is called by Azure AD B2C iframe, user context is removed from localStorage. If application is opened in different browser tab or window, it can to be notified about this change.

## Third party LocalStorage access 
Application logout endpoint, loaded from Azure AD B2C iframe, needs to access localStorage. Unless Azure AD B2C is configured using [Azure Front Door](https://learn.microsoft.com/en-us/azure/active-directory-b2c/custom-domain?pivots=b2c-custom-policy), to be available from the same domain as application, it can lead to some difficulties.

Some browsers, controls localStorage access using the third-party cookies settings. Once it is disabled, localStorage cannot be accessed from website iframed in a different domain. Since more and more browsers restricts cross domain access to storage, it needs to be considered.
![3rd party cookie](/assets/img/article5/google-cookies.jpg)
