![[Business Logic Vulnerabilities.png]]
# Recon

Logic flaw vulnerabilities are different than other vulnerabilities such as SQL injection - there is not a common signature to identify logic flaws as there is for SQLi. It requires walking through the entire application and all its functionalities and determine if there is a defect in the logic that was implemented in the application.

Generally, a programmer may have reasoned, "If A happens, then B must be the case, so I will do C". But there may not be any consideration for "But what if X occurs?".

There are 12 different logic flaw examples on the Web Hacker's Handbook 2 that show some real world example where logic flaws can happen.
# Logic Flaw Examples / Concepts

Remove parameters/cookies/query strings in each of the requests one at a time and analyze the responses. This ensures that all relevant code paths within the app are reached. 

In multistage processes, attempt to "force browse" and submit requests in different orders and analyse the responses. Think of the assumptions that may have been made. In a funds transfer app for example, try submitting negative values or very large values - analyse the effects it has on the app.

In any situation where sensitive values are adjusted based on user controllable criteria, analyse if there is a one-time process or if these values change based on further actions by the user such as getting a discount after a minimum total is reached, then removing items to lower that amount, while still using the discount.

Sometimes an escape character is used to escape malicious characters to protect against certain vulnerabilities such as command injection. You can inject your own escape characters "/" to neutralize the escape character that is used for defence.

For example, injecting the following payload (; ls) results in (\\; ls). However, if you inject the following payload (\\; ls) then the app still inserts an escape character, but is neutralized (\\\\; ls).

Identify any instances where the app either truncates, strips out, encodes or decodes user supplied data. Determine if malicious strings can be derived. Identify cases where the app is storing information in a static manner as opposed to per-thread/per-session-based.

For example, there is an endpoint that holds error messages that contains user info (not session based) so a user can potentially see details for another user. Race conditions in login functionalities where static values are used to determine the user in the backend, so it is possible for a user to see another user's data upon logging in.
# Excessive Trust in Client-Side Controls

Use a web proxy to bypass any client-side restrictions implemented on the app. You can change the values for critical parameters and potentially break the logic of the app. For example, changing the price of an item in the shopping cart to an arbitrary value. Another example may be changing the quantity value of an item purchased to a negative number, which may bring the total price of the shopping cart order down.
# Low-level Logic Flaw

Manipulate a numeric input field so that its value reaches a very large number. Analyse how the app responds - maybe there is a limit and once it is reached, it may be reverted back to zero or a negative number. Depending on the context, the logic flaw can be very critical.

For example, if the total price of a shopping cart reverts back to 0, then this can be bad for the app/company.
# Inconsistent handling of exceptional input

Similiar to the previous point, by submitting a very large value to an input field, the app may truncate the value to a certain character size limit. Depending on the context, this can be used to bypass some restrictions on the app.

When using the exploit server in the labs, this an example of the submitted payload for the email client:

- `GGGGGGG@dontwannacry.com.exploit-01fuhgh6885ygju85784787477406097501.exploit-server.net`

For example, if the app allows users to register an account, try to submit a very large value for the email address. The app may truncate the email after confirmation and set it as GGGGGGG@dontwannacry.com in the app.
# Inconsistent security controls

After registering an account, identify if there are any "Update email" functionalities available. Use this and identify if the app requires verification on the new email address specified before fully updating the email address. If it does not, then you can update the email to an arbitrary value and potentially bypass some access controls.
# Weak Isolation on Dual-Use Endpoint

Remove parameters completely from requests and analyse how the app responds. This can potentially bypass some restrictions or logic that the application is using. For example, in a "change password" functionality, remove the "current-password" parameter if there is one.
# Insufficient workflow validation and Authentication bypass via flawed state machine

When going through a workflow/functionality, skip a step and see how the app responds. You may be able to bypass a critical step in the process. For example, in a "Cart checkout" workflow, skip the "checkout" step and go straight to the "order confirmation" steps.

Another example, if after logging into an app you must select a role, drop all of the requests after logging into the application and analyse how the app responds. You may be able to bypass some access control related functions, since the "role" was never selected - essentially a "force browse" bypass.
# Flawed Enforcement of Business Rules

If the app has 2 coupons that can be used to get a discount on an order, but these coupons should only be allowed to be used once per order, try submitting them one after another in the same purchase order and analyse how the app responds. This may bypass some flawed logic in the app.
# Infinite Money Logic Flaw

If the app offers a coupon code that can be used when submitting an order, check if this coupon can be used an infinite number of times once per order. For example, you can purchase a gift card and use the coupon code when purchasing it, and when redeeming the gift card, you earn a profit. An infinite money logic flaw can be exploit if the coupon can be used multiple times.

If using Intruder, the Max Concurrent Requests config should be set to 1, as the order of the requests is important.

