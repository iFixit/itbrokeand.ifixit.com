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
So, late one night, I wrote [cimpler](https://github.com/danielbeardsley/cimpler).
We've been using it in production since <time datetime="2012-10-08">Oct. 2012</time>
and it's been rock solid.

## What it does
**Cimpler is mostly just a queue**
with a few useful special-purpose constructs provided for its plugins to utilize.

Cimpler relies entirely on plugins for:

  * Adding builds to the queue _(github plugin)_
  * Consuming and removing builds _(git-build plugin)_
  * Reporting success or failure _(github-commit-status plugin)_
  * Anything else you want to do.

In our setup, the github hook plugin adds builds to the queue when github
sends a "post-receive" notification. The git-build plugin takes builds off the
queue, does some merging and then runs a shell command that executes our test
script. We also have the github-commit-status plugin report the build results
to github so we get the positive **"Good to merge"** message and link to the test
log.

<img class="screenshot" src="/assets/build-success.png"/>

Configuration is simple JSON, see

