---
layout: with-comments
title: "CSRF - How we protect ifixit from request forgery"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: We've come up with a minimalist method of protecting our site
         from cross site request forgery.
---

We've come up with a minimalist method of
protecting our site from cross site request forgery ([CSRF][csrf])
and we'd like to share how we do it.

## What do we do?
We use a small amout of javascript
to automatically set a cookie and add a form field
immedaitely before a form is submitted.
This frees us from the hassle of ensuring *every* form 
includes a hidden csrf `<input>` in *every* template.

## What does a CSRF attack allow?
Cross-site request forgery allows an attacker
to make arbitrary requests to a target site
on behaf of another user,
with their privleges but without their consent.

## How is a CSRF attack executed?
An attacker first figures out what is sent during a
a particular POST request on the target site
(maybe something imoprtant like deleting a Guide).
Next, they create a similar form (via html or javascript) on their own site.
When a user who is logged in on the target site visits the attackers site,
it submits the form to the target site
and performs the action on behaf of the hapless user.

## How can it be prevented?
The root of CSRF prevention
is making the request submitter prove the request is from the allowed domain.
The easiest way to enforce this is to require the request submitter
to prove that they can read or write cookies on the allowed domain.

Often, this is done by having the server set a cookie to a random value
and include a hidden `<input>` tag in each `<form>` with the same value.
Upon submission, the server
compares the cookie with the value from the form.

### Problems with the standard approach

[csrf]:          http://en.wikipedia.org/wiki/CSRF

