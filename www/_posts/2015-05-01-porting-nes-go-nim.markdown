---
layout: post
title:  "Porting a NES emulator from Go to Nim"
categories: nim,nes
permalink: /blog/porting-nes-go-nim/
---

> Let me get this straight. We have an emulator for 1985 hardware that was written in a pretty new language (Go), ported to a language that isn't even 1.0 (Nim), compiled to C, then compiled to JavaScript? And the damn thing actually works? That's kind of amazing.
>
> &mdash; <cite>Summary by [haberman](https://news.ycombinator.com/item?id=9474030)</cite>

<!--more-->
I spent the last weeks working on [NimES](https://github.com/def-/nimes), a NES emulator in the [Nim](http://nim-lang.org/) programming language. As I really liked [fogleman's NES emulator in Go](https://github.com/fogleman/nes) I ended up mostly porting it to Nim. The source code is so clean that it's often easier to understand the internals of the NES by reading the source code than by reading documentation about it.

The choice of backend fell on SDL2 for me, contrary to GLFW + PortAudio that the Go version used. This was mainly motivated by the great portability promised by SDL2. Later we will see how porting to JavaScript and Android worked. If you're impatient and want to play a game, there's a [JS demo](/nimes/).

## Comparison of Go and Nim

Most Go concepts are quite trivial to translate to Nim. This made the porting process simple.

Let's compare some data that I found interesting:

| Metric             | Go                | Nim (clang backend) |
|--------------------|------------------:|--------------------:|
| Compile command    | `go build`        | `nimble build`      |
| Fresh compile time |      2.1 s¹       |          1.7 s      |
| Recompile time     |       1.5 s       |          0.5 s      |
| Binary size        | 16 MB (static)    | 136 KB + 1MB SDL2   |
| Source code size²  | 3260 lines, 74 KB | 2145 lines, 60 KB   |
| CPU usage          |        71 %       |           53 %      |
| Memory usage       |       79 MB       |          73 MB      |

¹ Excluding go-glfw, go-gl and portaudio, which take 17 s to compile<br>
² Emulation code only

It's nice to see Nim doing well. Even the compile time is shorter than that of Go, which is well known for its short compile times. Now that the port seems to be doing fine and should be running on all Desktop platforms, let's look into some other interesting things we can do with Nim:

## JavaScript port via emscripten

Nim has a JavaScript backend, but I don't trust it to be stable enough for this task yet. So I opted for emscripten instead, which can compile C code into JavaScript. Since Nim outputs C code, this sounds like a perfect fit. Luckily eeeee helped me with getting it started, since he had experience by porting my [DDNet](http://ddnet.tw/) client to [teewebs.net](http://teewebs.net/).

It turned out that [emsdk](https://kripken.github.io/emscripten-site/docs/getting_started/downloads.html) is the easiest way to use emscripten:

    $ ./emsdk update
    $ ./emsdk install latest
    $ ./emsdk activate latest
    $ source ./emsdk_env.sh

This may take a while, get a cup of tea. Afterwards we should have the `emconfigure`, `emmake` and `emcc` commands available. We can build regular Nim programs and look at the [resulting html file](/nimes/hello.html):

    $ cat hello.nim
    echo "Hello World"
    $ nim --cc:clang --clang.exe:emcc --clang.linkerexe:emcc \
      --cpu:i386 -d:release -o:hello.html c hello.nim
    $ ls -lha hello.{html,js}
    -rw-r--r-- 1 def users 101K Mai  1 19:02 hello.html
    -rw-r--r-- 1 def users 385K Mai  1 19:02 hello.js

That's a pretty cumbersome building command, so we'll slim it down later. The next step is to [build SDL2 for emscripten](https://hg.libsdl.org/SDL/file/e0e2e94ce5ea/docs/README-emscripten.md):

    $ hg clone https://hg.libsdl.org/SDL
    $ cd SDL
    $ emconfigure ./configure --host=asmjs-unknown-emscripten \
      --disable-assembly --disable-threads \
      --enable-cpuinfo=false CFLAGS="-O2"
    $ emmake make
    $ ls -lha build/.libs/libSDL2.a
    -rw-r--r-- 1 def users 1.6M Apr 29 06:58 build/.libs/libSDL2.a

I put the resulting `libSDL2.a` into the NimES repository under `emscripten/` for convenience.

Instead of increasing the cumbersomeness of our build command anymore, NimES's [nim.cfg](https://github.com/def-/nimes/blob/master/src/nim.cfg) specifies how to compile when `-d:emscripten` is set:

    @if emscripten:
      define = SDL_Static
      gc = none
      cc = clang
      clang.exe = "emcc"
      clang.linkerexe = "emcc"
      clang.options.linker = ""
      cpu = "i386"
      out = "nimes.html"
      warning[GcMem] = off
      passC = "-Wno-warn-absolute-paths -Iemscripten -s USE_SDL=2"
      passL = "-O3 -Lemscripten -s USE_SDL=2 --preload-file tetris.nes --preload-file pacman.nes --preload-file smb.nes --preload-file smb3.nes -s TOTAL_MEMORY=16777216"
    @end

Now a simple `nim -d:release -d:emscripten` builds the JavaScript port. Note that I'm preloading a few ROMs so that they can be loaded. The HTML then uses the `?nes=` parameter to pass the command line argument:

{% highlight js %}
var argument;
if (QueryString.hasOwnProperty("nes")) {
  argument = QueryString.nes;
} else {
  argument = "smb3.nes";
}

var Module;
Module = {
  preRun: [],
  postRun: [],
  arguments: [argument],
  canvas: (function() {
    var canvas = document.getElementById('canvas');
    canvas.addEventListener("webglcontextlost", function(e) { alert('WebGL context lost. You will need to reload the page.'); e.preventDefault(); }, false);
    return canvas;
  })(),
  totalDependencies: 0
};
{% endhighlight %}

Inside the Nim source code there are some interesting changes too. I quickly wrapped these functions as there is no emscripten wrapper for Nim yet:

{% highlight nimrod %}
when defined(emscripten):
  proc emscripten_set_main_loop(fun: proc() {.cdecl.}, fps,
    simulate_infinite_loop: cint) {.header: "<emscripten.h>".}

  proc emscripten_cancel_main_loop() {.header: "<emscripten.h>".}
{% endhighlight %}

Emscripten requires a slightly different execution style. Instead of actually looping, we define the main loop like this:

{% highlight nimrod %}
when defined(emscripten):
  emscripten_set_main_loop(loop, 0, 1)
else:
  while runGame:
    loop()
{% endhighlight %}

That's the main idea and with this we get a pretty playable [web version of NimES](/nimes/). I'm still getting 60fps in it, but just barely on my machine. Chrome seems to do a bit better than Firefox.

## Android port

Obviously the next step is to port NimES to Android as well. But since the original emulator is more accurate and nice than performant, we shouldn't expect runnable speed. Think of this more as a proof of concept:

We need a fresh clone of the SDL2 repository for this as well as the Android SDK (12 or later) and NDK (7 or later) installed. SDL2 has [building instructions for Android](https://wiki.libsdl.org/Android) as well:

    $ hg clone https://hg.libsdl.org/SDL
    $ cd SDL/build-scripts
    $ ./androidbuild.sh org.nimes /dev/null
    $ ls ../build/org.nimes
    gen/  src/                 build.properties    local.properties
    jni/  AndroidManifest.xml  build.xml           proguard-project.txt
    res/  ant.properties       default.properties  project.properties

That's our Android build directory now. I put this into the repository as well, under `android/`. Now we can add some ROM to the `assets/` directory and tell Nim to put the resulting C files into the correct directory and not to build them into binaries at all:

    @if android:
      cpu = "i386"
      nimcache = "./android/jni/src"
      compileOnly
      noMain
    @end

You may have noticed that I also defined `noMain`. Instead we define our own main function, as SDL is a bit weird with mains. Thanks to [yglukhov](https://github.com/yglukhov) for this little trick:


{% highlight nimrod %}
when defined(android):
  {.emit: """
  #include <SDL_main.h>

  extern int cmdCount;
  extern char** cmdLine;
  extern char** gEnv;

  N_CDECL(void, NimMain)(void);

  int main(int argc, char** args) {
      cmdLine = args;
      cmdCount = argc;
      gEnv = NULL;
      NimMain();
      return nim_program_result;
  }

  """.}
{% endhighlight %}

Another trick is how to access the assets we embed into our APK. Luckily SDL2 provides functions for that, which we can use as replacements for the regular file operations:

{% highlight nimrod %}
from sdl2 import rwFromFile, read, freeRW

proc newCartridge*(path: string): Cartridge =
  var file = rwFromFile(path.cstring, "r")
  defer: freeRW file

  var header: iNESHeader
  # Read directly into the header object
  if read(file, addr header, 1, sizeof header) != sizeof header:
    raise newException(ValueError, "header can't be read")
  ...
{% endhighlight %}

Finally we can build the project:

    $ nim -d:release -d:android c src/nimes
    $ cd android
    $ ndk-build
    $ ant debug

And the end result is a nice [nimes.apk](/nimes/nimes.apk). Of course it only shows some low FPS video for now and doesn't even have any controls, but it's a start.

## Conclusion

In the end I'm quite happy with the result: A truly portable emulator written in my favorite language. It compiles to C, C++ as well as JavaScript and runs on any Desktop platform as well as JavaScript and Android. The process for this was much easier than expected, mostly thanks to Nim and SDL2. I see a bright future for Nim as a practical language.

If you have any comments, suggestions or questions, feel free to ask them on [Hacker News](https://news.ycombinator.com/item?id=9473653) or [Reddit](https://www.reddit.com/r/programming/comments/34jnv1/porting_a_nes_emulator_from_go_to_nim/).
