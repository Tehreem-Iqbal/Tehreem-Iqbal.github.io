---
title: OAuth Code Interception Through Intent Hijacking
date: 2026-05-23 02:05:00 +0500
categories: [Mobile Security]
tags: [OAuth, Authorization Code Interception, PKCE, Mobile Security, Deep Links Vulnerabilities] 
---


## Introduction
While analyzing an Android application, I came across an authorization code interception vulnerability due to insecure deep link handling and a missing PKCE 
implementation. This blog explores how OAuth is implemented in Android applications, how a missing PKCE implementation combined with insecure deep link handling can allow an attacker to intercept authorization codes and gain full account access, and how PKCE protects against this attack.

## Normal OAuth 2.0 flow
OAuth 2.0 is an authorization framework that lets a third-party application access a user's resources on another service without the user sharing their credentials.

A normal oauth flow is:
1. User taps "Login with Google / Facebook"
2. App redirects user to the Authorization Server
3. User logs in and grants permission
4. Auth server returns an authorization code to the app
5. App exchanges the code for an access token
6. App uses the access token to call the bakend API

![Image](/assets/images/Posts/OAuth/oauthnormalflow.png)

On Android, apps receive the OAuth callback by registering a custom URI scheme in AndroidManifest.xml:
```xml
<activity
    android:name=".LoginActivity" >
    <intent-filter>
        ...
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="customscheme"/>
        <data android:host="oauth"/>
        <data android:pathPrefix="/callback"/>
    </intent-filter>
  </activity>
```
After the user authenticates, Google redirects to:
```
customscheme://oauth/callback?code=AUTH_CODE&provider=google
```
Android intercepts this URL and routes it to the registered app. If an attacker's app could intercept this callback, it would have the authorization code.

## OAuth Code Interception via deep link hijacking
On Android, any application installed on the device can also register to handle the exact same URI scheme and intercept the intent data (Authorization code). As documented in Android's security guidance on [unsafe use of deep links](https://developer.android.com/privacy-and-security/risks/unsafe-use-of-deeplinks), this creates a classic intent hijacking vulnerability. When an OAuth callback arrives, Android doesn’t automatically route it to the legitimate app instead, it asks the user which app should handle it. If a malicious app is installed claiming to handle the same scheme, the user gets a choice and might pick the wrong one.

![Image](/assets/images/Posts/OAuth/authcodeinterception.png)

The intercepted authorization code can then be sent to the authorization server to exchange it for an access token.

Normally, exchanging an authorization code requires a client secret to prove the request is coming from the legitimate app. But since the mobile apps are public client, this client secret can be extracted by the attacker.

#### The Attack in Action

Installed **both** apps (vulnerable app and attacker app) and initiated the login flow in the legitimate app:
1. User opens the legitimate app and clicks "Login with Google"
2. Google's OAuth consent screen appears - user authenticates
3. User grants permission for the app to access their profile
4. Google generates an authorization code and redirects to `customscheme://oauth/callback?code=XXXXX&state=YYYY&provider=google`
5. **Android shows a disambiguation dialog** listing both apps that can handle this URI
6. User selects the malicious app, the callback goes to the attacker app instead

![Image](/assets/images/Posts/OAuth/Im5.png)

Selecting the malicious app from the dialog, the authorization code is successfully intercepted and logged in the malicious app's logcat:
![Image](/assets/images/Posts/OAuth/Im1.png)

Since the app does not implement PKCE, the attacker app with the intercepted authorization code can directly call this endpoint to obtain an access token. Now attempt to exchange the code for an access token:
![Image](/assets/images/Posts/OAuth/Im8.png)

Now, the attacker app has a valid JWT access token for the victim's account which can be used to access protected resources and perform actions on behalf of the real user.
 
## Authorization code flow with PKCE  (Proof Key for Code Exchange)
PKCE is a security extension to OAuth 2.0 designed to prevent authorization code interception attacks,  especially important for public clients like mobile apps and SPAs that can't securely store a client secret. 

PKCE ensures only the original requester can do that exchange. [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)

The PKCE flow:
1. User taps login
2. App generates cryptographically-random `code_verifier` and `code_challenge`
```
code_verifier  = cryptographically random string
code_challenge = BASE64URL(SHA256(code_verifier))
```
3. App sends code_challenge to the Authorization Server with the login request
4. Authorization Server stores the challenge and issues an authorization code
5. App sends the authorization code and code_verifier to exchange for a token
6. Authorization Server hashes the verifier and checks it matches the stored challenge
7. Issue access token if verified

![Image](/assets/images/Posts/OAuth/oauthwithpkce.png)



Custom URI schemes have no ownership verification, making OAuth callbacks interceptable by any app on the device. Without PKCE, an intercepted authorization code is all an attacker needs, no additional credentials required, leading directly to a valid access token and full account access.

PKCE cryptographically binds the authorization code to the app that requested it, making an intercepted code useless without the `code_verifier`.
