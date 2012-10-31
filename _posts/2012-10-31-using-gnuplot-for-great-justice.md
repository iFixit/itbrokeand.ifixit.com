---
layout: with-comments
title: "Using Gnuplot for Great Justice!"
author: James Pearson
author_url: http://github.com/xiongchiamiov
summary: Gnuplot can be used to quickly draw ASCII graphs of information pulled
         from logs or other textual sources.
---

We here at iFixit are big fans of [Graphite], but sometimes I want to plot data
that hasn't made it's way into Graphite yet.  For instance, today I wanted to
see how our [CSRF] failures have changed over time.

Since we log every failure, we can pull out instances pretty easily with
`grep`:

    [$]> grep CSRF php_debug.log | head -n 5
    Oct  1 01:16:39 10.250.0.0 php: ◼|>>> [0.0.0.0] Unknown user, invalid CSRF.  Found: klFWI Expected: TdWnO Array
    Oct  1 01:16:39 10.250.0.0 php: ◼|>>> [0.0.0.0] Unknown user, invalid CSRF.  Found: klFWI Expected: TdWnO Array
    Oct  1 01:16:40 10.250.0.0 php: ◼|>>> [0.0.0.0] Unknown user, invalid CSRF.  Found: b1B35 Expected: 5PDl2 Array
    Oct  1 01:16:40 10.250.0.0 php: ◼|>>> [0.0.0.0] Unknown user, invalid CSRF.  Found: b1B35 Expected: 5PDl2 Array
    Oct  1 01:16:42 10.250.0.0 php: ◼|>>> [0.0.0.0] Unknown user, invalid CSRF.  Found: MTXff Expected: CQgPQ Array
    
    [$]> grep CSRF php_debug.log > csrf_failures.log

Grouping and counting these entries by date is easy with `awk` and `sort`:

    [$]> awk '{print $1, $2}' csrf_failures.log | uniq -c
     148275 Oct 1
     120430 Oct 2
      91808 Oct 3
      85287 Oct 4
      82859 Oct 5
      81987 Oct 6
      87585 Oct 7
      88378 Oct 8
      87490 Oct 9
      82574 Oct 10
      77563 Oct 11
      80779 Oct 12
      88841 Oct 13
      91723 Oct 14
      97711 Oct 15
      92099 Oct 16
      89417 Oct 17
      69037 Oct 18
       3683 Oct 19
       4958 Oct 20
       4308 Oct 21
       6540 Oct 22
       5514 Oct 23
       3561 Oct 24
       4818 Oct 25
      22023 Oct 26
       3915 Oct 27
       4487 Oct 28
       4657 Oct 29
       4833 Oct 30
       1804 Oct 31

By the way, if you're still scared of awk, go read [Why you should learn just a
little Awk][awk] - it gives you the basics you need to do really useful things
without requiring you to learn all the intricacies of the tool.

Now, that's an interesting set of data, but it's a bit hard to visualize.
[Gnuplot] is the standard solution for this problem, but its syntax is... a bit
difficult to wrangle.  Thankfully, Christian Wolf created [eplot], a tool that
takes stdin and munges it to be gnuplot-consumable.  For terminal use, you'll
want to apply this patch:

    --- eplot.orig  2012-10-12 17:07:35.000000000 -0700
    +++ eplot       2012-10-12 17:09:06.000000000 -0700
    @@ -377,6 +377,7 @@
                    # ---- print the options
                    com="echo '\n"+getStyleString+@oc["MiscOptions"]
                    com=com+"set multiplot;\n" if doMultiPlot
    +               com=com+"set terminal dumb;\n"
                    com=com+"plot "+@oc["Range"]+comString+"\n'| gnuplot -persist"
                    printAndRun(com)
                    # ---- convert to PDF

Now, run our previous command through `eplot` and... tada!

    [$]> awk '{print $1, $2}' csrf_failures.log | uniq -c | eplot 
    
    
      160000 ++---------+----------+-----------+----------+----------+---------++
             *          +          +  "/tmp/eplot20121031-4728-zc5s8k-0" ****** +
      140000 +*                                                                ++
             |*                                                                 |
             | *                                                                |
      120000 ++*                                                               ++
             |  *                                                               |
      100000 ++ *                                                              ++
             |   *                         ******                               |
             |   *****   ********      ****      *                              |
       80000 ++       ***        ******           *                            ++
             |                                     *                            |
       60000 ++                                    *                           ++
             |                                     *                            |
             |                                      *                           |
       40000 ++                                     *                          ++
             |                                      *                           |
       20000 ++                                     *                *         ++
             |                                       *              * *         |
             +          +          +           +     ***********  ** +* ********+
           0 ++---------+----------+-----------+-----*----+-----**---+-*-------+*
             0          5          10          15         20         25         30


[Graphite]: http://graphite.wikidot.com/
[CSRF]: http://en.wikipedia.org/wiki/Csrf
[awk]: http://gregable.com/2010/09/why-you-should-know-just-little-awk.html
[Gnuplot]: http://www.gnuplot.info/
[eplot]: http://liris.cnrs.fr/christian.wolf/software/eplot/index.html

