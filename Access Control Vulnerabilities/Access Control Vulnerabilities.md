![[Access Control.webp]]
# Recon

Map out the application and review the results of the mapping. You need to understand what the application's requirements are for access controls:

- Are there different levels of users in the app?
- Are users only given access to a subset of data belonging to them?
- Is there administrative functionality that can be accessed through the application?
- Are there any identifiers that may be used to determine a user's access? (e.g. ?admin=yes)

If there is only 1 user role, vertical privilege escalation may not be possible, but you can still find horizontal privilege escalation. For example, if a query parameter is used to determine which address to send your purchased items to (?addressId=7232) - is it possible to view other user's addresses by changing the value of the parameter?
# Testing with Different User Account

If you have access to a normal user and admin user, walk through the entire functionality that the higher privileged user can access and attempt to access that functionality with the lower privileged user. 

>[!warning]
>It requires strict comparison, functionality that both users should have access to is not vertical privilege escalation.

Use the "Compare Site Maps" feature to help map it out:

1. Login with admin creds and walk through the functionality of the app to populate the site map
2. Logout of the admin user
3. Login with a lower privileged user and make note of the session cookies.
4. Add the cookie to the Session Handling rules.
5. Go to Scope and select "Include all URLs".
6. Define the target scope (typically exclude login and logout endpoints).
7. Right click on the host and select "Compare site maps"
8. For site map 1, use the current site map (i.e. admin walk through)
9. For site map 2, request map 1 again in a different session context

If you have 2 accounts of the same privilege level, identify if horizontal privilege escalation is possible.
# Testing Multistage Process

Walk through a protected multistage process and use Burp Suite to access each one of the requests by a lower privileged user. Sometimes, the initial request in the process is protected but the subsequent requests are not protected from unauthorized access.
# Testing with Limited Access

If there is only 1 account to test with or no accounts, map out the app to identify any hidden sensitive/protected functionality. When pages are identified that may return different data depending on the user, try adding parameters/cookies such as "admin=true" or "debug=true".

Identify functionality where the app grants a user access to a subset of wider resources, such as emails, orders, documents, etc.. If the resources are retrieved through some predictable identifiers (?order=1234), try to determine the values that reference other resources you should not have access to and attempt to view the data.
# Testing Restrictions on HTTP Methods

An app's access controls may be bypassed by platform level controls. If there is a protected functionality that only a higher level privilege user can access, test whether this functionality can be accessed with a different HTTP method.

Then, determine if a lower level privilege user can bypass access controls using this method.
# Robots

Check the robots.txt page on the application and see if it reveals any sensitive locations on the application. Disclosed endpoints may be used to perform actions that you should not be authorized to do.
# Source Code Leak

Look at the source code of every page - there may be a client-side script or comments that disclose sensitive functionality or data.
# Sensitive Cookie

Identify if there are any client-side cookies that are being used to enforce your level of access on the application. Change the value and analyze how the app responds. As an example, a cookie such as "Admin=false".
# Mass Assignment

Identify if the app is vulnerable to a mass assignment vulnerability. For example, when updating your email address on the app, the response discloses a critical parameter called "roleID" - submit another request to update your email address, but also include the parameter "roleID" with a different value.

Analyze how it responds.
# HTTP Header Bypass - URL Based Access Control

Use non-standard headers to potentially bypass access control restrictions on endpoints. For example, the following header can be used to access the admin interface:

```html
GET /?username=carlos
X-Original-URL: /admin/delete
```
# HTTP Method Bypass

Use different HTTP methods when requesting a resource. This may bypass the access controls implemented on the endpoint. For example, use GET/PUT instead of POST.
# IDOR

IDOR allows you to specify a different resources belonging to another user by manipulating a query parameter. Always look into the source code of the app to identify if there is any sensitive data leakage. 

For example, ?id=user123 may be present - try changing it to ?id=456.

IDOR allows you to view another user's account information. For example, the parameter may be unpredictable, but there may be some functionality on the app which exposes other user's GUID or UUID values which can be used to view their account information in an endpoint like ?id=XXX-XXX.

If there is an endpoint for downloading items such as "/download/2.txt", attempt to change the file to "/download/1.txt" as it may disclose valuable information that may belong to other users on the application.
# Redirect Leakage

When submitting an invalid account on the following endpoint "/my-account?id=XXX", you may receive a 302 HTTP response with no body. If you submit a valid account, you may still receive a 302 HTTP response, but the body may disclose sensitive information for the specified user.
# Multi-Step Bypass

When testing multi-step processes with an admin account, test to see if a lower privilege user can bypass the access controls on any of those steps individually. If it is a 3-step process, the first 2 steps may be properly protected, but the last step may be left unsecured.
# Referer Header Bypass

The Referer header may be used to prevent unauthorized access to certain endpoints on the application. If you can figure out what the required value is, you may be able to bypass this restriction. As an example:

```html
GET /admin?delete?user=xxxx
Referer: https://vulnerable-app.com/admin
```

