---
layout: post
title:  "Nim Code Coverage"
tags: Nim Programming
permalink: /blog/nim-code-coverage/
---

Creating code coverage reports with Nim is surprisingly easy. You can simply
use the good old gcov and lcov tools. Nim can be told to insert its own line
information with the `--debugger:native` command line parameter.

Here's the small example program we're looking at:

{% highlight nim %}
var x = 0
if x > 1:
  echo "foo"
echo "bar"
{% endhighlight %}

<!--more-->

Note that if we change the condition to `if false:` the Nim compiler optimizes
the impossible code away and it will not count as uncovered. The same thing can
happen with entire uncovered functions when optimizations and dead code
elimination are enabled.

The file is saved as `x.nim`. Here's the script I use to create the code
coverage report:

{% highlight bash %}
#!/bin/sh
rm -rf *.info html nimcache
nim --debugger:native --passC:--coverage --passL:--coverage c x
lcov --base-directory . --directory . --zerocounters -q
./x
lcov --base-directory . --directory . -c -o x.info
lcov --remove x.info "lib/*" -o x.info # remove Nim system libs from coverage
genhtml -o html x.info
{% endhighlight %}

You can look at the [final html report](/public/coverage-minimal/) generated.
