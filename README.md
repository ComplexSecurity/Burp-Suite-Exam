![[README.jpg]]

The Burp Suite Certified Practitioner (BSCP) exam consists of two applications, with each application containing deliberate vulnerabilities. You have 4 hours to complete the exam. Each application can be completed in three stages:

1. Stage 1: Access any user account
2. Stage 2: Use your user account to access the admin interface perhaps by elevating your privileges or compromising the administrator account.
3. Stage 3: Use the admin interface to read the contents of /home/carlos/secret from the server's file system, and submit it using "submit solution".

>[!danger]
>Be careful not to delete your own account or a core system component with powerful functionality during the exam.

There is always an administrator account with the username "administrator", plus a lower privileged account usually called "carlos". If you find username enumeration, you may be able to break into a low-privileged account using the [username list](https://portswigger.net/web-security/authentication/auth-lab-usernames) and [password list](https://portswigger.net/web-security/authentication/auth-lab-passwords).

Each app has up to one active user who will be logged in either as a user or an administrator. You can assume they will visit the homepage of the site every 15 seconds, and click any links in any emails they receive from the app. 

>[!info]
>You can use the exploit server's "send to victim" functionality to target them with reflected vulnerabilities.

If you find an SSRF vulnerability, you can use it to read files by accessing an internal-only service, running on localhost on port 6566. 

Host header attacks are allowed, but the `_lab` and `_lab_analytics`cookies are part of the core exam functionality, so do not tamper with them. 

Some useful external resources:

- [PortSwigger Academy Cheat Sheets - ChrisM-X](https://github.com/ChrisM-X/PortSwigger-Academy-CheatSheets)
- [BSCP Exam Guide - DingyShark](https://github.com/DingyShark/BurpSuiteCertifiedPractitioner)
- [BSCP Exam Study - botesjuan](https://github.com/botesjuan/Burp-Suite-Certified-Practitioner-Exam-Study/blob/main/README.md)

It has also been stated to prepare extensions such as:

- ActiveScan++
- SQLiPy Sqlmap Integration
- HTTP Request Smuggler
- Java Deserialization Scanner
- JWT Editor
# Step 1

You must complete 1 practitioner level lab from each topic:

- [Web shell upload via extension blacklist bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass) ✅
- [OAuth account hijacking via redirect_uri](https://portswigger.net/web-security/oauth/lab-oauth-account-hijacking-via-redirect-uri)
- [SSRF via flawed request parsing](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-ssrf-via-flawed-request-parsing)
- [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft) ✅
- [Exploiting XSS to capture passwords](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-capturing-passwords) ✅
- [CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method) ✅
- [Blind XXE with out-of-band interaction via XML parameter entities](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-interaction-using-parameter-entities)
- [Multistep clickjacking](https://portswigger.net/web-security/clickjacking/lab-multistep)
- [SSRF with filter bypass via open redirection vulnerability](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)
- [CORS vulnerability with trusted insecure protocols](https://portswigger.net/web-security/cors/lab-breaking-https-attack)
- [Exploiting HTTP request smuggling to deliver reflected XSS](https://portswigger.net/web-security/request-smuggling/exploiting/lab-deliver-reflected-xss)
- [Server-side template injection in an unknown language with a documented exploit](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-in-an-unknown-language-with-a-documented-exploit)
- [Using application functionality to exploit insecure deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-application-functionality-to-exploit-insecure-deserialization)
- [File path traversal, traversal sequences stripped non-recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively) ✅
- [Multi-step process with no access control on one step](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step) ✅
- [Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block) ✅
- [Insufficient workflow validation](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-insufficient-workflow-validation) ✅
- [Manipulating the WebSocket handshake to exploit vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-handshake-to-exploit-vulnerabilities)
- [DOM XSS using web messages and a JavaScript URL](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-a-javascript-url)
- [Web cache poisoning with multiple headers](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-multiple-headers)
- [Information disclosure in version control history](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-in-version-control-history) ✅
- [Blind OS command injection with output redirection](https://portswigger.net/web-security/os-command-injection/lab-blind-output-redirection) ✅
- [Discovering vulnerabilities quickly with targeted scanning](https://portswigger.net/web-security/essential-skills/using-burp-scanner-during-manual-testing/lab-discovering-vulnerabilities-quickly-with-targeted-scanning) ✅
# Step 2

You must complete the following labs:

- [Exploiting XSS to capture passwords](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-capturing-passwords) ✅
- [Blind SQL injection with out-of-band data exfiltration](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration) ✅
- [Forced OAuth profile linking]() ✅
- [Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie) ✅
- [Exploiting HTTP request smuggling to capture other users' requests]() ✅
- [SSRF with blacklist-based input filter]() ✅
- [SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding) ✅
- [Discovering vulnerabilities quickly with targeted scanning]() ✅
# Step 3

You must complete 5 mystery lab challenges which will spawn randomized practitioner-level labs where you have to work out how to solve each challenge with no context.
# Step 4

You must pass a practice exam which contains one vulnerable application and it must be completed in 2 hours.