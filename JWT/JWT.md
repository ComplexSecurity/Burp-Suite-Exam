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