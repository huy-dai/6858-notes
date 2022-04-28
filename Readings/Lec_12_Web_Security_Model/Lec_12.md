# Readings about Web Security

## Basics of Same-Origin Policy
Link: <https://web.dev/same-origin-policy/>

**Same-origin policy** is a browser security features that restricts how documents and scripts from one origin can interact with resources from another origin.

An origin is defined by three elements:
 - Scheme (e.g protocol like HTTP or HTTPS)
 - Port if specified
 - Host

If all three components are the same for two URLs then they are considered same-origin.

For example, 

<http://www.example.com/foo> 
  - matches <http://www.example.com/bar> 
  - but not <https://www.example.com/bar> (Note the protocol)

Generally, embedding a cross-origin resource is permitted, but reading directly from a cross-origin resource is blocked. This applies to things like `iframes` and `images`. For `forms` cross-origin URLS can be used as the 'action' attribute of form elements (thus allowing writing form data to cross-origin destination). 

The difference between embedding and reading a resource is that when embedded, the resource is copied from the eternal origin and rendered locally, while reading the resource means their origin is preserved.

One common attack to work around cross-origin is **clickjacking**. This is when an attacker embeds a site in an `iframe` and overlays transparent buttons which link to a different and malicious destination. Thus users will think they are the displayed site when really they are sending data to the attackers.

## Deep dive into same-origin policy

Link: <https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy>

Same-origin policy help isolate malicious documents, and thus reducing possible attack vectors. 

Scripts executed from pages with an `about:blank` or `javascript:` URL **inherits** the origin of the document containing that URL, since normally these URLS do not contain information about an origin server.

### Cross-origin Interactions

Interactions between two different origins can be divided into three categories:

 - Cross-origin writes: Typically allowed. Includes links, redirects, and form submissions.
 - Cross-origin embedding: Typically allowed. Includes scripts, CSS stylesheets (need correct `Content-Type` header), images, video, fonts, iframes (can used `X-Frame-Options` to prevent cross-origin framing)
 - Cross-origin reads: Typically *disallowed*. However read access is often leaked by embedding.

### Allowing + Disallowing

To **allow** cross-origin access you can use CORS, which is a part of HTTP that let server specify any other hosts from which a browser should permit loading of content.

To **block** cross-origin access:
- For writes, check an unguessable token in the request known as the **Cross-Site Request Forgery (CSRF)** token. Dev must prevent cross-origin reads of pages that require this token.
- For reads, make sure it is not embeddable.
- For embeds, ensure resource cannot be interpreted as one of the embeddable formats. (For some browsers they may also not respect the `Content-Type` header, so you may also use CSRF token to prevent embedding)

### Documents Interactions

JavaScript APIS allow docments to directly reference each other (e.g. `window.parent`, `window.open`,`iframe.contentWindow`). When two documents do not have the same origin, these references provide very limited access to `Window` and `Location` objects.

However, documents from different origins can still communicate to each other using `window.postMessage`.

### Data storage access

Access to data stored in the browser like Web Storage and IndexedDB are separated by origin. Each origin gets its own separate storage, and Javascript in one origin cannot read from or write to the storage belonging to another origin.

However, **cookies** get a separate definition of origins. A page can set a cookie for its own domain or any *parent* domain, as long as the parent domain is not a public suffix. The browser will make a cookie available to any sub-domains, no matter the protocol or port used. 

## Using HTTP Cookies

Link: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies>

HTTP cookie is a small piece of data that a server sends to a user's web browser. The browser may store the cookie and send it back to the same server in alter requests. Typically, an HTTP cookie is used to tell if two requests come from the same browser. Thus it allows remembering **stateful** information for the stateless HTTP protocol.

### Usage of cookies

Cookies are mainly used for three purposes:

1. Session management
 - Logins, shopping carts, game scores, or anything else the server should remember
2. Personalization
 - User preferences, themes, and other settings
3. Tracking 
 - Recording and analyzing user behavior

### How cookies are created

Server sends back one or more `Set-Cookie` headers with the response, which causes browser to store the cookie and sends it with request afterwards made to the same server inside a `Cookie` HTTP header.

Cookies can have expiration date or time period, and additional restrictions to specific domains and path limits.

By default, the browser will send all previously stored cookies back to the server.

Cookies can have either **session** or **permanent** designation. Session cookies are deleted when the current session ends (which is defined by the browser), while permanent cookies are deleted at a specific date specified by `Expires` attribute.

### Restrict cookie access

Cookies with `Secure` attribute is only sent to the server with an encrypted request over HTTPS protocol (thus never HTTP). Additionally, cookies with `HttpOnly` attribute is not accessible to any client-side Javascript `Document.cookie` API; it's only sent to the server. This attribute helps with mitigating XSS attacks.

`Domain` and `Path` attributes define scope of the cookies (e.g. what URLS to send to). If `Domain` is not specified, the attribute default to only the host that sets the cookie, and **excludes subdomains**. Otherwise, subdomains under `Domain` list will have access.

`SameSite` attribute let server specify whether or when cookies are sent with cross-site requests. This provides some protection against CSRF attacks. There are three options:
- Strict: cookie only sent to site where it originated.
- Lax: Similar to strict, except cookies are sent when user navigates to the cookie's origin site.
- None: Cookies are sent on both originated and cross-site requests, but only in secure contexts (e.g. when `Secure` is set on cookie)

### Tracking and Privacy

If the domain and scheme of the cookie are different from the current page, it is referred to as a **third-party cookie**. While the server hosting a web page sets first-party cookies, the page may contain images or other components stored on servers in other domains. Usually these are mainly used for advertising and tracking across the web.

By default, Firefox blocks third-party cookies that are known to contain trackers.


## LocalStorage

Link: <https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage>

`localStorage` is the read-only property of the `window` interface which allows webpages to access a `Storage` object for the Document's origin. The stored data has no expiration time and saved across browser sessions.

This is in contrast to `sessionStorage`, which gets cleared when the page session ends (e.g. when the page is closed for Firefox

The keys and values stored with `localStorage` are always in the UTF-16 DOMString format, which uses two bytes per character.

## Cross-Origin Request Sharing

Often you need to load in certain resources from other origins that is blocked by same-origin policy. **CORS** allows this behavior in a standard way.

On the client side, when making a cross-origin request the browser adds an `Origin` header. When the server sees this header and wants to allow access, it adds an `Access-Control-Allow-Origin` header to the response specifying the requesting origin (or `*` to allow any origin). When the client browser receives this response, it will allow the response data to be shared with the client site.

### Sharing Cookies

By default, CORS is used as anonymous requests (e.g. without cookies). However, you can include cookies with the `credentials` option as part of the header info. Then when the server sends back info `Access-Control-Allow-Origin` *must* be set to a specific origin (no wildcard) and `Access-Control-Allow-Credentials` is set to True.

## Cross-site scripting attacks

XSS allows attackers to inject client-side scripts into web pages viewed by other users. XSS vulnerability is often used to bypass access controls like the same-origin policy. 

This works because the browser believes all data coming from the same source (or schema) is to be trusted and treated the same, and thus it operates under the permissions granted to that origin.

By injecting malicious scripts into web pages, an attacker can gain elevated access-privileges to sensitive page content, to session cookies, and to a variety of other information maintained by the browser on behalf of the user. 

### Types of XSS

Types of XSS includes:
- Non-persistent (reflected): This is when the vulnerability only affects the attacker themselves, but no other clients. 
- Persistent (stored): This occurs when the data provided by the attacker is saved on the server, and then is permanently served as part of the web page to other clients. A classic example is an online message board where users are allowed to post HTML formatted messages for other users to read.

### Preventative Measures

We can prevent XSS attacks with:
- Escaping of string inputs
- Safely validating untrusted HTML input
- Using additional security controls when handling cookie-based user authentication.
- Selectively or entirely disabling client-side scripts

## Clickjacking Attack

Clickjacking is the malicious technique of tricking a user into clicking something different from what the user perceives, thus potentially revealing confidential information or allowing others to take control of their computer.

Usually it's focused on loading a malicious page in a transparent layer on top of a real application, and then getting the users to interact with that malicious page unknowingly.

### Examples

- Classic: Overlay has Amazon page of product, so click (assuming user logged into Amazon and has 1-click ordering), will cause the order to go through.
- Likejacking: Causes users to like a certain page
- Cookiejacking: Trick the user to drag an object when seems harmless, but is in fact making the user select the entire content of the cookie being targeted.

### Mitigation
- Using client-side extensions and browser add-ons to detect clickjacking attempts
- Use frameworks to protect their apps against UI redressing
- Setting X-Frame-Options
- Setting content security options like same-origin policies

## Cross-site request forgery attacks

CSRF causes the user's browser to perform a request to the website's backend without the user's consent or knowledge. An attacker can use an XSS payload to launch a CSRF attack.

Example includes embedding some image into some page permanently that actually is a script which sends a request to the bank's server as you to withdraw money (this depends on you being logged into bank and the cookies are still valid).

### Mitigiation

- A CSRF token should be included in `<form>` elements via a hidden input field. This token should be unique per user and stored (for example, in a cookie) such that the server can look up the expected value when the request is sent. 
- Both CSRF tokens and SameSite cookies should be deployed. This ensures all browsers are protected and provides protection where SameSite cookies cannot help (such as attacks originating from a separate subdomain)

## Content Security Policy

Link: <https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP>

CSP is an added layer to detect and mitigate certain types of attacks like XSS and data injection attacks. To enable CSP, configure your web server to return the `Content-Security-Policy` HTTP header.

The primary goal of CSP is to mitigate and report XSS attacks. It does so by getting server admins to specify the domains that the browser should consider to be valid sources of executable scripts. The browser will then only execute scripts loaded in source files from those allowed domains, ignoring all other scripts. In addition, the server can also specify protocols are allowed to be used.

The entire policy is communicated as a value under the `Content-Security-Policy` header. A policy is described as a series of policy directives, each of us describes the policy for a certain resource type or policy area. 

The policy should have a `default-src` policy directive, which acts as a fallback for other resource types when they don't have policies of their own.

By default, violation reports aren't sent, but by specifying `report-uri` policy directive with at least URI you can get it to send the report to.