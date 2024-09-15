![[File Uploads.webp]]
# Recon

Identify any file upload functionalities on the app either direct or indirect accessible functions. Then, use the test cases here from the labs to identify vulnerabilities on the file upload process.

If you find a file upload functionality on an app, try out the following techniques. However, much more can be done depending on which part of the file the app is not validating, how the app is using the file (e.g. interpreters like XML parsers) and where the file is being stored.

In some cases, uploading the file is enough to cause damage. However, in other cases, the file will need to be executed somehow, such as requesting the file using an HTTP request. If the uploaded file is available within the webroot, try submitting HTML/JavaScript file, then view the file - it may introduce an easy XSS vulnerability.
# No Protections

The app has no protections against malicious file uploads and the server is configured to execute these files as code. Upload a file with the following properties:

- File extension - .php
- Content-Type header - application:x-httpd-php
- File Content - \<?php echo file_get_contents('/path/to/target/file'); ?>

```php
<?php echo file_get_contents('/path/to/target/file'); ?>
```

Now, view the uploaded file within the web root and you should see the contents of the file specified. The "viewing" of the file here caused its execution:

![[File Upload 1.png]]

![[File Upload 2.png]]
# Content-Type Header Restriction Bypass

The app is only allowing MIME types for JPEG. However, it does not block malicious file types from being uploaded such as .php. Simply keep the Content-Type header, within the subpart of the request body, with the allowed MIME type and upload the PHP file to bypass the restriction.
# Chain Multiple Vulns - Path Traversal

The app allows the user to upload malicious files, however, the directory that the file is uploaded to, is not configured with execution permissions. If the app is not performing input validation on the value used to determine the location of the uploaded file, you may be able to introduce a path traversal attack to get the uploaded file into a different directory that has execution permissions.

For example:

```bash
../../../file.php
```
# Overwriting a Config File / Extension Blacklist

Servers usually won't execute files unless they have been configured to do so. Many servers allow directory level configuration files to be used, which will override global configurations. In Apache, you can upload the following configurations into a file called ".htaccess":

```html
AddType application/x-httpd-php .php5
```

This maps the extension .php5 to the executable MIME type application/x-httpd-php. Now, you can upload a file with the .php5 file extension and the server will execute the code as PHP. This bypasses any reject list validations against files such as .php.

The file extension can be any arbitrary value as long as it is not blocked by the app.
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