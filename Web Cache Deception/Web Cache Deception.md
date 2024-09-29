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

Cache rules often target static resources. If there are discrepancies in how the cache and origin server map the URL path to resources or use delimiters, an attacker may be able to craft a request for a dynamic resource with a static extension that is ignored by the origin server but viewed by the cache.
# Path Mapping for Web Cache Deception

URL path mapping is a process of associating URL paths with resources on a server like files, scripts or command executions. Two common styles are traditional URL mapping and RESTful URL mapping.

Traditional represents a direct path such as:

```html
http://example.com/path/in/filesystem/resource.html
```

REST-style URLs don't directly match the physical file structure but abstract file paths into logical parts:

```bash
http://example.com/path/resource/param1/param2
```

Discrepancies in how the cache and origin server map the URL path to resources can result in web cache deception:

```html
http://example.com/user/123/profile/wcd.css
```

- Origin server using REST can interpret it as a request for /user/123/profile and returns the profile information for user `123`, ignoring wcd.css as a non-significant parameter
- Traditional URL mapping views it as a request for a file named `wcd.css` located in /profile under /user/123. Interprets the URL path as /user/123/profile/wcd.css. If cache is configured to store responses for requests where the path ends in css, it caches and serves profile info as if it were a CSS file.
# Exploiting Path Mapping Discrepancies

Try adding an arbitrary path to the URL of the target endpoint. If the response contains the same sensitive data, it indicates the origin server abstracts the URL and ignores the addition - e.g. `/api/orders/123` returns the same info as `/api/orders/123/foo`.

Test how the cache maps URL paths to resources by modifying the path to attempt to match a cache rule by adding an extension like `.js`. If response is cached it indicates the cache interprets the full URL path with extension and there is a rule to store responses for requests ending in `.js`.

>[!info]
>Burp Scanner detects this as well as Web Cache Deception Scanner.
# Exploiting Delimiter Discrepancies

The `?` character is generally used to separate URL path from query string, but variations can occur between frameworks. Consider:

```html
/profile;foo.css
```

- Spring framework uses the `;` character to add parameters (matrix variables)
- Other frameworks don't, therefore a cache that does not use Spring interprets `;` and everything after part of the path.

If a cache has a rule to store responses for requests ending in .css, it may cache and serve the profile info. For other frameworks like Ruby on Rails:

- `profile` - request is processed by the default HTML formatter
- `/profile.css` - request is recognized as a CSS extension, there is not a CSS formatter, so the request is denied and errors out
- `/profile.ico` - ico not recognized by Ruby, if cache is configured to store responses for requests ending in.ico, it would cache and serve profile info like a static file.

Encoded characters can also be used such as:

```html
/profile%00foo.js
```

- OpenLiteSpeed server uses %00 as a delimiter meaning it would interpret it as /profile
- Other frameworks respond with an error if `%00` is in the URL, but Akamai and Fastly caches interpret `%00` and everything after it as the path.

Try using a delimiter discrepancy to add a static extension to the path viewed by the cache, but not the origin server by identifying a character that is used as a delimiter by the origin server but not the cache.

Find characters used as delimiters by the origin server. Start by adding an arbitrary string to the URL - e.g. `/setting/users/listaaaa`. If response is redirected to the original response, it indicates the request is being redirected.

Add a possible delimiter character between the original path and the arbitrary string - e.g. `/settings/users/list;aaa`. If the response is identical, it indicates the `;` is used as a delimiter. If it matches the response to the path with the arbitrary string, it indicates that the `;` character is not used as a delimiter.

Test if they are also used by the cache by adding a static extension to the end of the path. If the response is cached, it indicates the cache does not use the delimiter and interprets the full URL path and there is a cache rule to store responses for requests ending `.js`.

>[!info]
>Test all ASCII characters and a range of common extensions (exe, ico, css, js). Use Intruder to quickly test the characters and turn off automated character encoding.

Construct an exploit that triggers the static extension cache rule - e.g. `/settings/users/list;aaa.js`:

- Cache interprets path as /settings/users/list;aaa.js
- Origin server interprets path as /settings/users/list

Origin server returns dynamic profile information stored in the cache. 
# Delimiter Decoding Discrepancies

Sites needs to send data in the URL that contains characters that have a special meaning such as delimiters which means encoding them. Some parsers decode them before processing the URL. If decoded, it may be treated as a delimiter, truncating the URL.

Consider `/profile%23wcd.css` which has the `#` character encoded:

- Origin server decodes %23 to # - uses # as a delimiter so it interprets the path as `/profile`
- Cache uses `#` as delimiter, but does not decode - interprets path as `/profile%23wcd.css`. If cache rule exists for CSS files, it stores the response.

Some cache servers decode URL and forward the request with decoded characters, others apply cache rules first based on encoded URL, then decode the URL and forward it. Consider `/myaccount%2fwcd.css`:

- Cache server applies the cache rules based on the encoded path `/myaccount%3fwcd.css` and stores the response as there is a rule for CSS files. Decodes `%3f` to `?` and forwards rewritten request to origin.
- Origin server receives the request `/myaccount?wcd.css` - it uses the `?` character as a delimiter so the path is interpreted as `/myaccount`.

>[!info]
>Use the same testing methodology to identify delimiter discrepancies, but use a range of encoded characters - also test encoded non-printable characters such as `%00`. `%0A` and `%09`.
# Static Directory Cache Rules

Common for servers to store static resources in specific directories. Cache rules target them by matching specific URL path prefixes like `/static`, `/assets`, `/scripts` and `/images`.
# Normalization Discrepancies

Normalization involves converting various representations of URL paths into a standardized format, sometimes including decoding encoded chars and resolving dot-segments. Discrepancies in how the cache and origin server normalize the URL can enable an attacker to construct a path traversal payload that is interpreted differently by each parser.

Consider `/static/..2fprofile`:

- Origin server that decodes slash characters and resolves dot-segments would normalize the path to /profile and return profile info.
- Ccache that does not resolve dot-segments or decode slashes would interpret the path as `/static/..%2fprofile`. If the cache stores responses for requests with the /static prefix, it would cache the and serve the profile info.

>[!info]
>An exploitable normalization discrepancy requires that either the cache or origin server decodes characters in the path traversal sequence as well as resolving dot-segments.

Test the origin server normalization by sending a request to a non-cacheable resource with a path traversal sequence and an arbitrary directory at the start of the path. Look for a non-idempotent method like POST (e.g. modify /profile to /aaa/...%2fprofile).

- If response matches base response and returns profile info, it indicates the path is interpreted as /profile. Origin server decodes the slash and resolves the dot segment.
- If response does not match the base response - e.g. 404 - it indicates the path has been interpreted as `/aaa/..%2fprofile`. Origin server does not decode the slash or esolve the dot segment.

>[!info]
>When testing for normalization, start by encoding only the second slash in the dot-segment. This is important because some CDNs match the slash following the static directory prefix.
>
You can also try encoding the full path traversal sequence, or encoding a dot instead of the slash. This can sometimes impact whether the parser decodes the sequence.

To test how the cache normalizes the path, start by identifying potential static directories. Look for requests with common static directory prefixes and cached responses. Focus on static resources by setting the HTTP history filter to only show messages with 2xx responses and script, images, and CSS MIME types.

Choose a request with a cached response and resend with a path traversal sequence. Choose a request with a response that contains evidence of it being cached - e.g. `/aaa/..%2fassets/js/stockCheck.js`.

- If response is no longer cached, indicates the cache is not normalizing the path before mapping to the endpoint, showing there is a cache rule based on /assets prefix.
- If response is still cached, it may indicate the cache is normalizing the path to `/assets/js/stockCheck.js`.

Try adding a path traversal sequence after the directory path such as `/assets/js/stockCheck.js` to `/assets/..%2fjs/stockCheck.js`:

- If response is no longer cached, it indicates the cache decodes the slash and resolves the dot segment during normalization, interpreting the path as /js/stockCheck.js and shows there is a cache rule based on /assets prefix.
- If response is cached, it may indicate the cache has not decoded the slash or resolved the dot segment, interpreting the path as `/assets/..%2fjs/stockCheck.js`.

In both cases, the response may be cached due to another cache rule (e.g. file extension). To confirm cache rule is based on static directory, replace path after directory prefix with arbitrary string - e.g. `/assets/aaa`. 

If still cached, it confirms the cache rule is based on the `/assets` prefix. If the response does not appear to be cached, it does not necessarily rule out a static directory cache rule since 404 responses may not be cached sometimes.
# Exploiting Normalization By Origin Server

If origin server resolves encoded dot segments, but cache does not, attempt to exploit by constructing a payload according to the following:

```html
/<static-directory-prefix>/..%2f<dynamic-path>
```

Consider the payload `/assets/..%2fprofile`:

- Cache interprets the path as `/assets/..%2fprofile`
- Origin interprets the path as `/profile`

Origin server returns the dynamic profile info, which is stored in cache.

Account details may include the API key of a specific user. Changing the account page URL to `/my-account/abc` may return a 404 not found response, indicating the origin server does not abstract the path. Attempting `/my-accountabc` also results in a 404 with no caching.

Fuzzing with a delimiter list without URL encoding results in the `?` character returning a 200 OK, indicating the origin server only uses `?` as a path delimiter - `?` is generally used as a path delimiter, so attempt normalization discrepancies.

Attempt to submit something like `/aaa/..%2fmyaccount` - if a 200 comes back, it indicates the origin server decodes and resolves the dot segment. Also attempt to look for a static resource path like `resources` - look for any indication of caching from this directory.

If true, attempt an encoded segment - `/resources/..%2fRESOURCE`. Send twice - if a cache hits, it indicates the cache does not decode or resolve the dot segment and has a cache rule based on `/resources` prefix. Test further to make sure it is using the directory to cache by modifying it to `/resources/aaa`.

Craft an exploit with an arbitrary parameter and send to victim:

```html
<script>document.location="https://YOUR-LAB-ID.web-security-academy.net/resources/..%2fmy-account?wcd"</script>
```
# Exploiting Normalization by Cache Server

If cache server resolves encoded dot segments but origin does not, attempt to exploit via the following:

```html
/<dynamic-path>%2f%2e%2e%2f<static-directory-prefix>
```

Path traversal alone may not work. Consider how the cache and origin server interpret it:

- Cache interprets path as `/static`
- Origin interprets path as `/profile%2f%2e%2e%2fstatic`

Origin server is likely to return an error instead of profile information. You must identify a delimiter used by the origin server but not the cache. Test possible delimiters by adding them to payload after dynamic path.

- If origin server uses a delimiter, it truncates the URL path and returns the dynamic info
- If cache does not use the delimiter, it resolves the path and caches response

Consider `/profile;%2f%2e%2e%2fstatic` with origin server using `;` as delimiter:

- Cache interprets it as `/static`
- Origin interprets it as `/profile`

Account details may include the API key of a specific user. Changing the account page URL to `/my-account/abc` may return a 404 not found response, indicating the origin server does not abstract the path. Attempting `/my-accountabc` also results in a 404 with no caching.

Fuzzing with a delimiter list without URL encoding results in the `#`, `?`, `%23` and `%3f` characters returning a 200 OK, indicating the origin server uses them as a path delimiter - `#` should be ignored due to the browser using it as a delimiter before forwarding to cache.

Attempt to submit something like `/myaccount?abc.js` - if the response does not have any caching it indicates the cache also uses it as a path delimiter or the cache does not have a rule based on JS extensions. Repeat using other characters.

Attempt to remove query string and add a directory followed by dot segment to start of path - `/aaa/..%2fmy-account` - if a 404 returns, it indicates the origin server does not decode/resolve the dot segment to normalize. Look at static resources and do the same thing - `/aaa/..%2fresources` - it may contain a cache header.

Test if it's the directory or the extension via `/resources/..%2fYOUR-RESOURCE`. If it no longer caches, it shows the cache decodes and resolves the dot segment and caches based on the /resources.

Test the delimiters via `/my-account?%2f%2e%2e%2fresources` - if it contains no caching, attempt the other characters until it does. Once it does, craft an exploit:

```html
<script>document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account%23%2f%2e%2e%2fresources?wcd"</script>
```
# Exploiting File Name Cache Rules

Certain files (robots, index, favicon) may be cached. Cache rules target them by matching the exact name. Test whether there is a file name cache rule by sending a GET request for a possible file and see if response is cached.

To test how origin server normalizes URL, use same methods as before. To test how the cache normalizes the URL, send a request with a path traversal sequence and a directory before the file - `/aaa%2f%2e%2e%2findex.html`.

- If cached, indicates the cache normalizes the path to `/index.html`
- If not cached, indicates the cache does not decode the slash and resolve the dot segment, interpreting path as `/aaa%2f%2e%2e%2findex.html`.

Since response is cached if request matches the exact name, exploitation is only possible where the cache server resolves encoded dot segments, but the origin does not. Use same method for static directory cache rules - replace static directory prefix with file name.

