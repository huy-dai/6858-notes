# Web Security

Web browser is responsible for managing multiple websites running on the computer at once in different tabs. It's important to keep these websites instances separated from each other and also from your underlying computer (local filesystem, other applications, etc.)

## Story of Web Evolution

At the start of the browser or web no one really know how it was going to progress. Initially they only needed support for text-only web application. Over time things such as cookies (for stateful requests), Javascript (as a way of running code in local browser), etc. was added, leading to the creation of many complex web applications & rapid evolution of features.

Thus much of the security implementations were constraint in the sense that they were done retroactively, and there was stringent requirements for compatibility. In addition, security choices and designs vary across browsers. In addition, one extra challenge of web browsers is that web applications has a very high level of sharing (thus requiring a lot of inter-connectivity).

## Threat Model

Suppose that the attacker controls a web site, like `attacker.com`, and you end up visiting the attacker's web site. In addition to the attacker's tab you also have other important web applications running in other tabs (like your banking account).

Things that we **trust**:
 - Browser (we can design it to contain attackers)
   - Assume that the browser implementation is non-buggy, e.g. stuff we say we add actually works as intended.
 - Network  
   - Assume that the network is secure, so no Man in the Middle attack

## Same-Origin Policy

Origin is defined by three components:
 - hostname, port, protocol

The browser labels each each resource (like network server, displayed page, JS variable, DOM elements, cookies, etc.) with an origin. The origin essentially tells us the web server the page (or iframe) came from. Whenever a web page wants to access a resource we will check its origin with the resource's origin and see if they are the same. If it is not the resource access will be denied.

Through the same-origin policy, a script, which runs on the behalf of some origin, can only access a resource if they have the same origin

Example: XMLHttpRequest()
  XMLHttpRequest(url) is a JavaScript call
  It fetches URL and lets JavaScript see the result.
  Often used to get at "web APIs", to fetch data for JS.
  Could attacker.com use it to steal data from gmail?
    No: browser enforces SOP, so it can only fetch from the
        same server the JavaScript came from.

### SOP Exceptions

- Links
  - Ordinary links are allowed without consideration for same-origin policy enforcement. This makes sense as we want to allow for different websites to link to each other. Plus, usually inter-domain links are harmless. But it should be noted that this in some sense gives the attackers a control over where the users is redirected to. This is not a security issue in itself, but it can be if the website you're visiting has some side effects (e.g. by clicking on `gmail.com` from attacker's website they effectively got you to log in to `gmail.com` at that moment), which may open up opportunities for other attacks.
- Images
  - Browser will fetch and display the image, despite different origins. 
  - This exception is necessary because we often have a lot of embedded images in web page. However, the browser still enforced SOP to the retrieved pixels, so that the IMG **can't** just fetch and inspect web pages of the parent website it is in. In addition, the parent page can't look into the composition of the child.
- Scripts
  - Often in your scripts you will want to load some public library hosted in a different origin.
  - This is still allowed as an exception to SOP. The script you load will end up running **as the same origin** as your page which calls it.
- iframe
  - It loads and displays a web page within a rectangle box on the page. The framed page can come from anywhere, and SOP is not imposed on fetch.
  - This exception is necessary as iframes are often used for advertisements, Facebook "like" buttons, etc.
  - While SOP is not applied on the fetching of contents for iframes, it is applied to the frame's actions one fetched. This means that if the main frame and frame's origins are different, then the SOP prevents them from interacting in most ways (defends each against the other). 
  - Also the iframe gets the SOP rights corresponding to its origin, just as you would see in a normal table.
  - However, the main page can still navigate the frame via JS.

## Cookies

Cookies allow servers to preserve state in the browser (make web applications from being stateless to stateful). Once authenticates to the web server, the web server will send back a `set_cookie` request asking the local browser to store the cookie in its table. The table will consist of columns `domain`, `key`, `value`. Cookies can be set to be used for a particular set of domains (ex. you log into mail.google.com but it wants to give you access to the rest of the Google ecosystem like Google Drive and Youtube). By default if no `domain` attribute is provided it is only applied for the domain the cookie came from.

The cookie is sent on all requests (matching the targeted URL). Javascript code running in a specific tab for a certain origin is also allowed to access cookies with matching origin. 

The complicated part is "Who can set a cookie?" Servers can only set cookies for itself or others inside its domain. This means a.com cannot set cookie for gmail.com since they are different domains, but www.a.com can set cookie for ftp.a.com. The reason why we restrict this behavior is because if a.com can set a cookie that gets sent to gmail.com there can be instances where this behavior causes `gmail.com` to *associate* services for the current user with the **attacker's account**. This is called a **session fixation attack**. 
  - Important! The part of the URL that you want to fix will depend on your use case. For example, .co.uk is too open (since many different government organizations) starts like this. Instead there is a **public suffix list** that is set universally and all browsers will have a copy of this list.


## Cross-Site Request Forgery (CSRF)

Suppose a page from attacker.com contains `<IMG SRC="https://bank.com/xfer?amount=500&to=attacker">`. The user doesn't see anything special, maybe a little broken image. However, if the user is logged in to the bank and has active cookies, then loading this image would cause for a request to be made on behalf of the user without the user knowing. Note: Usually it's more common to execute this attack with a `form` action link.

### Defenses against CSRF
- **CSRF token:** The real application requires both normal cookies and an extra CSRF token. When the user legitimately loads the real application in their browser, the page will be given a user-specific CSRF token which it stores in the **content of the real application website**. Then, for every request the token would then send this along the normal cookies. The server would have data structures to check if a token is valid. The main kicker is that this token would not be made available to other pages since it's not under the normal cookie table. This means the attacker must look inside the real website application to figure out the CSRF token, but according to the SOP this is not possible
  - Nowadays support for CSRF token is becoming more prominent. For some framework like Flask it is including as part of the default web page.

## Clickjacking

One security issue that is brought about by iframe is that it creates a level of ambiguity in terms of which website the user is currently interacting with at any time. In a clickjacking attack, the attacker will overlay a website with side effects on top of another application in a transparent layer, so that when the user thinks they are interacting with the bottom application they are actually interacting with the top application. This applies to more of one-click action, like liking a Facebook post or one-click Amazon buy.

For some websites they might choose to disallow themselves from being embedded as an iframe in a website (Amazon can do this in theory, Youtube can't), or somehow treat actions done in an iframe as being suspect.

## Cross Site Scripting (XSS)

Imagine if an attacker posts a comment with HTML content embedded. If FB isn't careful to sanitize the user's inputs then that HTML code will then be saved and later ran as actual code on the website (with FB's origin). This could potentially allow for the attacker to steal the user's FB cookies or perform some actions on their behalf (like submitting lots of spam posts).

