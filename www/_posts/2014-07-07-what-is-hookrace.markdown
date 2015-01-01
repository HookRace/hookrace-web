---
layout: post
title:  "What is HookRace?"
categories: hookrace, ddnet, nim
permalink: /blog/what-is-hookrace/
---

I have been running [DDraceNetwork](http://ddnet.tw) for 1 year, which started out as a little server of a [Teeworlds](http://teeworlds.com) modification called DDRace. It has turned into the biggest project within Teeworlds, offering servers in 6 countries with thousands of players and dozens of mappers. We also offer a custom server and client (Windows, Linux, OS X, Android).

Now it's time to start something new. I'm working on a new game called HookRace, which will be the successor to DDraceNetwork. It will be its own game, not based on Teeworlds.

Some of the plans I have so far for HookRace:

- Vector Graphics (SVG) ingame for a sharp image on any screen
- Whole physics is executed at the client as well as the server, for perfect prediction
- No lags with high player numbers
- Accounts to prevent faking

As the programming language for HookRace, I chose [Nim](http://nim-lang.org):

{% highlight nimrod %}
# Game of Life in terminal
import os, strutils, math

randomize()

var w, h: int
if paramCount() >= 2:
  w = parseInt paramStr 1
  h = parseInt paramStr 2
if w <= 0: w = 30
if h <= 0: h = 30

# Iterate over fields in the universe
iterator fields(a = (0,0), b = (h-1,w-1)) =
  for y in a[0]..b[0]:
    for x in a[1]..b[1]:
      yield (y,x)

# Create a sequence with an initializer
proc newSeqWith[T](len: int, init: T): seq[T] =
  result = newSeq[T] len
  for i in 0 .. <len:
    result[i] = init

# Initialize
var univ, univNew = newSeqWith(h, newSeq[bool] w)

for y,x in fields():
  if random(10) < 1: univ[y][x] = true

while true:
  # Show
  stdout.write "\e[H"
  for y,x in fields():
    stdout.write if univ[y][x]: "\e[07m  \e[m" else: "  "
    if x == 0: stdout.write "\e[E"
  stdout.flushFile

  # Evolve
  for y,x in fields():
    var n = 0
    for y1,x1 in fields((y-1,x-1), (y+1,x+1)):
      if univ[(y1+h) mod h][(x1 + w) mod w]:
        inc n

    if univ[y][x]: dec n
    univNew[y][x] = n == 3 or (n == 2 and univ[y][x])
  swap univ, univNew

  sleep 200
{% endhighlight %}

Nim is a rather new language, and it took me some time to choose it. What I love about the language:

- Python-like syntax
- Statically typed, compiled
- High performance (same ballpark as C/C++)
- Garbage Collector that can be controlled for soft real-time
- Produces executables without dependencies (compiles to C)
- Easily interfaces C libraries (compiles to C)
- Clean and powerful metaprogramming

On this blog I will report about my progress in the development of HookRace and my current Nim adventures.
