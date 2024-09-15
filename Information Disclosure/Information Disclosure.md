![[Information Disclosure.jpg]]
# Recon

Map out the app with the following steps for example:

- Walk through the entire functionality of the app as a regular user would. Make a note of every request, parameters/input fields, cookies and interesting headers that are being used. (Burp's site map can be helpful here to keep track of all endpoints/data that were found and/or a spreadsheet can help).
- Check the source code of the app and identify any JavaScript files, comments or any other resources that were not already discovered to see if they leak any internal system/sensitive information.
- Use enumeration tools to discover more content such as hidden directories, parameters or files. These resources may disclose some sensitive functionality. Some tools to use are Burp Pro's Discover Content, gobuster, FFuF and so on.
# What is Information Disclosure?

Information disclosure, also known as information leakage, is when a website unintentionally reveals sensitive information to its users. Depending on the context, websites may leak all kinds of information to a potential attacker, including:

- Data about other users, such as usernames or financial information
- Sensitive commercial or business data
- Technical details about the website and its infrastructure

The dangers of leaking sensitive user or business data are obvious but disclosing technical information can sometimes be just as serious. Although some of this information will be of limited use, it can potentially be a starting point for exposing an additional attack surface, which may contain other interesting vulnerabilities. The knowledge that you can gather could even provide the missing piece of the puzzle when trying to construct complex, high risk attacks.

Occasionally, sensitive information might be carelessly leaked to suers who are simply browsing the site in a normal way. More commonly however, an attacker needs to elicit the information disclosure by interacting with the website in unexpected or malicious ways. They will then carefully study the responses to try and identify interesting behaviour.
# Cheat Sheet

Sensitive information can be found even without explicitly looking for it. Sometimes, when probing for other vulnerabilities, there is a specific error message, notable difference in the response or a subtly time delay in the app's response. This info is important to note down and further engineer informative responses.

Check out the common files for web crawlers as well:

- robots.txt
- sitemap.xml

Web servers can be configured to automatically list the contents of directories that do not have an index page present. For example, if you see a path such as `/resources/static/files/23.jpg`. Look under all the folders to see if there is a listing of other existing resources:

- /resources
- /resources/static
- /resources/static/files

Submit unexpected characters into parameters, cookies, or headers and analyse the affect it has on the app. Maybe it discloses a stack trace or an overly verbose error message:

![[Stack Trace.png]]

Burp Pro has a search functionality under the "Engagement Tools" that can help to identify any sensitive information in the response of a specific target domain. You can search for some keywords such as password, secret, key, etc..

Look for keywords that are often contained in error messages - error, invalid, stack, not found, SQL, access, etc... Sometimes, the error messages will not be rendered to the screen so look at the raw responses.

When a server handles files with a particular extension such as .php, it will typically execute the code, rather than simply sending it to the client as text. However, in some situations, you can trick a website into returning the contents of the file instead. Appending a tilde (~) to the filename or adding a different file extension:

- `test.php~`

Use the HTTP method TRACE when submitting requests, as this can reveal sensitive debugging information that can be used to exploit the app:

![[ID 1.png]]

There may also be information disclosure in version control history. If a .git endpoint exists that is found through the "Discover Content" feature or other tools, you can download the entire directory using the following:

```bash
wget -r https://../path/to/file
```

Next, you can follow the steps below to obtain a potential password:

![[ID 2.png]]

