---
title: OAuth Authorization Code Interception Through Deep Links
date: 2026-04-18 17:35:00 +0500
categories: [Android Security]
tags: [OAuth, Authorization Code Interception, PKCE, Mobile Security, Deep Links Vulnerabilities] 
published: false
---

## Introduction

This write-up documents a real-world vulnerability discovered in a telecom Android application where poor OAuth implementation combined with insecure deep link handling allowed an attacker to intercept authorization codes, exchange them for JWT access tokens, and gain full account access.

## The Target Application
The target is a telecom application used by millions of users for account management, billing, and eSIM provisioning. The app offers social login functionality through Google and Facebook OAuth.

## Vulnerability Overview
The vulnerability chain involves:
1. **Insecure deep link configuration** allowing arbitrary apps to handle OAuth redirect URIs
2. **Missing PKCE validation** providing no protection against authorization code interception  [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)

The app implements Google/Facebook social login using OAuth 2.0 Authorization Code flow. The authorization code is delivered back to the app via the custom URI scheme `myapp://oauth/callback`. Since Android allows any app to register the same custom scheme, a malicious app installed on the victim's device can intercept the authorization code.

The app backend authorization server endpoint POST `/v1/auth/sso/google` accepts just auth_code, origin_type, with no cient secret or other client authentication. This means the intercepted authorization code alone is sufficient to obtain the victim's access token.
The JWT aaccess token is then used as the X-AUTH header for all authenticated API calls, granting full access to the victim's account including profile data, billing, plan management, eSIM details, and payment methods.

### Environment
I have tested this vulnerability on the following environment:
- **App Version:** Latest version available on the Play Store 
- **Tested on Android Version:**  Android 15
- **Test Device:** Pixel 9a

## Initial Discovery: Analyzing the Application
After extracting the app and decompiling it with JADX, I started by examining the **AndroidManifest.xml**, I found **LoginActivity** (exported) with an intent filter configured to accept custom deep links, that can possibly lead to intent hijacking vulnerability if not validated properly.

```xml
<activity android:name="com.telecom.app.LoginActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="oauth" android:path="/callback" />
    </intent-filter>
</activity>
```
The app is registering to handle OAuth callbacks through a custom URI scheme: `myapp://oauth/callback`. 

**On Android, any application installed on the device can also register to handle the exact same URI scheme and intercept the intent data**
As documented in Android's security guidance on [unsafe use of deep links](https://developer.android.com/privacy-and-security/risks/unsafe-use-of-deeplinks), this creates a classic **intent hijacking vulnerability**. When an OAuth callback arrives, Android doesn't automatically route it to the legitimate app instead, it asks the user which app should handle it. If a malicious app is installed claiming to handle the same scheme, the user gets a choice and might pick the wrong one.

#### The OAuth Handler function
I examined the **LoginActivity's onCreate() method** to see how the app processes the callback intent. The handleOAuthRedirect function handles the incoming intent:

```java
private void handleOAuthRedirect(Intent intent) {
    Uri uri = intent.getData();
    if (uri != null) {
        String authCode = uri.getQueryParameter("code");
        String state = uri.getQueryParameter("state");
        String provider = uri.getQueryParameter("provider"); // facebook or google
        if (authCode != null) {
            Log.d("OAuth", "Authorization code received: " + authCode);
            authSsoLogin(authCode, provider);
        }
    }
}
```
The handleOAuthRedirect function extracts the authorization code from the URI, determines which OAuth provider (Google or Facebook) the user authenticated with and exchanges the code for an access token by calling the **authSsoLogin** function.
If an attacker's app could intercept this callback before the legitimate app, it would have the authorization code.

For more context on how OAuth works and why authorization code interception is dangerous, see [Authorization Code Flow with Proof Key for Code Exchange (PKCE)](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce).

## Intercepting the Authorization Code
To test if this vulnerability could actually be exploited and intercept the authorization code, I created a minimal Android app designed to **claim the same deep link URI and capture whatever data comes through it**.
**Malicious App Manifest Configuration:** I copied the same intent filter from the legitimate app in the malicious app activity:
```xml
<activity
    android:name=".MainActivity"
    android:exported="true" > 
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="myapp"/>
        <data android:host="oauth"/>
        <data android:pathPrefix="/callback"/>
    </intent-filter>
</activity>
```

And replicated the same intent handler code (for receiving the authorization code) in the malicious app's activity:
```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    EdgeToEdge.enable(this);
    setContentView(R.layout.activity_main);
    handleOAuthRedirect(getIntent());
}

private boolean handleOAuthRedirect(Intent intent) {
    Uri data = intent.getData();
    String uriString = data.toString();
    Log.d("handleoauthredirectrequest", "Received OAuth redirect: " + uriString);
    String code = data.getQueryParameter("code");
    if(code != null && code.length() > 0) {
        String providerParam = data.getQueryParameter("provider");
        Log.d("handleOAuthRedirect", "Code: " + code);
    }
    return true;
}
```

#### The Attack in Action

Now for the actual test. I installed **both** apps on an Android emulator and initiated the login flow in the legitimate app:
1. User opens the legitimate telecom app and clicks "Login with Google"
2. Google's OAuth consent screen appears - user authenticates
3. User grants permission for the app to access their profile
4. Google generates an authorization code and redirects to `myapp://oauth/callback?code=XXXXX&state=YYYY&provider=google`
5. **Android shows a disambiguation dialog** listing both apps that can handle this URI
6. User selects the malicious app, the callback goes to the attacker app instead

![Image](/assets/images/Posts/AndroidSecurityWriteup/Im5.png)

Selecting the malicious app from the dialog, the authorization code is successfully intercepted and logged in the malicious app's logcat:
![Image](/assets/images/Posts/AndroidSecurityWriteup/Im2.png)

We have successfully stolen the authorization code. 

## Exchanging the Code for an Access Token
Before attempting to exchange the code, I needed to determine if the app implements PKCE and find the backend endpoint where I could exchange this code for an access token. Looking at the SocialApiService code, I found the endpoint for exchanging the authorization code:

![Image](/assets/images/Posts/AndroidSecurityWriteup/Im6.png)

The AUTH_SSO_LOGIN resolved to `v1/auth/sso/{provider}`. The request body contains the AuthSsoLoginRequest object which has the following fields:
- `authCode` - The OAuth authorization code
- `originType` - Device type identifier

There is no code_verifier or client authentication required to exchange the code for an access token - the app does not implement PKCE, . This means that anyone with the intercepted authorization code can directly call this endpoint to obtain an access token.

I tried sending a curl request to AUTH_SSO_LOGIN endpoint to exchange the authorization code for an access token. With the authorization code and the backend URL, I could now attempt to exchange the code for an access token:
![Image](/assets/images/Posts/AndroidSecurityWriteup/Im8.png)


MISSING_DEVICE_ID_HEADER. The server requires an additional header. Resending the request with the **X-Device-ID** header, I got a successful response and obtained a JWT access token for the victim's account:
With the required device ID header, the request succeeded:
![Image](/assets/images/Posts/AndroidSecurityWriteup/Im7.png)

I now had a valid JWT access token for the victim's account which I could use to access protected resources and perform actions on behalf of the user.

## Accessing Protected User Information
With a valid JWT access token, I tried to access the api endpoint that returns the user's profile information including the access token in the Authorization header:

![Image](/assets/images/Posts/AndroidSecurityWriteup/Im9.png)

As seen in the response, I successfully retrieved the victim's profile information including their name, email and account details.
From this point, I could access any endpoint that the legitimate app can access including billing and payment information, eSIM provisioning endpoints, account settings modification, subscription management

**Implementing the SSO login flow in attacker app**
To demonstrate the full attack, I implemented the SSO login flow in the malicious app. After intercepting the authorization code, I directly called the authSsoLogin function to exchange the code for an access token and retrieve the victim's profile information. The malicious app now has the same level of access to the victim's account as the legitimate app.

**The attacker now has complete control over the victim's account.**
This vulnerability represents a real threat to millions of users. While it requires a malicious app on the device, such an app could be:
- **Hidden in a legitimate-looking utility** (battery saver, cleaner, etc.)
- **Part of a supply chain compromise**
- **Social-engineered onto the victim's device**