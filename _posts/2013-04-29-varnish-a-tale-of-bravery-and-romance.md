---
layout: with-comments
title: "Varnish: A Tale of Bravery and Romance"
author: James Pearson
author_url: http://github.com/xiongchiamiov
summary: A story of failure and recovery, of ideal architectures and duct tape,
         of bravery and romance.  This is the story of Varnish.
---

When I joined iFixit two years ago, the upcoming release of a new [teardown]
was the source of much stress for those of us acting as the Web Operations
team.  The opportunity of a first look at the internals of Apple's newest gizmo
would drive thousands of geeks to our website, which would promptly crumble
under the weight of their combined curiosity.  A new teardown meant a day of
nursing injured servers wounded on the battlefield of Internet interest while
ordering new recruits into the fray, knowing full well they'd just be taken
down as well.  If you've ever seen the portrayal of the [Battle of Stalingrad]
in [Enemy at the Gates], you have a glimpse of what we were dealing with.

After [our site was obliterated][retina fail] by [our first 1/10 repairability
score][retina post], I was given a mission - make our infrastructure more
resilient before the release of the iPhone 5, upcoming in just a few months.  I
turned to a new tool, one that disguised its heroic nature underneath a simple
exterior.  That tool was [Varnish].

This is not a post about how to properly set up Varnish; if that is your
interest, start with [this lovely base from the NYTimes][nytimes] and then read
through [the Varnish Book].  This is a story of failure and recovery, of ideal
architectures and duct tape.  I tell this tale not as a model to strive
towards, but rather to convince the listener to stop procrastinating and start
doing - for despite the unexpected dangers and poor choices a hero must face in
any story, he must prevail, and in this case the sword that empowered him to do
so is available to any who wish to grab it.  Draw closer, my friends, and hear
of the day the dark forces of Reddit were driven back for good...

[teardown]: http://www.ifixit.com/teardown
[Battle of Stalingrad]: http://en.wikipedia.org/wiki/Battle_of_Stalingrad
[Enemy at the Gates]: http://www.imdb.com/title/tt0215750/
[retina fail]: http://www.reddit.com/r/technology/comments/uzt1n/apples_new_mbp_dubbed_the_least_repairable_laptop/c502sx7
[retina post]: http://ifixit.org/2753/macbook-pro-with-retina-display-teardown/
[Varnish]: https://www.varnish-cache.org/
[nytimes]: http://open.blogs.nytimes.com/2010/09/15/using-varnish-so-news-doesnt-break-your-server/
[the Varnish Book]: https://www.varnish-software.com/static/book/

# Preparations

As always seems to happen with deadlines, this one snuck up on me - the Retina
MBP teardown was in mid-June, but we didn't deploy Varnish until mid-August.
Even then, we turned off all caching, merely letting traffic flow untouched
through to HAProxy.  Finally, on September 12, 9 days before the iPhone 5
release, we enabled very conservative caching of our primary Wordpress blog -
the part of the site that fell the hardest during the previous teardown.  We
watched the logs, did some testing, made some configuration tweaks, and worked
feverishly on preparations for the main site.

The week passed by quicker than a jackrabbit pursued by .22-wielding teenagers.

The night before the release, while [Luke] waited outside the doors of an
Australian Apple store, we turned on caching of ifixit.com for anonymous users.
I was initially terrified of what the next day would bring, but the successful
handling of [large swarms of bees][bwmg] settled my mind enough to let me
sleep.  When I woke the next morning and continued making preparations for the
day to come, I found myself believing everything would work just as we wanted -
a sure sign of impending doom.

[Luke]: http://www.ifixit.com/User/Contributions/4/Luke+Soules
[bwmg]: https://github.com/newsapps/beeswithmachineguns

# The Day of Reckoning

Things did not go just as we wanted.

As we were about to publish the teardown, my nerves prompted me to double the
already-increased app machine pool.  This simple action inadvertently triggered
a cascade of errors - more machines meant more connections to memcached, more
than it was configured to handle.  And so as memcache calls began to timeout,
our application code fell back to fetching data from the database - but at a
rate far exceeding what it could handle.  An overloaded database is the
quickest way to bring down a website, and the laws of web operations held true
that day.

Of course, all I saw was the site operating normally for about 30 seconds (my
breath held captive the whole time) until response times shot through the roof
like fireworks stored in the flammable warehouse of our now-burning servers.
Frantic investigation lead to the discovery that memcached appeared to be dead
- and worse, every time I restarted it, memcached died again after only a few
brief minutes of life.  My dreams of a relaxing summer day vanished, replaced
by the destructive demons of DDoS.

It was time for Plan B.

# A Plan Composed of Duct Tape

Thankfully, our project manager had, with a well-tuned sense of caution,
insisted I make some sort of backup plan.  This was a single app machine, set
aside from standard rotation, with Apache turned off and Nginx configured to
serve static files.  I gave up on memcached for the time being and focused on
keeping the site available, through whatever means necessary.

I wrote a simple script that `wget`ed the multiple pages of our teardown into
the directory Nginx was serving.  This would act as our backend to Varnish,
serving "uncached" content.

Next, it was time to do something about Varnish.  My testing the previous night
had been overly simplistic; specifically, it used large numbers of completely
stateless requests, while actual visitors to our site were being assigned
sessions, thus bypassing our caching rules for every subsequent request.  As
with the connection limit in memcached, we only realized this in post-incident
analysis - in a crisis, humans operate sub-optimally and make poor
snap-judgements, if you don't have [structures in place to prevent
them][training resilience].  PSA: Please, *please* invest some time [training
your operations team][ooda] and [decreasing the probability of human failures
in your administrative tools][usability in firefighting].

Since we didn't realize how sessions were getting assigned, we thought there
were just too many logged-in users visiting the site.  With that in mind, we
decided to take a drastic step, one that ended up saving the site - we would
pretend all requests to the teardown were from anonymous users.

Unlike Apache, which has suffered the ill effects of an XML-based configuration
language growing features over time, Varnish uses a powerful C-like syntax
called VCL.  In VCL, this operation was as simple as putting something like
this in our `vcl_recv` function:

    if (req.url ~ "^/Teardown/iPhone+5+Teardown/10525/") {
        unset(req.http.Cookie);
        return (lookup);
    }

This strips off any cookies the client is sending and forces a cache-lookup on
the (now-simplified) request object.

In addition, we wanted to heavily cache the teardown page without altering the
caches on the rest of our site.  While the "correct" way to do this is through
the backend, Varnish again made it easy for us to make a quick change to the
same effect:

    sub vcl_fetch {
        if (req.http.host ~ "^(www\.)?ifixit\.com$") {
            if (req.url ~ "^/Teardown/iPhone+5+Teardown/10525/") {
                set beresp.ttl = 15m;
                return (deliver);
            } else {
                set beresp.ttl = 1m;
            }
        }
        # More rules about cookies, 404s, etc.
    }

And with that, we were set - operating a solution that'd make [Travis Taylor]
proud.

[training resilience]: http://www.kitchensoap.com/2011/05/10/training-organizational-resilience-in-escalating-situations/
[ooda]: http://blog.serverfault.com/2012/07/18/ooda-for-sysadmins/
[usability in firefighting]: http://changedmy.name/2012/01/05/the-importance-of-usability-in-regards-to-fire-fighting.html
[Travis Taylor]: http://en.wikipedia.org/wiki/Rocket_City_Rednecks

# Results

Things were a little odd.

Authenticated users were "logged-out" when visiting the teardown, but
mysteriously logged-in again as soon as they visited another page.  This was a
bit confusing, but normal actions continued to work, so nobody complained about
it too much.

Our tech writers could not see the changes they were making to the teardown on
the standard view page.  Their additions were, however, still displayed on the
edit page (which escaped our URL-matching), so they were able to continue to
write cooperatively.  When they had a good batch of updates, they'd ping me via
IM and I'd restart Varnish (to clear the cache) and temporarily route traffic
through to our normal application machines again just long enough for me to
take new snapshots for Nginx.

The rest of the site continued to function fairly normally - people could look
up repair guides for their devices, ask questions on [Answers] to get expert
opinions, and, most importantly, purchase parts and tools from the store. :)

And Varnish served up over 95% of the requests to our teardown in under a tenth
of a millisecond.  That is **fast**.

We analyzed our failures and fixed a number of things in the following few
days.  This was good, because a special on French television that aired the
next week gave us three times as many concurrent visitors as the Retina
teardown, our previous high point.  After I upped the number of connections in
Varnish, everything held up beautifully without further human intervention.

Now Varnish is an integral part of our infrastructure, handling edge caching
for a few popular pages during normal days.  It also serves as an important
safety net against unexpected traffic surges, allowing me to spend my days
improving other parts of our architecture instead of constantly worrying about
the influence of a single person talking about us in the wrong place.

But neither of these is the reason I love Varnish.  I love Varnish because it
is the Leatherman to my duct tape.  I love Varnish because it is a tool that
can be wielded in unexpected ways to make creative solutions at a moment's
notice.  I love Varnish because, when everything was going wrong, it allowed me
to engineer a solution that provided a high quality of service to our users.

I love Varnish, and I want you to, as well.

Reader, this is Varnish.  Varnish, this is Reader.  I'll let you two get
acquainted.

[Answers]:  http://www.ifixit.com/Answers

