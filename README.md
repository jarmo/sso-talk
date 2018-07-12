---
title: SSSO
theme: black
revealOptions:
    transition: zoom
---

![logo](/images/tobeornottobe.jpg)

----

# To OAuth or not to OAuth‽*

[* https://en.wikipedia.org/wiki/Interrobang](https://en.wikipedia.org/wiki/Interrobang) <!-- .element: style="float: right; font-size: 20px;" -->

---

## Customer Requirements #1

* Allow end user to access 3rd party application with their existing credentials <!-- .element: class="fragment" -->
* Do not leak user credentials to 3rd party application <!-- .element: class="fragment" -->

Let's use OAuth? <!-- .element: class="fragment" -->

----

## What is OAuth?

![logo](/images/oauth-logo.png) <!-- .element: style="max-width: 150px; float: right;" -->

OAuth is an open standard for access delegation, commonly used as a way for Internet users to grant websites or applications access to their information on other websites but without giving them the passwords. <!-- .element: style="text-align: left" -->

----

## YES, let's use OAuth!

---

![oauth-so](/images/oauth-so.png)

---

### Authentication vs Authorization

**Authentication** deals information about _"who one is"_. **Authorization** deals information about _"who grants what permissions to whom"_.

Since everyone else is also using OAuth for authentication (so-called social-login) we should be good! <!-- .element: class="fragment" -->

[If we avoid common pitfalls](https://oauth.net/articles/authentication/#common-pitfalls) <!-- .element: class="fragment" style="float: right; font-size: 22px;" -->

----

## Implementation

OAuth protocol implementation is relatively easy.

---

### User is authorized

* user requests a resource (app.example.org/home)
* server checks that user is authorized and responds with the requested resource <!-- .element: class="fragment" -->

---

### User is not authorized

* user requests a resource (app.example.org/home)
* server checks that user is not authorized and redirects to OAuth endpoint (auth.example.org) <!-- .element: class="fragment" -->
* user sees login form and posts it with its username and password (auth.example.org) <!-- .element: class="fragment" -->

---

### User is not authorized #2

* OAuth endpoint verifies user credentials and creates an authorization
* user is redirected back to original application using redirectUrl and a one-time authorization code (app.example.org/home) <!-- .element: class="fragment" -->

---

### User is not authorized #3

* application server exchanges one-time authorization code to access token by performing request to OAuth endpoint
* access token is stored in user session and is not revealed to the client <!-- .element: class="fragment" -->
  * or is when there is no server session, for example mobile applications <!-- .element: class="fragment" -->

EASY! <!-- .element: class="fragment" -->

----

## Customer Requirements #2

* 3rd party application should be shown in an iframe

By OAuth specification different applications have different ids, secrets and
so on, but user should be authenticated and authorized when opening in iframe. <!-- .element: class="fragment" -->

SOLUTION: let's store authenticated user id in OAuth endpoint session so that
we can reuse it for different applications when needed! <!-- .element: class="fragment" style="text-align: left" -->

---

### Reusing sessions between applications

* authorized user requests application page (app.example.org/settings)
* inside /settings an iframe is requested (settings.example.org) <!-- .element: class="fragment" -->
* since settings endpoint does not have an active user session, a redirect to OAuth endpoint happens <!-- .element: class="fragment" -->

---

### Reusing sessions between applications #2

* OAuth endpoint does not find any active authorizations for settings endpoint and should show login form
* BUT, it will see that it has an user id saved to its session and finds an active authorization for different endpoint (app.example.org) <!-- .element: class="fragment" -->
  * user is authorized now and it is redirected back to settings.example.org <!-- .element: class="fragment" -->

----

## Customer Requirements #3

* logging out from any application should log out from all applications

SOLUTION: create a token revocation endpoint to OAuth, which will remove all
user authorizations <!-- .element: class="fragment" style="text-align: left" -->

PROBLEM: user is not logged out from applications immediately, but only after they try to refresh their access tokens by contacting OAuth endpoint <!-- .element: class="fragment" style="text-align: left" -->

----

## Customer Requirements #4

* user should not be able to create two parallel authorizations belonging to
  same OAuth client

SOLUTION: revoke all tokens belonging to the same OAuth client before creating
a new one <!-- .element: class="fragment" style="text-align: left" -->

PROBLEM: conflicts with the requirement of reusing sessions between
applications <!-- .element: class="fragment" style="text-align: left" -->

---

![feature-toggle](/images/feature-toggle.jpg)

---

![smartass](/images/smartass.jpg) <!-- .element: style="width: 300px;" -->

We will find a way to add this functionality, we're smart!

----

## Customer Requirements #4

* show login form to the user in his/her locale

SOLUTION: add locale parameter support to auth.example.org <!-- .element: class="fragment" style="text-align: left" -->

----

## Customer Requirements #5

* initial application should show interface in correct locale after
  authentication
  * user can change its locale on login form too! <!-- .element: class="fragment" -->

SOLUTION: persist user locale to database and serve it as a part of
authorization response when authorization code is exchanged to access token <!-- .element: class="fragment" style="text-align: left" -->

----

## Customer Requirements #6

* show correct locale inside of an iframe when user changes its locale after login

SOLUTION: add locale parameter to iframe source URL and support that parameter... <!-- .element: class="fragment" style="text-align: left" -->

---

![wtf](/images/wtf.jpg)

----

## Are we even doing the right thing in here‽

* creating SSO (Single-Sign-On) for customer applications
  * no actual 3rd party applications are involved <!-- .element: class="fragment" -->
* mixing global locale with authorization logic <!-- .element: class="fragment" -->
* having delay between different applications when authorization is revoked <!-- .element: class="fragment" -->
* leaking access token to client side by using it as an iframe parameter and/or inside JavaScript <!-- .element: class="fragment" -->
* spaghetti comin' up! <!-- .element: class="fragment" -->

----

## What if there is a better way?

---

### Recap of customer requirements

* user should be authenticated between applications (SSO)
* logging out from one application should log out from all applications <!-- .element: class="fragment" -->
* user should not have more than one active session <!-- .element: class="fragment" -->
* user locale should be the same everywhere <!-- .element: class="fragment" -->
* [SEC] do not leak "access token" to client side <!-- .element: class="fragment" -->

----

## Alternative solution - no OAuth

Let's create our own SSSO (Simple-Single-Sign-On)!

---

### General Overview

* auth.example.org API is simple
  * GET / => shows login form <!-- .element: class="fragment" -->
  * POST / => creates authorization and redirects to redirectUrl <!-- .element: class="fragment" -->
  * GET /session => returns data about session <!-- .element: class="fragment" -->
  * POST /logout => revokes user authorization and shows login form <!-- .element: class="fragment" -->
* getting/setting locale is not part of authentication <!-- .element: class="fragment" -->

---

### GET auth.example.org/

* if authorization already exists then redirect to redirectUrl
* if authorization does not exist, then show login form <!-- .element: class="fragment" -->

---

### POST auth.example.org/

* if credentials are correct
  * if previous authorization exists for the same user, revoke that <!-- .element: class="fragment" -->
  * create an authorization <!-- .element: class="fragment" -->
  * create a cookie EXAMPLE\_SSO\_SESSION\_ID to domain example.org <!-- .element: class="fragment" -->
  * redirect to redirectUrl <!-- .element: class="fragment" -->
* if credentials are incorrect show login form with an error message <!-- .element: class="fragment" -->

---

### POST auth.example.org/logout

* revoke user authorizations
* remove cookie EXAMPLE\_SSO\_SESSION\_ID <!-- .element: class="fragment" -->
* redirect to login form <!-- .element: class="fragment" -->

---

### GET app.example.org/home

* if cookie EXAMPLE\_SSO\_SESSION\_ID does not exist
  * redirect to auth.example.org with redirectUrl app.example.org/home <!-- .element: class="fragment" -->
* if cookie EXAMPLE\_SSO\_SESSION\_ID exists <!-- .element: class="fragment" -->
  * GET auth.example.org/session => {userId: XXX, expiresAt: "2018-07-12T13:24:51.016Z", ...} <!-- .element: class="fragment" -->
    * expiresAt is prolonged with each request to /session <!-- .element: class="fragment" -->
  * respond with /home contents for the authorized user <!-- .element: class="fragment" -->

----

## What about locale?

Keep it separate from authentication API and reuse browser cookie functionality from client-side!

```javascript
let cookieParams = {
  EXAMPLE_LANG: "en",
  domain: "example.org",
  path: "/",
  "max-age": 365 * 24 * 60 *60
}
document.cookie = Object.entries(cookieParams)
                    .map(param => param.join("="))
                    .join("; ")
location.reload()
```

---

### Re-recap of customer requirements

* user should be authenticated between applications (SSO) ✓ <!-- .element: class="fragment" -->
* logging out from one application should log out from all applications ✓ <!-- .element: class="fragment" -->
* user should not have more than one active session ✓ <!-- .element: class="fragment" -->
* user locale should be the same everywhere ✓ <!-- .element: class="fragment" -->
* [SEC] do not leak "access token" to client side ✓ <!-- .element: class="fragment" -->

---

## Conclusion

* try to gather customer requirements as early as possible
* think before implementing anything
* reuse existing technologies instead of creating solutions which look simple
  first but end up being much more complicated in the end

----

## Thoughts and Questions
