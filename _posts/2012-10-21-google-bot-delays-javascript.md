---
layout: with-comments
title: "Google Bot delays executing your JS"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: We inadvertently discovered that the Google Bot delays execution of
         your javascript code for hours (even days) after downloading your pages.
---

__The Google bot delays execution of your javascript for hours and sometimes even days after it crawls your pages.__

### Lead up

We inadvertently discovered this after making a _mistake_.
We removed some code that reported a users current timezone to our servers via Ajax.
It had been used to format times in a given users's timezone.
It was old code and it was an old idea,
we primarily use relative times now.

### Woops

After deploying the code removal,
Our error logs immediately started lighting up with: 
`Unknown ajax response function: setTimezone`.
We knew this wasn't a huge deal.
It was an Ajax call, it would be invisible to users,
nothing would appear broken.

_But why were we seeing these errors?_
Pages that are rendered now will _never_ make that ajax call.

Ahah!, our [Varnish](https://www.varnish-cache.org/) cache
keeps old pages around for a few minutes.
After restarting Varnish, the error logs were a lot cleaner.
_Most_, but not _all_, of the errors stopped.
We were now consistently seeing several per second like this:

    Oct 18 23:40:40 php: ◼|>>> [66.249.76.39] makeprojects.com {UserError} /Project/See-Thru+Potato+Cannon/5/1 Exception - All:
    Oct 18 23:40:40 php: Unknown ajax response function: setTimezone in ... <<<|◼

### Google!

It took us a bit to realize that all the IPs were similar (`66.249.*.*`).
A reverse lookup revealed that it's an [IP block owned by Google](http://www.tcpiputils.com/browse/ip-address/66.249.76.39).
In particular, it's a block used by the Goole bot for crawling the web.

Pretty quickly we came to the conclusion that the Goole bot must
execute javascript separate from (and after) downloading web pages.
It makes sense when you're doing things at Google scale,
they must have a giant queue of downloaded pages whose javascript is waiting to be executed.
But we didn't realize how _long_ they waited in that queue. 

It's __now been 3 days__ and we're still seeing errors trickle through.

This is a minor error and didn't really affect our customers at all.
But we can imagine other scenarios where after a deploy,
using old html and old javascript could make part of our site non-functional.
The Google bot could easily just assume all our pages were broken or missing content.

---

### tl;dr ###
If you're removing code or changing an endpoint, be careful you don't screw
the Google bot, which might be "viewing" 3-day-old pages on your altered backend.
