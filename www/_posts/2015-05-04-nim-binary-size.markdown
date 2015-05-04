---
layout: post
title:  "Nim binary size from 160 KB to 150 Bytes"
categories: nim
permalink: /blog/nim-binary-size/
---

The size of binaries in the [Nim](http://nim-lang.org/) programming language seems [to be a](http://www.schipplock.software/2015/02/static-linking-with-nim.html) [popular](http://forum.nim-lang.org/t/679) [topic](http://forum.nim-lang.org/t/963) [recently](http://forum.nim-lang.org/t/1171). Nim's slogan is _expressive, efficient, elegant_, so let's examine the _efficient_ part in this post by exploring a few ways to reduce the size of a simple Nim `Hello World` binary on Linux. Along the way we will:

- Build a regular program into a 6 KB binary
- Disregard the C standard library and build a 952 byte binary
- Use a custom linker script and ELF header to build a 150 byte binary <br>([1 byte smaller than in Rust](http://mainisusuallyafunction.blogspot.de/2015/01/151-byte-static-linux-binary-in-rust.html))

The full source code of this post can be found in [its repository](https://github.com/def-/nim-binary-size). All measurements are done on Linux x86-64 with GCC 5.1 and Clang 3.6.0.

<!--more-->
## Using the C Standard Library


{% highlight nimrod %}
echo "Hello!"
{% endhighlight %}

By default Nim uses GCC as the backend C compiler on most platforms, and we dynamically link against glibc. We can try optimizing for speed and size, as well as strip unnecessary symbols after the compilation:

| Command (GCC backend) | Binary Size |
|---------|------------:|
| `nim c hello` | 160 KB |
| `nim -d:release c hello` | 61 KB |
| `nim -d:release --opt:size c hello` | 25 KB
| `nim -d:release --opt:size c hello && strip -s hello` | 19 KB

That's pretty nice and can be done with any Nim program to reduce binary size.

Now let's try to get rid of glibc, at least temporarily (we will come back to a more permanent solution later). Instead of glibc we're now statically linking against musl libc, which actually reduces our filesize slightly:

    $ nim -d:release --opt:size --passL:-static \
      --gcc.exe:/usr/local/musl/bin/musl-gcc \
      --gcc.linkerexe:/usr/local/musl/bin/musl-gcc hello
    $ strip -s hello
    18 KB

So that's a statically linked binary in 18 KB, which can be deployed without depending on any glibc version (or any other libraries)!

What about using Clang instead of GCC by setting `--cc:clang`:

| Command (Clang backend) | Binary Size |
|---------|------------:|
| `nim --cc:clang c hello` | 168 KB |
| `nim --cc:clang -d:release c hello` | 33 KB |
| `nim --cc:clang -d:release --opt:size c hello` | 29 KB
| `nim --cc:clang -d:release --opt:size c hello && strip -s hello` | 23 KB

The speed optimized binary is much smaller, but the one optimized for size isn't. The exact behaviour of Clang and GCC of course depends on their version, so expect to see (at least) slightly different numbers on your system.

Looks like the GCC backend is a better choice currently, so let's try stripping down the binary further with it:

As a first step we disable the garbage collector, not like we need it for this program anyway:

    $ nim --gc:none -d:release --opt:size c hello
    $ strip -s hello
    11 KB

Next we remove all the nice dynamic memory, error handling and other OS dependent goodies with `--os:standalone` (this implies `--gc:none`). We have to provide a `panicoverride.nim` that contains these two procs, about which we don't care anyway. Who needs error handling when you can have a 6 KB binary instead:

{% highlight nimrod %}
proc rawoutput(s: string) = discard
proc panic(s: string) = discard
{% endhighlight %}

    $ nim --os:standalone -d:release c hello
    $ strip -s hello
    6.1 KB

## Disregarding the C standard library

Now we have to start thinking more out of the box: If we want a program that really does nothing, not even print `Hello!`, we can just use an empty file. Now we don't rely on the C standard library anymore and can try to exclude it totally by using `-passL:-nostdlib` (passL simply passes that argument to GCC at the linking step):

    $ nim --os:standalone -d:release --passL:-nostdlib c hello
    CC: hello
    CC: stdlib_system
    [Linking]
    ld: warning: cannot find entry symbol _start; defaulting to 0000000000400160
    $ strip -s hello
    1.4 KB

Wow, that's small! Let's run our program, that does nothing, and enjoy:

    $ ./hello
    Segmentation fault (core dumped)

Ouch. That didn't go so well. Take another look at the linker output and notice what the problem is: We can't just expect our code to be run, the binary starts its execution at some random, wrong, position. Instead we have to take over the work of the C standard library now and supply our own `_start` function:

{% highlight nimrod %}
import syscall

proc main {.exportc: "_start".} =
  discard syscall(EXIT, 0)
{% endhighlight %}

We also have to explicitly exit the program, for which we use my [syscall library](https://github.com/def-/nim-syscall), that provides raw system calls into the Linux kernel in Nim. Let's wrap the syscalls we need and use them to write proper Nim code with them:

{% highlight nimrod %}
import syscall

const STDOUT = 1

proc write(fd: cint, buf: cstring, len: csize): clong
          {.inline, discardable.} =
  syscall(WRITE, fd, buf, len)

proc exit(n: clong): clong {.inline, discardable.} =
  syscall(EXIT, n)

proc main {.exportc: "_start".} =
  write STDOUT, "Hello!\n", 7
  exit 0
{% endhighlight %}

Now we can successfully compile like this:

    $ nim --os:standalone -d:release --passL:-nostdlib --noMain c hello
    $ strip -s hello
    1.5 KB
    $ ./hello
    Hello!

The final trick of this section is to tell GCC to optimize away unused functions. These functions are the initializations of Nim modules, like our `hello` module or the `system` module from the standard library, but they just end up empty here anyway. Maybe the Nim compiler could leave them out on its own, but usually you don't care about saving a few bytes and have something more important to do. Today we care, so first we tell GCC to put function and data items into separate sections (`-ffunction-sections` & `-fdata-sections`) at the compile step. At the linking step we tell Nim to tell GCC to pass `--gc-sections` to `ld`, our linker, which then removes sections that are not referenced anywhere:

    $ nim --os:standalone -d:release --passL:-nostdlib --noMain \
      --passC:-ffunction-sections --passC:-fdata-sections \
      --passL:-Wl,--gc-sections c hello
    $ strip -s hello
    952 B

Great! We're down from an initial binary size of 160 KB to just 952 bytes. Can we get even smaller? Yes, indeed, but not with the default tooling.

## Custom Linking to achieve 150 Bytes

This uses the exact method from the [151-byte static Linux binary in Rust](http://mainisusuallyafunction.blogspot.de/2015/01/151-byte-static-linux-binary-in-rust.html) blog post, except that Nim with GCC manage to get down 1 byte more. Meanwhile Clang needs 1 byte more than the Rust version.

We continue with the exact same program that we just got down to 952 bytes. But instead of letting Nim do all the work (with quite a lot of options by now) we simply create an object file from Nim (`--app:staticlib`), and go manually from there. A [custom linker script](https://github.com/def-/nim-binary-size/blob/master/script.ld) and a [custom ELF header](https://github.com/def-/nim-binary-size/blob/master/elf.s) do the main work. But the actual code that's executed is still fully provided by our Nim code:

    $ nim --app:staticlib --os:standalone -d:release --noMain \
      --passC:-ffunction-sections --passC:-fdata-sections \
      --passL:-Wl,--gc-sections c hello
    $ ld --gc-sections -e _start -T script.ld -o payload hello.o
    $ objcopy -j combined -O binary payload payload.bin
    $ ENTRY=$(nm -f posix payload | grep '^_start ' | awk '{print $3}')
    $ nasm -f bin -o hello -D entry=0x$ENTRY elf.s
    $ chmod +x hello
    $ wc -c < hello
    158
    $ ./hello
    Hello!

158 bytes! There's a final trick to get 8 bytes smaller. We put our string inside the padding in the [ELF header](https://github.com/def-/nim-binary-size/blob/master/elf.s) and access that memory manually:

{% highlight nimrod %}
proc main {.exportc: "_start".} =
  write STDOUT, cast[cstring](0x00400008), 7
  exit 0
{% endhighlight %}

    $ wc -c < hello
    150
    $ ./hello
    Hello!

150 bytes! That's the final size we're going to get. If you still don't have enough and want to write your binary manually you can use even more hacks to get down to 45 bytes as described in the excellent [Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux](http://www.muppetlabs.com/%7Ebreadbox/software/tiny/teensy.html).

## Summary

Nim is fine for writing small binaries. Now you also know how to write Nim without the C standard library. It might be fun to write your own runtime in Nim from scratch. You can check out the [repository](https://github.com/def-/nim-binary-size/) to get your own results:

    $ ./run.sh
    == Using the C Standard Library ==
    hello_unoptimized    163827
    hello_release         62131
    hello_optsize         25248
    hello_optsize_strip   18552
    hello_gcnone          10344
    hello_standalone       6208

    == Disregarding the C Standard Library ==
    hello2                 1776
    hello3                  952

    == Custom Linking ==
    hello3_custom           158
    hello4_custom           150

    $ objdump -rd nimcache/hello4.o
    ...
    0000000000000000 <_start>:
     0: b8 01 00 00 00          mov    $0x1,%eax
     5: ba 07 00 00 00          mov    $0x7,%edx
     a: be 08 00 40 00          mov    $0x400008,%esi
     f: 48 89 c7                mov    %rax,%rdi
    12: 0f 05                   syscall 
    14: 31 ff                   xor    %edi,%edi
    16: b8 3c 00 00 00          mov    $0x3c,%eax
    1b: 0f 05                   syscall 
    1d: c3                      retq 
    ...

Discussions on [Hacker News](https://news.ycombinator.com/item?id=9485526) and [Reddit](https://www.reddit.com/r/programming/comments/34tb7r/nim_binary_size_from_160_kb_to_150_bytes/).

_Addendum_: I did this for 32bit x86 now as well, which results in a 119 byte binary with GCC and 118 byte with Clang, more information in the [repository](https://github.com/def-/nim-binary-size#x86).

_Addendum 2_: With [a simple patch](https://github.com/Araq/Nim/pull/2657) to the Nim compiler the `{.noReturn.}` pragma now actually removes the final `retq` call that is useless after the `EXIT` syscall. So the final binary size now is 149 bytes with x86-64, 116 bytes with x86.
