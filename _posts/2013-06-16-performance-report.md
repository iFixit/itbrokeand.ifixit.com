---
layout: with-comments
title: "Performance Report: Summer 2013"
author: James Pearson
author_url: http://github.com/xiongchiamiov
summary: Etsy and others have recently been releasing very candid reports
         of their sites' performance.  Inspired by this, and wanting to both
         spur ourselves onward and to give others a view of how similar sites
         are doing, we present to you iFixit's first performance report.
---

[Etsy] and [others] have recently been releasing very candid reports of their
sites' performance.  Inspired by this, and wanting to both spur ourselves
onward and to give others a view of how similar sites are doing, we present to
you iFixit's first performance report.

We record certain sets of information about every request into [Graphite],
where it is rolled up into 10-second groupings.  Let's take a look at some
graphs of response times from last Friday, 2013-06-14.

![overall](/assets/2013-06-16-performance-report/overall.png)

Our overall response times stay at the 150ms mark.  Not bad.

![admin](/assets/2013-06-16-performance-report/admin.png)

The internal operations tools for handling stocking, shipping, and customer
service are used by far fewer people throughout the day than any other part of
our site; thus, the range of response times is much wider.  While our most
common operations are quite fast, less-used areas have suffered from neglect,
with their response times off this chart.

![answers](/assets/2013-06-16-performance-report/answers.png)

One of the main sections of the site is our Q&A product, [Answers].  Despite
being the most popular destination for visitors arriving from search engines,
Answers hasn't gotten as much work recently as it should have - which shows in
the 50ms increase of response times over the overall.

![search](/assets/2013-06-16-performance-report/search.png)

Thankfully, search requests somewhat make up for Answers, running almost
30-50ms faster than the average.  This is thanks in part to having a much
higher percentage of AJAX calls versus full page renders, as well as the
replacement of Sphinx for MySQL as the primary datasource.

Here are the averages calculated from Graphite across the whole day, using
Graphite's CSV output, a little unix shell magic, and [the average project]:

| Section | Median | Mean   | Std Deviation  | Min   | Max
|---------|--------|--------|----------------|-----------------
| Overall | 205.12 | 230.02 | 152.29         | 93.29 |  5683.25
| Admin   |  70.55 | 260.87 | 712.07         | 18.14 | 15437.67
| Answers | 233.92 | 243.19 | 102.22         | 79.15 |  6466.81
| Search  | 134.64 | 184.20 | 257.93         | 38.02 | 10752.06

This statistics are recorded inside our application, so let's grab a view from
outside:

![varnishhist](/assets/2013-06-16-performance-report/varnishhist.png)

This is a histogram from [Varnish], the caching server that sits at the very
edge of our control.  We avoid keeping disk logs to ensure it stays fast, but
the snapshot here is representative of a normal day.

Hash marks represent uncached requests - that is, those that go through to our
application.  Including the (internal) network time between the load balancer
and the application machines, uncached requests take between .01 and 1
seconds, biased just above .1 seconds (100ms).  This corroborates the data we
pulled from Graphite.

You'll also notice that cached pages (represented by vertical bars) are served
in less than *a tenth of a millisecond*.  At this point, the page response time
will effectively be that of the network round trip, which is about as good as
we can do from the backend.  Structuring our site such that Varnish can serve
pages from the cache is incredibly important to a responsive site.

--------------------------------------------------------------------------------

Our engineering team is more focused on and experienced with the backend than
the frontend.  However, we shouldn't ignore our site as soon as it passes off
to the user; considerations like the number and size of static assets and the
performance of our Javascript play an important role in how quickly a page is
perceived to load.

And frankly, we could do a lot better here.  Take a look at the stats Google
Analytics gives us from the last month:

Statistic              | Average # of Seconds
-----------------------|---------------------
Page Load Time         | 5.43
Redirection Time       | 0.06
Domain Lookup Time     | 0.03
Server Connection Time | 0.07
Server Response Time   | 0.50
Page Download Time     | 0.42

While our server times are reasonably low, the page load time is abysmal.  Five
and a half seconds?  Hrmph.

--------------------------------------------------------------------------------

Finally, let's take a look at our user-facing background jobs.  Since images
are such a big part of our guides, it's important for us to quickly respond to
users' requests to upload, crop, and mark-up images.

| Task           | Median | Mean | Std Deviation  | Min  | Max
|----------------|--------|------|----------------|--------------
| Upload         |   3.00 | 2.26 | 1.52           | 0.19 |  14.94
| Modify         |   8.72 | 9.54 | 7.50           | 0.21 | 105.22
| Generate Sizes |   2.92 | 2.23 | 1.21           | 0.25 |   5.89

While the new system [Cedric] just put into place improved our markup rendering
by an order of magnitude, it still seems high, both in the the max of almost
two minutes and the more reasonable median + deviation of seventeen seconds.

--------------------------------------------------------------------------------

Would you like to help us improve these numbers so we can [feed the repair
revolution][manifesto]?  We're always hiring talented folks just like you;
[give us a ring][jobs].

[Etsy]: http://codeascraft.com/category/performance/
[others]: http://engineering.wayfair.com/january-2012-site-performance-report/
[Graphite]: http://graphite.readthedocs.org/
[Answers]: http://www.ifixit.com/Answers/
[Varnish]: https://www.varnish-cache.org/
[the average project]: http://sourceforge.net/projects/average/
[Cedric]: https://github.com/cedmans
[manifesto]: http://www.ifixit.com/Manifesto
[jobs]: http://www.ifixit.com/Info/Jobs

