---
title:  Web Cache Poisoning
categories: [PortSwigger]
tags: [Web Appsec, PortSwigger]
order: 1
icon: fas fa-info-circle
---

- Basic Concepts and Components
	1. **Cache Key**: Parameters that determine cache **uniqueness** (usually URL + specific headers). It uses key inputs to decide whether two responses are the same.
	2. **Cache Buster**: Query parameters or headers excluded from the cache key (<span style="color:rgb(0, 112, 192)">keyed input</span>) and they are used to force the cache server to load the latest version from the web server while playing with the `Unkeyed Inputs` 
	3. **Unkeyed Inputs**: Headers or parameters processed by the origin server but not included in cache keys
	4. **Web Server:** Back-end (application framework).
	5. **Cache Server:** Front-end (e.g., CDN like Akamai, Cloudflare or reverse proxies like Nginx or Varnish).
	6. **TTL (Time To Live)**: How long poisoned content remains cached

### <span style="color:rgb(255, 0, 0)">The golden rule: Find an unkeyed input that influences the response, poison it, and let the cache do the rest and the exact steps to follow are:</span>
1. Identify a suitable cache oracle
2. Add a cache buster (in some cases, you may not be able to find it and you will do your work on the original response served to other users. So, be careful in such cases)
	![](../assets/img/Pasted%20image%2020260102175425.png)
3. Identify unkeyed inputs with Param Miner
	![](../assets/img/Pasted%20image%2020260102180805.png)
	
	![](../assets/img/Pasted%20image%2020260102175211.png)
	
4. Explore input potential
5. Elicit a harmful response & inject into cache


