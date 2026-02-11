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
