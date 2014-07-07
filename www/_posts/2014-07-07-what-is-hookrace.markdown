---
layout: post
title:  "What is HookRace?"
date:   2014-07-07 14:12:08
categories: hookrace, ddnet, nimrod
permalink: /blog/what-is-hookrace/
---

I have been running [DDraceNetwork](http://ddnet.tw) for 1 year, which started out as a little server of a [Teeworlds](http://teeworlds.com) modification called DDRace. It has turned into the biggest project within Teeworlds, offering servers in 6 countries with thousands of players and dozens of mappers. We also offer a custom server and client (Windows, Linux, OS X, Android) for the players and mappers.

Now it's time to start something new. I'm working on a new game called HookRace, which will be the successor to DDraceNetwork. It will be its own game, not based on Teeworlds.

As the programming language for HookRace, I chose [Nimrod](http://nimrod-lang.org):

{% highlight nimrod %}
# compute average line length
var count = 0
var sum = 0

for line in stdin.lines:
  count += 1
  sum += line.len

echo "Average line length: ",
  if count > 0: sum / count else: 0
{% endhighlight %}

On this blog I will report about my progress in the development of HookRace and my current Nimrod adventures.
