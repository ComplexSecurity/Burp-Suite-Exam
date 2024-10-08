#JWT #Completed

![[JWT.avif]]
# JSON Web Tokens (JWTs)

JWTs have 3 parts separated by a period (.):

- Header
- Payload
- Signature

```json
eyJraWQiOiI5MTM2ZGRiMy1jYjBhLTRhMTktYTA3ZS1lYWRmNWE0NGM4YjUiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTY0ODAzNzE2NCwibmFtZSI6IkNhcmxvcyBNb250b3lhIiwic3ViIjoiY2FybG9zIiwicm9sZSI6ImJsb2dfYXV0aG9yIiwiZW1haWwiOiJjYXJsb3NAY2FybG9zLW1vbnRveWEubmV0IiwiaWF0IjoxNTE2MjM5MDIyfQ.SYZBPIBg2CRjXAJ8vCER0LA_ENjII1JakvNQoP-Hw6GG1zfl4JyngsZReIfqRvIAEi5L4HV0q7_9qGhQZvy9ZdxEJbwTxRs_6Lb-fZTDpW6lKYNdMyjw45_alSCZ1fypsMWz_2mTpQzil0lOtps5Ei_z7mM7M8gCwe_AGpI53JxduQOaB5HkT5gVrv9cKu9CsW5MS6ZbqYXpGyOG5ehoxqm8DL5tFYaW3lB50ELxi0KsuTKEbD0t5BCl0aCR2MBJWAbN-xeLwEenaqBiwPVvKixYleeDQiBEIylFdNNIMviKRgXiYuAvMziVPbwSgkZVHeEdF5MQP1Oe2Spac-6IfA
```

Header and payload are base64-encoded JSON objects. Headers contain metadata about the token and payload contains actual "claims":

```json
{
    "iss": "portswigger",
    "exp": 1648037164,
    "name": "Carlos Montoya",
    "sub": "carlos",
    "role": "blog_author",
    "email": "carlos@carlos-montoya.net",
    "iat": 1516239022
}
```

Servers that issue tokens generate the signature by hashing headers and payloads. Some may also encrypt the hash generated. It also involves a secret signing key. Typically, the following can be stated:

- Since signature is derived from the rest of the token, changing a single byte results in a mismatch
- Without knowing the key, it should be impossible to generate the correct signature

JWT can be extended via JSON Web Signature (JWS) and JSON Web Encryption (JWE) specifications. 

>[!info]
>JWTs are usually either a JWS or JWE token. "JWT" tokens mostly mean a JWS token when talking about JWTs generally. 

The actual content of the JWEs are encrypted rather than encoded. 
# Recon

To identify JWTs, look for session tokens that start with "eY" and include three sections separated by periods (.). The [JWT Editor](https://portswigger.net/bappstore/26aaa5ded2f74beea19e2ed8345a93dd) can be used for easier editing inside of Burp Suite.
# Exploiting Flawed Signature Verification

Servers do not usually store any info about JWTs they issue, rather each token is a self-contained entity. The server rarely knows the original contents of the token, or the original signature used. If servers do not verify the signature properly, you can make changes to the rest of the token. 

As an example, you can modify the "isAdmin" role in the payload and try submitting it:

```json
{
    "username": "carlos",
    "isAdmin": true
}
```
## Accepting Arbitrary Signatures

There is typically one method for verifying tokens and one that decodes them. Devs can confuse them and only pass incoming tokens to the decode() method, meaning verification never happens.

As an example, try decoding the JWT and inspecting the payload inside:

```json
{
    "iss": "portswigger",
    "exp": 1727983510,
    "sub": "administrator"
}
```

If it includes a username parameter, try changing it to a different user and re-submitting a request. If no verification takes place, it may authenticate as a different user.
## Accepting No Signature

JWT headers contain `alg` parameters which tells the server what algorithm to use to sign and verify the token. Most servers has no option but to trust user-controllable input from the token which may not be verified yet, allowing you to influence how the server verifies the token.

JWTs can be left unsigned by specifying the `alg` parameter to `none` which indicates an unsecured JWT. Most servers reject these tokens, but they can be bypassed via obfuscation such as mixed capitalization - `nOnE` - or by unexpected encoding.

>[!info]
>Unsigned tokens must have a trailing dot to terminate it properly.

As an example, the payload may contain a username. Try decoding it and changing the username to "administrator" and the algorithm to `none`:

```json
{  
    "kid": "5f61c1de-665e-4c7a-b153-132c137e5599",  
    "alg": "none"  
}
```

```json
{
    "iss": "portswigger",
    "exp": 1727984152,
    "sub": "administrator"
}
```

Remove the signature from the original JWT by leaving the trailing dot and re-submitting. If the server accepts unsecure JWTs, it will authenticate as the admin user.
# Brute Forcing Secret Keys

Some algorithms like HS256 use a standalone string as the secret key. If easily guessable or brute-forceable, attackers can create JWTs with any header and payload values and use the key to re-sign with a valid signature. It may be trivial to brute-force using [well-known secrets](https://raw.githubusercontent.com/wallarm/jwt-secrets/refs/heads/master/jwt.secrets.list) as a wordlist.

To brute-force secret keys, use Hashcat:

```bash
hashcat -a 0 -m 16500 [JWT] [WORDLIST]
```

It signs the header and payload using each secret and compares the signature. If identified, use it to generate valid signatures for any JWT header and payload. If an extremely weak secret is in use, try brute-forcing it character-by-character rather than a wordlist.

If a JWT is found to be using a symmetric algorithm such as HS256, try copying the JWT and brute forcing the secret via hashcat such as:

```bash
hashcat -a 0 -m 16500 JWT.txt /path/to/jwt.secrets.list
```

If a key comes back, try using the JSON Web Tokens plugin to modify the payload to become an admin and re-sign it using the found key such as `secret1`:

```json
{
  "iss": "portswigger",
  "exp": 1728047784,
  "sub": "administrator"
}
```

![[Secret1.png]]
# Header Parameter Injections

The `alg` parameter is the only mandatory header parameter, but often contains other parameters including:

- `jwk` - provides embedded JSON object representing the key
- `jku` - provides URL from which servers can fetch a set of keys containing the correct key
- `kid` - provides an ID that servers can use to identify correct key in cases with multiple keys to choose from, depending on key may have a matching `kid` parameter.
# JWK Header Injection

Specification describes an optional `jwk` header which is used to embed a public key within the token itself in JWK format:

```json
{
    "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
    "typ": "JWT",
    "alg": "RS256",
    "jwk": {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
        "n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"
    }
}
```

Servers should use a limited whitelist of public keys to verify JWT signatures. If misconfigured, servers can use any key embedded in the `jwk` parameter which can be exploited by signing a modified JWT using an RSA private key you own and embedding the matching public key in the header.

To perform it:

1. Go to JWT Editor tab
2. Generate a new RSA key
3. Send JWT request to Repeater
4. Modify the token payload
5. Click Attack and "Embedded JWK" and select the RSA key
6. Send the request

>[!info]
>Manual exploitation can be done by adding the header yourself, but you need to update the JWT's kid header parameter to match the kid of the embedded key.

As an example, you may receive a JWT with a `jwk` header. Try sending it over the JWT Editor extension, generating a new RSA key, modifying the payload to become an administrator and signing the JWT with the new key.

>[!info]
>For the labs, this exploit works when the algorithm the app is using to sign the JWT token is asymmetric like RS256.g
# JKU Header Injections

Some servers let you use the `jku` header parameter to reference a JWK set containing the key which the server fetches the relevant key from when verifying:

```json
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",
            "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"
        },
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "d8fDFo-fS9-faS14a9-ASf99sa-7c1Ad5abA",
            "n": "fc3f-yy1wpYmffgXBxhAUJzHql79gNNQ_cb33HocCuJolwDqmk6GPM4Y_qTVX67WhsN3JvaFYw-dfg6DH-asAScw"
        }
    ]
}
```

JWK Sets are sometimes exposed publicly via a standard endpoint, such as `/.well-known/jwk.json`. More secure sites only fetch keys from trusted domains, but some can be used with URL parsing discrepancies to bypass the filtering.

As an example, try creating a new RSA key in JWT Editor and copying it. Then, replace the contents of the body in the exploit server to contain the keys:

```json
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "893d8f0b-061f-42c2-a4aa-5056e12b8ae7",
            "n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw"
        }
    ]
}
```

For it to work, replace the value of the `kid` parameter in the request to the one in the exploit server and add a new `jku` parameter to the header and set its value to the exploit server. 

```json
{
    "kid": "42971290-a070-4fe0-924a-af4e513b379d",
    "alg": "RS256",
    "jku": "https://exploit-0a2700f504d01bc18082d429016f00f8.exploit-server.net/exploit"
}
```

Finally, modify the payload, sign it with the RSA key without modifying headers and send the request.

>[!info]
>For the lab, this exploit works when the algorithm the application is using to sign the JWT token is Asymmetric like - "RS256"
# Kid Header Path Traversal

Severs can use cryptographic keys for signing different kinds of data. The header may contain a `kid` parameter to help identify which key to use. Verification keys are often stored as a JWK set where the server may simply look for the JWK with the same `kid` as the token.

The ID is just an arbitrary string that the dev chooses. A dev may use the `kid` parameter to point a particular entry in a databases or the name of a file. If the parameter is vulnerable to path traversal, you can force the server to use an arbitrary file from the filesystem as the key:

```json
{
    "kid": "../../path/to/file",
    "typ": "JWT",
    "alg": "HS256",
    "k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc"
}
```

If the server supports JWTs signed using a symmetric algorithm, an attacker can point the `kid` parameter to a predictable, static file and sign the JWT using a secret matching the contents of the file. The `dev/null` could be used which is an empty string, meaning it would result in a valid signature with an empty string.

>[!info]
>JWT Editor does not allow this, but try Base64 encoding a null byte to bypass it.

For example, try generating a new symmetric key in JWK format and replace the `k` property with an empty string or base64 encoded null byte - `AA==`.  Attempt to change the value of `kid` with a path traversal sequence pointing to /dev/null.

Sign with the symmetric key and try submitting it.
# Other Header Parameters

Some other headers includes:

- cty (Content Type) - used to declare a media type for content in the JWT payload. Usually omitted from the header, but the underlying parsing library may support it. If bypassing signature verification is possible, try injecting a `cty` header to change the content type to `text/xml` or `application/x-java-serialized-object` to open new vectors like XXE and deserialization.
- x5c (X.509 Certificate Chain) - used to pass X.509 public key certificate or certificate chain used to sign the JWT. Can be used to inject self-signed certs. Parsing these certs can also introduce vulnerabilities such as [CVE-2017-2800](https://talosintelligence.com/vulnerability_reports/TALOS-2017-0293) and [CVE-2018-2633](https://mbechler.github.io/2018/01/20/Java-CVE-2018-2633).
# Algorithm Confusion

You may be able to forge valid JWTs by signing the token using an algorithm the developers do not anticipate. JWTs have a range of algorithms available including HS256 which uses a symmetric key which uses a single key to sign and verify the token. 

Others include RS256 which are asymmetric consisting of a private key which is used to sign and a public key used to verify. To exploit an algorithm confusion attack:

1. Obtain server's public key
2. Convert public key to a suitable format
3. Create a malicious JWT with a modified payload and the `alg` header set to HS256
4. Sign the token with HS256, using public key as the secret

Some public keys are exposed as JWK objects via an endpoint like `/jwks.json` or `/.well-known/jwks.json`:

```json
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",
            "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"
        },
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "d8fDFo-fS9-faS14a9-ASf99sa-7c1Ad5abA",
            "n": "fc3f-yy1wpYmffgXBxhAUJzHql79gNNQ_cb33HocCuJolwDqmk6GPM4Y_qTVX67WhsN3JvaFYw-dfg6DH-asAScw"
        }
    ]
}
```

It may be possible to extract it from a pair of existing JWTs.

When verifying the signature of a token, it will use its own copy of the key from its local filesystem or database which can be stored in a different format. For an attack to work, the version of the key used to sign the JWT must be identical to the server's local copy. Every single byte must match, including non-printable characters.

For example, you may need a key in X.509 PEM format - can convert a JWK to a PEM using JWT Editor:

1. Click New RSA key and paste the JWK obtained earlier
2. Select PEM radio button and copy resulting PEM key
3. Base64 encode the PEM
4. Click new symmetric key
5. Replace generated value for `k` with base64-encoded PEM key you copied.
6. Save the key

Once public key is in suitable format, modify the JWT and make sure the `alg` header is set to HS256. Finally, try signing the token using the HS256 algorithm with the RSA pubic key as the secret.

If the public key is not available, test for it by deriving the key from a pair of existing JWTs using a tool like [jwt_forgery](https://github.com/nu11secur1ty/rsa_sign2n/blob/main/jwt_forgery.py). To run it:

```bash
docker run --rm -it portswigger/sig2n TOKEN1 TOKEN2
```

It uses JWTs to calculate potential values of `n`. Only one of them matches the value of `n` used by the server's key. For each value, the script outputs:

- Base64-encoded PEM key in both X.509 and PKCS1 format
- Forged JWT signed using each of the keys

Use Repeater to send a request containing each of the forged JWTs - only one will be accepted by the server. If so, use the matching key to construct an algorithm confusion attack.