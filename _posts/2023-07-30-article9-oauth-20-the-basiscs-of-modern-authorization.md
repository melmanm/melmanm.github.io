---
layout: post
title: "OAuth 2.0 - the basics of modern authorization"
categories: misc
tags:
- OAuth2.0
- Authorization
- OAuth
cover-img: /assets/img/article9/token.png
---

Application wants to access your account! Permission requested – application would like to read your profile and calendar!
OAuth 2.0 is an authorization protocol that simplifies accessing user’s resources in modern applications.
This article explains into OAuth 2.0 protocol from application developer, presenting how modern applications provides effortless login, empowers APIs integration, boosting users trust while accessing their data.

> This article is created as written form of a presentation that I had a pleasure to present recently.

## Back in 2007
To understand the need for authorization framework, we need to go back to year 2007, when OAuth 1.0 was being created. Just to give you the best feeling I need to mention that 2007 was the year of IPhone 1 premiere, it was the year when Google get interested in Android OS, and mySpace was at its prime. 
Back then web applications strived to provide integration with 3rd party services.

For instance, facebook introduced a feature enabling users to invite all friends from their gmail contact list, by sending an invitation email to them all.
In order to do so, facebook required user to enter gmail credentials on facebook site. Then, facebook used these credentials to log into gmail on user behalf and send invitation emails. 
The way of how user authorized facebook to access user's gmail resources, is today considered as anti-pattern. In general, user never knows, if authorized service uses given credentials only in the way it promises (having gmail credentials services gets full access to user's account). Moreover, user cannot assume credentials are stored securely by authorized service.

## The need of authorization protocol
OAuth goal was to solve the issue of authorization. The main goals of modern authorization was to:
* Provide a process for end user to authorize third-party access to their resources without sharing their credentials.
* Provide a method for applications to access resources on behalf of user.
* Provide granular levels (scopes) of third-party application access to user resources, which user can consent to.

![oauth-goals](/assets/img/article9/oauth-goals.svg)


## What Authorization means?
**Authorization** focuses around an access to resources. Authorization process verifies if resources can be accessed or not. In case of authorization service does not consider user identity, just the access rights. Unlocking front door of our houses is good example of authorization. Doors are not verifying the identity of person who is trying to open it. The key is what is verified. Note that others can unlock the door by using the same key.

Authorization is often mistaken with **Authentication**, which focuses around the user identity. Authentication helps the service to understand who the user is.


## The authorization we are used to
Today OAuth becomes an industry standard. Users are getting used to modern authorization solutions.

For instance, recently I passed a certificate. I was awarded with digital badge, available on _credly.com_. Credly enables users to share their achievements on LinkedIn. After I chose this option, Credly redirected me to LinkedIn, where I needed to enter credentials and allow Credly to post the content on LinkedIn on my behalf.
![credly-linkedin](/assets/img/article9/consent-credly-linkedin.PNG)

Another example of OAuth implementation could be the Google Maps integration with Spotify. Google Maps can integrate with Spotify in order to pause the music for the moment when navigation is providing voice guidance. After integration is initiated in Google Maps Application, user is redirected to Spotify. Google is asking for allowance to perform following actions on Spotify on users behalf

![credly-linkedin](/assets/img/article9/consent-googlemaps-spotify.jpg)

## OAuth 2.0 Roles
OAuth 2.0 defines a handful of roles, taking part in the process of authorization:

### Resource owner
![resource-owner](/assets/img/article9/user.png)
An entity able to grant an access to resources. Usually user.
### Resource server
![resource-server](/assets/img/article9/resource-server.png)
Server containing protected resources owned by resource owner. Able to accept the resource request using provided access token.
### Access Token *
![access-token](/assets/img/article9/token.png)
Representation of authorized access to specific resources. Describes access level to specific resources.
> **_NOTE:_** According to specification token is not OAuth 2.0 role, however it was included here for better understanding of OAuth roles.

### Client
![client](/assets/img/article9/client.png)
Application making requests for protected resources on behalf of resource owner. Access token is opaque to the client. Client retrieves the access token and uses it to access to resource server but it does not validate or check its content.
It is important to note that OAuth 2.0 distinguish two types of clients:
* Confidential - able to store securely store a secret, e.g. classic web application which backend code is running in secure environment.
* Public - not able to securely store a secret, e.g. SPA web application, which js code can be inspected, or desktop application, which can be decompiled and inspected.

### Authorization Server
![authorization-server](/assets/img/article9/authorization-server.png)
Server that issues access tokens to client.

Depends on application it is possible that resource server and authorization server are provided by the same party. OAuth 1.0 did not distinguish these roles, but OAuth 2.0 becomes more flexible by separating these roles. There are multiple use cases, where authorization server and resource server are provided by different parties.

## Authorization Code Grant 
Authorization Code Grant allows third party application to securely access users resources on their behalf.

![authorization-code-grant](/assets/img/article9/authorization-code-grant.png)

Entire flow consists following steps:
1. User initiates authorization flow in client application.
2. Client redirects user to authorization server providing following parameters in http call
   ```
   https://{authorization_server}/authorize?
   client_id={client_id}
   &redirect_uri=https://{client_url}
   &response_type=code
   &scope={access_scope}
   ```

   | parameter | description |
   | :--- | :--- |
   |`client_id`| represents unique client identifier that is assigned to the client during registration process (which will be described later in this article) |
   |`redirect_uri`| specifies an address (client address) to which user will be redirected to after authentication and consent on authorization server. **For security reasons redirect uri should be restricted only to approved uris, which belongs to client applications** Approved uris can be configured during client registration process. |
   |`response_type`| it specifies the way of authorization server response. In case of Authorization Code Grant it must be `code` |
   |`scope`| defines the resources and access level which client application is trying to access. For instance if client application tries to get readonly access to user gmail email the scope is https://www.googleapis.com/auth/gmail.readonly. Scope can include multiple, space delimited, entries. |
   
3. User authenticates to authorization server and allows/denies access to the resources defined by `scope`.
4. Authorization server generates authorization code and redirects to redirect_uri with authorization code as a parameter
5. Client application sends the authorization code to the authorization server in order to exchange it for a token
   ```
   POST https://{authorization_server}/token?
   client_id={client_id}
   &client_secret={client_secret}
   &redirect_uri=https://{client_url}
   &grant_type=authorization_code
   &code={code}
   ```
   | parameter | description |
   |:--- | :--- |
   |`client_secret`| applies only for confidential clients, represents client password that is assigned for the client during registration process (which will be described later in this article)|
   |`redirect_ur`| needs to be identical as in initial authorization request |
   |`grant_type`| in case of Authorization Code Grant it must be  `authorization_code`|
   |`code`| authorization code received by the client in step 4. reflecting rights consent by the user in step 3.|
6. Authorization server responds with access token available in the response body
7. Authorization token can be used by the client to access user resources.

## Why Authorization Code Grant is so complex?
One could think the entire procedure is way to complex, considering the fact its only goal is to provide an access token to the client. Though its complexity secures the flow and makes it more robust. 
In Authorization code grant takes advantage of two separate communication channels.
### Front Channel
Front channel can be defined as communication interface between client and authorization server performed using redirection in browser navigation bar. Client application redirects user to authorizations server, where user can authenticate and consent. Authorization server redirects user back to the client application with authorization code.
#### Front channel is not secure medium. 
Front-channel communication involves user browser which is running in the environment, that client application has no control over. If user does not protect the environment communication can be sniffed. Moreover URL redirections are stored in browser history where can be inspected.
#### Front channel responsibility
Apart from its disadvantages, front channel is a very important building block in Authorization Code Grant. Since it is based on browser redirection, it ensures that actual user authenticates and gives consent. It enables authorization server to include as many authentication steps as needed, e.g. multi factor authentication or captha.
Redirects allows client to have non-routable address. It is especially important for mobile or desktop applications. For instance, client application can be hosted on `localhost`. If localhost is specified as `redirect_uri`, local machine will be able to receive authorization code, after authorization server redirects user to `localhost` (with authorization code).
## Back channel
Back channel is secure channel. It is server to server communication between authorization server and client application server. If we consider standard web application (e.g. written in ASP.NET) there is a backend code running on the server, and frontend code running in client browser. Backend server establishes secure communication with authorization server.
Thanks to back channel client application can securely provide a secret, in access token request, to authorization server, and securely receive an access token.

![authorization-code-grant-front-back-channel](/assets/img/article9/authorization-code-grant-front-back-channel.png)

## What about public clients?
Public clients has some limitations comparing with confidential clients.
1. **Public clients can't store client secret.** It is more the nature of public clients than limitation, however lack of client secret leads to some security tradeoffs. It is really important to mention that `client_secret` is an optional parameter in Authorization Code Grant, so in theory it can be applied to public clients, but 
2. **Usually no secure channel available for public clients**
3. **CORS (historically)**. Historically browser were not able to perform a cross origin request in js code. Due to CORS limitation, it was not possible to perform Authorization code grant flow. It requires HTTP POST request to authorization server located outside of client domain.

## Implicit grant
Considering these limitations, OAuth 2.0 introduces simplified way to gather access token - Implicit grant. Authorization code is not exchanged for access token in implicit grant. Instead, access token is passed directly as parameter in redirection to `redirect_uri`.
In order to initiate implicit flow, client needs to provide `response_type=token` in authorization request.

![implicit-grant](/assets/img/article9/implicit-grant.png)

### Implicit grant security tradeoffs
Implicit grant flow provides simplified way for the client to gather access token. Though, there are some security considerations
1. **Access token can be leaked.** Access token is provided to the client via front channel. It becomes easier to intercept in transit. Moreover, it is available in browser history. 
If public client application wants to reuse access token, it usually stores it in browser cookie, from where it could be potentially stolen. Authorization server should serve short lived tokens, which are valid only for short period to minimize the ri
2. **Malicious client obtains authorization.** Missing `client_secret` makes client application unable to proof its identity to authorization server. In order to prevent malicious client obtain access token, by using the same authorization server request as original client, server should validate `redirect_uri`. Redirection should be only possible to approved `redirect_uri`.
3. **CSRF attack with attacker token**. Attacker can prepare same url as authorization server `redirect_url` and inject own access token into it. After user is tricked into navigating to prepared url, client application can access resource server on behalf of attacker, not original user. To prevent this attack client application should include `state` parameter, with non-guessable value, in authorization request, and validate if `state` parameter is present with the same value in authorization server redirection parameters. This type of attack can be mitigated using PKCE.

> **_NOTE:_** PKCE is an extension to Authorization Code Grant. It is described outside of OAuth 2.0 specification in [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636). It is now recommended to use with both public and confidential clients

## Client registration
Client registration is a process o building trust between client and authorization server. During client registration it is required to provide following client information to authorization server:
* approved `redirect_uri`s
* client type - based on that authorization server should decide if `client_secret` should be associated with client
* any other information that identifies or describes client application, like name, image, legal terns etc.

> **_NOTE_:** The way of providing these information to authorization server is outside of OAuth 2.0 documentation scope. Client registration can be usually performed by client application developer. Some authorization servers supports dynamic registration approach so client can self-register.

From client application perspective registration process provides `client_id` and optionally `client_secret`. Moreover `redirect_uri`s should be limited to uris used belongs to client.

![client-registration](/assets/img/article9/client-registration.PNG)

## Access tokens
The format of access token is not specified in the scope of OAuth 2.0. The key information about access token provided in documentation is: 

> Access tokens are credentials used to access protected resources. An
access token is a string representing an authorization issued to the
client. The string is usually opaque to the client. Tokens
represent specific scopes and durations of access, granted by the
resource owner, and enforced by the resource server and authorization
server.

There are two general types of access tokens:
### Reference access token
Reference access can be consider as a database key, which is associated with granted permissions. Authorization server saves every generated access token, with corresponding permissions.
Generated token is passed to the client and eventually to resource server. Reference token is completely opaque to the client since it represents a reference to some data that client can't access.
Resource server needs to access the original database in order to get permissions, related with access token, and validate the client request.

![reference-token](/assets/img/article9/reference-token.PNG)

### Self contained access token
In contrast with reference tokens, self contained tokens are not pointing to permission related data. Self contained tokens contains the data.

![reference-token](/assets/img/article9/self-contained-token.PNG)

JWT (JSON Web Tokens) is the most popular implementation of self contained access token. JWT defines a structure and encoding of access token. It provides mechanism that protects token from being modified in transit.

## Other OAuth 2.0 grants
Besides Authorization Code and Implicit grants OAuth 2.0 proposes
* **ROPC (Resource Owner Password Credential)** - Allows client to gather user credentials and send it to authorization server to exchange it for a token. Because of security consideration ROPC is not recommended. It is excluded from OAuth 2.1 specification draft.
* **Client Credentials** - Used for client applications, which are not running in the context of user (e.g. deamon applications). Application exchanges `client_id` and `client_secret` for access token.

## Where to find more about OAuth
* OAuth 2.0 docs [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
* PKCE docs [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
* [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/draft-ietf-oauth-cross-device-security/)
* _OAuth 2.0 Simplified_ book by Aaron Parecki

