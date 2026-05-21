# OSL Social Login

A unified multi-platform social login solution for Unity.

## Overview
OSL Social Login provides a single, unified interface for integrating various social login platforms (Apple, Facebook, Google, Steam) into your Unity project. It abstracts specific SDK implementations behind a common interface, making it easy to selectively enable, disable, and manage platform-specific code without breaking your build.

## Features
- **Unified Interface**: Use the ISocialLoginProvider adapter to perform login operations across any supported platform with identical method signatures and callbacks.
- **Centralized Editor Setup**: Manage all API keys and application IDs in one convenient Editor Window via Tools > OSL > Social Login > Setup.
- **Soft Dependencies**: Leveraging Assembly Definitions (smdef) and Scripting Define Symbols (e.g., OSL_GOOGLE, OSL_FACEBOOK), the package safely excludes unselected SDKs from compiling.
- **Automated Build Process**: Pre-built processors automatically inject required elements (e.g., AndroidManifest values, .androidlib configurations, iOS Info.plist URL schemes) dynamically during build time.

## Getting Started

1. **Import SDKs**: Import the requisite SDKs (Google Sign-In, Facebook SDK for Unity, Steamworks.NET / Facepunch) depending on your target platforms.
2. **Configure Settings**: 
   - Open Tools > OSL > Social Login > Setup.
   - Enable the providers you plan to use.
   - Enter your App IDs, Client Tokens, and Client IDs.
   - Click **Apply Settings**.
3. **Usage Example**:
    `csharp
    // Call unified facade via specific provider adapter
    var googleProvider = new GProvider(); 
    googleProvider.Signin((result) => 
    {
        if(result.IsSuccess) 
        {
            Debug.Log("Logged in successfully! Token: " + result.Token);
            // Send this Token to your Backend Server for verification!
        }
    });
    `

---

## ⚠️ Important Precautions & Platform Specifics
While the OSL package simplifies the Unity side of things, each platform maintains strict rules regarding security and implementation.

### Global Security Standard
- **Backend Verification is Mandatory**: Never trust the Identity Token (JWT), Access Token, or Session Ticket returned directly to the Unity client. You **MUST** send these tokens to your secure backend server and verify them using the respective platform's Official Auth API. Relying solely on client-side authentication is a critical vulnerability.

### 🍎 Apple Sign-In
- **Supported Platforms**: iOS (Native via AuthenticationServices).
- **Profile Scope Constraint**: Apple only provides the user's name and email on the **very first** successful sign-in. Subsequent sign-ins will only return the Identity Token. Do not design your database to expect the email string on every login attempt.
- **No Silent Sign-In**: Apple strictly enforces explicit user intent via FaceID/TouchID. Calling SilentSignIn() will automatically fallback to the standard SignIn() UI prompt.

### 📘 Facebook Login
- **Supported Platforms**: iOS, Android, WebGL. (PC/Standalone is explicitly excluded to comply with Meta's deprecation of desktop native OAuth flows).
- **WebGL HTTPS Requirement**: Facebook's JavaScript SDK strictly requires an https:// secure connection. Running local tests on an http:// endpoint will result in Facebook blocking the login attempt entirely.

### 🇬 Google Sign-In
- **Supported Platforms**: iOS, Android, WebGL, Standalone (PC).
- **Android SHA-1 Config**: You must copy the SHA-1 fingerprints for BOTH your local Unity Debug Keystore and your Release App Signing certificate to the Google Cloud Console. Failure to do so will result in an ApiException (Status Code 10).
- **EDM4U Required**: For mobile, use External Dependency Manager (EDM4U) to resolve Android (play-services-auth) and iOS (CocoaPods) library dependencies automatically.
- **PC Local Loopback**: The Standalone PC module uses a local HttpListener. You must add exactly http://127.0.0.1:50000/ to the *Authorized redirect URIs* in the Google Console 'Desktop app' credentials. 

### 💨 Steamworks
- **Supported Platforms**: Standalone (Windows, macOS, Linux, Steam Deck).
- **Steam Client Dependency**: Steam authentication cannot happen independently. The Steam Client application MUST be installed, running, and logged into a valid account on the development PC.
- **steam_appid.txt Required**: You must create a text file named exactly steam_appid.txt in the physical root folder of your project (outside Assets, near your .sln file) and enter only your API ID (e.g., 480) to test in the Unity Editor.
