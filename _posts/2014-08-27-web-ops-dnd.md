---
layout: with-comments
title: "Web Operations D&D"
author: James Pearson
author_url: https://github.com/xiongchiamiov
summary: One of the most obvious, although hopefully infrequent, responsibilities
         of a Web Operations Engineer is firefighting - diagnosing issues that
         are critically affecting production services.  Unfortunately, most of
         us are pretty bad at it because every situation is different and the
         only time we practice is "on the job."  At iFixit, we've started a
         program to help with this, known as "Web Ops D&D".
---

One of the most obvious, although hopefully infrequent, responsibilities of a Web
Operations Engineer is firefighting - diagnosing issues that are critically
affecting production services.  Unfortunately, most of us are pretty bad at it
because every situation is different and the only time we practice is "on the
job".  At iFixit, we've started a program to help with this, known as "Web Ops
[D&D]".

The basic premise is that a senior engineer (the Dungeon Master) crafts a
scenario that a group of 2-4 other engineers (players) have to solve.

[D&D]: https://en.wikipedia.org/wiki/D%26D

# An Example

Rather than making up situations on the fly, I've created a set of index cards
with basic information about scenarios; the specific details are up to the DM
to create.  Here's an example card:

> Trigger: Alertra calls at 3:03 AM saying ifixit.com is unavailable.
>
> Direct Cause: HAProxy has taken all app machines out of rotation.
>
> Root Cause: We deployed a new SSL cert with a different filename the previous
> day.  We didn't update the symlink to the cert, so Apache failed to start
> when syslog bounced it.

The trigger is the only thing the DM provides to the players at the start; it
is whatever would cause us to realize that there is a problem.

For engineers unfamiliar with this sort of troubleshooting, it can be a bit
disorienting to start.  I found it's helpful to cover the basics of [the OODA
loop] and include one or two engineers in the group who have gone through the
activity before.

From there on, the DM's primary role is to answer questions without giving any
hints.  For instance, if the players ask "What does `mysql show processlist`
show?", "There are lots of queries stacked up" is a bad answer; "There are 129
rows; 80% of them are in the query state with a time of 80 or more, while the
rest are sleeping" is better.  Additionally, the players need to know *how* to
get the information they're asking for, so "And how do you get that?" is a
valid answer to "What is the MySQL status?".  Remember: the DM is functioning
merely as a verbal interface to an imaginary computer screen.  Use your
judgement here: sometimes it's important to make sure the players know the
specific syntax of a command, but if it's clear they know what they're doing,
don't make them spell it out exactly.

If you're feeling evil enough, the DM can also introduce new events in the
midst of the players' troubleshooting - there's nothing like a cascading
failure. :)

After the players succeed, then you get to have a post-mortem time.  This is
important for two reasons.  First, it allows the DM to give suggestions to the
players about additional tools and processes that would've been useful in the
scenario.  Secondly, now that the problem has been solved, the DM can fully
answer player questions about the architecture, the output of various tools,
etc.

[the OODA loop]: http://www.mindtools.com/pages/article/newTED_78.htm

# Results

After going through a few instances, separated by several weeks, engineers seem
to have a better grasp on our architecture and the process for tracking down
common issues.  Thankfully, we haven't had any unfortunately-timed outages to
force a real-world test of this knowledge yet. However, we have already been
able to improve the tools available for diagnosing issues with the production
architecture.

----

Have you done anything similar?  We'd love to hear about it!
