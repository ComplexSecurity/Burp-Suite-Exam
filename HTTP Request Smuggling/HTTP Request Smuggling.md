![[HTTP Request Smuggling.jpg]]
# Important Notes

These techniques are only possible using HTTP/1 request. Browsers and other clients, including Burp, use HTTP/2 by default to communicate with servers that explicitly advertise support for it via ALPN as part of the TLS handshake. As a result, when testing sites with HTTP/2 support, you need to manually switch protocols in Repeater.

>[!info]
>Done via the Request attributes section of the Inspector panel.

When working with TE.CL payloads - to send this request is Burp Repeater, you will first need to go to the Repeater menu and ensure that the "Update Content-Length" option is unchecked.

When submitting request smuggling payloads, it is often required to include an arbitrary body parameter (x=) at the end so that the next normal submitted requested does not break the smuggled request, as it will be appended to the parameter (for example, this would avoid duplicate header issues).

All of the headers in the smuggled request are important such as the Host, Content-Type, and Content-Length. The values for these headers need to be considered when capturing other user's requests, etc..
# Basic CL.TE payload

Include both Content-Length and Transfer-Encoding headers. Here, the front end server is processing the request length using the CL header, which will process the entire body. The back-end server receives the same request but uses the TE header to process the request's length.

Since the terminating byte 0 is provided in the beginning of the body, the rest of the data will be left unprocessed and will remain in the connection queue. The next request that is submitted will be appended to this left over request data. So, the back-end server essentially see's 2 requests in the payload submitted.

![[HTTP 1.png]]

![[HTTP 2.png]]

![[HTTP 3.png]]
# Basic TE.CL Payload

Includes both Content-Length and Transfer Encoding headers. The front-end server is using the TE header to determine the length of the request. The HTTP request smuggler extension can be used here to automatically update the bytes required. It will add in the start and end bytes (9d and 0) in this case.

When the back-end server receives this request, it will use the CL header to determine the length of the request. Since the value is 4, it will leave the rest of the body unprocessed and will remain in the connection queue. The next request that is submitted will be appended to the x=1 parameter. The server here essentially see's 2 requests.

>[!info]
>To send this request in Burp Repeater, you first need to go to the Repeater menu and ensure that the "Update Content-Length" option is unchecked.

![[HTTP 4.png]]

![[HTTP 5.png]]
# Basic TE.TE Payload (Obfuscating TE Header)

In this scenario, both the front-end and back-end servers support the TE header. We need to submit a payload that will obfuscate the TE header and identify if either of the servers reject the TE header and use the CL header for processing.

CL.TE payload - the app times-out when using this method, which means the front-end app is using the TE header.

![[HTTP 6.png]]

TE.CL payload - the app does not time out and the backend server processes the request using the CL header. This is the direction for exploitation since the obfuscated TE header prevented the back-end server from using it.

![[HTTP 7.png]]

![[HTTP 8.png]]
# Confirming CL.TE Vulnerability via Differential Responses

A request to GET / endpoint normally returns the home page of the app. A request to a random endpoint like GET/404 will return a 404 Not Found response. 

This behaviour will be used to identify if the request smuggling payload works. The payload consists of a smuggled request that will be sent to the /404 endpoint, which would return a 404 error. The follow up request will be submitted to the GET / endpoint, however, instead of returning the home page, a 404 error is returned, proving the smuggled payload worked.

![[HTTP 9.png]]

![[HTTP 10.png]]
# Confirming TE.CL Vulnerability via Differential Responses

A request to GET / endpoint normally returns the home page of the app. A request to a random endpoint like GET /404 or POST /404 will return a 404 Not Found response. This behaviour can be used to identify if the request smuggling payload works.

The payload consists of a smuggle request that will be sent to the /404 endpoint, which would return a 404 error. The CL header contains the value of 4, which covers the data up to the beginning of line 20.

The follow up request receives a 404 error, even though the request is made to the home page / of the application. This proves that the payload worked as expected.

![[HTTP 11.png]]

![[HTTP 12.png]]
# Using TE.CL Payload to Bypass Front-End Controls

The /admin endpoint is only available to local users. A TE.CL payload was crafted where the smuggled request will be to the /admin endpoint, the Host header containing the value of localhost.

Submitting a follow up request will return the normal response to the /admin request. You can send another request to delete the user, Carlos:

![[HTTP 13.png]]
# Using CL.TE Payload to Capture Other User's Requests

You need to identify if there is a request with parameters whose values are being reflected or stored in a response. In this scenario, the app has a blog where users can leave comments that can be viewed in the application. The "comment" parameter was intentionally injected last in the payload so the follow request will appear in the app's UI.

The CL header in the smuggled request payload matters and needs to be adjusted in order to capture all the data in the follow up request. A lot of trial/error can happen here. Since the victim user's request contains the session cookie header, this will be captured in the smuggled request and can be used to access the app as that user.

![[HTTP 14.png]]

![[HTTP 15.png]]

The rest of the payloads for the other practitioner labs can be found in the sections of the PDF document. The following can be found:

- Exploiting HTTP request smuggling to bypass front-end security controls, CL.TE vulnerability
- Exploiting HTTP request smuggling to reveal front-end request rewriting
- Exploiting HTTP request smuggling to deliver reflected XSS
- Response queue poisoning via H2.TE request smuggling
- H2.CL request smuggling
- HTTP/2 request smuggling via CRLF injection
- HTTP/2 request splitting via CRLF injection
- CL.0 request smuggling
- Exploiting HTTP request smuggling to perform web cache poisoning
- Exploiting HTTP request smuggling to perform web cache deception

