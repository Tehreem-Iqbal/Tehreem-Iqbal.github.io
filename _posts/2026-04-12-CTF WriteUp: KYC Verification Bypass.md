---
title: CTF WriteUp: KYC Verification Bypass
date: 2026-04-12 03:32:51 +5000
categories: [Android Security, CTF Writeups]
tags: [KYC bypass, Identity verification bypass, face embedding]    
---
## Introduction
In this writeup, I'll walk through the process of solving the "KYC verification bypass" challenge from the Mobile Hacking Lab CTF. The challenge revolves around an Android app called MobileGuard that implements a KYC flow using face recognition and liveness detection. The goal is to bypass the KYC verification and retrieve a hidden flag by impersonating one of the whitelisted users.

## Challenge Description
Your goal is to analyze the app's identity flow and find a way to unlock the hidden flag, by impersonating the whitelisted users, which are the founder and co-founders of Mobile Hacking Lab.

**Challenge file** : `MobileGuard.apk`


## How the Identity Flow mechanism works
After loading the APK into JADX, most of the interesting code is in `MainActivity` and `KYCApiClient`. The app uses the camera to capture video frames, runs them through a local liveness detection model, and then sends a face embedding to a backend server for verification. If the embedding matches one of the whitelisted users, the server returns a flag.

**High level flow:**
1. App generates a random session ID on startup (`UUID.randomUUID()`)
2. App opens the front camera and starts processing frames
3. Performs liveness detection locally (eye blinks and head turns)
4. Generates a face embedding 
5. Sends data to backend server in three steps: 
   - Get session HMAC with a JWT token ```/api/liveness```
   - Send face embedding and HMAC for verification ```/api/verify-face```
   - If verified, get the flag ```/api/get-flag```


**Liveness check (Client side)** The app uses `LivenessDetector` that loads a MediaPipe face landmark model and checks: eye blinks and a head turn. Once the conditions are met, it flips `isLivenessPassed` to true.

**Issue:** **The server does not validate whether liveness checks actually occurred** 

**Server verification flow.** After liveness passes, the app talks to a backend at `2026.mhc-ctf.workers.dev` in three steps:
1. Sends a JWT to `/api/liveness`, gets back a `session_hmac`
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/liveness.png)
2. Generates a face embedding and sends it with the HMAC to `/api/verify-face`, gets back a `face_token` (if the face embedding matches a whitelisted user)
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/verifyface.png)
3. Sends the `face_token` to `/api/get_flag`, gets the flag
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/getflag.png)

#### Generating a forged JWT for liveness bypass
In `KYCApiClient.generateLivenessToken()`, the application constructs a JWT with the following claims:

```json
{
  "sub": "LivenessVerified",
  "blink_passed": true,
  "head_turn_passed": true
}
``` 
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc-jwttoken.png)


The JWT token is signed using HS256 with a hardcoded secret, ```LIVENESS_SECRET = "MobileGuard2025_SuperSecretKey!!!"```. 
With this harcoded secret, we can forge a valid JWT token with the required claims to bypass the liveness check on the server. This allows us to proceed to the next step without actually performing any liveness checks on a real face.

## Generating the Face Embedding 
The server accepts a client-generated face embedding without capture integrity or source authenticity validation.

Basically, the app ships with a FaceNet TFLite model (`facenet.tflite`) in its assets. When a camera frame passes liveness, the app does some image preprocessing on it, resizing, normalization, the usual ML pipeline stuff, feeds it through the model, and gets back a 512-dimensional float array. That array is what gets sent to the server. 
This means if I can get a photo of the whitelisted user and run it through the same model with the same preprocessing, I'll get the same embedding the server expects.

## Exploitation

1. Extract hardcoded JWT secret
2. Forge liveness token
3. Generate valid face embedding
4. Submit embedding to backend
5. Receive valid face_token
6. Access protected endpoint


#### `generatefaceembedding.py`
The script replicates what the app's `FaceEmbedder` does on-device. It takes a photo, runs it through the same preprocessing and TFLite model extracted from the APK, and dumps the resulting 512-dim embedding vector to a JSON file.

```bash
unzip MobileGuard.apk assets/facenet.tflite
python3 generatefaceembedding.py --photo whitelisteduser.jpeg --model assets/facenet.tflite --save-embedding whilelistembedding.json
```
This produces the same 512-dimensional embedding expected by the server.

#### `kyc.py`
The kyc.py script handles the server side. It forges a JWT liveness token using the hardcoded secret, sends it to `/api/liveness` for the session HMAC, submit the generated embedding to `/api/verify-face`, retrieves the face_token, and fetches the flag from `/api/get_flag`.

```bash
python3 kyc.py --embedding whilelistembedding.json
```

#### Finding the Right Whitelisted users
As the challenge says the whitelisted users are the founder and co-founders of Mobile Hacking Lab. A quick search on the website's about page revealed three names: Umit Aksu, Jelmer Hulsman, and Arno Miedema. I found their photos online, ran them through the embedding generation script, and submitted each embedding to the server.

**Umit Aksu**
Response: `"similarity": 0.417, "Similarity too low"`.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc1.png)

**Jelmer Hulsman** 
Response: `"similarity": 0.417, "Similarity too low"`.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc2.png)

**Arno Miedema** 
Response: `"similarity": 0.856, "Face verified"`.
![Image](/assets/images/Posts/CTF%20writeup%20-%20Mobilegurd%20kyc%20verification%20bypass/kyc3.png)
