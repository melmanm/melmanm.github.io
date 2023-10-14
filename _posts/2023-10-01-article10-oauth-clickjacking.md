---
layout: post
title: "OAuth clickjacking attack (with example)"
categories: misc
tags:
- OAuth2.0
- vulnerability
- hacking
- OAuth
cover-img: /assets/img/article10/html-page-opacity50.png
---

In OAuth clickjacking attack the attacker tricks the user into authorizing malicious application to access user's resources. Clickjacking attack is usually hard to notice by the user, though it can be very harmful. 

> NOTE: This article is created with an intention to increase users and developers awareness of clickjacking attack, including its technique, consequences and best mitigation practice.

## Table of contents <!-- omit from toc --> 
- [Introduction](#introduction)
- [Iframe](#iframe)
- [How the attack is performed](#how-the-attack-is-performed)
  - [Malicious application](#malicious-application)
  - [Html page](#html-page)
  - [Trick the user to authorize malicious application](#trick-the-user-to-authorize-malicious-application)
- [Clickjacking attack mitigation](#clickjacking-attack-mitigation)
  - [The user perspective](#the-user-perspective)
  - [Authorization Server](#authorization-server)
  - [X-Frame-Options HTTP header](#x-frame-options-http-header)
- [Sometimes iframe is intended](#sometimes-iframe-is-intended)
- [About the examples](#about-the-examples)


## Introduction
Clickjacking, in general, is not OAuth specific attack. Clickjacking is performed by tricking the user to click invisible/transparent element, which performs malicious action, like downloading malware or performing CRSF attack. Website, used for attack, is designed to trick the user to click into the malicious element, by displaying other content under malicious, transparent element, which is fabricated to convince user to click it.

![clickjacking](/assets/img/article10/clickjacking-img.png)

In this article I will present how clickjacking can be used to abuse OAuth-based flow. OAuth clickjacking attack is usually performed to obtain user's authorization to access protected resources by malicious application.

## Iframe
Iframe is main OAuth Clickjacking attack vector.

> The iframe HTML element represents a nested browsing context, embedding another HTML page into the current one.
![iframe](/assets/img/article10/iframe.png)
source: [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe)

The example above shows how iframe can be used to nest html document originating from external website in original html document.

## How the attack is performed

### Malicious application
First, the attacker prepares malicious application. Application is designed to obtain user's authorization to access user's protected resources, e.g. user mailbox account, personal information, file storage. Next, the attacker [registers](https://melmanm.github.io/misc/2023/07/30/article9-oauth-20-the-basiscs-of-modern-authorization.html#client-registration) malicious application at the Authorization Server.

![clickjacking-registration](/assets/img/article10/clickjacking-registration.png)

### Html page
Attacker prepares hmtl page, with an intention to trick the user into authorizing malicious application.
Html page includes iframe element, which `src` points at the authorization server's `/authorize` endpoint uri. `/authorize` endpoint is used to initiate the authorization flow.

This attack can be successful if user was already authenticated to the Authorization Server. When active login session is already established, Authorization Server usually does not require user to re-authenticate. 

Once above precondition is met, iframe prepared by the attacker instantly loads the consent form, where user is asked for authorizing malicious application application.

![clickjacking-consent](/assets/img/article10/clickjacking-consent.png)

### Trick the user to authorize malicious application
Html page is designed to make the iframe with the consent form transparent to the user and display some other elements underneath, to convince the user to click into `authorize` button.

Let's consider following html page which could potentially be used to perform clickjacking attack:
![html-page](/assets/img/article10/html-page.png)

Attacker designs the website to convince the user to click in `click here to get an award` button.

The html source of the page is:

```html
<h1 id="header">
    Congratulations! You won an Iphone!
</h1>

<button id="button">click me</button>

<div id="arrow">
    <span>&#8594;</span>
</div>
<div>
    <button id="button">Click here to get an award</button>
</div>


<iframe 
    id="iframe" 
    src="https://<authorization_server>/authorize?client_id=<client_id>&redirect_uri=<redirect_uri>&scope=<scope>&response_type=<token/code>" 
    frameBorder="0" 
    height="100%" 
    width="100%">
</iframe>

<style>
    #iframe {
        opacity: 0%;
    }
    /*other styles were omitted for simplicity*/
</style>
```

The `opacity` of the iframe element, which covers all other elements, is set to `0%`, what makes it invisible.
Let's change the opacity to `50%`:

![html-page-opacity50](/assets/img/article10/html-page-opacity50.png)

Now the top layer of the website - the iframe with the consent form, which normally was invisible, is revealed.

## Clickjacking attack mitigation
### The user perspective
Clickjacking attack can be very hard to detect by the user. Of course users can inspect html of every website they visit, but normally nobody should be required to do that. 
### Authorization Server
Protecting users against OAuth clickjacking attack is the Authorization Server responsibility. Authorization server can prevent loading html pages it returns to the user browser in the iframe element. It can be achieved by adding `X-Frame-Options` header to the HTTP responses, which include html pages. The `X-Frame-Options` HTTP response header can be used to indicate if browser should be allowed to render a page in a `iframe` element and eventually prevent clickjacking attack.

### X-Frame-Options HTTP header
`X-Frame-Options` header can take following values:
* `X-Frame-Options:` `DENY` - page cannot be displayed in iframe
* `X-Frame-Options:` `SAMEORIGIN`- page can be displayed in iframe only if iframe source and original website originates have the same origin.

## Sometimes iframe is intended
In some scenarios, especially when OAuth is used with OpenId - for authentication purposes, it is an application developer intention to include the login page in the iframe. Below there is an example of such design.

![login-iframe](/assets/img/article10/iframe-login.png)

Because of that some authorization servers leave the configuration of `X-Frame-Options:` header value to the application developer.

However, since clickjacking attack is very hard to detect and can be very harmful for the user, majority of Authorization Servers do not enable application developer to configure `X-Frame-Options:` header value. `X-Frame-Options: DENY` header is always included in the Authorization Server's HTTP responses, which include html pages (especially login and consent page).

## About the examples
Examples presented in this article was created using Auth0 as an Authorization Server. As for now Auth0 supports two models of user's login and authorization experience. New - `New Universal Login`, and old (no new features added) `Classic Universal Login`. `New Universal Login` [(read more)](https://auth0.com/docs/authenticate/login/auth0-universal-login/new-universal-login-vs-classic-universal-login). `X-Frame-Options:` header value can be configured by application developer only in `Classic Universal Login` mode, using `Settings->Advanced->Migrations->Disable clickjacking protection for Classic Universal Login` option [(read more)](https://auth0.com/docs/troubleshoot/product-lifecycle/past-migrations/clickjacking-protection-for-universal-login).

![auth0-clickjacking-protection](/assets/img/article10/auth0-clickjacking-protection.png)
