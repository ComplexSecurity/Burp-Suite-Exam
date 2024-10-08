#HTTP-Host-Header #Completed

![[HTTP Host Header Attacks.png]]
# HTTP Host Header

The Host header is mandatory as of HTTP/1.1 and specifies the domain the client wants to access. For example:

```bash
GET /web-security HTTP/1.1
Host: portswigger.net
```

It helps identify which back-end component the client wants to communicate with. If not specified, it could lead to issues when routing incoming requests. If a server hosts multiple sites, each one will have a domain name, but all share a common IP.

A common scenario is when sites are hosted on distinct back-end servers, but traffic between is routed through an intermediary system, prevalent in cases where clients access the website via a CDN. Even though sites are hosted on separate back-end servers, all domain names resolve to one IP of the intermediary component.

When a browser sends a request, the target URL is resolved to the IP of a server. When the server receives it, it refers to the Host header to determine the intended back-end.
# Recon

To test whether a website is vulnerable to attack via the HTTP Host header, you will need an intercepting proxy, such as Burp Proxy and manual testing tools like Repeater and Intruder. 

In short, you need to identify whether you are able to modify the Host header and still reach the target application with your request. If so, you can use this header to probe the app and observe what effect this has on the response.

First, test what happens when supplying an arbitrary, unrecognized domain name. Some proxies derive the target IP from the Host header. Burp Suite maintains separation between Host header and target IP address, allowing supply of malformed Host header.

Sometimes you can access the target when using an unexpected Host header, as servers are sometimes configured with a default or fallback option. If so, try studying what the app does with the Host header and whether it is exploitable.

If the request is blocked for security, understand how it parses the Host header. Some parsing algorithms omit the port from the Host header. If you can supply a non-numeric port, you can leave the domain untouched and inject a payload via the port:

```html
GET /example HTTP/1.1
Host: vulnerable-website.com:bad-stuff-here
```

Some sites will apply matching logic to allow for subdomains. Try to bypass the validation entirely by registering a domain name that ends with the same sequence:

```html
GET /example HTTP/1.1
Host: notvulnerable-website.com
```

Or try taking advantage of a less-secure subdomain:

```html
GET /example HTTP/1.1
Host: hacked-subdomain.vulnerable-website.com
```

Additionally, try adding duplicate Host headers. It is common for one of the two headers to be given precedence, overriding the other value. If systems disagree on which, it leads to discrepancies:

```html
GET /example HTTP/1.1
Host: vulnerable-website.com
Host: bad-stuff-here
```

You could use the first header to ensure the request is routed to the target and use the second to pass a payload into server-side code.

Many servers also understand requests for absolute URLs. The request line should be given precedence when routing but not always the case. You may be able to exploit the discrepancies such as:

```html
GET https://vulnerable-website.com/ HTTP/1.1
Host: bad-stuff-here
```

Try indenting HTTP headers with a space character. Some servers interpret the indented header as a wrapped line and treat it as part of the preceding header's value or it may ignore it altogether. There may be discrepancies between different systems. For example:

```html
GET /example HTTP/1.1
    Host: bad-stuff-here
Host: vulnerable-website.com
```

Site may block requests with multiple Host headers, but bypassed by indenting one. If the front-end ignores it, the request is processed as ordinary. If the back-end ignores the leading space and gives precedence to the first  header, you may pass values via the wrapped Host header.

More options include injecting the payload via one of several other HTTP headers. The front-end may inject the `X-Forwarded-Host` header, containing the original value of the Host header. Many frameworks refer to this header instead. 

Sometimes, you can use `X-Forwarded-Host` to inject malicious input and circumvent any validation on the Host header:

```html
GET /example HTTP/1.1
Host: vulnerable-website.com
X-Forwarded-Host: bad-stuff-here
```

Some other headers include:

- X-Host
- X-Forwarded-Server
- X-HTTP-Host-Override
- Forwarded
# Basic Password Reset Poisoning

If the app exposes a password reset functionality, determine how the app is generating the password reset URL. It may be possible to inject a malicious domain in that URL by manipulating the Host header or including other headers such as X-Forwarded-Host.

If the URL sent to the user is dynamically generated based on controllable input, an attacker can submit a password reset request and change the Host header. Victim receives a genuine password reset email with a domain pointing to our site. If clicked, the token may be sent to us.

For example, try using the "Forgot password" functionality and check if it sends an email. Analyse the password reset link for any interesting parameters such as `temp-forgot-password-token`. Additionally, check if it includes the URL.

Try performing the full password reset process and analyse the requests in Repeater. Check for interesting requests such as `POST /forgot-password` and check if it includes the username or email of the account being reset.

Check if the app uses any user controllable data via the link generation. Try changing the host header such as `example.com` and check the reset email for an indication of the Host value.

```bash
https://example.com/forgot-password?temp-forgot-password-token=4h5gjkc3y59h2ctshua86505rrl87a73
```

If changed, change the Host header to a domain and the username to the victim. Analyse the logs and check for any password reset tokens.
# Web Cache Poisoning via Ambiguous Requests

When probing, you may find some seemingly vulnerable behaviour that is not directly exploitable such as the Host header being reflected in the response without HTML encoding or even inside script imports. Reflected XSS via a Host header is not typically a finding since you can't force the user's Host header to change.

If the site uses a web cache, it can be exploited. To do it, elicit a response from the server that reflects a payload while preserving a cache key that is mapped to other user's requests. If successful, try and cache the malicious response to serve to other users.

Standalone caches typically include the Host header in the cache key. 

For example, analyse the responses and test if the Host header is validated by tampering with it. If not successful, scan and enumerate for unkeyed parameters and add a cache buster (`/?cb=123`). Observe if there are any caching headers such as `Age` or `X-Cache`.

Attempt to add a second Host header and observe the responses to see if it is ignored. Check if it is reflected anywhere in the response to load other files such as JavaScript files.

Try removing the second Host header and sending the response again to check if the response is cached and contains the injected value. If it caches the injection, create a malicious file matching the one using the injected Host value such as:

```javascript
alert(document.cookie)
```

Try adding a second Host header containing the malicious domain and send until it is stored in the cache. Attempt to navigate to the URL with the cachebuster in the browser. 

To exploit victims, remove cachebusters and replay the request until the cache is poisoned.
# Host Header Authentication Bypass

It's common for sites to restrict access to certain functionality to internal users only. Some sites access control features make flawed assumptions allowing you to bypass these restrictions by making simple modifications to the `Host` header.

For example, the `robots.txt` file may reveal an admin endpoint. If the `/admin` endpoint is only available to localhost users, try injecting the local host values  such as:

- `Host: localhost`
- `X-Forwarded-Host: localhost`

>[!info]
>If failing, try encoding/obfuscating the header value or adding double Host headers.

A session handling rule can help in Burp Suite by automatically adding/modifying the `Host` header to be the same value for all requests sent.

In some cases, public and private sites are on the same server. Servers typically have a public and private IP address such as:

```bash
www.example.com: 12.34.56.78
intranet.example.com: 10.0.0.132
```

Sometimes the internal site may not have a public DNS record, but an attacker can typically access any virtual host on any server they have access to, provided they can guess the hostnames. If a domain name is discovered, it can be requested directly.

Intruder could also be used to brute force virtual hosts using a wordlist.
# Routing-Based SSRF

Routing-based SSRF relies on exploiting the intermediary components that are prevalent in many cloud-based architectures, including in-house load balancers and reverse proxies. If insecurely configured to forward requests based on an unvalidated `Host` header, they can be manipulated into misrouting requests to another system.

Burp Collaborator can be used to help identify them. Try supplying the domain in the `Host` header and analyse if a DNS lookup happened from the target server or another in-path system which may indicate you can route requests to arbitrary domains.

If so, try to exploit the behaviour to access internal systems by identifying private IP addresses used on the internal network. In addition, also try scanning hostnames belonging to the company to check if any resolve to a private IP address. 

Finally, try identifying valid IP addresses by simply brute forcing standard private ranges including `192.168.0.0/16`.

For example, try replacing the original Host header value with another domain and determine if the app initiates a request to the domain. Burp Collaborator can be used here for testing.  If this works, then it means the app is vulnerable to SSRF through the Host header.

Try sending to Intruder and adding a private subnet range such as `192.168.0.0` and fuzz the last octet with a number list to determine a valid internal address. Analyse the response and look for a way to delete users. It may require a POST request, but a GET request equivalent could be done including:

```html
GET /admin/delete?csrf=QCT5OmPeAAPnyTKyETt29LszLL7CbPop&username=carlos
```

Change the request method to POST and send it.
# SSRF via Flawed Request Parsing

The `Host` header can be used to perform an SSRF attack. Try replacing the Host header value with another domain and determine if the app initiates a request to the domain (i.e. Collaborator). If it does not appear to query the domain, try using an additional `Host` header or `X-Forwarded-Host` header

Another technique to try is to supply an absolute URL in the request line:

- `GET https://vulnerable-website.com/admin HTTP/2`
- `Host: bad-stuff-here - 192.168.0.1`

If this works, then it means the app is vulnerable to SSRF through the Host header, while injecting the absolute URL of the application in the request line.

For example, try accessing the home page by supplying the absolute URL such as:

```html
GET https://vulnerable-website.com/admin HTTP/2
```

If successful, try adding another Host header and observe if it bypasses the block and returns a timeout. If so, the absolute URL may be validated instead of `Host` value. To confirm, try submitting a Collaborator domain as the `Host` value and observe any queries made.

If successful, try accessing internal IPs by fuzzing the last octet such as `192.168.0.x` and observing the responses for a difference. Analyse the response if successful for any admin functionality like deleting users.

If a POST request is needed, try changing the absolute URL by appending `/admin` to it and adding the appropriate parameters such as:

```bash
GET https://YOUR-LAB-ID.web-security-academy.net/admin/delete?csrf=QCT5OmPeAAPnyTKyETt29LszLL7CbPop&username=carlos
```

Try submitting it as a GET request. If it fails, change the request method to POST or whatever is required.
# Host Validation Bypass via Connection State Attack

Some website reuse connections for multiple request/response cycles with the same client. Poorly implemented servers sometimes work on the assumption that certain properties, such as the `Host` header are identical for all HTTP/1.1 requests sent over the same connection.

It may be true for browsers but not necessarily the same in Repeater. 

You may encounter servers that only perform thorough validation on the first request they receive. Try bypassing the validation by sending an innocent-looking initial request then following up with a malicious one down the same connection.

Some reverse proxies use the `Host` header to route requests to the correct back end. If it assumes all requests on the connection are intended for the same host as the initial request, it can provide a useful vector including routing-based SSRF, password reset poisoning and cache poisoning.

For example, try changing the host header to an IP address such as `192.168.0.1` and the path to `/admin` and check what happens (i.e. redirection to home page). Try duplicating the tab and add them toa  group in Repeater.

For the first request, try changing the path to `/` and the `Host` back to the original and then try sending the group in sequence as a single connection. Analyse what happens with the second request to `/admin` - check if it succeeds.

Study the response for anything interesting such as a function to delete users and the attributes associated with it such as:

- `/admin/delete`
- `username` as input
- `csrf` token present

Use them to craft a POST request and potentially access admin functionality by sending requests in sequence down a single connection.
# SSRF via Malformed Request Line

Custom proxies can fail to validate request lines properly which can allow you to supply unusual, malformed input. A reverse proxy may take the path from the request line, prefix it with `http://backend-server` and route it to the upstream URL. If the path starts with a character other than `/`, it may end up with something like:

```html
GET @private-intranet/example HTTP/1.1
```

The resulting URL could be:

```html
http://backend-server@private-intranet/example
```

Which most libraries interpret as a request to access `private-internet` with the username `backend-server`.

























Poorly implemented HTTP servers sometimes work on the dangerous assumption that certain properties, such as the Host header, are identical for all HTTP/1.1 requests sent over the same connection. 

For example, you may occasionally encounter servers that only perform thorough validation on the first request they receive over a new connection. In this case, you can potentially bypass this validation by sending an innocent-looking initial request then following up with the malicious one down the same connection.

To attack the app using the connection state attack::

- Use Repeater to place 2 different tabs into a new group and change the send mode to "Send group in sequence (single connection)".
- Ensure the first tab contains the normal HTTP request. The second tab can contain the malicious HTTP request. Same technique as previous examples, can be used here.

