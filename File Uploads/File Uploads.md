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









































# Bypass File Extension Allow List

The null byte injection (%00) may bypass the file extension restriction as this can alter the intended logic of the app. If the app is only allowing files that have the extension ".png", you can supply the following filename:

```bash
malicious.php%00.png
```
# Bypass JPEG Signature Validation

Here, the app ma be checking that the file's contents begin with a certain byte structure. Simply inject the malicious code after the beginning bytes of the file to bypass this validation. Tools can be used to inject this malicious code in the metadata to avoid "breaking" the file/image.

![[File Upload 3.png]]

![[File Upload 4.png]]

# Other Methods

Stored XSS by uploading HTML page with JavaScript. Exploiting vulnerabilities specific to the parsing or processing of different file formats to cause XXE injection (.doc, .xls). Finally, using the PUT method to upload files, use the OPTIONS request method to determine what methods are accepted.