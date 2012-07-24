---
layout: with-comments
title: "Forker: a PHP Library"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
---

A while back we released [Forker](http://github.com/ifixit/Forker), a small PHP
library that enables easy parallel processing.  

How We Use It
------------
Right now, we use it for network IO operations.  Particularly, we use it to
connect via SSH and execute commands on a bunch of machines in parallel.  This
has decreased the time it takes us to deploy new code to 2-3 seconds, instead
of 2-3*n seconds that it took before.

Here's an example of how you can use Forker to run a command on many machines
in parallel:
{% highlight php startinline %}
require "Forker.php";

$servers = array('machine1.example.com', 'machine2.example.com');

/**
 * Run a shell command on an array of servers using SSH,
 * returning the output and exit code.
 */
function runCmd($command, $servers) {
    return Forker::map($servers,
    function($index, $server) use ($command) {
        $sshCommand = "ssh $server -c '$command' 2>&1";
        exec($sshCommand, $exitCode, $output);
        return array(
            'output' => implode("\n", $output),
            'exitCode' => $exitCode
        );
    });
}

$results = runCmd("hostname", $servers);
print_r($results);
{% endhighlight %}

This would produce something along the lines of:
    Array
    (
        [0] => Array
            (
                [output] => machine1.example.com
                [exitCode] => 0
            )
        [1] => Array
            (
                [output] => machine1.example.com
                [exitCode] => 0
            )
    )


How It Works
------------

### fork()

Forker has three functions to provide the functionality
behind its one public function.  A protected method called
`fork()` creates a connected socket stream pair, forks the current
process and returns one socket to the parent, and one to
child. These sockets are used for one-way communication from
the child to the parent.

{% highlight php startinline %}
protected static function fork() {
   $results = array();
   $sockets = stream_socket_pair(STREAM_PF_UNIX,
                                 STREAM_SOCK_STREAM,
                                 STREAM_IPPROTO_IP);
   $pid     = pcntl_fork();

   if ($pid == -1) {
      die('Could not fork');

   } else if ($pid) {
        /* parent */
       fclose($sockets[1]);
       $results = array(
          'stream' => $sockets[0],
          'parent' => true
       );
   } else {
       /* child */
       fclose($sockets[0]);
       $results = array(
          'stream' => $sockets[1],
          'parent' => false
       );
   }

   return $results;
}
{% endhighlight %}


### mapStream()

Up one more level there is the `mapStream()` function.
This implements a typical map function over a provided array.
The map function passes to the callback the array entry and a connected stream.
The data written to the stream during each callback is read
from the other end by the parent and returned from the original
`mapStream()` call.

{% highlight php startinline %}
protected static function mapStream($things, $callback) {
   $children = array();

   foreach($things as $key => $value) {
      $info = self::fork();
      if ($info['parent']) {
         $children[$key] = $info;
      } else {
         $callback($key, $value, $info['stream']);
         fclose($info['stream']);
         exit;
      }
   }

   return self::getChildrenOutput($children);
}
{% endhighlight %}

The final public function 'map()' wraps the `mapStream()`
function and allows the caller jsut to return a php
object instead of writing to a stream.  Returned objects are serialized
and written to the stream and then deserialized before being returned
in the resultant array.

{% highlight php startinline %}
public static function map($things, $callback) {
   $outputStrings = self::mapStream($things,
   function($key, $value, $stream) use ($callback){
      $data = $callback($key, $value);
      fwrite($stream, serialize($data));
   });

   $results = array();
   foreach ($outputStrings as $key => $output) {
      if ($output === null || $output === '')
         $results[$key] = null;
      else
         $results[$key] = unserialize($output);
   }
   return $results;
}
{% endhighlight %}

The end result is an easy to use interface to [forking in
PHP](https://github.com/iFixit/forker), `Forker::map()`.

