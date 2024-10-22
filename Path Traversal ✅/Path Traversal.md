+![[Path Traversal.png]]
# Path Traversal

Path traversal allows an attacker to read arbitrary files on the server running an application, including:

- Application code and data
- Credentials for back-end systems
- Sensitive OS files
# Recon

Identify any app functionality that is likely to involve retrieval of data from a server's filesystem. Look for any input fields that reference a file or directory name, or that contain any file extensions such as:

```html
?file=test
?item=test.html
```

After identifying potential targets for path traversal testing, you must test every instance individually to determine if user controllable data is used to interact with the server filesystem in an unsafe way.
# Reading Files via Path Traversal

An app may display images of items such as:

```html
<img src="/loadImage?filename=218.png">
```

It takes a filename parameter and returns the contents of the specified file. The image files are typically stored at `/var/www/images`. To return an image, the app appends the requested filename to the base directory and uses an API to read it.

If no defenses exist, attackers can request the following URL to retrieve the `/etc/passwd` file from the server filesystem:

```bash
https://insecure-website.com/loadImage?filename=../../../etc/passwd
```

On Windows, both `../` and `..\` are valid traversal sequences. An example on Windows may be:

```bash
https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini
```

For example, there may be a filename parameter that loads images:

![[Filename Path.png]]

Try modifying the request to get a non-existent file or try navigating to the base /image folder. Try injecting a simple path traversal payload to grab a system file such as `/etc/passwd`:

![[Simple Trav.png]]

Observe the response for the contents of the file.
# Absolute Path Bypasses

Many apps that place user input into file paths implement defenses. If an app strips or blocks traversal sequences from user-supplied filenames, it may be possible to bypass by using an absolute path from the root (e.g. `/etc/passwd`) to directly reference a file.

For example, similiar to above, there may be a filename parameter grabbing an image file. A simple traversal sequence may be blocked, but an absolute path may work:

```http
filename=/etc/passwd
```

![[Absolute Path.png]]
# Non-Recursive Filtering Bypass

Another common defense is to filter out the `../` characters, but not in a recursive manner. You may be able to use nested traversal sequences such as `....//` or `....\/` to bypass it.

For example, back to the filename parameter, it may not work with absolute paths or traversal sequences. If so, attempt to use double traversal sequences to test for recursive stripping:

```http
filename=....//....//....//....//....//....//etc//passwd
```

![[Double Trav.png]]
# URL Encoding Bypass

In some contexts, servers may strip any directory traversal sequences before passing input to the app. This can be bypassed by URL encoding or double URL encoding the `../` characters, resulting in `%2e%2e%2f` and `%252e%252e%252f`. 

Various non-standard encodings such as `..%c0%af` and `..%ef%bc%8f` may also work.

For example, if no other payloads work, try encoding the traversal sequence such as the following:

![[URL Encode.png]]

If still no success, try double URL encoding such as:

![[Double URL.png]]
# Validation of Start of Path

An app may require the user-supplied filename to start with the expected base folder, such as `/var/www/images`. If so, it may be possible to include the required base folder followed by suitable sequences:

```http
filename=/var/www/images/../../../etc/passwd
```

For example, a filename parameter may include the full path as part of the request. If so, try leaving the full path and appending traversal sequences to the end. If still no luck, try URL encoding or other bypasses discussed:

![[Validation Start.png]]
# File Extension Validation Bypass

An application may require the user-supplied filename to end with an expected file extension. If so, it may be possible to use a null byte to terminate the path before the required extension such asL

```bash
../../../etc/passwd%00.png
```

For example, if the previous methods do not work, try combining them and adding a null byte at the end to terminate the file extension:

![[Null Byte Ending.png]]

