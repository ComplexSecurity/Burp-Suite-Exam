![[Web Cache Deception.jpg]]
# Overview

A web cache sits between the origin server and user. When requesting a static resource, it is directed to the cache - if it does not have a copy (cache miss), it is forwarded to the origin server which processes and responds.

Response is sent to the cache before going to the user. The cache typically uses a preconfigured set of rules to determine whether to store the response. Any future requests for the same resource uses the stored copy stored on the cache (cache hit).

When caches receive a request, it decides whether there is a cached response or whether it has to forward to the origin server. The decision made by generating a "cache key" from elements of the request - typically including the URL path and query parameters, but can also have other elements (headers/content type).

If the request cache key matches a previous request, it considers them equal and servers a copy of the cached response.

Cache rules determine what is cached and the time it is cache - often set up to store static resources which don't change and reused many times. Dynamic content is not cached as it is more likely to contain sensitive info.

Some different types of rules include:

- Static file extension rules - match file extension of the requested resource
- Static directory rules - match all URL paths that start with a specified prefix, often used for specific directories that contain static resources (/assets, /images).
- File name rules - match specific file names to target files universally required for web operations and rarely change (robots.txt, favicon.ico).
# Recon

Various response headers may indicate that a response is cached. For example:

- X-Cache provides info about whether a response was served from the cache with typical values being:
	- X-Cache: hit - response served from cache
	- X-Cache: miss - fetched from origin server with response then cached (to confirm, send the request again to see if it hits)
	- X-Cache: dynamic - origin server dynamically generated the content (not suitable for caching)
	- X-Cache: refresh - cached content was outdated and needs refreshed/revalidated

If there is a big difference in response time for the same request, it can indicate that the faster response is served from the cache.