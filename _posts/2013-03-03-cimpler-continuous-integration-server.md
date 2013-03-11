---
layout: default
title: "Cimpler - A continuous integration server"
author: Daniel Beardsley
author_url: http://github.com/danielbeardsley
summary: Cimpler is a continuous intergration server in node.js
         that is simple to configure, install, and use.
---

Cimpler: [Github Repo](https://github.com/danielbeardsley/cimpler)

At iFixit, we spent far too long
trying to configure [Jenkins](http://jenkins-ci.org/)
and its many plugins _just right_.
Jenkins and friends ended up being unintuitive (at least for our purposes)
and had far more features than we needed.

I love node.js and needed something simple, single-purpose, and extensible.
So, late one night, I wrote [cimpler][repo]
We've been using it in production since <time datetime="2012-10-08">Oct. 2012</time>
and it's been rock solid (we haven't even bothered to move it out of a screen
session).

## What it does

**Cimpler is mostly just a queue**
with a few useful special-purpose constructs provided for its plugins to utilize.

Cimpler relies entirely on plugins for:

  * Adding builds to the queue &mdash; _[github plugin][p-github]_,
     _[cli plugin][p-cli]_, _[post-receive hook][p-cli]_
  * Consuming and removing builds &mdash; _[git-build plugin][p-git-build]_
    * Also creates individual logs of each test run
  * Reporting success or failure &mdash; _[github-commit-status plugin][p-status]_
  * Anything else you want to do.

Plugins interact with cimpler through a straightforward API:

  * `cimpler.addBuild()`
  * `cimpler.consumeBuild()`
  * `registerMiddleware()` &mdash; register a connect HTTP middleware
  * Events:
    * `buildAdded`
    * `buildStarted`
    * `buildFinished`
    * `shutdown`

See the well documented [example config][config].

## How iFixit Uses Cimpler

The github plugin adds builds to the queue
when it receives a [post-receive][hooks]
notification from github.
The git-build plugin takes builds off the queue,
does some merging,
and then runs a shell command that executes our test script.
We also have the github-commit-status plugin report the build results to github
so we get the positive **"Good to merge"** message and a link to the test log on each of our pull requests.

### Like this :)
<img class="screenshot" src="/assets/build-success.png"/>

### Though sometimes like this :(
<img class="screenshot" src="/assets/build-failed.png"/>


## Configuration

Configuration is done via a javascript file that exports a JS object.
This is the entire configuration file we use at iFixit.

{% highlight javascript %}
module.exports = {
   /**
    * All plugin can register connect middleware that
    * will listen on this port.
    */
   httpPort: 12345,

   plugins: {
      'github-commit-status': {
         auth: {
            // Anything accepted github.authenticate()
            // https://github.com/ajaxorg/node-github
            type: 'basic',
            username: "[REDACTED]",
            password: "[REDACTED]"
         }
      },

      /**
       * So we can use the `cimpler status` command
       */
      'build-status': true,

      /**
       * github post-receive notifications plugin
       */
      github: true,

      /**
       * enable commandline plugin
       */
      cli: true,

      'git-build': {
         // 2 build dirs == 2 jobs can run in parallel
         repoPaths: ['/home/ci-1/Code/',
                     '/home/ci-2/Code/'],
         cmd: 'Tests/continuous-integration.sh',
         logs: {
            path: '/var/www/pub/cimpler-logs/',
            url:  'http://[REDACTED]/cimpler-logs/'
         },
         timeout: 20 * 60 * 1000 // 20 minutes
      }
   }
};
{% endhighlight %}

## The Future

We'll continue contributing to cimpler as we discover more features that we want.
We'll gladly accept contributions, so please fork the [Github Repo][repo] and
hack away!

[repo]:          https://github.com/danielbeardsley/cimpler
[p-github]:      https://github.com/danielbeardsley/cimpler/blob/master/plugins/github.js
[p-cli]:         https://github.com/danielbeardsley/cimpler/blob/master/plugins/cli.js
[p-git-build]:   https://github.com/danielbeardsley/cimpler/blob/master/plugins/git-build.js
[p-status]:      https://github.com/danielbeardsley/cimpler/blob/master/plugins/github-commit-status.js
[config]:        https://github.com/danielbeardsley/cimpler/blob/master/config.sample.js
[hooks]:         https://help.github.com/articles/post-receive-hooks

