---
layout: post
title:  "What is special about Nim?"
categories: nim
permalink: /blog/what-is-special-about-nim/
---

The [Nim programming language](http://nim-lang.org/) is exciting. While the [official tutorial](http://nim-lang.org/tut1.html) is great, it slowly introduces you to the language. Instead I want to quickly show what you can do with Nim that would be more difficult or impossible in other languages.

I've discovered Nim in my quest to find the right tools to write a game, HookRace, the successor of my current [DDNet](http://ddnet.tw) game/mod of Teeworlds. Since I'm busy with other projects for now, this blog is now officially about Nim instead, until I find time to continue developing the game.

## Easy to get running

Ok, this part is not exciting yet, but I invite you to follow along with the post:

{% highlight nimrod %}
for i in 0..10:
  echo "Hello World"[0..i]
{% endhighlight %}

If you want to do so, [get the Nim compiler](http://nim-lang.org/download.html). Save this code as `hello.nim`, compile it with `nim c hello` and finally run the binary with `./hello`. To immediately compile and run, use `nim -r c hello`. To use an optimized release build instead of a debug build use `nim -d:release c hello`.

## Run regular code at compile time

To implement an efficient CRC32 procedure you need a lookup table. You could compute it at runtime or write it into your code as a magic array. Clearly we don't want any magic numbers in our code, so we'll do it at runtime (for now):

{% highlight nimrod %}
type CRC32* = uint32
const initCRC32* = CRC32(-1)

proc createCRCTable(): array[256, CRC32] =
  for i in 0..255:
    var rem = CRC32(i)
    for j in 0..7:
      if (rem and 1) > 0: rem = (rem shr 1) xor CRC32(0xedb88320)
      else: rem = rem shr 1
    result[i] = rem

# Table created at runtime
var crc32table = createCRCTable()

proc crc32(s): CRC32 =
  result = initCRC32
  for c in s:
    result = (result shr 8) xor crc32table[(result and 0xff) xor ord(c)]
  result = not result

proc `$`(c: CRC32): string = int64(c).toHex(8)

echo crc32("The quick brown fox jumps over the lazy dog")
{% endhighlight %}
Great, this works and we get `414FA339` as the output. But it would be even better if we could compute the CRC table at compile time. This is as easy as it gets in Nim, instead of the current crc32table creation we use:

{% highlight nimrod %}
# Table created at compile time
const crc32table = createCRCTable()
{% endhighlight %}

Yes, that's right: All we had to do was replace the `var` with a `const`. Beautiful, isn't it? We can write the exact same code and have it run at runtime and compile time. No template metaprogramming necessary.

## Extend the language
[Templates](http://nim-lang.org/manual.html#templates) and [macros](http://nim-lang.org/manual.html#macros) can be used to get rid of boilerplate, by transforming your code at compile time.

Let's start with a simple logger, not using templates so far:

{% highlight nimrod %}
import os
const debug = false

proc log*(msg: string) =
  if debug:
    echo msg

proc expensive(): string =
  sleep(milsecs = 1000)
  "That was difficult"

10.times:
  log expensive()
{% endhighlight %}

What's the problem? At the invocation of log the `expensive` procedure is still executed, even though `debug` is set to `false`. To prevent this we can use a log template instead:

{% highlight nimrod %}
template log*(msg: string) =
  if debug:
    echo msg
{% endhighlight %}

Yes, again we had to change just one word to convert a `proc` to a `template`. Templates just replace their invocations with their code, similar to C macros.

If you're still going along with compiling this post's examples, you will have noticed that `times` is not defined. That's not a problem though, since we know templates now, we can define it ourselves:

{% highlight nimrod %}
template times(x: expr, y: stmt): stmt =
  for i in 1..x:
    y
{% endhighlight %}

Macros go a step further and allow you to analyze and manipulate the AST. There are no list comprehensions in Nim, for example, but it's possible to [add them to the language by using a macro](https://github.com/def-/nimrod-unsorted/blob/master/listcomprehensions.nim). Now instead of this:

{% highlight nimrod %}
var result: seq[tuple[a,b,c: int]] = @[]
for x in 1..n:
  for y in x..n:
    for z in y..n:
      if x*x + y*y == z*z:
        result.add((x,y,z))
echo result
{% endhighlight %}

You can write:

{% highlight nimrod %}
echo lc[(x,y,z) | (x <- 1..n, y <- x..n, z <- y..n,
                   x*x + y*y == z*z), tuple[a,b,c: int]]
{% endhighlight %}

## Add your own optimizations to the compiler
Instead of optimizing your code, wouldn't you prefer to make the compiler smarter? In Nim you can!

{% highlight nimrod %}
var x: int
for i in 1..1_000_000_000:
  x += 2 * i
echo x
{% endhighlight %}

This (pretty useless) code can be sped up by teaching the compiler two optimizations:

{% highlight nimrod %}
template optMul{‘*‘(a,2)}(a: int): int =
  let x = a
  x + x

template canonMul{‘*‘(a,b)}(a: int{lit}, b: int): int =
  b * a
{% endhighlight %}

In the first [term rewriting](http://nim-lang.org/trmacros.html) template we specify that `a * 2` can be replaced by `a + a`. In the second one we specify that `int`s in a multiplication can be swapped if the first is an integer literal, so that we can potentially apply the first template.

More complicated patterns can also be implemented, for example to optimize boolean logic:

{% highlight nimrod %}
template optLog1{a and a}(a): auto = a
template optLog2{a and (b or (not b))}(a,b): auto = a
template optLog3{a and not a}(a: int): auto = 0

var
  x = 12
  s = x and x
  # Hint: optLog1(x) --> ’x’ [Pattern]

  r = (x and x) and ((s or s) or (not (s or s)))
  # Hint: optLog2(x and x, s or s) --> ’x and x’ [Pattern]
  # Hint: optLog1(x) --> ’x’ [Pattern]

  q = (s and not x) and not (s and not x)
  # Hint: optLog3(s and not x) --> ’0’ [Pattern]
{% endhighlight %}

`s` gets optimized to `x` directly, `r` gets optimized to `x` in 2 successive pattern applications and `q` ends up as `0` immediately.

## Bind to your favorite C functions and libraries
Since Nim compiles down to C, [foreign function interfaces](http://nim-lang.org/manual.html#foreign-function-interface) are fun.

You can easily use your favorite functions from the C standard library:

{% highlight nimrod %}
proc printf(formatstr: cstring)
  {.header: "<stdio.h>", varargs.}
printf("%s %d\n", "foo", 5)
{% endhighlight %}

Or use your own code written in C:

{% highlight c %}
void hi(char* name) {
  printf("awesome %s\n", name);
}
{% endhighlight %}

{% highlight nimrod %}
{.compile: "hi.c".}
proc hi*(name: cstring) {.importc.}
hi "from Nim"
{% endhighlight %}

Or any library you please with the help of [c2nim](https://github.com/nim-lang/c2nim):

{% highlight nimrod %}
proc set_default_dpi*(dpi: cdouble) {.cdecl,
  importc: "rsvg_set_default_dpi",
  dynlib: "librsvg-2.so".}
{% endhighlight %}

<!--- ## Compiler analysis
Coming from Haskell
.noSideEffect, .tags, .raises --->

## Control the Garbage Collector
To achieve soft realtime, you can tell the [garbage collector](http://nim-lang.org/gc.html) when and for how long it is allowed to run. The main game logic can be implemented like this in Nim, to prevent the garbage collector from causing stutters:

{% highlight nimrod %}
gcDisable()
while true:
  gameLogic()
  renderFrame()
  gcStep(us = leftTime)
  sleep(restTime)
{% endhighlight %}

## Unified Call Syntax

This is just syntactic sugar, but it's definitely nice to have. In Python I always forget whether `len` and `append` are functions or methods. In Nim I don't have to remember, because there is no difference. Nim uses [Unified Call Syntax](http://nim-lang.org/manual.html#method-call-syntax), which has now also been proposed for C++ by [Herb Sutter](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4165.pdf) and [Bjarne Stroustrup](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4174.pdf).

{% highlight nimrod %}
var xs = @[1,2,3]

# Procedure call syntax
add(xs, 4_000_000)
echo len(xs)

# Method call syntax
xs.add(0b0101_0000_0000)
echo xs.len()

# Command invocation syntax
xs.add 0x06_FF_FF_FF
echo xs.len
{% endhighlight %}

<!---
## Breaking out of nested loops

I've been missing this in Python as well:

{% highlight nimrod %}
block loops:
  for x in 0..9:
    for y in 0..9:
      echo x * y
      if x * y > 50:
        break loops
{% endhighlight %}

## Index arrays with any enumerable type

## Use your favorite naming convention
--->

## Good performance
It's easy to write fast code in Nim, as can be seen in the [Longest Path Finding Benchmark](https://github.com/logicchains/LPATHBench/blob/master/writeup.md), in which Nim is competing with [some cute code](https://github.com/logicchains/LPATHBench/blob/master/nim.nim).

I made some measurements on my machine (Linux x86-64, Intel Core2Quad Q9300 @2.5GHz, state of 2014-12-20):

| Lang   |Time [ms] |Memory [KB] |Compile Time [ms] |Compressed Code [B] |
|--------|---------:|-----------:|-----------------:|-------------------:|
| Nim    |     1400 |       1460 |              893 |                486 |
| C++    |     1478 |       2717 |              774 |                728 |
| D      |     1518 |       2388 |             1614 |                669 |
| Rust   |     1623 |       2632 |             6735 |                934 |
| Java   |     1874 |      24428 |              812 |                778 |
| OCaml  |     2384 |       4496 |              125 |                782 |
| Go     |     3116 |       1664 |              596 |                618 |
| Haskell|     3329 |       5268 |             3002 |               1091 |
| LuaJit |     3857 |       2368 |                - |                519 |
| Lisp   |     8219 |      15876 |             1043 |               1007 |
| Racket |     8503 |     130284 |            24793 |                741 |

Compressed code size with `gzip -9 < nim.nim | wc -c`. Removed unused functions in Haskell. Compile times are clean compiles, if you have a nimcache with the standard library precompiled it's only 323 ms for Nim.

Another small benchmark I did, calculating which numbers of the first 100 million are prime, in Python, Nim and C:

### Python (runtime: 35.1 s)
{% highlight python %}
def eratosthenes(n):
  sieve = [1] * 2 + [0] * (n - 1)
  for i in range(int(n**0.5)):
    if not sieve[i]:
      for j in range(i*i, n+1, i):
        sieve[j] = 1
  return sieve

eratosthenes(100000000)
{% endhighlight %}

### Nim (runtime: 2.6 s)
{% highlight nimrod %}
import math

proc eratosthenes(n): auto =
  result = newSeq[int8](n+1)
  result[0] = 1; result[1] = 1

  for i in 0 .. int sqrt(float n):
    if result[i] == 0:
      for j in countup(i*i, n, i):
        result[j] = 1

discard eratosthenes(100_000_000)
{% endhighlight %}

### C (runtime: 2.6 s)
{% highlight c %}
#include <stdlib.h>
#include <math.h>
char* eratosthenes(int n)
{
  char* sieve = calloc(n+1,sizeof(char));
  sieve[0] = 1; sieve[1] = 1;
  int m = (int) sqrt((double) n);

  for(int i = 0; i <= m; i++) {
    if(!sieve[i]) {
      for (int j = i*i; j <= n; j += i)
        sieve[j] = 1;
    }
  }
  return sieve;
}

int main() {
  eratosthenes(100000000);
}
{% endhighlight %}

## Compile to JavaScript

You can [compile Nim to JavaScript](http://nim-lang.org/backends.html#the-javascript-target), instead of C, as well. This allows you to write the client as well as the server directly in Nim. Let's make a small visitor counter on the server that gets displayed in the browser. This is our `client.nim`:

{% highlight nimrod %}
import htmlgen, dom

type Data = object
  visitors {.importc.}: int
  uniques {.importc.}: int
  ip {.importc.}: cstring

proc printInfo(data: Data) {.exportc.} =
  var fooDiv = document.getElementById("foo")
  fooDiv.innerHTML = p("You're visitor number ", $data.visitors,
    ", unique visitor number ", $data.uniques,
    " today. Your IP is ", $data.ip, ".")
{% endhighlight %}

We define a `Data` type that we use to pass data from the server to client. The `printInfo` procedure will be called with this `data` and display it. Compile this with `nim js client`. The result javascript file ends up in `nimcache/client.js`.

For the server we need to [get nimble](https://github.com/nim-lang/nimble) and run `nimble install jester`. Now we can use the [Jester web framework](https://github.com/dom96/jester) and write our `server.nim`:

{% highlight nimrod %}
import jester, asyncdispatch, json, strutils, times, sets, htmlgen

var
  visitors = 0
  uniques = initSet[string]()
  time: TimeInfo

routes:
  get "/":
    resp body(
      `div`(id="info"),
      script(src="/client.js", `type`="text/javascript"),
      script(src="/visitors", `type`="text/javascript"))

  get "/client.js":
    const execResult = staticExec "nim js client"
    const clientJS = staticRead "nimcache/client.js"
    resp clientJS

  get "/visitors":
    let newTime = getTime().getLocalTime
    if newTime.monthDay != time.monthDay:
      visitors = 0
      init uniques
      time = newTime

    inc visitors
    uniques.incl request.ip

    let json = %{"visitors": %visitors,
                 "uniques": %uniques.len,
                 "ip": %request.ip}
    resp "printInfo($#)".format(json)

runForever()
{% endhighlight %}

The server delivers the main website. It also delivers the `client.js`, by compiling and reading the `client.nim` at compile time. The logic is in the `/visitors` handling. Compile and run with `nim -r c server` and open [http://localhost:5000/](http://localhost:5000/) to see the code in effect.

You can see our code in action on the [Jester-generated site](http://visitors.hookrace.net/) or inline here:

<div id="info"></div>
<script src="http://visitors.hookrace.net/client.js" type="text/javascript"></script>
<script src="http://visitors.hookrace.net/visitors" type="text/javascript"></script>

## Conclusion

I hope I could pick your interest in in the Nim programming language.

For comments use [Reddit](http://reddit.com/r/programming), [Hacker News](//news.ycombinator.com/news) or just write me a mail at [dennis@felsin9.de](mailto:dennis@felsin9.de).
