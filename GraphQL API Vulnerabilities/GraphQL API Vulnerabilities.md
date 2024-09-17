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

