---
layout: with-comments
title: "How we use php-call-site-stats to get cache hit ratios per key"
author: Daniel Beardsley
author_url: https://github.com/danielbeardsley
summary: We use the php call-site-stats utility to collect and analyze cache
         hit ratios and DB query times for every get() and set() to our caching
         layer and every query() of our database.
---

We've been using [php-call-site-stats] in production since June 2013
and it's been a great asset to our team. We hpe you find it useful as well.

# What We Get

The [php-call-site-stats] project has enabled us to provide information like
mean,min,max,std-dev of the execution time for each and every place in our code
base where we make a database query. Note: units are in milliseconds and these
stats wer collected over a 24hour period.

    // avg:1.328 count:111 sum:147.361 std:3.753 min:0.465 max:35.607 - callsite(2013-11-25):
    $db->execute($q_insert, array(T_S, $text, T_S, $tagtype));

Hit ratios for all of our cache gets: (diff is the number of misses)

    // diff:476696 1508814 / 1985510 = 75.99% - callsite(2013-11-25):
    if (($reputation = $cache->get($key)) === false) {

Cache replacement times for each set() (i.e. time between a get
with a cache miss and the following set() of the same key.

    // avg:69.28 count:158 sum:10947.06 std:12.56 min:51.95 max:149.79 - callsite(2013-11-25):
    $cache->set($key, $device, CACHE_SHORT);

And lastly, we get query times for each usage of ::find() through our ORM.

    // avg:0.921 count:8457 sum:7792.918 std:2.332 min:0 max:44.624 - callsite(2013-11-25):
    $author = User::find($guide->authorid);

Every few months we turn on the recording of these stats for a day, aggregate
them and replace the existing stats in our entire codebase. The stats recording
incurs minimal overhead for each request (we track it too) with an average
additional time of 5ms. The process is very automated; Downloading the log from
each machine and aggregating them is a single command. Replacing existing
stats with new aggregated numbers is another single command.

# How It Works

1. Code in your caching layer records a stat
   `self::recordCallSite("cache-get", $gets = 1, $hit ? 1 : 0);`
1. php-call-site-stats examines a stack trace, walks back up the stack and
   records the first .php file after leaving the current class (i.e. the calling
   class).
1. A line like: "/path/to/file.php:241 cache-get 1 0" is written to the log
1. Logging is turned off and the all the log files are concatenated.
1. The [summarize] tool is used to aggregate by call-site
   (file/path:linenumber)
1. summarized results are inserted into source files at the appropiate places
   via some carefully crafted sed commands.

# How It's Used

There's a much more thorough explanation in the [README], but it only takes a
few lines of code to start logging lots of useful data:

Imagine you have a `Cache` class, decorate it like this and you'll get a record
of hits and misses for every place in our code that `$cache->get(...)` is
called.
{% highlight php startinline %}
class Cache {
   use CallSiteStats
   
   // Needed to record call-sites outside of this file
   protected function isExternalCallSite($file) {
      return $file != __FILE__;
   }
   
   public function get($key) {
     //...
     $this->recordCallSite('cache-get', $gets = 1, $cacheHit ? 1 : 0);
   }
}

// Then, during shutdown:
file_put_contents('cache-gets.log', $cache->getCallSiteStats(), FILE_APPEND);
{% endhighlight %}

[php-call-site-stats]: https://github.com/danielbeardsley/php-call-site-stats 
[README]: https://github.com/danielbeardsley/php-call-site-stats/blob/master/README.md
[summarize]: https://github.com/danielbeardsley/php-call-site-stats/blob/master/summarize.php 
