---
layout: post
title: "Progressive profiling with Azure AD B2C"
categories: misc
---

# Progressive Profiling with Azure AD B2C

## Table of contents
- [Introduction](#introduction)
- [The concept of Progressive Profiling](#the-concept-of-progressive-profiling)
- [Progressive Profiling implementation using Azure AD B2C](#progressive-profiling-implementation-using-azure-ad-b2c)
- [Progressive Profiling UserJourney - Custom Policy implementation](#progressive-profiling-userjourney---custom-policy-implementation)

## Introduction 
Azure AD B2C provides advanced tools for user identity and access management. Modern applications often delegate identity and access management to external service. At the same time authorization and authentication needs to be highly flexible providing features as: 
* Integration with external identity providers like GitHub or Facebook etc. 
* Customization of user experience during sign-in, sign-up, profile edit etc. 
* Extension of user identity with application-specific information 
* Incorporate custom services/API calls into authorization or authentication flow

Azure AD B2C supports all these scenarios and even more. In this article I will focus on progressive profiling implementation, taking advantage from Azure AD B2C support for highly customized user experience.  

Progressive Profiling is a gradual way of collecting user preferences. As application collects user’s preferences it can provide more personalized, better suited experience.

## The concept of Progressive Profiling
Imagine user is singing-up for gym website. User is asked to fill standard sign-up form including email and name. Gym website added personalized training offer recently. In result, during sign-up process, user is additionaly asked about favorite exercises, training preferences, age, calories eaten per day, health condition and past training record. At this point user would probably resign from signing-in. The main reason could be user doesn’t trust the website yet so to share such detailed information.  

Progressive profiling is a way to ask these questions gradually as trust to website increases and personalization benefits are observed by user. Let's assume user signs-up to the website with minimum information required. After several visits on the website, during log-in process, user is informed about personalized training offer and asked to choose favorite exercise from the list. This question can be not answered, however after few visits on the website and maybe on the gym itself, there is a better chance this information will be provided. As users observe benefits form personalization, they become more eager to share their preferences.

## Progressive Profiling implementation using Azure AD B2C 
Azure AD B2C allows to incorporate progressive profiling into sign-in process. It is possible to customize sign-in process and extend it with gathering and processing additional user’s input. 
![Azure AD B2C login 1](/assets/img/article1/azure-b2c-progressive-profiling-diagram-1.png)
![Azure AD B2C login 2](/assets/img/article1/azure-b2c-progressive-profiling-diagram-2.png)

Provided information can be stored in Azure AD B2C tenant and served within id_token. Having preferences as a part of user identity enables personalization of application services. OAuth 2.0 OpenId token can be extended with progressive profiling information as follows: 
![Azure AD B2C login 2](/assets/img/article1/azure-b2c-open-id-token.jpg)

Azure AD B2C provides two ways of configuring identity related flows.
* **User flows** - predefined, build-in policies designed to handle most common scenarios; can be configured within Azure Portal UI. 
* **Identity Experience Framework (Custom Policies)** - Configuration of identity-related processes like sign-in, sign-up, reset password or profile edit is called policy. Policies are defined in xml files. Xml-based policies are flexible and can be highly customized.

![Azure AD B2C login 2](/assets/img/article1/azure-b2c-policies.jpg)

Progressive profiling scenario is not supported by user flows and needs to be implemented using Identity Experience Framework. **TrustFrameworkPolicy*** is top-level xml element of policy file. It defines claims schema, claims transformation, token generation steps and more. It is well described on https://learn.microsoft.com/en-us/azure/active-directory-b2c/trustframeworkpolicy.  

**TrustFrameworkPolicy** uses concept of UserJourneys. **UserJourney** includes steps (Orchestration Steps) needs to be taken to perform identity related actions like sign-in or sign-up. In the context of signing-in, the result of UserJourney is generated id_token.

**OrchestrationSteps** process claims using TechnicalProfiles. **TechnicalProfiles** generates output claims based on input claims by interacting with AD, calling external API, gathering data from user, etc. TechnicalProfiles are connected together in a chain by Orchestration Steps, creating UserJourney.

UserJourney starts when OAuth 2.0 flow is initiated (in most cases by calling corresponding /authorize endpoint). Once all UserJourney steps are finished, user is redirected to *RedirectUrl* with generated token. RedirectedUrl can be configured in Azure AD B2C tenant for registered application. 

## Progressive Profiling UserJourney - Custom Policy implementation 
The concept solution of progressive profiling policy is available on my [GitHub](https://github.com/melmanm/azure-b2c-custom-policy-progressive-profiling). 
Repository contains following files implementing policies 

* TrustFrameworkBase.xml (B2C_1A_TrustFrameworkBase policy) 
* TrustFrameworkLocalization.xml (B2C_1A_TrustFrameworkLocalization policy) 
* TrustFrameworkExtensions.xml (B2C_1A_TrustFrameworkExtensions policy) 

These files originate from [Azure custom policy samples repository](https://github.com/Azure-Samples/active-directory-b2c-custom-policy-starterpack). They are recommended as a foundation for building custom policies. 
Additionally, my repository contains xml files with custom progressive profiling policy implementation. 

* **ProgressiveProfileTrustFrameworkBase.xml**(B2C_1A_ ProgressiveProfile_TrustFrameworkBase policy) defines base claims and technical profile to handle progressive profiling, exchange claims with AAD and handle user input. 
* **ProgressiveProfileTrustFrameworkExtensions.xml** (B2C_1A_ ProgressiveProfile_TrustFrameworkExtensions policy) defines application-specific claims which will be gathered from user in progressive profiling flow. It implements UserJourney. 
* **ProgressiveProfileSignUpOrSignin.xml**(B2C_1A_ProgressiveProfile_SignUpOrSignIn policy) defines RelyingParty which returns final token claims. 

Policies can be inherited. Inheritance allows to use or extend elements defined in parent policies. Following diagram presents inheritance of policies in this implementation: 
![Azure AD B2C login 2](/assets/img/article1/b2c-progressive-profiling-policies-inheritance.png)

Below diagram shows the concept of UserJourney that incorporates progressive profiling into user sign-in flow. 

---
*Presented UserJourney uses several custom claims, to control, gathering progressive profiling information*

* **PPCounter** – used to count user sign-in actions. Once counter reaches specific value, progressive profiling form should be displayed to the user after signing-in. Next PPCounter is reset. It is saved in AD user attributes, so value is preserved between UserJourney executions. 
* **PPShouldExecute** – determines if PPCounter reaches specific level after which progressive profiling form should be displayed to the user. In given example PPShouldExecute is set once PPCounter equals 3. It means user is asked to fill progressive profiling form after every 3rd sign-in. 
* **PPExecuted** – indicates that progressive profiling information was already gathered from user in current UserJourney. It helps to prevent from asking user for multiple information in single UserJourney execution.

---

![Azure AD B2C login 2](/assets/img/article1/b2c-progressive-profiling-user-journey.png)

**Step 1** utilizes base self-asserted technical profile which displays sign-in form in the browser.  Once user fills email and password *SelfAsserted-LocalAccountSignin-Email* credentials are validated by executing ROPC (Resource Owner Password Credentials) flow. ROPC flow gets user access token form https://login.microsoftonline.com/{tenant}/oauth2/token endpoint. Received token indludes *objectId* attribute, which identifies user in AD tenant.

**Step 2** is not directly related with progressive profiling and it is excluded from diagram. Step 2 guides user through sign-up action in case user account does not exist. 

**Step 3** reads user related data from AD tenant. It uses extended base *AAD-UserReadUsingObjectId* TechnicalProfile to gather user data based on *objectId* from Step1.

1. B2C_1A_ ProgressiveProfile_TrustFrameworkBase

   ```xml
   <TechnicalProfile Id="AAD-UserReadUsingObjectId">
       <OutputClaims>
           <OutputClaim ClaimTypeReferenceId="extension_PPCounter" DefaultValue="0"/>
       </OutputClaims>
       <OutputClaimsTransformations>
           <OutputClaimsTransformation ReferenceId="IncrementProgressiveProfileCounter" />
           <OutputClaimsTransformation ReferenceId="SetProgressiveProfilingShouldExecute" />
       </OutputClaimsTransformations>
   </TechnicalProfile>  
   ```
   *PPCounter* claim is read from AD. If it does not exist its value is set to 0. Additionaly, two output claims transformations are executed: 

   *PPCounter* claim is incremented

   ```xml
   <ClaimsTransformation Id="IncrementProgressiveProfileCounter" TransformationMethod="AdjustNumber">
       <InputClaims>
           <InputClaim ClaimTypeReferenceId="extension_PPCounter" TransformationClaimType="inputClaim" />
       </InputClaims>
       <InputParameters>
           <InputParameter Id="Operator" DataType="string" Value="INCREMENT" />
       </InputParameters>
       <OutputClaims>
           <OutputClaim ClaimTypeReferenceId="extension_PPCounter" TransformationClaimType="outputClaim" />
       </OutputClaims>
   </ClaimsTransformation> 
   ```

   *PPShouldExecute* claim is added and set to *true* when *PPCounter* equals 3. **It enables progressive profiling prompt on every third log-in**

   ```xml
   <ClaimsTransformation Id="SetProgressiveProfilingShouldExecute" TransformationMethod="AssertNumber">
       <InputClaims>
           <InputClaim ClaimTypeReferenceId="extension_PPCounter" TransformationClaimType="inputClaim" />
       </InputClaims>
       <InputParameters>
           <InputParameter Id="Operator" DataType="string" Value="GreaterThanOrEqual" />
           <InputParameter Id="CompareToValue" DataType="int" Value="3" />
           <InputParameter Id="throwError" DataType="boolean" Value="false" />
       </InputParameters>
       <OutputClaims>
           <OutputClaim ClaimTypeReferenceId="PPShouldExecute" TransformationClaimType="outputClaim" />
       </OutputClaims>
   </ClaimsTransformation>
   ```
2. ProgressiveProfile_TrustFrameworkExtensions 

   ```xml
   <TechnicalProfile Id="AAD-UserReadUsingObjectId">
       <OutputClaims>
           <OutputClaim ClaimTypeReferenceId="extension_PreferedTraining"/>
           <OutputClaim ClaimTypeReferenceId="extension_PreferedExcercises"/>
       </OutputClaims>
   </TechnicalProfile>
   ```
   Progressive profiling, implementation specific, claims are read from AD as well. 

**Step 4** executes SubJourney which is designed to gather and store user preferences. Each Orchestration Step correspond with single progressive profiling claim. Its structure can be easily extended when new application specific claims are added. SubJourney is executed when *PPShouldExecute* equals true. 

SubJourney Steps are skipped if corresponding claim is already set or different claim was set in current UserJourney. 
```xml
<OrchestrationStep Order="1" Type="ClaimsExchange">
    <Preconditions>
        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
            <Value>extension_PreferedTraining</Value>
            <Action>SkipThisOrchestrationStep</Action>
        </Precondition>
        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
            <Value>PPExecuted</Value>
            <Action>SkipThisOrchestrationStep</Action>
        </Precondition>
    </Preconditions>
    <ClaimsExchanges>
        <ClaimsExchange Id="ProfileUpdateExchangePreferedTraining" TechnicalProfileReferenceId="SelfAsserted-ProgressiveProfiling-PreferedTraining" />
    </ClaimsExchanges>
</OrchestrationStep> 
```
It calls *SelfAsserted* technical profile which displays progressive profiling form and saves selected value in output claim. Additionaly, technical profile sets *PPExecuted* output claim to true. It prevents from showing next progressive profiling forms during the same sign-in flow.

**Step 5** writes output claims associated with user to Azure AD. 

**Step 6** resets (deletes) *PPCounter* claim from AD if progressive profiling SubJourney were executed. 

```xml
<OrchestrationStep Order="6" Type="ClaimsExchange">
    <Preconditions>
        <Precondition Type="ClaimEquals" ExecuteActionsIf="true">
            <Value>PPExecuted</Value>
            <Value>false</Value>
            <Action>SkipThisOrchestrationStep</Action>
        </Precondition>
    </Preconditions>
    <ClaimsExchanges>
        <ClaimsExchange Id="DeleteClaimUpdateExchange" TechnicalProfileReferenceId="AAD-DeleteProgressiveProfilingCounter" />
    </ClaimsExchanges>
</OrchestrationStep> 
```
**Step 7** creates JWT token based on current output claims. 

---
*All policy configuration files are available on my [GitHub](https://github.com/melmanm/azure-b2c-custom-policy-progressive-profiling).*

---