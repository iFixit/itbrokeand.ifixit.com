---
layout: with-comments
title: "CSRF - How we protect ifixit.com from request forgery"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: We've come up with a minimalist method of protecting our site
         from cross site request forgery.
---

We've come up with a minimalist method of
protecting our site from cross site request forgery ([CSRF][csrf]).
Here's how we do it.

## What do we do?
We use a small amount of javascript
to automatically set a cookie and add a form field
immediately before a form is submitted.
The server then compares the form field with the cookie
and ignores the request if they don't match.
This frees us from the hassle of ensuring *every* form 
includes a hidden csrf `<input>` in *every* template.

## What does a CSRF attack allow?
Cross-site request forgery allows an attacker
to make arbitrary requests to a target site
on behalf of another user,
*with* their privileges but *without* their consent.

## How is a CSRF attack executed?
An attacker first figures out what is sent during a
a particular POST request on the target site
(maybe something important like deleting a Guide).
Next, they create a similar form (via html or javascript) on their own site.
When a user who is logged in on the target site visits the attackers site,
the form is submitted to the target site
and performs the action on behalf of the hapless user.

## How can it be prevented?
The root of CSRF prevention
is making the request submitter prove the request is from the allowed domain.
The easiest way to enforce this is to require the request submitter
to prove that they can read or write cookies on the allowed domain.

Often, this is done by having the server set a cookie to a random value
and include a hidden `<input>` in each `<form>` with the same value.
Upon submission, the server
compares the cookie with the value from the form
and ignores the request if they don't match.

### Problems with the standard approach
The standard approach requires
adding an `<input>` tag to each `<form>` in every template.
That's a lot of repeated code,
even if it's boiled down to a function call
or a single partial template;
there are often hundreds of forms in a typical web-app.
At the very least, that's just more boiler-plate code
that slows development progress.
At the worst, it's easy to forget to add the CSRF `<input>` tag
to some seldom-used forms and they'll effectively be broken.

### Our approach to prevention
We use javascript to dynamically set a cookie
and add an `<input>` element to each form
immediately before it's submitted.
Because we just have to prove that we can alter or read cookies,
we don't need to depend on the server-side for setting the cookie;
the client can do it just fine.
We hook into forms by replacing the `form.submit()` function
and listening for the `onsubmit` event.
This frees us from worrying about CSRF inputs and tokens in our templates.
It reduces the boiler-plate code for creating new pages and forms
and reduces the chance of mistakes.
One caveat: If forms are created dynamically in javascript,
an input element must be added while creating the form.
Luckily, this is trivial: `form.grab(CSRF.formField())`.

Here's the entire javascript implementation of our CSRF protection (Github [gist][gist]):
{% gist 6060418 %}

*Note: we use Mootools &mdash; refactoring for jQuery or no library would be
fairly straightforward.*

## How we deployed this
We designed it, tested it extensively, ran our test suite many times,
then soft-deployed it &mdash; silently reporting CSRF failures to a database.
Once we'd worked out a few issues
and were confident that we'd covered all our bases,
we flipped the enforcement switch on
and it's been working well ever since.

[csrf]:          http://en.wikipedia.org/wiki/CSRF
[gist]:          https://gist.github.com/danielbeardsley/6060418

