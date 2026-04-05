# MobileGuard KYC Verification Bypass
---
title: WriteUp: MobileGuard KYC Verification Bypass
date: 2026-04-06 03:32:51 +5000
categories: [Android Security, CTF Writeups]
tags: [MobileGuard, KYC bypass, face embedding, JWT forgery]    
---
## Challenge Description
Your goal is to analyze the app's identity flow and find a way to unlock the hidden flag, by impersonating the whitelisted users, which are the founder and co-founders of Mobile Hacking Lab.

Challenge file : `MobileGuard.apk`.
## Analyzing the APK
I started by loading the APK in JADX and start analyzing the AndroidMenifest.xml. Nothing fancy - one activity (`MainActivity`), with some services, receivers and content providers. Everything lives in `MainActivity`, so that's where I started reading.


## How the App Works
The `MainActivity` constructor generates a random session ID (`UUID.randomUUID()`) right away, before anything else. Then in `onCreate`, it sets up a `LivenessDetector` and a `FaceEmbedder`, and opens the front camera. Camera frames come in through `processImageProxy()`, throttled to once per second - and go through a two-phase pipeline.

**Phase one is liveness.** There's a class called `LivenessDetector` that loads a MediaPipe face landmark model and watches for two things: eye blinks (both eyes need to score above 0.4 on the blendshape, twice) and a head turn (yaw angle above ~28 degrees from the transformation matrix). Standard anti-spoofing stuff. Once you blink twice and turn your head, it flips `isLivenessPassed` to true.

The catch? This all happens on the device locally. The server never checks if someone actually blinked or liveness check was passed or not.

**Phase two is server verification.** After liveness passes, the app talks to a backend at `2026.mhc-ctf.workers.dev` in three steps:
1. Sends a JWT to `/api/liveness`, gets back a `session_hmac`
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/liveness.png)
2. Generates a face embedding and sends it with the HMAC to `/api/verify-face`, gets back a `face_token`
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/verifyface.png)
3. Sends the `face_token` to `/api/get-flag`, gets the flag
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/getflag.png)

So the question becomes: can we fake steps 1 and 2?

## The Hardcoded Secret
In `KYCApiClient.generateLivenessToken()`, the app constructs a token with `sub: "LivenessVerified"`, `blink_passed: true`, `head_turn_passed: true`, signs it with HS256 using a harcoded secret LIVENESS_SECRET:
```java
public static final String LIVENESS_SECRET = "MobileGuard2025_SuperSecretKey!!!";
```

![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc-jwttoken.png)

## The Face Embedding 
The server still needs to see a face embedding that matches one of the whitelisted users.
Basically, the app ships with a FaceNet TFLite model (`facenet.tflite`) in its assets. When a camera frame passes liveness, the app does some image preprocessing on it -- resizing, normalization, the usual ML pipeline stuff -- feeds it through the model, and gets back a 512-dimensional float array. That array is what gets sent to the server. Not the image itself -- just 512 numbers representing the face.

This means if I can get a photo of the whitelisted user and run it through the same model with the same preprocessing, I'll get the same embedding the server expects.

## Building the Exploit

### generatefaceembedding.py
This one replicates what the app's `FaceEmbedder` does on-device. It takes a photo, runs it through the same preprocessing and TFLite model extracted from the APK, and dumps the resulting 512-dim embedding vector to a JSON file.

```bash
unzip MobileGuard.apk assets/facenet.tflite
python generatefaceembedding.py --photo whitelisteduser.jpeg --model assets/facenet.tflite --save-embedding whilelistembedding.json
```
### kyc.py
The kyc.py script handles the server side. It forges a JWT liveness token using the hardcoded secret, sends it to `/api/liveness` for the session HMAC, posts the embedding to `/api/verify-face` for the face token, and hits `/api/get-flag` for the flag.

```bash
python kyc.py --embedding whilelistembedding.json
```
## Finding the Right Whitelisted users
As the challenge says the whitelisted users are the founder and co-founders of Mobile Hacking Lab, but doesn't tell you which one the server is actually expecting. So I had to go through them.
**Umit Aksu** - Found his photo, generated the embedding, sent the embedding to the server. Response: `"similarity": 0.417, "Similarity too low"`.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc1.png)
**Jelmer Hulsman** - Generated the embedding, sent it. Also similarity too low.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc2.png)
**Arno Miedema** - Generated the embedding, sent the request, and this time the similarity was high enough. The server returned a `face_token`, sent the face_token to `/api/get-flag`, and got the flag.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc3.png)
