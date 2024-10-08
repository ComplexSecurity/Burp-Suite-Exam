#Race-Conditions #Completed

![[Race Conditions.png]]
# Overview

Race conditions are a common type of vulnerability closely related to business logic flaws. They occur when websites process requests concurrently without adequate safeguards. This can lead to multiple distinct threads interacting with the same data at the same time, resulting in a "collision" that causes unintended behaviour.

A race condition attack uses carefully timed requests to cause intentional collisions and exploit this unintended behaviour for malicious purposes. The period of time during which a collision is possible is known as the "race window" - this could be a fraction of a second between two interactions with the database.
# Limit Overrun Race Conditions

The most well-known type of race condition enables you to exceed some kind of limit imposed by the business logic of the ap. For example, consider an online store that lets you enter a promotional code during checkout to get a one-time discount on your order.

To apply the discount, the app may perform the following high-level steps:

- Check that you have not already used the code
- Apply the discount to the order total
- Update the record in the database to reflect the fact that you have now used the code

If you send 2 or more requests concurrently, you can try to abuse the "race window" that is before the app updates the database, in order to use the same discount code twice.

There are many variations of this attack including:

- Redeeming a gift card multiple times
- Rating a product multiple times
- Withdrawing or transferring cash in excess of your account balance
- Reusing a single CAPTCHA solution
- Bypassing an anti-brute-force rate limit

>[!info]
>Limit overruns are a subtype of "time-of-check to time-of-use" (TOCTOU) flaws".

The process of detecting and exploiting limit overrun race conditions is relatively simple. In high-level terms, all you need to do is:

- Identify a single use or rate limited endpoint that has some kind of security impact or other useful purpose.
- Issue multiple requests to this endpoint in quick succession to see if you can overrun this limit.

The primary challenge is timing the requests so that at least two race windows line up, causing a collision. This window is often just milliseconds.

>[!info]
>Sending requests in parallel - [https://portswigger.net/burp/documentation/desktop/tools/repeater/send-group#sending-requests-in-parallel](https://portswigger.net/burp/documentation/desktop/tools/repeater/send-group#sending-requests-in-parallel)

Burp Suite can send a group of parallel requests to reduce the impact of one of the factors - network jitter:

- HTTP/1 - last-byte synchronization technique
- HTTP/2 - single-packet attack technique

Single-packet allows you to neutralize interference by a single TCP packet to complete request simultaneously. Try to send a large number of requests to help mitigate internal latency (server-side jitter).

An example lab's purchasing flow contains a race condition that enables you to purchase items for an unintended price. To start, use all of the app's available functionality, including using the PROMO code when purchasing an item. Try adding items to cart, checking out and applying the PROMO code.

There may be a POST request for the coupon itself. Send the request to Repeater:

```bash
POST /cart/coupon HTTP/2
Host: 0a77008c045c4af180f9e44f00ac00d1.web-security-academy.net
REDACTED...

csrf=UfjDdxlSAUrINBasJasUgfvqCR&coupon=PROMO20
```

Attempt to submit the coupon code multiple times and observe the error message. Try to identify other things such as sending the `GET /cart` request with and without the session token. Try to infer if:

- The state of the cart is stored server-side via the session
- Operations on the cart are keyed on the session token or user ID

Try benchmarking the behaviour by applying a PROMO code and then using a Repeater group to send 20-30 coupon code requests in sequence - does the coupon code work multiple times or just the first?

>[!info]
>In Repeater, Send --> "Send group (separate connections)".

Try submitting a group of requests in parallel and observe for any differences.

You can use the "single packet" attack to complete around 20-30 requests simultaneously to see if you can exploit the "race window", which is before the app updates the database with info confirming coupon has already been used in the order.

This may take a couple of tries, but eventually you can submit multiple coupons in the same order to purchase an expensive item. The coupon is only supposed to be used once per order, but exploiting a race condition vulnerability allows for a bypass.
# Turbo Intruder

Turbo Intruder is more efficient for complex attacks that require multiple retries, staggered request timing or a large number of requests. For a single-packet attack:

1. Ensure target supports HTTP/2
2. Set `engine=Engine.BURP2` and `concurrentConnections=1`
3. Group requests by assigning to a named gate using `gate` argument for `engine.queue()` method.
4. Open respective gate to send all requests in a given group via `engine.openGate()` method:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            engine=Engine.BURP2
                            )
    
    # queue 20 requests in gate '1'
    for i in range(20):
        engine.queue(target.req, gate='1')
    
    # send all requests in gate '1' in parallel
    engine.openGate('1')
```

As an example, there may a lockout after 3 incorrect logins. Logging in with another username may not appear to be blocked - rate limiting on usernames and not sessions. Consider a race window between:

- Submitting login request
- When website increments failed login counter

Try creating many tabs in Repeater and sending the group of requests in sequence and observe - i.e. after 2 failed login attempts, lockout occurs. Try sending many tabs in parallel and observe again - if more requests don't appear to be locked out, indicates a race condition.

To exploit, send to Turbo Intruder, change the username to desired user and select `race-single-packet-attack.py` script and modify it so it queues once using each password:

```python
def queueRequests(target, wordlists):

    # as the target supports HTTP/2, use engine=Engine.BURP2 and concurrentConnections=1 for a single-packet attack
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2
                           )
    
    # assign the list of candidate passwords from your clipboard
    passwords = wordlists.clipboard
    
    # queue a login request using each password from the wordlist
    # the 'gate' argument withholds the final part of each request until engine.openGate() is invoked
    for password in passwords:
        engine.queue(target.req, password, gate='1')
    
    # once every request has been queued
    # invoke engine.openGate() to send all requests in the given gate simultaneously
    engine.openGate('1')


def handleResponse(req, interesting):
    table.add(req)
```

Copy list of passwords to clipboard and launch. If no logins, try waiting for lockout to reset and attempt again. If 302 occurs, it may indicate success.
# Hidden Multi-Step Sequences

In practice, a single request may initiate an entire multi-step sequence behind the scenes, transitioning the application through multiple hidden states that it enters and then exits again before request processing is complete - referred to as "sub-states".

If you can identify one or more HTTP requests that cause an interaction with the same data, you can potentially abuse these sub-states to expose time-sensitive variations of the kinds of logic flaws that are common in multi-step workflows.

An example vulnerable workflow:

```bash
session['userid'] = user.userid
if user.mfa_enabled:
session['enforce_mfa'] = True
# generate and send MFA code to user
# redirect browser to MFA code entry form
```

The app may transition through a sub-state in which the user has a temporary valid session, but MFA is not yet enforced. It's exploitable by sending a login request along with a request to a sensitive endpoint.

Predict potential collisions - is this endpoint security critical? Many endpoints do not touch critical functionality, so they are not worth testing. Is there any collision potential? For a successful collision, you need two or more requests that trigger operations on the same record.

To recognize clues, you first need to benchmark how the endpoint behaves under normal conditions. You can do this in Repeater by grouping all of the requests and using the "Send group in sequence (separate connections)" option.

Next, send the same group of requests at once using the single-packet attack (or last-byte sync if HTTP/2 is not supported) to minimize network jitter. You can do this in Repeater by selecting the Send group in parallel option.

Anything at all can be a clue - just look for some form of deviation from what you observe during benchmarking. Try to understand what is happening, remove superfluous requests and make sure you can still replicate the effects.
# Multi-Endpoint Race Conditions

The most intuitive form of these race conditions are those that involve sending requests to multiple endpoints at the same time. A variation of this can occur when payment validation and order confirmation are performed during the processing of a single request.

In this case, you can potentially add more items to your basket during the race window between when the payment is validated and when the order is finally confirmed. 

When testing for multi-endpoint race conditions, you may encounter issues trying to line up the race windows for each request, even if you send them all at exactly the same time using the single packet technique.

This common problem is primarily caused by the following two factors:

- Delays introduced by network architecture - for example, there may be a delay whenever the front-end server establishes a new connection to the back-end. The protocol used can also have a major impact.
- Delays introduced by endpoint-specific processing - different endpoints inherently vary in their processing times, sometimes significantly so, depending on what operations they trigger.

Fortunately, there are potential workarounds to both of these issues.

One way to do this is by "warming" the connection with one or more inconsequential requests to see if this smooths out the remaining processing times. In Repeater, you can try adding a GET request for the homepage to the start of the tab group, then using the "Send group in sequence (single connection)" option.

If the first request still has a longer processing time, but the rest of the requests are now processed within a short window, you can ignore the apparent delay and continue testing as normal. If there are still inconsistent response times on a single endpoint, it indicates the back-end delay is interfering with the attack - try using Turbo Intruder to send some connection warming requests.


As an example, there may be a purchasing flow that has a race condition, enabling you to purchase items for an unintended price. It may have 2 requests to interact with the cart - POST /cart and POST/cart/checkout. 

In this case, try adding a gift card to the cart and sending the request to Repeater to send the GET /cart request with and without the session cookie. Try to confirm that if the session token is not present, you can only access an empty cart - means the cart is stored server-side and operations on the cart are keyed on the session ID.

This indicates that there is potential for a collision. There may be a race window between when the order is validated and when it is confirmed.

Try sending POST /cart and POST /cart/checkout to Repeater, grouping them and sending them in sequence a few times - the first request may take significantly longer than the second:

Send group (Single connection) request:
- POST /cart - 471 millis
- POST /cart/checkout - 173 millis

If so, try adding an arbitrary request to the start of the request and sending them as a single connection (Here we are "warming" the connection by including the GET request to the beginning of the Group list, the last 2 requests were now processed in similar times):

- GET /academyLabHeader - 447 millis
- POST /cart - 180 millis
- POST /cart/checkout - 174 millis

If the time is acceptable, remove the GET request, add a gift card to the cart. Modify the POST /cart request so the productID is set to the product of choice and send in sequence - the order is rejected due to insufficient funds.

To solve the lab, send the following request in Burp Repeater using the "Send group (parallel)" option: (before submitting this payload, ensure that there is a gift card already in your cart)

- GET /
- POST /cart/checkout
- POST /cart (ensure that the productId parameter is set to 1, as this is the ID for the jacket)

>[!info]
>Play around with the order of the requests that are submitted in Burp Repeater. For example, if requests in the order of 1, 2, 3 is not working, try to switch them around like 1, 3, 2.

If warming didn't make a difference, try introducing a short client-side delay using Turbo Intruder. It won't work with a single-packet attack due to splitting it across multiple TCP packets. On high-jitter targets, the attack is unlikely to work regardless of delay set.

Another way is by sending a large number of dummy requests to intentionally trigger the rate or resource limit and cause a suitable server-side delay, making single-packet viable.
# Single-Endpoint Race Conditions

Sending parallel requests with different values to a single endpoint can sometimes trigger powerful race conditions. Consider a password reset function that stores the user ID and reset token in the user session. Sending two parallel password reset requests from the same session, but different usernames, could cause a collision.

![[Pass Reset Collision.png]]

>[!info]
>Session contains the victim user ID but valid reset token is sent to attacker.

Email address confirmations or any other email-based operations are generally a good target for single-endpoint race conditions. Emails are often sent in a background thread after the server issues the HTTP response to the client, making race conditions more likely.

For example, an app contains functionality that allows us to update the email for the user wiener. The app sends a confirmation email to the email client that is available. There is a race condition vulnerability within this functionality.

Attempt to change the email to something and observe a confirmation email is sent. Try submitting two different email addresses in succession and observe the email client. Attempting to use the first link sent results in it no longer being valid.

The site may only store one pending email at a time. Since submitting a new one edits the entry rather than appending, there may be a collision. 

To benchmark behaviour, send a change email request to Repeater, add to a group, duplicate it and change the email address in each. Send them as separate connections and observe a single confirmation email for each change request.

Attempt to send again in parallel to try and change the email to many values at once. The recipient address may not always match the pending new email address. 

There is a race window between when the website:

- Kicks off a task that eventually sends an email to the provided address
- Retrieves data from the database and uses this to render the email template (the database stores only 1 email address info at a time, can be confirmed by trying to access an older confirmation email request in the client).

When a parallel request changes the pending email address in the database, it results in confirmation emails being sent to the wrong address.

To exploit, create a group containing two change email requests. Change email value of one to anything and another to `carlos@ginandjuice.shop` and send in parallel. If a confirmation email is received with the body matches the owned address, retry. If not, click the link to update the email.

An overview of the exploit:

- POST /my-account/change-email
- Send around 15 more requests to Burp Repeater for the same endpoint to change the email address. Make every email address in each request unique. Group all of the requests and select the "Send group (parallel)" option, then submit requests.
- Next, go to the email client and notice that the confirmation message contains an email address that differs from the email address to which the confirmation message was sent to. For example, confirmation message contains [test555@attacker.com](mailto:test555@attacker.com), while the message was sent to the email address [test777@attacker.com](mailto:test777@attacker.com). (Note - the email client is meant for us to retrieve all emails sent to any exploit server sub-domain)

Now to gain access to the email address - [carlos@ginandjuice.shop](mailto:carlos@ginandjuice.shop)
- Send 2 requests to repeater for the change email address function - POST /my-account/change-email
- The body payloads for each requests:
    - Request 1 - email=test999%40exploit-0aa1009a0479c5cb8180f74601a100da.exploit-server.net
    - Request 2 - email=[carlos@ginandjuice.shop](mailto:carlos@ginandjuice.shop)
- Select the option "Send group (parallel)" in Burp Repeater and submit the requests. This step may need to be initiated many times since the latest email confirmation message needs to contain the value for - [carlos@ginandjuice.shop](mailto:carlos@ginandjuice.shop). (This is because in the database there only exists one value at a time.)
- Once the latest email confirmation message contains the value for [carlos@ginandjuice.shop](mailto:carlos@ginandjuice.shop) process the link and gain access to an admin account.
# Session-Based Locking Mechanisms

Certain frameworks can attempt to prevent accidental data corruption by using some form of request locking such as PHP's native session handler module which only processes one request per session at a time. If you see all of the requests are being processed sequentially, try sending each using a different session token.
# Partial Construction Race Conditions

Some apps create objects in multiple steps, which introduce a temporary middle state where it is exploitable. As an example, when registering a new user, an app may create the user in the database and set the API key using two SQL statements, leaving a window where the user exists, but the API key is uninitialized.

It can be exploits by injecting an input value that returns something matching the uninitialized database value, like an empty string or null and is compared as part of a security control. 

Frameworks often let you pass arrays and other non-string data using non-standard syntax like in PHP:

- `param[]=foo` is equivalent to `param = ['foo']`
- `param[]=foo&param[]=bar` is equivalent to `param = ['foo', 'bar']`
- `param[]` is equivalent to `param = []`

Ruby does similiar by providing a query or POST parameter with a key but no value. For example `param[key]` results in the following:

```ruby
params = {"param"=>{"key"=>nil}}
```

During the race window, you can potentially make authenticated API requests as follows:

```html
GET /api/user/info?user=victim&api-key[]= HTTP/2
Host: vulnerable-website.com
```
# Time-Sensitive Attacks

Sometimes the techniques for delivering requests with precise timing can reveal other vulnerabilities such as when high-resolution timestamps are used instead of cryptographically secure random strings to generate security tokens.

Consider a reset token that is only randomized using a timestamp. It may be possible to trigger two password resets for two different users, which both use the same token by timing the requests so they generate the same timestamp.

As an example, the lab can be exploited via its broken cryptography by sending timed requests. The app contains a password reset function that sends an email which includes the name and token. Try observing that every request results in a different token.

Try to determine if:

- The token is a consistent length - either randomly generated or could be a hash
- The token is different each time - if so, likely a hash digest that has an internal state like RNG, counter or timestamp

Try creating a group with 2 tabs and send then in parallel. If there is a significant delay between each response and a different token is generated every time, it may infer that the requests are still processed sequentially.

Also see if the session token suggests what backend is being used such as PHP. If so, it can mean it only processes one request at a time per session. Try sending a new reset request, remove the cookie and send it and check the response.

Copy the new token and CSRF token and replace the values in the requests in Repeater to make two different session requests. Try sending in parallel again and see if the process times are closer or identical. Check the email for any duplicate or identical tokens from testing. If so, a timestamp is likely part of the hashing process.

If you knew the other inputs, the token would be predictable. If there is a separate username parameter, it may mean the user is not part of the hash and two users can have identical tokens. Try changing the username in one request and sending in parallel - if works, both users have the same token.

Try copying the reset link and changing the username to the victim. If it works, reset the password for the victim user.