![[Insecure Deserialization.jpg]]
# Recon

The Java serialization format can be the following. Serialized Java Objects always begin with the same bytes:

- Hexadecimal: ac ed
- Base64: ro0

For PHP serialization format, serialized objects are usually base-64 encoded. As an example, consider a User object with the attributes:

- $user->name = "carlos";
- $user->isLoggedIn = true;

When serialized, the object may look something like this:

```javascript
O:4:"User":2:{s:4:"name":s:6:"carlos"; s:10:"isLoggedIn":b:1;}
```
# Tools

- Ysoserial ([https://forum.portswigger.net/thread/ysoserial-stopped-working-b5a161f42f](https://forum.portswigger.net/thread/ysoserial-stopped-working-b5a161f42f))
- PHP Generic Gadget Chains (PHPGGC)
- Java Deserialization Scanner - [https://portswigger.net/bappstore/228336544ebe4e68824b5146dbbd93ae](https://portswigger.net/bappstore/228336544ebe4e68824b5146dbbd93ae)

# Modifying Serialized Objects - PHP

Identify if there is a PHP Object that is being used in any HTTP requests sent to the app. Decode the object and check if there are any sensitive fields such as "isAdmin", "role" or more. If there is, modify the object and re-encode it.

For example:

```javascript
O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}
```

Modify the serialized object, encode it and submit it back in the HTTP request. Here, the "isAdmin" field was changed to the value of 1, which equals to true:

```javascript
O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:1;}
```