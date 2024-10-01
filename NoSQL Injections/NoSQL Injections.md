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