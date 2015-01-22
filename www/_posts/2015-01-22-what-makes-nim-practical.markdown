---
layout: post
title:  "What makes Nim practical?"
categories: nim
permalink: /blog/what-makes-nim-practical/
---

In [my last post](../what-is-special-about-nim/) I showed what makes the [Nim](http://nim-lang/) programming language special. Today, let's consider Nim from another angle: What makes Nim a practical programming language?

## Binary Distribution

Programs written in interpreted languages like Python are difficult to distribute. Either you require Python (in a specific version even) to be installed already, or you ship it with your program. This even causes some to [reconsider Python as a teaching language](http://prog21.dadgum.com/203.html).

How does Nim ship around this problem? For starters your program gets statically linked against the Nim runtime. That means you end up with a single binary that depends solely on the standard C library, which we can take for granted on any operating system we're interested in.

Let's write a small program and give this a try:

{% highlight nimrod %}
echo "Hello World"
{% endhighlight %}

If you want to follow along, [get the Nim compiler](http://nim-lang.org/download.html). Save this code as `hello.nim`. Let's compile it now so that we can distribute the `hello` binary:

    $ nim -d:release c hello
    CC: hello
    CC: stdlib_system
    [Linking]
    $ ./hello
    Hello World
    $ ls -lha hello
    -rwxr-xr-x 1 def def 57K Jan 22 09:37 hello*
    $ ldd hello
        linux-vdso.so.1 (0x00007fffd5973000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f0f92c6b000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f0f928c3000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f0f92e6f000)

Now our little program starts growing and we get interested in using a few Nim libraries. Do we have to add compiled versions of these libraries to our distributions now? Do not fret! Nim libraries are statically compiled into our binary as well. Let's get a library using Nim's package manager, [nimble](https://github.com/nim-lang/nimble):

    $ nimble install strfmt
    Installing strfmt-0.5.4
    strfmt installed successfully.

And use the library in our, admittedly not very useful, program:

{% highlight nimrod %}
import strfmt

echo "Hello {} number {:04.1f}".fmt("World", 6.0)
{% endhighlight %}

If we want to compile with Clang instead of GCC as the backend, that's easy as well and Clang is usually much faster to compile and yields a smaller binary:

    $ nim --cc:clang -d:release c hello
    $ ./hello
    Hello World number 06.0
    $ ls -lha hello
    -rwxr-xr-x 1 def def 54K Jan 22 10:06 hello*

The only dynamic libraries we have to worry about are the C libraries we use, but that's a story for another time.

## Source Distribution

We can't compile our program for all possible combinations of operating systems and CPU architectures of course. But we can be prepared for them and create a source distribution that can be compiled on the target systems with just a C compiler:

    $ tree
    .
    ├── hello.ini
    ├── hello.nim
    └── lib
        └── nimbase.h
    $ cat hello.ini
    [Project]
    Platforms: """
      windows: i386;amd64
      linux: i386;amd64;powerpc64;arm;sparc;mips;powerpc
      macosx: i386;amd64;powerpc64
      solaris: i386;amd64;sparc
      freebsd: i386;amd64
      netbsd: i386;amd64
      openbsd: i386;amd64
      haiku: i386;amd64
    """

    [Lib]
    Files: "lib/nimbase.h"

    [Windows]
    binPath: "bin"
    $ niminst csource hello.ini -d:release

Here's the resulting build script if you want to try it out on any of these platforms: [hello-csources.zip](/public/hello-csources.zip)

    $ unzip hello-csources.zip
    $ ./build.sh
    SUCCESS
    $ bin/hello
    Hello World

`niminst` can also be used to [build full installers](http://nim-lang.org/niminst.html) for your program. It's what the Nim compiler itself uses.

## Debug and Release Builds

Debug and release builds behave quite differently in Nim by default. Here's an overview:

| Option         | Debug build | Release build |
|----------------|-------------|---------------|
| objChecks      | On          | Off           |
| fieldChecks    | On          | Off           |
| rangeChecks    | On          | Off           |
| boundChecks    | On          | Off           |
| overflowChecks | On          | Off           |
| assertions     | On          | Off           |
| stackTrace     | On          | Off           |
| lineTrace      | On          | Off           |
| lineDir        | On          | Off           |
| deadCodeElim   | Off         | On            |

As you develop your program you will be thankful for the range, bound and overflow checks. As you release your program your users will be thankful for the additional speed of disabling those.

If this is not what you want, and you require runtime checks even in release builds, you can enable them project wide or for parts of your code.

For example here we have an expensive max function that takes most of the running time:

{% highlight nimrod %}
proc max[T](x: varargs[T]): T =
  if x.len == 0:
    return

  result = x[0]

  for i in 1 .. x.high:
    if result < x[i]:
      result = x[i]

var s = newSeq[int]()
for i in 1..1_000:
  s.add(i)

echo max(s)
{% endhighlight %}

We want our program to never access out of sequence bounds, so we want to compile with `nim -d:release --checks:on max`. As it's unreasonable to write this every time, we can create a `nim.cfg` file in our directory instead and enable runtime checks for every compilation of our program:

    checks: on

Now we run into the problem that the `max` function runs too slowly (it doesn't, but let's imagine). So we disable runtime checks just for this function:

{% highlight nimrod %}
{.push checks: off.}
proc max[T](x: varargs[T]): T =
  if x.len == 0:
    return

  result = x[0]

  for i in 1 .. x.high:
    if result < x[i]:
      result = x[i]
{.pop.}
{% endhighlight %}

Great, now we can control which part of our program has runtime checks and which doesn't. `checks` enables and disables all runtime checks at once, but there are more [fine-grained controls](http://nim-lang.org/manual.html#compilation-option-pragmas) as well.

## Nim instead of Python + C++

High performance languages like C++ may require some boilerplate. A higher level language can be used at compile time to automatically create the boilerplate. In [DDNet](http://ddnet.tw/) (which inherited them from [Teeworlds](http://teeworlds.com/)) there are [many examples](https://github.com/def-/ddnet/tree/DDRace64/datasrc) for this.

A really simple use case is to get the current git revision at compile time in Python and put it into the config.h with a `#define` so it can be referred to at runtime in the program. In Nim you need neither Python nor a C preprocessor and can instead do it all directly at compile time in Nim:

{% highlight nimrod %}
const gitHash = staticExec "git rev-parse HEAD"
echo "Revision: ", gitHash
{% endhighlight %}

## Use as a scripting language

With TinyCC as the backend Nim makes for a nice scripting language. All we need is a `nimscript` file and the TinyCC compiler:

    #!/bin/sh
    nim --cc:tcc --verbosity:0 -d:release -r c $*

To automatically update DDNet's map files on our upcoming HTTP map server I'm using this script (with this [crc32](https://github.com/def-/nim-unsorted/blob/master/crc32.nim) module):

{% highlight nimrod %}
#!/usr/bin/env nimscript

import os, crc32, strutils

const
  baseDir = "/home/teeworlds/servers"
  mapdlDir = "/var/www-maps"

for kind, path in walkDir baseDir/"maps":
  if kind != pcFile:
    continue

  let (dir, name, ext) = splitFile(path)

  if ext != ".map":
    continue

  let
    sum = crc32FromFile(path).int64.toHex(8).toLower
    newName = name & "_" & sum & ext
    newPath = mapdlDir / newName
    tmpPath = newPath & ".tmp"

  if existsFile newPath:
    continue

  copyFile path, tmpPath
  moveFile tmpPath, newPath
{% endhighlight %}

Of course you can always compile your program if you need the full speed of Clang or GCC. But for quick tests while developing, or scripts you just need occasionally, this is good enough.

## Debugging Nim

GDB works rather well with Nim. The only problem I've had so far is that the variables get unique identifiers in C, but using tab completion you can get along fine:

{% highlight nimrod %}
echo "Hello World"
var x = 10
echo "Value of x: ", x
{% endhighlight %}

    $ nim --lineDir:on --debuginfo c hello
    $ gdb ./hello
    (gdb) break hello.nim:3
    Breakpoint 1 at 0x41f886: file /home/def/hello.nim, line 3.
    (gdb) run
    Starting program: /home/def/hello
    Hello World
    Breakpoint 1, helloInit () at /home/def/hello.nim:3
    3       echo "Value of x: ", x
    (gdb) print x_89010
    $1 = 10
    (gdb) print x_89010 = 200
    $2 = 200
    (gdb) c
    Continuing.
    Value of x: 200

## Final words

That's Nim from a more practical angle. Hopefully you'll consider Nim for your next project, there are [many libraries](http://nim-lang.org/lib.html) available already. Also, the community always needs more helping hands!

For comments use [Reddit](TODO), [Hacker News](TODO) or get in touch with the [Nim community](http://nim-lang.org/community.html) directly. You can reach me personally at [dennis@felsin9.de](mailto:dennis@felsin9.de).
