---
title: Getting started with AAD integration in JavaScript
date: 2021-04-14 08:05:13
updated: 2021-09-16 17:52:00
description: My sharing for the UX team about the auth process, components and some empirical knowledges from the business.
categories:
- Web 前端
tags:
- oauth
- azure
- aad
- auth
- msal
- microsoft-identity
- TechTalk
---


## Relationship between OAuth2.0, AAD and MSAL

### Definitions

#### OAuth 2.0

OAuth 2.0 is the industry-standard **protocol** for authorization. OAuth 2.0 focuses on client developer simplicity while providing specific authorization flows for web applications, desktop applications, mobile phones, and living room devices.

#### Azure Authentication Directory

One of OAuth 2.0 and OpenID Connect standard-compliant authentication **service** enabling developers to authenticate work or school accounts. It's the implementation which supports OAuth 2.0. It's the component of Microsoft Identity platform.

#### Microsoft Authentication Libraries

Open source **libraries** for **several clients** to authenticate users using AAD, Microsoft personal accounts (MSA), and social identity providers like Facebook, Google, LinkedIn, Microsoft accounts, etc.

### Usage scenario

![oauth-scenario](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6IHQAAAAAAJbw7JxUahyBCOI?width=1141&height=754)



## Workflow and implementation

### Authentication flow

![auth-flow](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6IXQAAAAAAPmBRQmNY7L1M_8?width=1820&height=1089)

#### Two APIs

Only two APIs requested from client-side, very simple. Notice that `/oauth2/v2.0/authorize` is `document` type.

![auth-document](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6HXQAAAAAADtAdchzfpJcMlc?width=1928&height=1161)

#### Redirect URI setup

For CORS and redirect target from sign-in page usage, should configure it on both Azure Portal and codebase.

#### Configuration on Azure Portal

![reply-uris-config](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6H3QAAAAAAEdVZ-zUMZ6cCHA?width=1319&height=863)

#### oauth.html

Usually we will set two reply URIs, one is probably the homepage as redirect target after signed in, another is used for requesting token silently. The silent token refresh loads an iframe using that empty `oauth.html` file, and that same `oauth.html` file needs to receive the response back, so the acquire token silent flow provides the `/oauth.html` endpoint as the redirect URI. This is different from the user-facing flow because there is no iframe that the middleware loads into the page, redirecting back to `oauth.html` will not hook into React or MSAL and auth would halt -- it must redirect back to your react app in the user-facing flow.

#### Code configuration

```typescript
const config = {
    auth: {
        // The application ID of your AAD application
        clientId: "11111111-1111-1111-111111111111",
        // The authority URL for your application
        authority: "https://login.microsoftonline.com/common",
        // Defaults to application start page
        redirectUri: "https://localhost:3000",
        postLogoutRedirectUri: "https://localhost:3000/logout"
    }
}

const loginRequest = {
    scopes: [`${resourceEndpoint}/.default`]
}
```

#### Authority

The authority is a URL that indicates a directory that MSAL can request tokens from, for more details check the [MSAL Application configuration options](https://docs.microsoft.com/en-us/azure/active-directory/develop/msal-client-application-configuration).

#### Scopes and permissions

Applications that integrate with the Microsoft identity platform follow an authorization model that gives users and administrators control over how data can be accessed. `/.default` represents OpenID Connect scopes, for more details check the [Permissions and consent in the Microsoft identity platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent).

### Sign-in with redirect

It's no need to redirect to sign-in page every time if auth is not expired, just go ahead. You can also [sign-in with a pop-up window](https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-spa-sign-in?tabs=javascript2#sign-in-with-a-pop-up-window).

```typescript
let username = "";

const msalInstance = new PublicClientApplication(config);

function handleResponse(response) {
    //handle redirect response

    // In case multiple accounts exist, you can select
    const currentAccounts = msalInstance.getAllAccounts();

    if (currentAccounts === null) {
        // No accounts detected, need to login
        msalInstance.loginRedirect(loginRequest);
    } else if (currentAccounts.length > 1) {
        // Add choose account code here
    } else if (currentAccounts.length === 1) {
        username = currentAccounts[0].username;
    }
}

msalInstance.handleRedirectPromise().then(handleResponse);
```

#### Authorization code

After signed in successfully, page will redirect to the redirect URI we set before with the authorization code. MSAL.js library will handle it automatically, we can observe the code on the navigation bar.

![hash-code](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6HHQAAAAAAHpK1DZ6S5AguZg?width=1928&height=70)

Then the library will parse the authorization code and send the request to get token back directly, all logics encapsulated. Here is the structure of the decoded code.

![auth-code-deserialized](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6JHQAAAAAABGEAriQXq2iNsU?width=1181&height=227)

### Acquire a token with a redirect

The pattern for acquiring tokens for APIs with MSAL.js is to first attempt a silent token request by using the `acquireTokenSilent` method. When this method is called, the library first checks the cache in browser storage to see if a valid token exists and returns it. When no valid token is in the cache, it sends a silent token request to AAD from a hidden iframe.

The silent token requests to AAD might fail for reasons like an expired Azure AD session or a password change. In that case, we need to redirect or using a pop-up window to acquire tokens.

```typescript
const accessTokenRequest: AuthenticationParameters = {
    scopes: [`${resourceEndpoint}/.default`],
    authority: this.authority,
    redirectUri: "https://localhost:3000/oauth.html",
    account: this.account
}

msalInstance.acquireTokenSilent(accessTokenRequest).then(function(accessTokenResponse) {
    // Acquire token silent success
    // Call API with token
    let accessToken = accessTokenResponse.accessToken;
}).catch(function (error) {
    //Acquire token silent failure, and send an interactive request
    console.log(error);
    if (error.errorMessage.indexOf("interaction_required") !== -1) {
        msalInstance.acquireTokenRedirect(accessTokenRequest);
    }
});
```

#### Why use redirect way?

If users have browser constraints or policies where pop-ups windows are disabled, you can use the redirect method. Use the redirect method with the Internet Explorer browser, because there are [known issues with pop-up windows on Internet Explorer](https://github.com/AzureAD/microsoft-authentication-library-for-js/wiki/Known-issues-on-IE-and-Edge-Browser).

### Choice of MSAL.js

You will find so many packages when you visit MSAL.js repository, APIs used above should be compatible across the packages, then question comes: how to make a choice?

Besides node package using in the server-side, I recommend `msal-browser` ([Microsoft Authentication Library for JavaScript v2.x](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser)) or other UI framework wrappers based on it, for its implementation of OAuth 2.0 Authorization Code Flow with PCKE as well as it's OpenID-compliant.

![msal-package-structure](https://1drv.ms/i/c/7a756318060faeec/UQTsrg8GGGN1IIB6InQAAAAAAOR5onzJTSk7gxw?width=1554&height=993)

These packages are just official engineering implementations, not so well-documented yet, I had to read the source code for debugging before. Anyway as long as you master the workflow of authentication, it would not be too complicated.



## Migration from AdalJS to MSAL.js

AdalJS is the legacy library for AAD authentication used by so many dated products. If you don't have any idea about this section, the official document explains [Why switch to MSAL](https://docs.microsoft.com/en-us/azure/active-directory/develop/msal-migration#why-switch-to-msal) clearly.

### Tips

- You may need to change the reply URL type from `Web` to `SPA` as MSAL.js required. AdalJS supports both two types, so this change will not block online client services.
- Need to specify the `accessTokenAcceptedVersion` to `2` and `signInAudience` to `AzureADandPersonalMicrosoftAccount` in the manifest if you want to support both AAD account and MSA account. These settings will not break online client services as well.
- You can just import the MSAL.js distribution script to your app then test if the current AAD configurations are satisfying both SDKs.
- If there is an auth service for the app, you can just do [silent login with `ssoSlient()`](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/login-user.md#silent-login-with-ssosilent).
- Remember to test all kinds of audiences of the customers and all provided resouce endpoints with access token.

### Quick login verification on you app page

PoC (Proof of concept) or verfication always needs quick moves. For the migration scenario, we have to make sure the AAD configurations are correct as well as compatible to make both AdalJS clients and MSAL.js clients login successfully at first stage. To do the quick login verification, the authentication client class need to be exposed to `Window` global no matter AdalJS or MSAL.js, then we can do experiment on the console of browser.

#### For MSAL.js

Please refer to the [document](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/initialization.md), it's similar to the codes demonstrated above.

#### For AdalJS

The docs for AdalJS are much fewer than MSAL.js probably becasue it has been deprecated. And different versions of AdalJS seems to have few subtle API as well as behavior changes, so you may need to update the code below to make it work. My test code is based on AdalJS v1.0.8.

```javascript
var authContext = new AuthenticationContext({
            instance: "https://login.microsoftonline.com/",
            clientId: "Client ID of AAD app",
            redirectUri: "Reply URL of AAD app",
            postLogoutRedirectUri: window.location.href,
            cacheLocation: 'localStorage',
            endpoints: { graphApiResourceEndpoint: "https://graph.microsoft.com/" },
        })

// Implement the pop-up login flow
authContext.config.displayCall = function (url) {
      authContext.config.displayCall = null;
      var popup = window.open(url, 'auth-popup', 'width=800,height=500');
      var intervalId = window.setInterval(function () {
        try {
          // Assume your reply URL contains "/signin"
          if (popup.location.pathname.indexOf('/signin') >= 0) {
            window.clearInterval(intervalId);
            // Pass the ID token for AdalJS to parse
            window.location.hash = popup.location.hash;
            // But AdalJS will not clean the "#" tag in url, so it will redirect in the handleWindowCallback().
            // I'd like to add a breakpoint inside to update the value of "adal.login.request" key to stop redirection.
            // Just read the source code if you don't know what I mean.
            authContext.handleWindowCallback();
            popup.close();
            var user = authContext.getCachedUser();
            if (user) {
              console.log(JSON.stringify(user))
            } else {
              console.error(authContext.getLoginError());
            }
          }
        } catch (whatever) {
          if (popup.closed) {
            console.error(whatever)
          }
        }
      }, 100);
    };

// Login via a pop-up window
authContext.login();

// Inspect the access token after logining
authContext.acquireToken("https://graph.microsoft.com/", function() { console.log(arguments); debugger })
```



## Frequently asked issues

### Debugging and verification approach

You can't verify your change until the production release, also the change for the production environment can't be verified on other staging environment since they are using different hosts. So the question is how to debug or do verification?

Basically you can run a local server to host your application with adding a host record which points the production host to your local IP address like `127.0.0.1` to make it. Remember to enable HTTPS protocol for your local server.

And it's better to verify your change on both common window and incognito window after the release.



### Difference between access token, ID token, and refresh token

We observed that `/oauth2/v2.0/token` return 3 tokens: `access_token`,  `id_token` and `refresh_token`.

`access_token` enables clients to securely call protected web APIs, and are used by web APIs to perform authentication and authorization. For more details please check [Microsoft identity platform access tokens](https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens).

`id_token` should be used to validate that a user is who they claim to be and get additional useful information about them, it can be sent alongside or instead of an access token. We have not used it. For more details please check [Microsoft identity platform ID tokens](https://docs.microsoft.com/en-us/azure/active-directory/develop/id-tokens).

`access_token` and `id_token` are both JWT which consists of a header, payload, and signature portion, that's why they look so similiar though these two strings are base64 encoded. We mainly use access token for the application, it contains more information than ID token, and we can get user info like name, E-mail from it.

Because access tokens are valid for only a short period of time, authorization servers will sometimes issue a refresh token at the same time the access token is issued. MSAL.js will exchange the `refresh_token` while request token silently for a new access token when needed.



### How to change token expiration time?

Please check this [answer from StackOverflow](https://stackoverflow.com/questions/31162257/azure-oauth-how-to-change-token-expiration-time).



### InteractionRequiredAuthError: AADSTS50058: A silent sign-in request was sent but no user is signed in.

This error need interaction sign in, thus catch the error and call `acquireTokenRedirect` method to sign in.

If the error message contains like this "The cookies used to represent the user's session were not sent in the request to Azure AD. This could happen if the user is using Internet Explorer or Edge, and the web app sending the silent sign-in request is in different IE security zone than the Azure AD endpoint (login.microsoftonline.com).", upgrade your MSAL.js v1.x to v2.x. Check more details [here](https://github.com/AzureAD/microsoft-authentication-library-for-js/issues/1765).



### The frame attempting navigation of the top-level window is sandboxed, but the flag of 'allow-top-navigation' or 'allow-top-navigation-by-user-activation' is not set.

This error usually occurred while requesting token, mostly it's caused by wrong reply URL which iframe could not load it to request token silently. So make sure your reply URL pointing to the blank static HTML resource is right. Additional, this error message is somehow confusing cause it does not tell the root cause directly, check more details [here](https://github.com/AzureAD/microsoft-authentication-library-for-js/issues/1199).



### BrowserAuthError: pcke_not_created: The PCKE code challenge and verifier could not be generated.

Check the protocol and be sure it's HTTPS.



## References

- [Microsoft identity platform and OAuth 2.0 authorization code flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow)
- [What is the Microsoft identity platform?](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-overview)
- [OAuth 2.0 authentication with Azure Active Directory](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/auth-oauth2)
- [Single-page application: Sign-in and Sign-out](https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-spa-sign-in)
- [Single-page application: Acquire a token to call an API](https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-spa-acquire-token)
- [MSAL Application configuration options](https://docs.microsoft.com/en-us/azure/active-directory/develop/msal-client-application-configuration)
- [Permissions and consent in the Microsoft identity platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent)
- [Configurable token lifetimes in the Microsoft identity platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-configurable-token-lifetimes)
- [Microsoft identity platform ID tokens](https://docs.microsoft.com/en-us/azure/active-directory/develop/id-tokens)
- [Microsoft identity platform access tokens](https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens)
- [Quickstart: Register an application with the Microsoft identity platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)

