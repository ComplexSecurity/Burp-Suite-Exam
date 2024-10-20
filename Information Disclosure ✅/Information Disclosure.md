![[Information Disclosure.jpg]]
# Information Disclosure

Information disclosure, also known as information leakage, is when a website unintentionally reveals sensitive information to its users. Depending on the context, websites may leak all kinds of information to a potential attacker, including:

- Data about other users, such as usernames or financial information
- Sensitive commercial or business data
- Technical details about the website and its infrastructure

The dangers of leaking sensitive user or business data are obvious but disclosing technical information can sometimes be just as serious. Although some of this information will be of limited use, it can potentially be a starting point for exposing an additional attack surface, which may contain other interesting vulnerabilities.

Occasionally, sensitive information might be carelessly leaked to suers who are simply browsing the site in a normal way. More commonly however, an attacker needs to elicit the information disclosure by interacting with the website in unexpected or malicious ways.

Some examples include:

- Names of hidden directories via a robots.txt file or directory listing
- Providing access to source code files via temporary backups
- Mentioning database table or column names in errors
- Exposing highly sensitive information like credit cards
- Hard-coding API keys, IP addresses, database credentials in source code
- Hinting at existence or absence of resources, usernames via subtle differences in app behaviour
# Recon

Walk through the entire functionality of the app as a regular user would. Make a note of every request, parameters/input fields, cookies and interesting headers that are being used. (Burp's site map can be helpful here to keep track of all endpoints/data that were found and/or a spreadsheet can help).

For interesting parameters, try submitting unexpected data types and crafted fuzz strings and pay attention to responses including the time. Burp Scanner can also alert you of any sensitive information found in a response, as well as finding backup files, directory listings and more.

Check the source code of the app and identify any JavaScript files, comments or any other resources that were not already discovered to see if they leak any internal system/sensitive information.

Use enumeration tools to discover more content such as hidden directories, parameters or files. These resources may disclose some sensitive functionality. Some tools to use are Burp Pro's Discover Content, gobuster, FFuF and so on.
# Crawlers

Many sites provide files like `/robots.txt` and `/sitemap.xml` to help web crawlers which can include specific directories that crawlers should skip. Always attempt to navigate to these files to find potentially interesting information or hidden directories.
# Directory Listings

Some servers may automatically list the contents of directories that don't have an index page which can increase the exposure of sensitive files within the directory.

>[!info]
>It is not necessarily a vulnerability, but if there are also improper access controls, leaking the existence and location of sensitive resources can be an issue.
# Developer Comments

HTML comments are sometimes present which are easily accessed by Burp or via the DevTools. Comments in source code may contain useful information like a hidden directory or provide clues as to how the web app functions.
# Error Messages

Content of errors can reveal information about what input or data type is expected. Verbose errors can also provide information about different technologies in use like a templating engine, database type or server the website is using.

Try checking for any common configuration errors or dangerous default settings as well. 

Differences between error messages can reveal different behaviour. Observe the differences to potentially lead to other vulnerabilities such as SQL injection, username enumeration and so on. 

For example, trying to input a number when it expects a string may result in an error indicating a version running:

![[Apache Struts.png]]
# Debug Data

Some sites generate custom errors and logs containing large amounts of information. Debug messages can contain sensitive info such as:

- Values for key session variables that can be manipulated via user input
- Hostnames and creds
- File and directory names
- Keys used to encrypt data transmitted

For example, there may be a sensitive page like `phpinfo.php` in a common directory like `cgi-bin` that contains a secret or private key value that is easily accessible:

![[Secret Key.png]]
# User Account Pages

Some sites contain logic flaws that allow attackers to leverage user pages to view other user data. Consider a site that determines which user's account page to load via a parameter:

```http
GET /user/personal-info?user=carlos
```

An attacker may not be able to load another user's page entirely, but the logic for certain functions like rendering the user's registered email may not check the user parameter matches the current user, allowing attackers to change the user parameter and display another user's email address:
# Backup Files

Sometimes it may be possible to cause the site to expose its own source code - there may be some source code files that are referenced explicitly. In some cases, you can trick a site to return the contents of a file (e.g. PHP) by appending a tilde `~` to the filename or using a different extension.

Requesting code files using a backup file extension can sometimes allow the reading of contents of the file in the response.

For example, there may even be a backup directory that contains some source code. The source code could contain sensitive information like database credentials:

![[Database Pass.png]]
# Insecure Configuration

Developers may forget to disable various debugging options. The HTTP `TRACE` method is used for diagnostic purposes which can make the web server respond to requests that use it by echoing the exact request that was received which can lead to disclosure.

For example, the `TRACE` method may reveal an unknown header:

![[Custom IP.png]]

The custom header could be used to specify the localhost and gain access to administrative functionality.
# Version Control History

Most sites use version control like Git. Git projects store all version control data in a `.git` folder. If a site exposes this directory, it can be downloaded and opened locally to view all the version control history which can include sensitive information such as hardcoded credentials.

For example, the `.git` directory can be downloaded recursively:

```bash
wget -r https://0af1005403f24d998106025e00800011.web-security-academy.net/.git/
```

Git Cola can then be used to look through all commits for sensitive information:

![[Admin Pass.png]]

