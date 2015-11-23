---
layout: post
title:  "Make a Lisp in Nim"
categories: nim
permalink: /blog/make-a-lisp-in-nim/
---

I spent the last weekend working through the amazing
[guide](https://github.com/kanaka/mal/blob/master/process/guide.md) for [Make a
Lisp](https://github.com/kanaka/mal), writing a Lisp interpreter in Nim. The
[final result](https://github.com/kanaka/mal/tree/master/nim) just made it into
the repository.

Running the Nim version is pretty simple. You need the Nim compiler from the [devel branch](https://github.com/araq/nim) and [nre](https://github.com/flaviut/nre) which can be installed through [nimble](https://github.com/nim-lang/nimble). After installing those you can build and run the MAL interpreter:

    $ cd nim
    $ make
      # OR
    $ nimble build
    $ ./stepA_mal
    Mal [nim]
    user> 12
    12
    user> (+ 2 3)
    5

<!--more-->
Running the tests:

    $ cd ..
    $ make "test^nim^step1" # A single test
    $ make "test^nim" # All tests
    $ make "perf^nim" # All benchmarks

There is a nice presentation in MAL about MAL you can read:

    $ cd nim
    $ ./stepA_mal ../examples/clojurewest2014.mal

And you can run MAL implented in MAL itself using the Nim interpreter:

    $ ./stepA_mal ../mal/stepA_mal.mal
    Mal [nim-mal]
    mal-user>

The benchmark results from [Joel Martin](https://github.com/kanaka), the author
of MAL, don't look bad for Nim. _Edit_: Note that these are just rough
measurements to see that the Nim implementation is doing fine. Don't judge the
other languages for their numbers, which may not have ideal implementations
performance-wise.

In the short benchmarks Nim is the fastest, in the long benchmark only the JVM
can beat it. I didn't really try to optimize and oriented mostly on the Python
implementation, which is quite a lot slower as you can see. So it's pretty nice
to see idiomatic Nim performing well:

                 perf1  perf2       perf3
                macros   math      macros
                    ms     ms   iters/sec
    java             6     24       17969
    scala           47     87       15963
    nim              1      1       11121
    ocaml            1      3        7063
    cs              10     11        5414
    vb              12     13        4523
    rust             2      5        4084
    c                1      4        3649
    go               1      6        3048
    racket           3     10        2461
    coffee           5     11        2326
    js               6     14        1726
    ruby             4     15        1255
    haskell          4     14        1163
    clojure         11     23        1174
    forth            9     29         563
    php             13     51         331
    python          14     51         304
    bash          1673   9000         276
    perl            18     69         215
    ps              41    332          48
    R              116    490          28
    miniMAL        779   3448           4
    matlab        1688   5844           2
    make          3427  28453           0


I also didn't really aim for a low amount of code but rather good readability,
but it may still be interesting to look at. I guess Nim's size being so close
to Python shows how much I oriented on the Python implementation:

                    Lines   Words   Chars
    mal               280     913    7075
    clojure           369    1164   10099
    ruby              481    1513   13247
    coffee            532    2020   15653
    racket            550    1937   17229
    python            670    1997   19632
    nim               715    2453   20263
    r                 812    2283   21644
    lua               914    2640   23102
    ocaml             597    3704   24371
    scala             871    3060   24885
    js                920    3059   26437
    php               940    3011   27332
    matlab            959    2485   28322
    perl             1081    3511   29091
    miniMAL           857    3104   29983
    haskell           985    4452   30115
    bash             1347    3586   31717
    go               1297    4744   36321
    ps               1409    6117   38651
    forth            1609    6826   44715
    cs               1249    4136   45039
    java             1591    4966   53223
    rust             1897    5702   56516
    vb               1556    5175   58099
    make             1593    6055   62310
    c                2304    6957   73047

Thanks to Joel Martin for this guide. This was my first time getting close to
Lisp and I recommend others to work through the MAL guide as well. But be
aware, once you get through the first steps, it becomes addictive.

Comments on [Reddit](https://www.reddit.com/r/programming/comments/2xx4wq/make_a_lisp_in_nim/) and [Hacker News](https://news.ycombinator.com/item?id=9145360).
