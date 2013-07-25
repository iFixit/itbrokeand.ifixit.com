---
layout: with-comments
title: "CSRF - How we protect ifixit.com from request forgery"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: We've come up with a minimalist method of protecting our site
         from cross site request forgery.
---

We've come up with a minimalist method of
protecting our site from cross site request forgery ([CSRF][csrf])
and we'd like to share how we do it.

## What do we do?
We use a small amount of javascript
to automatically set a cookie and add a form field
immediately before a form is submitted.
This frees us from the hassle of ensuring *every* form 
includes a hidden csrf `<input>` in *every* template.

## What does a CSRF attack allow?
Cross-site request forgery allows an attacker
to make arbitrary requests to a target site
on behaf of another user,
*with* their privleges but *without* their consent.

## How is a CSRF attack executed?
An attacker first figures out what is sent during a
a particular POST request on the target site
(maybe something important like deleting a Guide).
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
The standard approach requires
adding a `<input>` tag to each `<form>` in every template.
That's a lot of repeated code,
even if you boil it down to a function call
or a single partial template;
there are likely hundreds of forms across your application.
At the very least, that's just more boiler-plate code
that slows development progress.
At the worst, you may forget to add the CSRF `<input>` tag
to some seldom-used forms and they'll efectively be broken.

### Our approach to prevention
We use javascript to dynamically set a cookie
and add an `<input>` element to the form
immediately before it's submitted.
Because we just have to prove that we can alter or read cookies
we don't need to depend on the server-side for setting the cookie,
the client can do it just fine.
We hook into forms by replacing the `form.submit()` function
and listening for the `onsubmit` event.
This frees us from worrying about CSRF inputs and tokens in our templates.
It reduces the boiler-plate code for creating new pages and forms
and reduces the chance of mistakes.

Here's the entire javascript implementation of our CSRF protection [gist][gist]:
{% highlight javascript %}
var CSRF = (function() {
   window.addEvent('domready', function() {
      /**
       * Setup triggers on the .submit() function and the `submit` event so we
       * can add csrf inputs immediately before the form is submitted.
       */
      $$('form').each(function(form) {
         // Ensure all forms submitted via a traditional 'submit' button have
         // an up-to-date CSRF input
         form.addEvent('submit', function() {
            ensureFormHasCSRFFormField(form);
         });
         // Ensure all forms submitted via form.submit() have an up-to-date CSRF
         // input
         var oldSubmit = form.submit;
         // Wrap the default submit() function with our own
         form.submit = function () {
            ensureFormHasCSRFFormField(form);
            return oldSubmit.apply(form, Array.from(arguments));
         }
      });
   });

   /**
    * Generate a new token and store it in the cookie
    */
   function resetToken() {
      /**
       * random() generates a number [0..1] so the first two chars are always
       *'0.' no matter what the base.
       */
      var token = Math.random().toString(36).substring(2,12); // 10 char string

      // 30 days is pretty arbitrary, it could be 3 minutes or 3 years.
      Cookie.write('csrf', token, { duration: 30 }); // 30 days
      return token;
   }

   /**
    * Ensure the provided Form element has a CSRF token
    * input who's value is up to date.
    */
   function ensureFormHasCSRFFormField(form) {
      csrfInputForForm(form).set('value', CSRF.get());
   }

   /**
    * Return the csrf input element for the given form or give it one.
    */
   function csrfInputForForm(form) {
      var csrfInput = form.getElement('.csrf');
      if (!csrfInput) {
         csrfInput = CSRF.formField();
         form.grab(csrfInput);
      }

      return csrfInput;
   }

   return {
      /**
       * ## CSRF.get()
       *
       * Read the value from the cookie or create and return one
       * if it doesn't exist
       */
      get: function() {
         return Cookie.read('csrf') || resetToken();
      },

      /**
       * Returns a new hidden input field with the CSRF token as a value.
       * Used when dynamically creating forms.
       */
      formField: function() {
         return new Element("input", {
            type: 'hidden',
            name: 'csrf',
            'class': 'csrf',
            value: CSRF.get()
         });
      }
   };
})();
{% endhighlight %}

*Note: we use Mootools, but refactoring for no jQuery or no library would be
fairly straightforward.*

## How we deployed this
We designed it, tested it extensively, ran our test suite many times, then
soft-deployed it, silently reporting CSRF failures to a database. Once we'd
worked out a few issues and were confident that we'd covered all our bases we
flipped the switch on enforcement and it's been working well since.

[csrf]:          http://en.wikipedia.org/wiki/CSRF
[gist]:          https://gist.github.com/danielbeardsley/6060418

