![[JWT.avif]]
# Recon

The JWT Burp Extension: [https://portswigger.net/burp/documentation/desktop/testing-workflow/session-management/jwts](https://portswigger.net/burp/documentation/desktop/testing-workflow/session-management/jwts)
# JWT Auth Bypass via Unverified Signature

The app is not properly verifying the signature of the JWT token. Simply manipulate the JWT's payload and use it to attack the application. Use the JWT Editor plugin to easily manipulate the JWTtokens. Changing the value of the "sub" key in the payload section will gain us access to the admin account.

![[JWT 1.png]]
# JWT Auth Bypass via Flawed Signature Verification

The app is trusting the algorithm sent in the header of the JWT token. Change the "alg" key in the header to the value of "none". To bypass dis-allow list validations, it may be required to obfuscate the value "none" -> "NonE", etc..

When using the JWT token in the attack, omit the entire signature of the token but leave the preceding dot "." at the end:

![[JWT 2.png]]
# JWT Auth Bypass via Weak Signing Key

The app is using a weak secret to both sign and verify tokens. The secret can be brute forced using a tool like "hashcat". Once the secret is cracked, you can use it to create your own symmetric key using Burp's JWT Editor Keys and use it to sign our tampered tokens to attack the app.

To create symmetric keys with the known secret:

- Go to the "JWT Editor Keys" tab and select the option "New Symmetric Key"
- Base64 encode the known secret and include it within the "k" key's value of the generated symmetric key

>[!info]
>For the labs, this exploit works when the algorithm the app is using to sign the JWT token is symmetric like HS256.

The following wordlist can be used for brute forcing - [https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list)
# JWT Auth Bypass via JWK Header Injection

The server supports the JSON Web KEy (JWK) header which provides an embedded JSON object representing the key. The server fails to ensure the key is coming from a trusted source. The original JWT token used in the app is using the RS256 algorithm.

Using Burp, you can create a new RSA key to use in our attack. Manipulate the JWT payload then sign the tampered token using that same RSA key, ensuring to also select the option that updates the "alg", "typ" and "kid" parameters automatically.

Burp has the option to use the "embedded JWK" attack method, which will automatically update the header with the JWK key and we can choose to use the RSA key. Now that the JWT token is signed with our own privateRSA key, when the server uses the public key in the JWK header we injected, the verification will succeed and the exploit will work.

>[!info]
>For the labs, this exploit works when the algorithm the app is using to sign the JWT token is asymmetric like RS256.g
# JWT Auth Bypass via JKU Header Injection

The server supports the JSON Web Key Set URL (JKU) header, which provides a URL from which servers can fetch a set of keys containing the correct key. The server fails to check that the provided URL is coming from a trusted domain. The original JWT token used in the app is using the RS256 algorithm. Using JWT Editor Keys, you can create a new RSA key to use.

You can place the new created RSA key within the Exploit Server to host the key:

- Go to the JWT Editor Keys tab and right click on the created RSA key and select the option "Copy Public Key as JWK".
- Inject it within a JSON "keys" array in the exploit server:

```json
{
    "keys": [
      {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "b1d95cf1c2a9",
        "n": "sfVWMUmmiXR_7K1SRWoqQ"
      }
    ]
}
```

In Repeater, change the "kid" value of the JWT tokens you are manipulating so that it matches the same value of the key generated, then inject the "jku" header that points to the exploit server's URL that is hosting the RSA key.

Finally, when manipulating the JWT token, sign it with the generated RSA key. Now that the JWT token is signed with your own private RSA key, when the server reaches out to the URL within the JKU header it will grab the public key with the same "kid" value and use it to verify the token which will now succeed.

>[!info]
>For the lab, this exploit works when the algorithm the application is using to sign the JWT token is Asymmetric like - "RS256"
# JWT Auth Bypass via Kid Header Path Traversal

The "kid" header which is a string that indicates the key that was used to digitally sign the JWT is vulnerable to a path traversal attack. The server is using a symmetric key (HS256) algorithm to sign the token, which means a single key is used to both sign and verify the token.

If you can point the kid header to the /dev/null file, this will return an empty string:

```bash
"kid": "../../../../../../../dev/null"
```

Then, use JWT Editor Keys to create a new symmetric key and change the "k" value to "AA\=="" which is a base64 encoded null byte. Sign the tampered JWT token with this new symmetric key and use a path traversal payload in the "kid" header to attack the app. Essentially, the same "key" is being used here to both sign and verify the token so the exploit will work.

>[!info]
>For the lab, this exploit works when the algorithm the application is using to sign the JWT token is Symmetric like - "HS256"
# Algorithm Confusion - Public Key Exposed

The server is using the "alg" header to determine which algorithm to use when verifying the token, however only the RS256 or HS256 is allowed. Originally, the JWT token is using the RS256 token (2 different keys) to sign and verify the token. The exact public key used to verify the token is being exposed within the app's web root.

You can use this same exposed key to generate a new symmetric key using JWT Editor Keys. Use this generated symmetric key to sign the tampered JWT token, while also changing the "alg" to the value of "HS256". Since the algorithm "HS256" uses the same key to both verify and sign the token this exploit will work, as the server will fetch the same key we used to sign the token to verify the signature.
# Algorithm Confusion - Public Key Not Exposed

See the steps in the section/document for more details.


![[JWT attacks.pdf]]


