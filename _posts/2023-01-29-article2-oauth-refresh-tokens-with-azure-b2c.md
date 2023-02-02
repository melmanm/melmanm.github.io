---
layout: post
title: "OAuth refresh tokens with Azure AD B2C"
categories: misc
tags:
- Azure-AD-B2C
- OpenId-Connect
---

Refresh tokens are commonly used in OAuth based authorization scenarios. The purpose of refresh token is to retrieve new id/access token from authorization server, without user interaction. In simple scenarios once access token expires user is forced and reauthenticate to Auth server to get new token. Using refresh tokens approach, expired access token can be replaced with fresh one in the background, without user interaction. 

Using refresh token improves application security. One might think it would be just enough to extend an expiration time of id/access token. By making it long lived, user avoids often reauthentication. This approach may put your application at greater risk of token being intercepted and used by attacker. 

Once id/access token is retrieved from Auth server, it can be used until it expires against resource server (e.g. personal calendar service). Risk of token interception and malicious usage increases as the same long-lived id/access token is used over and over again.  

![wrong way](/assets/img/article2/azure-b2c-refresh-token-access-token-usage.png)

With short-lived access token, risk of its interception still exists, however their maliciues usage can be minimized as they shortly expire and cannot be used against resource server anymore. In Azure AD B2C default access token lifetime is 60 minutes and can be configured in a range of 5 minutes to 24 hours. 

Besides external risks long-lived access tokens, expose resources to internal risks. Once user access scope to specific resource is changed or revoked by Auth server administrator, it will be reflected in access token generated after this change. With long lived access tokens user might keep the access to the resource for long time after it was changed or revoked. 

## Table of contents
- [Refreshing tokens by the book](#refreshing-tokens-by-the-book)
- [Refresh tokens settings in Azure AD B2C](#refresh-tokens-settings-in-azure-ad-b2c)
    - [User Flows](#user-flows)
    - [Custom Policies](#custom-policies)
- [Refresh token revocation](#refresh-token-revocation)
    - [Revocation and access tokens](#revocation-and-access-tokens)
    - [Azure AD B2C revocation scenarios](#azure-ad-b2c-revocation-scenarios)
    - [One time use refresh token](#one-time-use-refresh-token)
- [Customizing refresh token flow](#customizing-refresh-token-flow)

## Refreshing tokens by the book
Below diagram presents how refresh token can be used in Authorization Code Grant flow. 
![refresh token](/assets/img/article2/azure-b2c-refresh-token-authorization-code-flow.png)

In order to get refresh token from authorization server following request needs to be sent to /token endpoint 

```
POST https://{tenant_id}.b2clogin.com/{tenant_id}.onmicrosoft.com/{policy}oauth2/v2.0/token 
?grant_type=refresh_token 
&client_secret={client_secret} 
&client_id={client_id} 
&scope=offline_access //in addition, openid or specific access can be specified 
&refresh_token={refresh_token} 
```

Auth server response includes new refresh token plus access or id token (depending of specified request scope).  

---
**Note 1**

*Authorization code grant flow is presented as example. Note that refresh token can be used in OAuth flows other than that, except implicit grant flow.*

---
**Note 2**

*Refresh token is encrypted during generation, and decrypted by Auth server once it is used. Its payload is transparent for client application and can be understood solely by Auth server.*

---

## Refresh tokens settings in Azure AD B2C 
Azure AD B2C governs refresh tokens and controls their behavior. Refresh token can be configured using 3 properties 

* ***refresh_token_lifetime_secs*** – describes how long single refresh token is valid. Once refresh token lifetime expires it cannot be used to gather new refresh token and will be refused by Auth server. Minimum value is 24h, maximum is 90 days.
* ***rolling_refresh_token_lifetime_secs*** – describes time frame in which token refreshing can be continued. Note that refresh token can be used to get new refresh token. New refresh token can be used to get another refresh token creating kind of chain. This property specifies maximum time of refresh token chain. Minimum value is 24h, maximum is 365 days. 
* ***allow_infinite_rolling_refresh_token*** – once this property is set to true, refresh token chain is not time limited.  

These properties can be configured for both User Flows and Custom Policies (Identity Experience Framework).

#### User Flows
User Flows allows to configure refresh token properties under Settings/Token Lifetime 
![refresh token](/assets/img/article2/azure-b2c-refresh-token-settings-user-flow.jpg)

#### Custom Policies
Custom Policies can take advantage of extending OpenId Connect *TechnicalProfile*. Azure AD B2C custom policies [starter pack](https://learn.microsoft.com/en-us/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#custom-policy-starter-pack) implements *TrustFrameworkBase* custom policy, which defines *JwtIssuer* - OpenId Connect *TechnicalProfile*. It can be inherited and extended with refresh token settings like below
```xml
<ClaimsProvider>
    <DisplayName>Token Issuer</DisplayName>
    <TechnicalProfiles>
        <TechnicalProfile Id="JwtIssuer">
            <Metadata>
                <!-- 86400s = 24h -->
                <Item Key="refresh_token_lifetime_secs">86400</Item>
                <!-- 172800s = 48h -->
                <Item Key="rolling_refresh_token_lifetime_secs">172800</Item>
                <Item Key="allow_infinite_rolling_refresh_token">false</Item>
            </Metadata>
        </TechnicalProfile>
    </TechnicalProfiles>
</ClaimsProvider> 
```
Refresh token lifetime configuration is reflected in response from /token endpoint. Response includes *refresh_token_expires_in* property that corresponds with *refresh_token_lifetime_secs* setting.  Once end application receives this information, it can request new refresh token before current expires. 

## Refresh token revocation
 Azure AD B2C does not provide OAuth */revocation* endpoint which is normally used to inform the Auth server specific token should not be accepted anymore.  

Token revocation can be achieved in three different ways using
##### 1. Using Powershell 
```
Revoke-AzureADUserAllRefreshToken -ObjectId <String> 
```
##### 2. Using Azure portal
 *Revoke session* button is available in B2C tenant user settings (Handling of token revocation, after using this option, requires additional implementation. It is described in further section).
 ![refresh token](/assets/img/article2/azure-b2c-refresh-token-revoke-in-azure-portal.jpg)

##### 3. Using Graph API
 ```
 POST https://graph.windows.net/myorganization/users/{user_id}/invalidateAllRefreshTokens?api-version 
 ```
 Graph Api call requires caller to have User.ReadWrite permission. Azure AD B2C does not support delegated (sign in user-related) User.ReadWrite permission. From the other hand User.ReadWriteAll permissions can be granted as Application permission, however since it is not related with application sign in user, application will be entitled to read or write all users data.  This solution is rather suited for administrator usage rather than for application to revoke the token on behalf of signed in user. 

---
#### Revocation and access tokens
Revocation does not invalidate the access token. User does not lose the access immediately. Access tokens are valid until expiration. Once application gets access token, there is no way to stop the application from accessing resource with this access token.
Revocation prevents from acquiring new id/access tokens without reauthentication.  

#### Azure AD B2C revocation scenarios
Azure AD B2C token revocation possibilities seem to be designed for administrator usage scenarios. For example, when user loss device, administrator can revoke tokens, so reauthentication will be required.  

This approach doesn’t work well with scenarios, where users need to revoke their own tokens. However, it is valid requirement considering security reasons. After user logs out from application, refresh tokens are usually removed from application memory but they are not invalidated on Auth Server and still can be used. It creates a risk of malicious usage of intercepted refresh tokens. 

#### One time use refresh token 
Staying in the context of security, there are [recommendation](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) for Auth servers to generate one time use refresh tokens. It means refresh token can be used only once to request new access and refresh token pair. New refresh token can be used only once as well. This approach mitigates refresh token usage risk, even if it is intercepted by attacker. However, Azure AD B2C allows to use the same refresh token multiple times. 

## Customizing refresh token flow 
Customization of token claims can be implemented using Azure AD B2C custom policies.  It is possible to customize claims generation process, when the tokens are requested specifically using refresh token.  

This functionality gives variety of possibilities, for example 

* Refresh user claims in returned id token using Azure AD, or other identity provider data. Some claims might have changed after they were initially generated during sign in process. 
* Call external API to check if user, who is trying to refresh the token, is on some backlist. Interrupt token generation if that is the case. 
* Check if user’s refresh token was revoked.

In example below, I will present how to incorporate revoked session check into refresh token flow.

After requesting */token* endpoint with *grant_type=refresh_token* Azure AD B2C triggers *UserJourney* which is specified in *RelyingParty* as an Endpoint with *Id=Token*.  
```xml
<RelyingParty>
    <DefaultUserJourney ReferenceId="SignUpOrSignIn" />
    <Endpoints>
        <Endpoint Id="Token" UserJourneyReferenceId="RedeemRefreshToken" />
    </Endpoints> 
... 
</RelyingParty> 
```
*RedeemRefreshToken* Journey is implemented in Azure AD B2C [starter pack](https://learn.microsoft.com/en-us/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#custom-policy-starter-pack) in *TrustFrameworkBase.xml* policy file. (Starter pack is recommended by Azure as a foundation for Custom Policies) Of course, you can extend, customize or use different *UserJourney* instead.  

*RedeemRefreshToken* UserJourney includes three steps, 
```xml
<UserJourney Id="RedeemRefreshToken">
    <PreserveOriginalAssertion>false</PreserveOriginalAssertion>
    <OrchestrationSteps>
        <OrchestrationStep Order="1" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="RefreshTokenSetupExchange" TechnicalProfileReferenceId="RefreshTokenReadAndSetup" />
            </ClaimsExchanges>
        </OrchestrationStep>
        <!-- Extra steps can be added before or after this step for REST API or claims transformation calls-->
        <OrchestrationStep Order="2" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="CheckRefreshTokenDateFromAadExchange" TechnicalProfileReferenceId="AAD-UserReadUsingObjectId-CheckRefreshTokenDate" />
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="3" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />
    </OrchestrationSteps>
</UserJourney>
```

which utilize following *TechnicalProfiles*



1. ***RefreshTokenReadAndSetup*** – it just returns refresh token parameters like *objectId* and *refreshTokenIssuedOnDateTime* in output claims. Before executing refresh token UserJourney, refresh token claims seems to be already preloaded into claims bag. (It is different approach comparing to [UserInfoEndpoint implementation](https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/active-directory-b2c/userinfo-endpoint.md), where dedicated technical profile loads access token claims into claims bag explicitly). 
2. ***AAD-UserReadUsingObjectId-CheckRefreshTokenDate*** – retrieves user information stored in azure AD. Since it uses base *AAD-UserReadUsingObjectId* TechnicalProfile, basic user data will updated in returned id/access token. Additionally, *refreshTokensValidFromDateTime* property is read from AD. When user’s refresh token is revoked *refreshTokensValidFromDateTime* is set to time it happened.   
   ```xml
   <TechnicalProfile Id="AAD-UserReadUsingObjectId-CheckRefreshTokenDate">
       <OutputClaims>
           <OutputClaim ClaimTypeReferenceId="refreshTokensValidFromDateTime" />
           <OutputClaim ClaimTypeReferenceId="displayName" />
       </OutputClaims>
       <OutputClaimsTransformations>
           <OutputClaimsTransformation ReferenceId="AssertRefreshTokenIssuedLaterThanValidFromDate" />
       </OutputClaimsTransformations>
       <IncludeTechnicalProfile ReferenceId="AAD-UserReadUsingObjectId" />
   </TechnicalProfile> 
   ```
   By comparing *refreshTokensValidFromDateTime* property with *refreshTokenIssuedOnDateTime* it is possible to validate if used refresh token should be accepted by auth server. This logic is implemented by output claim transformation. Once *refreshTokensValidFromDateTime* is greater that *refreshTokenIssuedOnDateTime*, exception is thrown and UserJourney is interrupted. In that case, Auth server returns **400-Bad request** http response, including body: 
   ```json
   { 
       "error": "invalid_grant", 
       "error_description": "AADB2C90129: The provided grant has been revoked. Please reauthenticate and try again.Correlation ID: xxxxxx-xxxx-xxxx-xxxx-xxxxxxxx\r\nTimestamp: xxxx-xx-xx xx:xx:xxx\r\n" 
   } 
   ```
3. ***JwtIssuer*** - OpenIdConnect technical profile that issues requested token and returns it to relaying party.  Refresh token properties described above, in Refresh tokens properties in Azure AD B2C section, can be specified in its definition. 