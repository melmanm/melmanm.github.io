---
layout: post
title: "Entra ID (Azure AD) app registration - client secret vs certificate. What to choose?"
categories: misc
tags:
- OAuth2.0
- Azure AD
- Entra ID
- Active Directory
- OAuth
- client_secret
- certificate
cover-img: /assets/img/article12/cover.png
---

In this article I will compare two methods, which can be used by this client application prove its identity to Entra ID (Azure AD) in the process of OAuth authorization.

## Introduction

In the process of OAuth authorization client application can prove its identity to Authorization Server (e.g. Entra ID) by presenting some secret asset (which only the client application can be in possession of). From now on lets call this secret asset **client credential**.

Client credential enables Authorization Server to ensure it communicates with legitimate application. Additionally, it minimizes the risk of client impersonation attack, where malicious application obtains the access token, by presenting stolen client credential to Authorization Server. 

Note that the client credential can be used only by the applications running in the environments, where it is possible to keep the it secret.

Client credential should **not** used by the applications running on devices under end-users control (e.g. desktop application, or mobile application). Even if client credential is well hidden or encrypted on end-user device, end-user can always listen the outgoing communication to the Authorization Server and intercept it.

After this introduction let's take a look on how client secret and certificate can be used by client application as client credential.

## Client credential in Entra ID App registration

Client credential can be configured in App registration `Certificates and secrets` blade.

![azure-ad-client-credentials](/assets/img/article12/azure-ad-client-credentials.png)

## client_secret as client credential

The idea of `client_secret` is very straightforward. Authorization Server (Entra ID) generates a secret (password) for specific App registration. Next, application developer or owner embeds the secret in application code (or in application environment from application can access it) so application can provide it to Authorization Server within access token request. If provided secret matches the initially generated secret, Authorization Server returns the access token to the application.

## Certificate as client credential

Alternatively, a certificate can be used to prove the client identity to the Authorization server. The idea of certificate is based on asymmetric cryptography, which utilize public and private key pair. 

First we need a X.509 certificate with private key.

> For production application certificate should be signed by TA (trusted authority). For testing it can be self-signed certificate.

Public certificate key should be uploaded to Entra ID under specific app registration, while private key should be available for application. 

### Client assertion
In the process of authorization application generates an object called `client assertion`, which will be provided to Authorization Server within token request (instead of `client_secret`). `client assertion` is formatted as JWT (json web token). For instance

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "x5t": "gx8tGysyjcRqKjFPnd7RFwvwZI0"
}
.
{
  "aud": "https: //login.microsoftonline.com/contoso.onmicrosoft.com/oauth2/v2.0/token",
  "exp": 1484593341,
  "iss": "97e0a5b7-d745-40b6-94fe-5f77d35c6e05",
  "jti": "22b3bb26-e046-42df-9c96-65dbd72c1c81",
  "nbf": 1484592741,
  "sub": "97e0a5b7-d745-40b6-94fe-5f77d35c6e05"
}
.
"SIGNATURE"
```
_example form https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials#example-of-a-decoded-jwt-assertion_

`client assertion` includes
* `aud` - audience; Authorization Server assertion is created for
* `exp` - expiration; The time when assertion expires and should not be accepted by Authorization Server
* `iss` - issuer; application id

### Signature

How the `client assertion` proves the application identity? By JWT signature. `client assertion` is signed by the private key using RS256 algorithm (specified in the JWT header). If you are not cryptography geek, you can consider RS256 signature as a hash of the token payload, where hashing algorithm takes private key as an input parameter.

Thanks to RS256 (and asymmetric cryptography in general) it is possible to use public key to check if signature was created using corresponding private key. This is the validation which Entra ID performs once it receives the `client assertion`. Once validation passes, Entra ID can assume all information included in `client assertion` payload originate form legitimate application.

## Security

When it comes to security certificate and `client assertion` is way more secure than `client secret`. There are two main reasons:

* `client_secret` is transmitted through the wire within the token request. Once communication is intercepted by the attacker, `client_secret` can be easily extracted, and re-used by the attacker to obtain access tokens, until `client_secret` expires. Certificate is never exposed in communication.
* `client assertion` signed using certificate is short-lived, its expiration time can be defined in `client assertion` payload. In contrast `client secret` can be valid up to 2 years. Even if `client assertion` is intercepted, short expiration time limits the risk of its re-usage.

Because of that, Microsoft recommends using certificates over `client_secret`. 

> _We strongly recommend that you use x509 certificates issued by Trusted Certificate Authority as the only credential type for getting tokens for your application._
> 
> ~ https://devblogs.microsoft.com/microsoft365dev/client-secret-expiration-now-limited-to-a-maximum-of-two-years/
