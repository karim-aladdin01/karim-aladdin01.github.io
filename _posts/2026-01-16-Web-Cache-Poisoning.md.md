---
icon: fas fa-info-circle
order: 1
title: Web Cache poisoning
categories: PortSwigger
tags:
---

- Basic Concepts and Components
	1. **Cache Key**: Parameters that determine cache ==uniqueness== (usually URL + specific headers). It uses key inputs to decide whether two responses are the same.
	2. **Cache Buster**: Query parameters or headers excluded from the cache key (<span style="color:rgb(0, 112, 192)">keyed input</span>) and they are used to force the cache server to load the latest version from the web server while playing with the `Unkeyed Inputs` 
	3. **Unkeyed Inputs**: Headers or parameters processed by the origin server but not included in cache keys
	4. **Web Server:** Back-end (application framework).
	5. **Cache Server:** Front-end (e.g., CDN like Akamai, Cloudflare or reverse proxies like Nginx).
	6. **TTL (Time To Live)**: How long poisoned content remains cached
