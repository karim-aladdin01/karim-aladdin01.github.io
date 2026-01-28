---
title: JWT Attacks
categories: PortSwigger
tags:
  - Web Appsec
  - PortSwigger
  - JWT Attacks
order: 3
---

## What's the difference between JWTs and sessions?
Unlike with classic session tokens, all of the data that a server needs is stored client-side within the JWT itself. This makes JWTs a popular choice for highly distributed websites where users need to interact seamlessly with multiple back-end servers.

---
## Exploiting flawed JWT signature verification
JWTs are self-contained, meaning the server usually does not store any state about issued tokens. so if the server doesn’t properly verify the signature, attackers can modify the token. Example:
```json
{  
"username": "carlos",  
"isAdmin": false  
}  
```

Changing it to another username or setting `"isAdmin": true` can lead to impersonation or privilege escalation.

A common bug is using `decode()` instead of `verify()` (e.g., in jsonwebtoken). decode() doesn’t check the signature, so forged tokens are accepted.
<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. Due to implementation flaws, the server doesn't verify the signature of any JWTs that it receives. To solve the lab, modify your session token to gain access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- After logging in as `wiener`, try to access the `/admin` endpoint, but you get `Admin interface only available if logged in as an administrator`
- Send the `/admin` to `Repater` and edit the `"sub":"wiener"` to `"sub":"administrator"` and hit `Apply Changes` them `Send`
	![](../assets/img/Pasted%20image%2020260128014457.png)
- You get `200 OK` and in the response, you will find the direct links to delete both users carlos and wiener
	![](../assets/img/Pasted%20image%2020260128014715.png)
- Edit the request to `/admin/delete?username=carlos` and hit `Send` to solve the lab.

---
## JWT authentication bypass via flawed signature verification
JWT header has an alg field that tells the server how to verify the signature, for example:  
```json
{  
"alg": "HS256",  
"typ": "JWT"  
}
```

If an attacker changes `alg` to `"none"`, the token becomes unsigned (unsecured JWT). If the server trusts this value and doesn’t properly block it, it may accept a token with no signature, but payload is still used (must end with a trailing dot)
<br/>
Some servers try to block this, but weak string checks can be bypassed using tricks like mixed case or encoding.
<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. The server is insecurely configured to accept unsigned JWTs. To solve the lab, modify your session token to gain access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- You need to change 3 things in the JWT token:
```json
"alg":"RS256" --> "alg":"none"
"sub":"wiener" --> "sub":"administrator"
Remove the signature leaving the trailing dot
```
**Note: If the `alg:none` didn't work, you can try different letter casing such as `None, NoNe`**
- You get `200 OK` and in the response, you will find the direct links to delete both users carlos and wiener
	![](../assets/img/Pasted%20image%2020260128030323.png)
- Edit the request to `/admin/delete?username=carlos` and hit `Send` to solve the lab.

---
## JWT authentication bypass via weak signing key
Some JWT algorithms like HS256 use a secret key. If this key is weak, default, or hardcoded, an attacker can brute-force it and then sign their own tokens with any data they want. 
- Example: using hashcat with a known JWT and a wordlist:  
`hashcat -a 0 -m 16500 <jwt> <wordlist>`

<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. It uses an extremely weak secret key to both sign and verify tokens. This can be easily brute-forced using a [wordlist of common secrets](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list).To solve the lab, first brute-force the website's secret key. Once you've obtained this, use it to sign a modified session token that gives you access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- After logging in as `wiener`, from burp try to access the `/admin` and observe that you can't, we can brute force the `secret key` that was used to sign the jwt token and use it to forge a new one. We can crack the signature using various tool such as `jwt_tool` or `hachcat` 
```bash
jwt_tool "eyJraWQiOiJmYTJjMWY0Mi01Y2M0LTRhNmEtOTJlOS00Y2U0MTI0NjNjNTMiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc2OTU2ODA3NSwic3ViIjoid2llbmVyIn0.0HzimWHpykPLjiL5LLox65oNpA0rOmDiRudU_tSxNn8" -C -d jwt.secrets.list
```
- OR, you can use the `Jwt Editor` extension from burpsuit. It adds a `json web token` in the `Repeater`,hit the `Attack` button, then select `Weak HMAC secret` option. The result looks like:
	![](../assets/img/Pasted%20image%2020260128043114.png)
- Follow the steps numbers in the following screenshot to generate a new key to sign the token after editing it
	![](../assets/img/Pasted%20image%2020260128043557.png)
- Get back to the `json web token` tab in the `Repeater` and edit the `Payload` of the token, then `sign`
	![](../assets/img/Pasted%20image%2020260128044042.png)
- Hit `Send` and you receive `200 Ok` and follow the steps in the previous labs to solve it

---
## JWT header parameter injections
According to the JWS specification, only the `alg` header parameter is mandatory. In practice, however, JWT headers (also known as JOSE headers) often contain several other parameters. The following ones are of particular interest to attackers.
- `jwk` (JSON Web Key) - Provides an embedded JSON object representing the key.
- `jku` (JSON Web Key Set URL) - Provides a URL from which servers can fetch a set of keys containing the correct key.
- `kid` (Key ID) - Provides an ID that servers can use to identify the correct key in cases where there are multiple keys to choose from. Depending on the format of the key, this may have a matching `kid` parameter.
<br/>
As you can see, these user-controllable parameters each tell the recipient server which key to use when verifying the signature. In this section, you'll learn how to exploit these to inject modified JWTs signed using your own arbitrary key rather than the server's secret.

---
## JWT authentication bypass via jwk header injection
Some servers accept the public key directly from the JWT itself using the jwk header instead of verifying the signature against a trusted key store. This allows an attacker to create a self-signed token and make the server trust it. A JWT header with embedded public key looks like:
```json
{  
"kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",  
"typ": "JWT",  
"alg": "RS256",  
"jwk": {  
"kty": "RSA",  
"e": "AQAB",  
"kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",  
"n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"  
	}  
}
```
- Although you can manually add or modify the `jwk` parameter in Burp, the [JWT Editor extension](https://portswigger.net/bappstore/26aaa5ded2f74beea19e2ed8345a93dd) provides a useful feature to help you test for this vulnerability:
	1. With the extension loaded, in Burp's main tab bar, go to the **JWT Editor Keys** tab.
	2. Generate a new RSA key
	3. Send a request containing a JWT to Burp Repeater.
	4. In the message editor, switch to the extension-generated **JSON Web Token** tab and modify the token's payload however you like.
	5. Click **Attack**, then select `Embedded JWK`. When prompted, select your newly generated RSA key.
	6. Send the request to test how the server responds.
<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. The server supports the `jwk` parameter in the JWT header. This is sometimes used to embed the correct verification key directly in the token. However, it fails to check whether the provided key came from a trusted source. To solve the lab, modify and sign a JWT that gives you access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- After logging in as `Wiener` and trying to access the `/admin` endpoint, you get `401 Unauthorized`.
- Follow the above steps to solve the lab.

---
## JWT authentication bypass via jku header injection
Some servers use the jku (JWK Set URL) header in a JWT to tell them where to download the public key used to verify the signature. Instead of trusting a fixed key, they fetch a JWK Set from the provided URL and use the key with the matching kid. A JWK Set is a JSON object that contains multiple public keys, for example:  
```json
{  
"keys": [  
{  
"kty": "RSA",  
"e": "AQAB",  
"kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",  
"n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"  
},  
{  
"kty": "RSA",  
"e": "AQAB",  
"kid": "d8fDFo-fS9-faS14a9-ASf99sa-7c1Ad5abA",  
"n": "fc3f-yy1wpYmffgXBxhAUJzHql79gNNQ_cb33HocCuJolwDqmk6GPM4Y_qTVX67WhsN3JvaFYw-dfg6DH-asAScw"  
}  
]  
}
```
- These sets are often hosted at endpoints like  `/.well-known/jwks.json`
- If a server blindly trusts the jku URL, an attacker can:
	1. Host their own JWK Set containing their own public key.
	2. Create a JWT signed with the matching private key.
    3. Set the jku header to their malicious JWK Set URL.
    4. Set kid to match one of the keys in that set.
- More secure websites will only fetch keys from trusted domains, but you can sometimes take advantage of URL parsing discrepancies to bypass this kind of filtering. We covered some [examples of these](https://portswigger.net/web-security/ssrf#ssrf-with-whitelist-based-input-filters) in our topic on SSRF.
<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. The server supports the `jku` parameter in the JWT header. However, it fails to check whether the provided URL belongs to a trusted domain before fetching the key. To solve the lab, forge a JWT that gives you access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- Go to `JWt Editor` tab and generate a `New RSA Key`(you don't need to select a key size as this will automatically be updated later), then right-click on the generated key and select `Copy Public Key as JWK`
- Go to the exploit server and paste the copied text in the following template, then store it
```json
{ "keys": [
	// insert the jwk here 
 ] 
}
```
- Go to the `JSOn Web Token` generated tab in the Repeater and update:
	- header
	```json
{  
    "kid": "2600ff2a-5928-41a0-8e13-5bd604648993",  
    "alg": "RS256",  
    "jku": "https://exploit-0a2c002603dbcd6f82ecc41c01540071.exploit-server.net/exploit"  
}
	```
	- payload
```json
{  
    "iss": "portswigger",  
    "exp": 1769574248,  
    "sub": "administrator"  
}
```
**Note**: `kid` is updated to match the `kid` of our generated `RSA key` 
- Sign it with our generated `RSA key` and hit `Send`, you gonna receive `200 Ok`
	![](../assets/img/Pasted%20image%2020260128065335.png)

---
## JWT authentication bypass via kid header path traversal
Some servers use the kid (Key ID) header in a JWT to decide which key to use for signature verification. The kid is just an arbitrary string, so developers might use it to reference a <span style="color:rgb(0, 112, 192)">database entry</span>, a <span style="color:rgb(0, 112, 192)">JWK in a set</span>, or even a <span style="color:rgb(0, 112, 192)">file path</span>.

If the server uses the kid value to load a key from the filesystem and doesn’t properly sanitize it, this can lead to <span style="color:rgb(0, 112, 192)">path traversal</span>. An attacker can make the server load an arbitrary file as the verification key. Example JWT header:  
```json
{  
"kid": "../../path/to/file",  
"typ": "JWT",  
"alg": "HS256",  
"k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc"  
}
```
- The attacker can set the kid to a predictable file on the server and use its known contents as the HMAC secret to sign a forged JWT.
- A simple example is pointing kid to `/dev/null` on Linux, which is an empty file, so its content is an empty string. If the server uses it as the HMAC key, signing the JWT with an empty string will produce a valid signature.
<br/>
- **Lab Description**: This lab uses a JWT-based mechanism for handling sessions. In order to verify the signature, the server uses the `kid` parameter in JWT header to fetch the relevant key from its filesystem. To solve the lab, forge a JWT that gives you access to the admin panel at `/admin`, then delete the user `carlos`. You can log in to your own account using the following credentials: `wiener:peter`
<br/>
### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- Log in to your account and send the **GET /my-account** request to Repeater.
- Change the path to **/admin** and confirm it requires admin access (Returns `401 Unauthorized`)
- Open **JWT Editor → Keys** and create a **New Symmetric Key**.
- Generate a key in JWK format, then replace the value of **k** with `AA==` (Base64 for null byte). this is just a workaround because the JWT Editor extension won't allow you to sign tokens using an empty string.
	![](../assets/img/Pasted%20image%2020260128071555.png)
- Go back to the **GET /admin** request and open the JWT editor tab.
- In the JWT header, change **kid** to `../../../../../../../dev/null`
- In the payload, change **sub** to: `administrator`
	![](../assets/img/Pasted%20image%2020260128072046.png)
- Click **Sign**, select the generated symmetric key, keep **Don't modify header** enabled, and confirm.
- Send the request and verify admin panel access.
- From the response, open `/admin/delete?username=carlos` to complete the lab.

---
