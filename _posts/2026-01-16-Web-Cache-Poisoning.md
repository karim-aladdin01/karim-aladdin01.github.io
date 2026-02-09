---
title: Web Cache Poisoning
categories:
  - PortSwigger
tags:
  - Web Appsec
  - PortSwigger
order: 1
icon:
---
# How Does Cache Work?

Web application stores data in a database. Reading data from the database needs network calls and I/O operations which is a time-consuming process. Cache reduces the network calls to the database and speeds up the performance of the system.

- When the first time a request is made a call will have to be made to the database to process the query. This is known as a cache miss.
- Before giving back the result to the user, the result will be saved in the cache.
- When the second time a user makes the same request, the application will check your cache first to see if the result for that request is cached or not.
- If it is then the result will be returned from the cache. This is known as a cache hit.
- The response time for the second time request will be a lot less than the first time.
<br/>
<br/>
- Basic Concepts and Components
	1. **Cache Key**: Parameters that determine cache **uniqueness** (usually URL + specific headers). It uses key inputs to decide whether two responses are the same.
	2. **Cache Buster**: Query parameters or headers excluded from the cache key (<span style="color:rgb(0, 112, 192)">keyed input</span>) and they are used to force the cache server to load the latest version from the web server while playing with the `Unkeyed Inputs` 
	3. **Unkeyed Inputs**: Headers or parameters processed by the origin server but not included in cache keys
	4. **Web Server:** Back-end (application framework).
	5. **Cache Server:** Front-end (e.g., CDN like Akamai, Cloudflare or reverse proxies like Nginx or Varnish).
	6. **TTL (Time To Live)**: How long poisoned content remains cached
<br/>
<br/>
- Why can't we use cache keys inputs for poisoning?
	- As any payload injected via keyed inputs would act as a `cache buster`, meaning your poisoned cache entry would almost certainly never be served to any other users.
<br/>
<br/>

# <span style="color:rgb(255, 0, 0)">The golden rule: Find an unkeyed input that influences the response, poison it, and let the cache do the rest and the exact steps to follow are:</span>
1. Identify a suitable cache oracle: Identify a page or an endpoint that provides feedback about the cache's behavior and whether the response comes from the cache server or from the backend server such as `X-Cache: Hit` & `X-Cache: Miss`. If you identify a 3rd party like Akami or Cloudflare, you may check the documentation to know how the default cache key is constructed and you can find some tricks to know about the cache-key such as:
	- Akami-based websites: may support the header `Pragma: akamai-x-get-cache-key`which displays the cache-key in the response headers. The default cache keys of Akami: 
	![](../assets/img/Pasted%20image%2020260116201623.png)
	- Default Cache key of Cloudflare
	![](../assets/img/Pasted%20image%2020260116201734.png)

2. Add a cache buster (in some cases, you may not be able to find it and you will do your work on the original response served to other users. So, be careful in such cases)
	![](../assets/img/Pasted%20image%2020260102175425.png)
	- If you use Param Miner, you can select the options "Add static/dynamic cache buster" and "Include cache busters in headers".
	- Most probably adding a query param `cb=cache-buster` will be enough if the query string is a keyed input, but what if it isn't? Fortunately, there are alternative ways of adding a cache buster, such as adding it to a keyed header that doesn't interfere with the application's behavior. Some typical examples include:
```
Accept-Encoding: gzip, deflate, cachebuster 
Accept: */*, text/cachebuster 
Cookie: cachebuster=1 
Origin: https://cachebuster.vulnerable-website.com
```

3. Identify unkeyed inputs with Param Miner
	![](../assets/img/Pasted%20image%2020260102180805.png)
	![](../assets/img/Pasted%20image%2020260102175211.png)

4. Explore input potential
5. Elicit a harmful response & inject into cache

---
# Exploiting cache design flaws

# Exploiting cache implementation flaws
- Many websites and CDNs perform various transformations on keyed components when they are saved in the cache key. This can include:
	- Filtering out specific query parameters or even excluding the whole query string 
	- Normalizing input in keyed components
	- Removing the port from the `Host` header
	- Removing the Request method (results in fat GET request)
<br/>
These transformations may introduce a few unexpected quirks. These are primarily based around <span style="color:rgb(0, 112, 192)">discrepancies between the data that is written to the cache key and the data that is passed into the application code</span>, even though it all stems from the same input. These cache key flaws can be exploited to poison the cache via inputs that may initially appear unusable.

### Unkeyed port
If the Host is part of the cache-key but the port is excluded from it, then you can perform a **DOS** attack.
- Consider the case where a redirect URL is dynamically generated based on the `Host` header.

```http
GET / HTTP/1.1
Host: vulnerable-website.com

HTTP/1.1 302 Moved Permanently 
Location: https://vulnerable-website.com/en 
Cache-Status: miss
```
- You can poison the cache by appending a dummy port at the end of the `HOST`:

```http
GET / HTTP/1.1
Host: vulnerable-website.com:3879
```
 - Now, All the users who browse to `https://vulnerable-website.com` will be redirected to `https://vulnerable-website.com:3879/en` , which is a dud port.  

### Web cache poisoning via an unkeyed query string
Excluding the query string from the cache key can actually make these reflected XSS vulnerabilities even more severe.
Usually, such an attack would rely on <span style="color:rgb(255, 0, 0)">inducing the victim to visit a maliciously crafted URL</span>. However, poisoning the cache via an unkeyed query string would cause the payload to be served to users who visit what would otherwise be a perfectly normal URL. This has the potential to impact a far greater number of victims with no further interaction from the attacker.
<br/>
- **Lab Description**: This lab is vulnerable to web cache poisoning because the query string is unkeyed. A user regularly visits this site's home page using Chrome. To solve the lab, poison the home page with a response that executes `alert(1)` in the victim's browser.
<br/>
#### 💡<span style="color:rgb(0, 176, 80)">Solution</span>
- First, observe the cache oracle via the headers:
	![](../assets/img/Pasted%20image%2020260116235934.png)
- Try adding a cache buster to the homepage `GET /?cb=cache-buster` and observe that you never get a `miss`, which means that the query param is unkeyed.
- Try probing the request till you get a cache miss. I tried the Cookie, Accept, Accept-Encoding headers but none of them worked. So, I added `Origin: example.com` and I got a `miss`
	![](../assets/img/Pasted%20image%2020260117001811.png)
- You can see that the query param `cb=cache-buster` is reflected in the response. Let's try to close the link tag and inject a script tag to trigger the alert. The working payload is:
```html
?cb=test'/><script>alert(1)</script>
```
- Right-click on the request from Burp and select `request in browser` then paste the link in the browser and observe that the alert pops up.
  Note: Copying the url directly won't work as this doesn't set the origin header.
- Now, remove the origin header and re-poison the cache and the lab should be solved.

----
### Web cache poisoning via an unkeyed query parameter
So far we've seen that on some websites, the entire query string is excluded from the cache key. But some websites only exclude specific query parameters that are not relevant to the back-end application, such as parameters for analytics or serving targeted advertisements. UTM parameters like `utm_content` are good candidates to check during testing.
<br/>
- **Lab Description**: This lab is vulnerable to web cache poisoning because it excludes a certain parameter from the cache key. A user regularly visits this site's home page using Chrome. To solve the lab, poison the cache with a response that executes `alert(1)` in the victim's browser.
<br/>
#### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- First, observe the cache oracle via the headers:
	![](../assets/img/Pasted%20image%2020260116235934.png)
- Try adding a cache buster to the homepage `GET /?cb=cache-buster` and observe that you get a `miss`, which means it's a cache buster.
- Use `param miner`  to get unkeyed inputs by guessing query params and observe that the  `utm_content` appears in the output.
	![](../assets/img/Pasted%20image%2020260117010040.png)
- As expected, the value of `utm_content` is reflected in the response:
	![](../assets/img/Pasted%20image%2020260117010243.png)
- Try this payload:
```html
/?cb=cache-buster&utm_content=test'/><script>alert(1)</script>
```
- Copy URL and paste it in the browser and observe that the alert pops up
- Remove the cache-buster, re-poison the cache and the lab should be solved

----
### Cache parameter cloaking
Abusing discrepancies between how the cache and the origin servers use characters and strings as delimiters to separate parameters and strip unwanted ones.

For example, it's known that a query param is placed after `?` or `&` in the query string, but Ruby accepts `;` as a delimiter as well. 
<br/>
Consider the following request:
```http
GET /?keyed_param=abc&excluded_param=123;keyed_param=bad-stuff-here
```
- As the names suggest, `keyed_param` is included in the cache key, but `excluded_param` is not. Many caches will only interpret this as two parameters, delimited by the ampersand:
	1. `keyed_param=abc`
	2. `excluded_param=123;keyed_param=bad-stuff-here`
- Once the parsing algorithm removes the `excluded_param`, the cache key will only contain `keyed_param=abc`
<br/>
- On the back-end, however, Ruby on Rails sees the semicolon and splits the query string into three separate parameters:
	1. `keyed_param=abc`
	2. `excluded_param=123`
	3. `keyed_param=bad-stuff-here`

But now there is a duplicate `keyed_param`. This is where the second quirk comes into play. If there are duplicate parameters, each with different values, <span style="color:rgb(0, 112, 192)">Ruby on Rails gives precedence to the final occurrence</span>. The end result is that the cache key contains an innocent, expected parameter value, allowing the cached response to be served as normal to other users. <span style="color:rgb(255, 0, 0)">On the back-end, however, the same parameter has a completely different value, which is our injected payload.</span> 
<br/>

- This exploit can be especially powerful if it gives you control over a function that will be executed. For example, if a website is using JSONP to make a cross-domain request, this will often contain a `callback` parameter to execute a given function on the returned data:
```http
GET /jsonp?callback=innocentFunction
```

In this case, you could use these techniques to override the expected callback function and execute arbitrary JavaScript instead.
<br/>
- **Lab Description**: This lab is vulnerable to web cache poisoning because it excludes a certain parameter from the cache key. There is also inconsistent parameter parsing between the cache and the back-end. A user regularly visits this site's home page using Chrome. To solve the lab, use the parameter cloaking technique to poison the cache with a response that executes `alert(1)` in the victim's browser.
<br/>
#### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- First, I identified a cache oracle from the response headers
- I appended `?cb=cache-buster` to the homepage and I got a `miss` so it's a cache buster
- I used `param miner` to guess query params and it identified `utm_content` 
	![](../assets/img/Pasted%20image%2020260117220308.png)
- let's try to break out of the link tag and inject a script tag
	![](../assets/img/Pasted%20image%2020260117220434.png)
	- It seems that the output is encoded.
- We need to search for another endpoint, this one `/js/geolocate.js?callback=setCountryCookie` looks interesting as the `setCountryCookie` is reflected in the response
	![](../assets/img/Pasted%20image%2020260117221623.png)
- Since it's a js file, we don't need a script tag. We only need to inject `alert(1)` directly
	![](../assets/img/Pasted%20image%2020260117222646.png)
- The `alert(1)` is reflected as intended, but we got a `X-Cache: miss` which means that the `callback` param is a keyed input.
- What if we try `parameter pollution`?
	![](../assets/img/Pasted%20image%2020260117223313.png)
- We still get a `X-Cache: miss`, which means that the 2nd callback param can still be seen by the caching server. So, we need to figure out some way to <span style="color:rgb(255, 0, 0)">hide it from the front-end caching server</span> ,aka making it `unkeyed input`. Let's try adding the `utm_content`, we still get a `X-Cache: miss`
	![](../assets/img/Pasted%20image%2020260117223706.png)
- To make sure that the `callback=alert(1)` is still a `keyed input` change `alert(1)` to `alert(2)` and if it's `unkeyed` you should get the same response aka a `X-Cache: hit`, but unfortunately you get a `X-Cache: miss`
	![](../assets/img/Pasted%20image%2020260117224140.png)
- Here where the `parameter cloaking` technique comes into play --> You get a `X-Cache: hit` again and again when you hit the `send` button in Burpsuite 
	![](../assets/img/Pasted%20image%2020260117224331.png)
- Now, wait till the `Age: 35` and hit send to poison the cache and you should get a `X-Cache: hit`  
	![](../assets/img/Pasted%20image%2020260117225029.png)

-----
### Web cache poisoning via a fat GET request
- In select cases, the <span style="color:rgb(0, 112, 192)">HTTP method may not be keyed</span>. This might allow you to poison the cache with a `POST` request containing a malicious payload in the body. Your payload would then even be served in response to users' `GET` requests. Although this scenario is pretty rare, you can sometimes achieve a similar effect by simply adding a body to a `GET` request to create a "fat" `GET` request: `GET /?param=innocent HTTP/1.1 … param=bad-stuff-here`

In this case, the <span style="color:rgb(0, 112, 192)">cache key would be based on the request line</span>, but <span style="color:rgb(255, 0, 0)">the server-side value of the parameter would be taken from the body.</span>

- This is only possible if a website accepts `GET` requests that have a body, but there are potential workarounds. You can sometimes encourage "fat `GET`" handling by overriding the HTTP method, for example:
```http
GET /?param=innocent HTTP/1.1 
Host: innocent-website.com 
X-HTTP-Method-Override: POST 
… 
param=bad-stuff-here
```
As long as the `X-HTTP-Method-Override` header is unkeyed, you could submit a pseudo-`POST` request while preserving a `GET` cache key derived from the request line.

<br/>
- **Lab Description**: This lab is vulnerable to web cache poisoning. It accepts `GET` requests that have a body, but does not include the body in the cache key. A user regularly visits this site's home page using Chrome. To solve the lab, poison the cache with a response that executes `alert(1)` in the victim's browser.
<br/>
#### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- append `&cb=cache-buster` to the `GET /js/geolocate.js?callback=setCountryCookie` and observe that you get `X-Cache: miss`, which means that it's cache buster
- Add `callback=alert(1)` to the `body` of the request and observe that it's reflected
	![](../assets/img/Pasted%20image%2020260117232024.png)
- Remove the cache buster and wait till `Age: 35`, poison the cache and the lab should be solved
	![](../assets/img/Pasted%20image%2020260117232141.png)

----
### URL normalization
Cache key normalization can make otherwise unexploitable reflected XSS exploitable. If the cache normalizes encoded and unencoded parameters to the same key
```
GET /example?param="><test>
GET /example?param=%22%3e%3ctest%3e
```
 An attacker can poison the cache with an unencoded XSS payload, and when a victim requests the encoded version, the cache serves the poisoned response, causing the payload to execute.

<br/>
- **Lab Description**: This lab contains an XSS vulnerability that is not directly exploitable due to browser URL-encoding. To solve the lab, take advantage of the cache's normalization process to exploit this vulnerability. Find the XSS vulnerability and inject a payload that will execute `alert(1)` in the victim's browser. Then, deliver the malicious URL to the victim.
<br/>
#### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- After identifying a cache oracle, I added `?cb=cache-buster` to the homepage, and I got a cache `miss` which means that it's a cache buster.
- I used `param miner` to guess for `unkeyed inputs` in the homepage, but nothing was found. Additionally, the cache buster is not reflected in the response at all.
- In such cases we check for `URL Normalization` 
- Instead of `GET /` , we request `GET %2f` which is the `URL-Encoded` version of the homepage
	![](../assets/img/Pasted%20image%2020260118005206.png)
- Now, what if the front-end caching server normalizes both the normal `/` and the encoded `%2f` versions, which means if we are able to cache the `404 Not Found` response from the backend server to the front-end caching server, then we are able to perform a `DOS` attack as any normal user fetches `GET /` will then served the poisoned version `GET %2f` as the cache server treats them as identical.
	![](../assets/img/Pasted%20image%2020260118005908.png)
- During the caching window, if a normal user visits the homepage
	![](../assets/img/Pasted%20image%2020260118010153.png)
- As the `%2f` is reflected within a `<p>` tag, what if we inject a `<script>` tag and this is a kind of XSS attacks which is trying to trigger an error results in returning `4XX or 5XX`, meanwhile our injected string is reflected and thus we can trigger an alert by injecting a malicious payload after the string that caused the error initially
	![](../assets/img/Pasted%20image%2020260118011015.png)
- During the caching window, which is a little bit short for this lab 10s, Copy the link as paste it in the browser
	![](../assets/img/Pasted%20image%2020260118011714.png)
- Now, send the `URL-encoded` version of the link to victim, which is `https://0a9700fd0465f0d681f9cfa600f30032.web-security-academy.net/%3Cscript%3Ealert(1)%3C/script%3E` and the lab should be solved.
- Note that if you send the `un-encoded` or `normal` version of the URL to the victim, which is `https://0a9700fd0465f0d681f9cfa600f30032.web-security-academy.net%2f<script>alert(1)</script>`, the attack won't work.

----
### Cache key injection
- Keyed headers can become exploitable via cache poisoning if cache key delimiters are not properly escaped, allowing different requests to resolve to the same cache key and serve a poisoned response. You can exploit this by first poisoning the cache with a request containing your payload in the corresponding keyed header:
```http
GET /path?param=123 HTTP/1.1 
Origin: '-alert(1)-'__

HTTP/1.1 200 OK 
X-Cache-Key: /path?param=123__Origin='-alert(1)-'__ 

<script>…'-alert(1)-'…</script>
```
- If you then induce a victim user to visit the following URL, they would be served the poisoned response:
```http
GET /path?param=123__Origin='-alert(1)-'__ HTTP/1.1 

HTTP/1.1 200 OK 
X-Cache-Key: /path?param=123__Origin='-alert(1)-'__ 
X-Cache: hit 

<script>…'-alert(1)-'…</script>
```
<br/>
- **Lab Description**: This lab contains multiple independent vulnerabilities, including cache key injection. A user regularly visits this site's home page using Chrome. To solve the lab, combine the vulnerabilities to execute `alert(1)` in the victim's browser. Note that you will need to make use of the `Pragma: x-get-cache-key` header in order to solve this lab
<br/>
#### 💡 <span style="color:rgb(0, 176, 80)">Solution</span>
- 