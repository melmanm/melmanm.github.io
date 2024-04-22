---
layout: post
title: "ASP.NET AddAuthentication and AddAuthorization explained"
categories: misc
tags:
- dotnet
- webapi
- OAuth
- .NET
- C#
- ASP.NET
cover-img: /assets/img/article15/cover-image.png
---

In this article I will describe the concepts of authentication and authorization in ASP.NET, and why it still can be confusing, even knowing authentication and authorization concepts.


## Table of Contents <!-- omit from toc -->
- [In general](#in-general)
- [AddAuthentication()](#addauthentication)
  - [Single and Multiple Authentication scheme](#single-and-multiple-authentication-scheme)
  - [Authentication scheme to protect the endpoint](#authentication-scheme-to-protect-the-endpoint)
- [AddAuthorization()](#addauthorization)
  - [Forbid and Challenge](#forbid-and-challenge)
    - [What happens if policy is not fulfilled?](#what-happens-if-policy-is-not-fulfilled)
    - [What happens if policy requires authenticated user, but there is none?](#what-happens-if-policy-requires-authenticated-user-but-there-is-none)
    - [What happens if endpoint has `RequireAuthorization()` extension method with no policy specified?](#what-happens-if-endpoint-has-requireauthorization-extension-method-with-no-policy-specified)

## In general

The difference between Authentication and Authorization is in general:
 * Authentication - process of determining user's (or client in general) identity
 * Authorization - process of determining if user's (or client in general) has an access to specific asset


## AddAuthentication()

ASP.NET authentication is based on, what is called, `scheme`. Scheme can be considered as a specific way of authentication; specific way of how user (or api client in general) proves its identity to ASP.NET application and how the application collects identity data related with the user (or api client in general).

For instance, ASP.NET MVC application, usually involves a user who proves his/her identity by passing some cookie or OpenIDConnect id token within the request to backend API. That is intuitive when it comes to the authentication.

However, in case of standalone ASP.NET Web API, authentication scheme is used to validate and collect some information related with the client who, calls the API. For instance, if the client application attaches OAuth access token within the request to Web API, authentication scheme is used to validate the token and collect data included in the token. This approach can be confusing, since OAuth access tokens are used by API client to prove, that client has **authorization** to access some assets (API endpoints in general). But even in such case, access token is validated using authentication scheme.

Now let's focus on the scheme itself. Each authentication scheme should have unique name. Moreover, it should provide an implementation of `IAuthenticationHandler` of. `IAuthenticationHandler` interfaces includes following methods

```csharp
namespace Microsoft.AspNetCore.Authentication
{
  public interface IAuthenticationHandler
  {
    Task InitializeAsync(AuthenticationScheme scheme, HttpContext context);

    Task<AuthenticateResult> AuthenticateAsync();

    Task ChallengeAsync(AuthenticationProperties? properties);

    Task ForbidAsync(AuthenticationProperties? properties);
  }
}
```

`AuthenticateAsync()` is the key method, which validates the user (or client) identity. It returns `AuthenticateResult` (Success/Failed/NoResult). In case of successful validation `AuthenticationResult()` needs to include user's (or client) identity related information, called `Claims` in `AuthenticationResult`. `Claim` is key-value pair, for instance `UserName:John`.

`Claims` collection is wrapped in `ClaimsIdentity`. Finally `ClaimsIdentity` is included in `ClaimsPrincipal` object and included into `AuthenticateResult`. Result of authentication and claims, if there are any, are attached to HttpContext.

Just to make easier to understand, let's map it to real life example:
* `ClaimPrincipal` is can be an user, a person who has many ways to proof his/her identity
* `ClaimIdentity` is a single way of how person can proof his/her identity, for instance by presenting ID, passport or driving license
* `Claim` single information included in `ClaimIdentity`. For instance, considering passport as `ClaimIdentity`, the claim could be `name:John`,`surname:Doe`,`Citizenship:Poland`,`NumberOfVisas:3`.

`AuthenticateAsync()` method is called in `AuthenticationMiddleware`, which is executed before client request reaches Web API's endpoint.

> NOTE: Before dotnet 7, it was required to explicitly register `AuthenticationMiddleware` by using `app.AddAuthentication()`. Starting from dotnet 7, framework adds the middleware for you, and `app.AddAuthentication()` it no longer required. The same applies for `app.AddAuthorization()`. See [documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/security?view=aspnetcore-8.0#enabling-authentication-in-minimal-apps) for details.


### Single and Multiple Authentication scheme 
In order to add authentication via schemes to ASP.NET project following extension method can be used;

```csharp
builder.Services
    .AddAuthentication("schema1") //schema1 is set as default authentication scheme
    .AddScheme<AuthenticationSchemeOptions, SomeAuthenticationHandler1>("schema1", options) //SomeAuthenticationHandler1 implements IAuthenticationHandler
    .AddScheme<AuthenticationSchemeOptions, SomeAuthenticationHandler2>("schema2", options); //SomeAuthenticationHandler2 implements IAuthenticationHandler
```

> Before dotnet 7, it was required to specify default authentication scheme name in `AddAuthentication` method parameter. Since dotnet 7, if there is only one registered schema, it is treated as default. (Read more in [documentation](https://learn.microsoft.com/en-us/dotnet/core/compatibility/aspnet-core/7.0/default-authentication-scheme)). **Authentication middleware verifies the user (or client) identity using only default authentication scheme.**

It is very important to be aware, that if there is no default scheme specified and there are more then one authentication schemes specified, AuthenticationMiddleware does not process incoming request by calling scheme's `AddAuthentication()` method.

### Authentication scheme to protect the endpoint

AuthenticationMiddleware is not designed to protect API endpoints. Of course, based on what was already said, (that `Result of authentication and claims, if there are any, are attached to HttpContext`). We could make use of `HttpContext` and design following API code:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication()
    .AddScheme<AuthenticationSchemeOptions, SomeAuthenticationHandler1>("schema1", options);

///...

var app = builder.Build();

=
///..

app.MapGet("/GetAll", Results<UnauthorizedHttpResult, Ok<string>> (HttpContext context) =>
{
    if (context.User?.Identity?.IsAuthenticated != true)
    {
        return TypedResults.Unauthorized();
    }

    return TypedResults.Ok("Ok");
});

app.Run();
```

But this is not how ASP.NET authentication mechanisms should be used. The proper way to control an access to API endpoints is authorization.

## AddAuthorization()

ASP.NET provides other useful extension method- `AddAuthorization()` to register authorization-related services, and `AuthorizationMiddleware` which checks if API caller has an access to endpoint. 

> Since dotnet 7 `AuthorizationMiddleware` is added by default and it is not required to use `app.UseAuthorization()`.

The idea of authorization is quite simple. Based of authentication result and collected claims, it is possible to create conditions to decide if caller can access specific endpoint. Combination of such conditions is called `policy`.
The simplest policy I can imagine can only check if API caller passed the authentication process. For instance:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication()
    .AddScheme<AuthenticationSchemeOptions, SomeAuthenticationHandler1>("schema1", options);

builder.Services.AddAuthorization(config =>
{
    config.AddPolicy("myPolicy", policy =>
    {
        policy.AuthenticationSchemes.Add("schema1");
        policy.RequireAuthenticatedUser();
    });
});

///...

var app = builder.Build();

///..

app.MapGet("/GetAll", () =>
{
    return TypedResults.Ok("Ok");
}).RequireAuthorization("myPolicy");

app.Run();
```

Specific policy can be apply to protect the the endpoint by using extension method `RequireAuthorization("policyName")`.
Client can access the endpoint only if all policy conditions are satisfied.

Seems straightforward, right?

### Forbid and Challenge

Let's make a slightly more complex

#### What happens if policy is not fulfilled?

The framework makes use of the `IAuthenticationHandler` (related with authentication scheme) and its `ForbidAsync` method. 
Yes! In case of failed authorization, ASP.NET uses authentication scheme handler to process it O.o. `ForbidAsync` usually returns just `Unauthorized` HTTP result code.

#### What happens if policy requires authenticated user, but there is none?

We can find ourselves in such situations due to two reasons
1. **There is no default authentication scheme specified.** In this case `AuthorizationMiddleware` just tries to perform `AuthenticateAsync()` method from `IAuthenticationHandler` , which corresponds with the authentication scheme.
1. **There is default scheme specified, but authentication in `AuthenticationMiddleware` failed.**
In this case `AuthorizationMiddleware` performs `ChallengeAsync` method from `IAuthenticationHandler`, which corresponds with the scheme. `ChallengeAsync` usually re-tries authentication, and returns `Unauthenticated` HTTP status code if it fails again.

#### What happens if endpoint has `RequireAuthorization()` extension method with no policy specified?
Then the framework requires user to be authenticated via default authentication scheme. So default authentication scheme needs to be specified. If there is no default authentication scheme following exception is thrown, and returned to the client with 500 Http status code:
`System.InvalidOperationException: No authenticationScheme was specified, and there was no DefaultChallengeScheme found. The default schemes can be set using either AddAuthentication(string defaultScheme) or AddAuthentication(Action<AuthenticationOptions> configureOptions).`





