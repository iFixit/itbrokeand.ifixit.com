---
layout: with-comments
title: "Memcache Stats in Graphite"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: Use [Graphite](http://graphite.wikidot.com/) and
         [Memcache](http://memcached.org/)? Want to track your memcache server
         stats in Graphite?  Bash has got your back.
---

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

