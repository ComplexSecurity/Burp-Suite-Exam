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

The process of detecting and exploiting limit overrun race conditions is relatively simple. In high-level terms, all you need to do is:

- Identify a single use or rate limited endpoint that has some kind of security impact or other useful purpose.
- Issue multiple requests to this endpoint in quick succession to see if you can overrun this limit.

The primary challenge is timing the requests so that at least two race windows line up, causing a collision. This window is often just milliseconds and can be even shorter.

>[!info]
>Sending requests in parallel - [https://portswigger.net/burp/documentation/desktop/tools/repeater/send-group#sending-requests-in-parallel](https://portswigger.net/burp/documentation/desktop/tools/repeater/send-group#sending-requests-in-parallel)

An example lab's purchasing flow contains a race condition that enables you to purchase items for an unintended price. To start, use all of the app's available functionality, including using the PROMO code when purchasing an item.

Send the following request to Repeater (ensure to have added the expensive jacket into your cart before this step):

```bash
POST /cart/coupon HTTP/2
Host: 0a77008c045c4af180f9e44f00ac00d1.web-security-academy.net
REDACTED...

csrf=UfjDdxlSAUrINBasJasUgfvqCR&coupon=PROMO20
```

This request is the request that is used to process the coupon in the purchase order. You can use the "single packet" attack to complete around 20-30 requests simultaneously to see if you can exploit the "race window", which is before the app updates the database with info confirming coupon has already been used in the order.

Send around 20 of the same request to Repeater and create a "group" that will include all of the tabs for the same request. Finally, select the option "Send group in parallel (single-packet attack)", and submit the requests.

This may take a couple of tries, but eventually you can submit multiple coupons in the same order to purchase an expensive item. The coupon is only supposed to be used once per order, but exploiting a race condition vulnerability allows for a bypass.
# Hidden Multi-Step Sequences

In practice, a single request may initiate an entire multi-step sequence behind the scenes, transitioning the application through multiple hidden states that it enters and then exits again before request processing is complete - referred to as "sub-states".

If you can identify one or more HTTP requests that cause an interaction with the same data, you can potentially abuse these sub-states to expose time-sensitive variations of the kinds of logic flaws that are common in multi-step workflows.
## Methodology

Predict potential collisions - is this endpoint security critical? Many endpoints do not touch critical functionality, so they are not worth testing. Is there any collision potential? For a successful collision, you need two or more requests that trigger operations on the same record.

To recognize clues, you first need to benchmark how the endpoint behaves under normal conditions. You can do this in Repeater by grouping all of the requests and using the "Send group in sequence (separate connections)" option.

Next, send the same group of requests at once using the single-packet attack (or last-byte sync if HTTP/2 is not supported) to minimize network jitter. You can do this in Repeater by selecting the Send group in parallel option.

Anything at all can be a clue - just look for some form of deviation from what you observe during benchmarking. Try to understand what is happening, remove superfluous requests and make sure you can still replicate the effects.
## Multi-Endpoint Race Conditions

The most intuitive form of these race conditions are those that involve sending requests to multiple endpoints at the same time. A variation of this can occur when payment validation and order confirmation are performed during the processing of a single request.

In this case, you can potentially add more items to your basket during the race window between when the payment is validated and when the order is finally confirmed. 
## Aligning Multi-Endpoint Race Windows

When testing for multi-endpoint race conditions, you may encounter issues trying to line up the race windows for each request, even if you send them all at exactly the same time using the single packet technique.

This common problem is primarily caused by the following two factors:

- Delays introduced by network architecture - for example, there may be a delay whenever the front-end server establishes a new connection to the back-end. The protocol used can also have a major impact.
- Delays introduced by endpoint-specific processing - different endpoints inherently vary in their processing times, sometimes significantly so, depending on what operations they trigger.

Fortunately, there are potential workarounds to both of these issues.

One way to do this is by "warming" the connection with one or more inconsequential requests to see if this smooths out the remaining processing times. In Repeater, you can try adding a GET request for the homepage to the start of the tab group, then using the "Send group in sequence (single connection)" option.

If the first request still has a longer processing time, but the rest of the requests are now processed within a short window, you can ignore the apparent delay and continue testing as normal.

For the lab, its purchasing flow contains a race condition that enables you to purchase items for an unintended price. The 2 main requests that are used to interact with the user's cart are - for example, a POST /cart request adds items to the cart and a POST /cart/checkout request submits your order.

The state of the user's cart is stored server-side within the user's session cookie. Any operations on the cart revolve around the user's session. This indicates that there is potential for a collision.

A summary of the requests that were submitted to benchmark the requests behaviour and exploit the lab are below. These requests were sent to Burp Repeater and placed in a Group and processed using the different available configurations:

Send group (Single connection) request:
- POST /cart - 471 millis
- POST /cart/checkout - 173 millis

Send group (Single connection) request: (Here we are "warming" the connection by including the GET request to the beginning of the Group list, the last 2 requests were now processed in similar times)

- GET /academyLabHeader - 447 millis
- POST /cart - 180 millis
- POST /cart/checkout - 174 millis

To solve the lab send the following request in Burp Repeater using the "Send group (parallel)" option: (before submitting this payload, ensure that there is a gift card already in your cart)

- GET /
- POST /cart/checkout
- POST /cart (ensure that the productId parameter is set to 1, as this is the ID for the jacket)

>[!info]
>Play around with the order of the requests that are submitted in Burp Repeater. For example, if requests in the order of 1, 2, 3 is not working, try to switch them around like 1, 3, 2.
# Single-Endpoint Race Conditions

Sending parallel requests with different values to a single endpoint can sometimes trigger powerful race conditions. Email address confirmations or any other email-based operations are generally a good target for single-endpoint race conditions.

Emails are often sent in a background thread after the server issues the HTTP response to the client, making race conditions more likely.

For the lab, the app contains functionality that allows us to update the email for the user wiener. The app sends a confirmation email to the email client that is available. There is a race condition vulnerability within this functionality.

The information that is sent to the email versus the confirmation message sent in the body of the request are not matching. For example, the email sent to the address of test123@attacker.com contains the confirmation message in scope for the email of test987@attacker.com

There is a race window between when the website:

- Kicks off a task that eventually sends an email to the provided address
- Retrieves data from the database and uses this to render the email template (the database stores only 1 email address info at a time, can be confirmed by trying to access an older confirmation email request in the client).

A summary of the exploitation steps:

Send the following request to Burp Repeater:

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
# Partial Construction Race Conditions

- [https://portswigger.net/web-security/race-conditions/lab-race-conditions-partial-construction](https://portswigger.net/web-security/race-conditions/lab-race-conditions-partial-construction)

This lab contains a user registration mechanism. A race condition enables you to bypass email verification and register with an arbitrary email address that you do not own.
# Time-Sensitive Attacks

Sometimes, you may not find race conditions, but the techniques for delivering requests with precise timing can still reveal the presence of other vulnerabilities. One such example is when high-resolution timestamps are used instead of cryptographically secure random strings to generate security tokens.

The lab contained a password reset mechanism. Although it did not contain a race condition, you can exploit the mechanism's broken cryptography by sending carefully timed requests.