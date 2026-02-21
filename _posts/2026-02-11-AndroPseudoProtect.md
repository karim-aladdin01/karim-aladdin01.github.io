---
title: AndroPseudoProtect
categories: 8kSec
tags: [Mobile Appsec, 8kSec]
order: 5

---
## Objective
Create a malicious application that exploits the AndroPseudoProtect application by targeting vulnerabilities in its IPC mechanisms. Your goal is to develop an Android application that can silently disable the encryption protection without the user's knowledge or consent. The attacker should also be able to steal unencrypted files otherwise considered encrypted on the external filesystem. The exploit should ensure that even when users believe they've activated the advanced protection, it remains ineffective because the victim application turns it off in the background, undermining the app's publicized security claims. All this without needing any action from the victim!  

Successfully completing this challenge demonstrates a critical vulnerability in service authentication that could allow attackers to silently disable security protections, putting sensitive user data at risk and potentially enabling further device compromise.

## Restrictions

Your exploit must work on Android versions up to Android 15 and must not require any runtime permissions to be granted by the victim except the standard external storage access permissions and notification permissions on the device. Your attacker PoC should demonstrate the ability to extract and reuse any application-generated or hardcoded tokens from the victim application through normal user interaction, rather than hardcoding those tokens directly into the PoC.

## <span style="color:rgb(0, 176, 80)">Walkthrough</span> 
- By analyzing the `AndroidManifest.xml` file, we got the following:
### App Metadata
- **Package Name**: `com.eightksec.andropseudoprotect`
- **Version**: 1.0 (versionCode=1)
- **Compiled with**: Android SDK 33 (Android 13)

### SDK Compatibility
- **Minimum SDK**: API 26 (Android 8.0 Oreo)
- **Target SDK**: API 33 (Android 13)

### Permissions Requested
The app requests several sensitive permissions:
1. **`FOREGROUND_SERVICE`** - Allows the app to run foreground services
2. **`POST_NOTIFICATIONS`** - Can post notifications (Android 13+)
3. **`READ_EXTERNAL_STORAGE`** - Can read files from external storage
4. **`WRITE_EXTERNAL_STORAGE`** - Can write to external storage (only up to Android 10/API 29)
5. **`MANAGE_EXTERNAL_STORAGE`** - Full access to manage all files (very powerful permission)
6. **Custom permission** - Defines and uses its own signature-level permission for security

### App Components
#### MainActivity
- The launcher activity (starts when app is opened)
- Exported (can be launched by other apps)
#### SecurityService
- A foreground service with type `dataSync`
- Exported (can be started by other apps)
- Likely runs security monitoring in the background
#### SecurityReceiver
- A broadcast receiver that responds to custom actions:
    - `START_SECURITY`
    - `STOP_SECURITY`
- Exported (can receive broadcasts from other apps)
#### Content Provider
- Uses AndroidX startup library for initializing components at app startup

---
### Analyzing the workflow of the code:
- 
