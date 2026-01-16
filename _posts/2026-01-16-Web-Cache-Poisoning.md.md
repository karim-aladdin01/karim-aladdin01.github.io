---
title:  Web Cache Poisoning
categories: [PortSwigger]
tags: [Web Appsec, PortSwigger]
order: 1
icon: fas fa-info-circle
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
Many websites and CDNs perform various transformations on keyed components when they are saved in the cache key. This can include:
- Filtering out specific query parameters or even excluding the whole query string 
- Normalizing input in keyed components
- Removing the port from the `Host` header
These transformations may introduce a few unexpected quirks. These are primarily based around <span style="color:rgb(0, 112, 192)">discrepancies between the data that is written to the cache key and the data that is passed into the application code</span>, even though it all stems from the same input. These cache key flaws can be exploited to poison the cache via inputs that may initially appear unusable.

### Unkeyed port
If the Host is part of the cache-key, but the port is excluded, then you can perform a DOS attack.
Consider the case where a redirect URL is dynamically generated based on the `Host` header.
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
 - Now, All the users who browse to https://vulnerable-website.com will be redirected to https://vulnerable-website.com:3879/en , which is a dud port.  

### 