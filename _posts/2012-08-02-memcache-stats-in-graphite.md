---
layout: with-comments
title: "Memcache Stats in Graphite"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: Use [Graphite](http://graphite.wikidot.com/) and
         [Memcache](http://memcached.org/)? Want to track your memcache server
         stats in Graphite?  Bash has got your back.
---

------
### Update (08-25-2012) ###

This concept has been extended further and genericized into a friendly bash
component called [pipe-to-graphite](https://github.com/iFixit/pipe-to-graphite)
that allows you to easily pipe the output of any script to graphite.  This has
been [done before](https://github.com/BrightcoveOS/Diamond) and quite well, so
I'd recommend using Diamond, but this was fun to hack on anyway.

The memcache script is now MUCH simpler than the one below and allows for an
optional "extended" mode that reports more stats:

{% highlight bash %}
#!/bin/bash
argument="$1"
(
    sleep 1
    [ "$argument" == "extended" ] &&
      echo "stats slabs" &&
      echo "stats items"
    echo "stats"
    sleep 1
    echo "quit"
) | telnet localhost 11211 2>/dev/null |
grep STAT |
grep -v version |
sed -re 's/STAT (items:)?([0-9]+):/memcache.slabs.\2./' \
     -e 's/STAT /memcache./'
{% endhighlight %}

There are also scripts included to monitor Gearman and Mysql.  
You can clone the repo and start monitoring things quickly:

{% highlight bash %}
$ > git clone git://github.com/iFixit/pipe-to-graphite.git
$ > cd pipe-to-graphite
$ > ./pipe-to-graphite.sh scripts/memcache-stats.sh
Running 'scripts/memcache-stats.sh' as a test.. SUCCESS

Redirecting stdout to /dev/null so it doesn't mess up your
terminal.  Redirect it somewhere else if you wan't to save it.

Command: scripts/memcache-stats.sh
is being piped to graphite every 10 seconds
Background PID: 18637
$ >
{% endhighlight %}

------

### Original Post ###

Use [Graphite](http://graphite.wikidot.com/) and
[Memcache](http://memcached.org/)?  Want to track your memcache server
stats in Graphite? This bash snippet is the answer and you'll end up
with beautiful graphs of memcache numerical goodness like this:

![Memcache stats](/assets/memcache-stats-graph.png "Why the high write / read
ratio?")

We use both Graphite and Memcache here at [iFixit](httpd://www.ifixit.com)
and we wanted to get the stats straight from memcached's telnet interface
into grahite.

A quick hacking about in bash and we've got what we want.
This script gets some stats from memcache, does a bit of string
munging, and sends them to your Graphite server using netcat.  It
echo's to stdout too, in case you want to `log everything`.

### memcache-stats.sh -- [Public Gist](https://gist.github.com/3235099)
{% highlight bash %}
#!/bin/bash

if [ "$1" != "report" ]; then
   echo "Usage:" >&2
   script="`basename $0`"
   echo "  nohup $script report > /var/log/memcache-stats.log &" >&2
   exit 1
fi

GRAPHITE_SERVER=localhost
GRAPHITE_PORT=2003
GRAPHITE_INTERVAL=10

while true; do
   # Do it in a backgrounded subshell so we can move
   # directly on to sleeping for $GRAPHITE_INTERVAL
   (
      # Get a timestamp for sending to graphite
      ts=`date +%s`

      # memcache gives us some decent stats in the form of 
      # STAT bytes_read 4535820
      output=`service memcached status 2>/dev/null |
              grep STAT |
              grep -v version |
              sed "s/STAT /memcache\./"` 

      # Pipe the output through sed, using a regex to
      # append a $ts timestamp to the end of each line,
      # and then to the correct server and port using netcat
      echo "$output" |
       sed "s/\$/ $ts/" |
       nc $GRAPHITE_SERVER $GRAPHITE_PORT

      # Echo this data too in case we want to record
      # it to a log
      echo `date "+%Y-%m-%d_%H:%M:%S"`
      echo "$output"
      echo; echo
   ) &
   sleep $GRAPHITE_INTERVAL
done;
{% endhighlight %}

