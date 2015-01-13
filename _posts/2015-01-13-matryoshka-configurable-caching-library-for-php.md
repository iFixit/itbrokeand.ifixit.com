---
layout: with-comments
title: "Matryoshka: A Configurable Caching Library for PHP"
author: Marc Zych
author_url: https://github.com/marczych
summary: We recently open-sourced Matryoshka, a configurable caching library
         for PHP which makes common operations easier and allows for on-the-fly
         configuration.
---

Like most websites, we make heavy use of caching to reduce load on our servers and decrease page response times.
Our caching daemon of choice is [memcached].
The PHP extensions are certainly usable and provide all of the core functionality that you could need.
However, we use a lot of patterns to make our day to day caching much easier that aren't provided by the extensions.

In comes [Matryoshka], an open source caching library for PHP which makes common operations easier and allows for on-the-fly configuration.

# Configurable Behavior

Matryoshka is designed to be very configurable.
You can add functionality on the fly simply by wrapping an existing `Backend` with a new one.
We use this extensively to prefix keys, modify expiration times, disable gets, gather metrics, etc.

Start off with a `Memcache` instance:

{% highlight php startinline=true %}
// From the native extension.
$memcache = new Memcache();
$memcache->pconnect('localhost', 11211);
$cache = Matryoshka\Memcache::create($memcache);
{% endhighlight %}

Then prefix all the keys:

{% highlight php startinline=true %}
$prefixedCache = new Matryoshka\Prefix($cache, 'prefix-');
// The key ends up being "prefix-key".
$prefixedCache->set('key', 'value');
$value = $prefixedCache->get('key');
{% endhighlight %}

Finally double all expiration times for the prefixed backend:

{% highlight php startinline=true %}
$doubleExpiration = function($expiration) {
   return $expiration * 2;
};
$cache = new Matryoshka\ExpirationChange($prefixedCache,
 $doubleExpiration);
// Results in an expiration time of 20 for "prefix-key".
$cache->set('key', 'value', 10);
{% endhighlight %}

By using composition, caching configurations can be assembled on the fly quite easily.
The core API is identical for all backends so the caller doesn't need to be aware of the exact configuration.
Common configurations and base backends (like memcached connections) can be made into singletons or provided using dependency injection in your application.
Additionally, this architecture results in very maintainable and testable code because each class has exactly one job.

# Scopes

Cache invalidation is hard.
To make it easier, Matryoshka provides "cache scopes" to invalidate a group of keys at once.
[This works][Scope.php] by prefixing all keys with a unique value that is stored in the backend using the scope name.

{% highlight php startinline=true %}
$cache = new Matryoshka\Scope($memcachedBackend, 'name');

// This results in a get request to memcached for 'scope-name'
// which results in something like '0fb4ae36'. This `set` call
// then results in a key of '0fb4ae36-key'.
$cache->set('key', 'value');
$cache->set('key2', 'value2');
$value = $cache->get('key'); // '0fb4ae36-key' => 'value'

// Deleting the scope results in a new scope value e.g. 'e093f71e'.
$cache->deleteScope();

// Both of these result in a miss because the scope has a new
// value so the keys are now prefixed with 'e093f71e-'.
$value = $cache->get('key'); // 'e093f71e-key' => false
$value2 = $cache->get('key2'); // 'e093f71e-key2' => false
{% endhighlight %}

We have found this to be particularly useful for scoping keys to code deploys.
We simply put any caches that should be invalidated under the `'deploy'` scope which is deleted anytime we deploy code.
You can also have dynamic scopes such as `"post-{$postid}"` which can be cleared anytime a specific post is modified.
This is an implementation of [generational caching] which we make [heavy use of][caching thesis] throughout our application.

Cache scopes help with cache invalidation but unfortunately don't make [naming things] any easier.

# Helper Functions

Matryoshka adds a few helper functions to make common operations easier.
`getAndSet` makes populating the cache dead simple:

{% highlight php startinline=true %}
$cache = new Matryoshka\Ephemeral();
// Calls the provided callback if the key is not found and sets
// it in the cache before returning the value to the caller.
$value = $cache->getAndSet('key', function() {
   return 'value';
});
{% endhighlight %}

Similarly, `getAndSetMultiple` makes doing multi-gets significantly easier:

{% highlight php startinline=true %}
// Array of key => id. The id's can be anything used to identify
// the resource that the key represents.
$keys = [
   'key1' => ['id1a', 'id1b'],
   'key2' => ['id2a', 'id2b']
];
// Calls the provided callback for any missed keys so the missing
// values can be generated and set before returning them to the
// caller. The values are returned in the same order as the
// provided keys.
$values = $cache->getAndSetMultiple($keys, function($missing) {
   // Use the id's to fill in the missing values.
   foreach ($missing as $key => $primaryKey) {
      $missing[$key] = getValueFromDb($primaryKey);
   }

   // Return the new values to be cached and merged with the hits.
   return $missing;
});
{% endhighlight %}

# Try it out!

You can install Matryoshka with composer from [Packagist] or by cloning the [repo][Matryoshka] into your project.
A complete list of backends as well as more examples are available in the [readme].
[memcached], specifically the [Memcache extension], is the only supported caching daemon right now but adding others is very easy.
We encourage you to try it out and contribute any caching techniques that you find useful in your own applications.

Happy caching!

[memcached]: http://memcached.org/
[Matryoshka]: https://github.com/iFixit/Matryoshka
[readme]: https://github.com/iFixit/Matryoshka#readme
[Scope.php]: https://github.com/iFixit/Matryoshka/blob/master/library/iFixit/Matryoshka/Scope.php
[naming things]: http://martinfowler.com/bliki/TwoHardThings.html
[Packagist]: https://packagist.org/packages/ifixit/matryoshka
[Memcache extension]: http://php.net/manual/en/book.memcache.php
[generational caching]: https://signalvnoise.com/posts/3113-how-key-based-cache-expiration-works
[caching thesis]: http://digitalcommons.calpoly.edu/theses/1002/
