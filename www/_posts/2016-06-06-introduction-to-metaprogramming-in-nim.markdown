---
layout: post
title:  "Introduction to Metaprogramming in Nim"
tags: Nim
permalink: /blog/introduction-to-metaprogramming-in-nim/
---

## Introduction to the Introduction (Meta-Introduction)

[Wikipedia](https://en.wikipedia.org/wiki/Metaprogramming) gives us a nice
description of metaprogramming:

> Metaprogramming is the writing of computer programs with the ability to
> treat programs as their data. It means that a program could be designed to
> read, generate, analyse and/or transform other programs, and even modify
> itself while running.

In this article we will explore [Nim](http://nim-lang.org/)'s metaprogramming
capabilities, which are quite powerful and yet still easy to use. After all
great metaprogramming is one of Nim's main features. The general rule is to use
the least powerful construct that is still powerful enough to solve a problem,
in this order:

<!--more-->
1. [Normal procs](#normal-procs) and [inline iterators](#inline-iterators)
2. [Generic procs](#generic-procs) and [closure iterators](#closure-iterators)
3. [Templates](#templates)
4. [Macros](#macros)

So before looking at Nim's two main metaprogramming constructs, templates and
macros, we'll look at what we can do with procs and iterators as well.

## Regular Programming Constructs
### Normal procs

We're in normal programming land here. Regular procedures are what you know as
*functions* elsewhere and they're pretty easy to define and use:

{% highlight nimrod %}
proc sayHi(name: string) =
  echo "Hello ", name

sayHi("World")
sayHi "World"
"World".sayHi
{% endhighlight %}

### Generic procs

With generics we can define procs that work on multiple types. Actually a new
proc will be generated based on our generic definition for each instantiation:

{% highlight nimrod %}
proc min[T](x, y: T): T =
  if x < y: x else: y

echo min(2, 3) # more explicitly: min[int](2, 3)
echo min("foo", "bar") # min[string]("foo", "bar")
{% endhighlight %}

### Inline iterators

Inline iterators are the default iterators in Nim. They get compiled into high
performance loops:

{% highlight nimrod %}
iterator reverseItems(x: string): char =
  for i in countdown(x.high, x.low):
    yield x[i]

for c in "foo".reverseItems:
  echo c
{% endhighlight %}

So this code gets compiled into:

{% highlight nimrod %}
let x = "foo"
for c in countdown(x.high, x.low):
  let c = x[i]
  echo c
{% endhighlight %}

Of course we can make iterators generic too:

{% highlight nimrod %}
iterator reverseItems[T](x: T): auto =
  for i in countdown(x.high, x.low):
    yield x[i]
{% endhighlight %}

### Closure iterators

Inline iterators simultaneously have the advantage and disadvantage of being
translated into loops. This means you can not pass them around. This limitation
can be lifted by using closure iterators instead:

{% highlight nimrod %}
import math

proc powers(m: int): auto =
  #return iterator: int {.closure.} = # Make a closure explicitly
  return iterator: int = # Compiler makes this a closure for us
    for n in 0..int.high:
      yield n^m

var
  squares = powers(2)
  cubes = powers(3)

for i in 1..4:
  echo "Square: ", squares() # 0, 1, 4, 9
for i in 1..4:
  echo "Cube: ", cubes()     # 0, 1, 8, 27
echo "Square: ", squares()   # 16
echo "Cube: ", cubes()       # 64

for x in squares(): # Go through all the remaining squares
  echo "Square: ", x         # 25, 36, 49, 64, ...
{% endhighlight %}

As you can see closure iterators keep their state. You can call them again and
get the next value, or use them inside of a for-loop to get out many values.

## Templates

You can think of templates as Nim's equivalent to the C preprocessor. But
templates are written in Nim itself and fit well into the rest of the language.

Templates simply insert their code at the invocation site, working at the level
of the abstract syntax tree. They can be used in just the same way as procs.

### Logger

A common example are loggers, which we looked at in [another
article](/blog/writing-an-async-logger-in-nim/) already. Consider that you want
to have extensive debug logging in your program. A trivial implementation would
look like this:

{% highlight nimrod %}
import strutils, times, os

type Level* {.pure.} = enum
  debug, info, warn, error, fatal

var logLevel* = Level.debug

proc debug*(args: varargs[string, `$`]) =
  if logLevel <= Level.debug:
    echo "[$# $#]: $#" % [getDateStr(), getClockStr(),
      join args]

proc expensiveDebuggingInfo*: string =
  sleep(milsecs = 1000)
  result = "Everything looking good!"

debug expensiveDebuggingInfo()
{% endhighlight %}

    [2016-06-05 22:00:50]: Everything looking good!

We have to call `expensiveDebuggingInfo` to get the debugging info, which is
fine right now since our `logLevel` is set to `Level.debug`. But it stops being
fine when we instead set `logLevel` to anything higher than `debug`. Then it
still takes a full second to evaluate the `expensiveDebuggingInfo` parameter
for `debug`, but inside of `debug` nothing is done with that information. This
is of course a consequence of call-by-value argument evaluation, which Nim
uses, just as most other languages do. A notable exception would be lazy
evaluation in Haskell, where this kind of logger would work perfectly fine,
only calling `expensiveDebuggingInfo` when its value is actually needed.

But let's stay in Nim-land and use a template instead of a proc to magically
fix this:

{% highlight nimrod %}
template debug*(args: varargs[string, `$`]) =
  if logLevel <= Level.debug:
    const module = instantiationInfo().filename[0 .. ^5]
    echo "[$# $#][$#]: $#" % [getDateStr(), getClockStr(),
      module, join args]
{% endhighlight %}

    [2016-06-05 22:01:30][logger]: Everything looking good!

Note that we also conveniently use `instantiationInfo()` to find out at what
location in the program our template was instantiated, something we could not
do using a procedure.

We can still call the template in the exact same way as the proc. But now we
have the advantage that the template is inlined at compiletime, so
`expensiveDebuggingInfo` is only called if the runtime `logLevel` actually
requires it. Perfect.

### Safe locking

Another problem that can be solved with a template is automatically acquiring and releasing a system [lock](http://nim-lang.org/docs/locks.html):

{% highlight nimrod %}
import locks

template withLock(lock: Lock, body: stmt) =
  acquire lock
  try:
    body
  finally:
    release lock
{% endhighlight %}

Compile with `--threads:on` for platform independent lock support.

This looks pretty simple, we just acquire the lock, execute the passed
statements and finally release the lock, even if exceptions have been thrown.
We can pass any set of statements as the `body`. The usage is as easy as using
a built-in if statement:

{% highlight nimrod %}
var lock: Lock
initLock lock

withLock lock:
  echo "Do something that requires locking"
  echo "This might throw an exception"
{% endhighlight %}

When our template accepts a value of type `stmt` we can use the colon to pass an entire indented block of code. When we have multiple parameters of type `stmt` the [do notation](http://nim-lang.org/docs/manual.html#procedures-do-notation) can be used.

This gets transformed into:

{% highlight nimrod %}
var lock: Lock
initLock lock

acquire lock
try:
  echo "Do something that requires locking"
  echo "This might throw an exception"
finally:
  release lock
{% endhighlight %}

Now we will never forget to call `release lock`. You could use this to make a
higher level locking library that only exposes `withLock` instead of the
lower-level `acquire` and `release` primitives.

## Macros
Just like templates, macros are executed at compiletime. But with templates you
can only do constant substitutions in the AST. With macros you can analyze the
passed arguments and create a new AST at the current position in any way you
want. A nice property of Nim is that these compiletime macros are also written
in the regular Nim language, so there is no need to learn another language.

A simple way to create an AST is to use `parseStmt` and `parseExpr` to parse
the regular textual representation into a NimNode. For example
`parseStmt("result = 10")` returns this AST:

    StmtList
      Asgn
        Ident !"result"
        IntLit 10

A very useful way to find the AST of a piece of code is `dumpTree`:

{% highlight nimrod %}
import macros

dumpTree:
  result = 10
{% endhighlight %}

This is the same output as you get with `treeRepr`:

{% highlight nimrod %}
import macros

static:
  echo treeRepr(parseStmt("result = 10"))
{% endhighlight %}

Alternatively you can use `lispRepr` to get a lisp-like representation:

    StmtList(Asgn(Ident(!"result"), IntLit(10)))

Finally there is also the `repr` proc, which turns a NimNode AST back into its
textual representation.

Many beginners start by piecing strings together and finally calling
`parseStmt` on them. While this works it is inefficient and prone to bugs.
Instead you can use the [macros module](http://nim-lang.org/docs/macros.html)
to create NimNodes of all kinds yourself. `dumpTree` gives you a hint if you're
not sure how a specific piece of code will look in its AST representation.

### JSON Parsing

JSON is pretty popular, so let's improve the support for it in Nim. What we
want is to have a magical `%*` so that we can write JSON directly in Nim source
code and have it checked at compile time, like this:

{% highlight nimrod %}
var j1 = %*
  [
    {
      "name": "John",
      "age": 30
    },
    {
      "name": "Susan",
      "age": 31
    }
  ]
{% endhighlight %}

So far if you want to use JSON in Nim, you have to use the JSON constructor `%`
a lot:

{% highlight nimrod %}
import json

var j2 =
  %[
    %{
      "name": %"John",
      "age": %30
    },
    %{
      "name": %"Susan",
      "age": %31
    }
  ]
{% endhighlight %}

Looks annoying. How can we implement `%*`? As a macro of course!:

{% highlight nimrod %}
macro `%*`*(x: expr): expr = toJson(x)
{% endhighlight %}

Ok, that doesn't do anything interesting yet. We just call the still
unspecified compile time proc `toJson` and return the result. We want `toJson`
to traverse the passed AST `x` and create a new AST, which inserts a `%` call
at just the right places, exactly as it would happen if we added the `%` calls
manually.

For this purpose we print the AST of `j2` by putting it into `dumpTree` from
the [macros module](http://nim-lang.org/docs/macros.html):

{% highlight nimrod %}
import json, macros

dumpTree:
  %[
    %{
      "name": %"John",
      "age": %30
    },
    %{
      "name": %"Susan",
      "age": %31
    }
  ]
{% endhighlight %}

We get the following AST printed when compiling this program:

    Prefix
      Ident !"%"
      Bracket
        Prefix
          Ident !"%"
          TableConstr
            ExprColonExpr
              StrLit name
              Prefix
                Ident !"%"
                StrLit John
            ExprColonExpr
              StrLit age
              Prefix
                Ident !"%"
                IntLit 30
        Prefix
          Ident !"%"
          TableConstr
            ExprColonExpr
              StrLit name
              Prefix
                Ident !"%"
                StrLit Susan
            ExprColonExpr
              StrLit age
              Prefix
                Ident !"%"
                IntLit 31

This turned out quite big, but from here we can see how the AST we want to
construct looks like. We do the same for `j1` to see what we're working with:

    StmtList
      Bracket
        TableConstr
          ExprColonExpr
            StrLit name
            StrLit John
          ExprColonExpr
            StrLit age
            IntLit 30
        TableConstr
          ExprColonExpr
            StrLit name
            StrLit Susan
          ExprColonExpr
            StrLit age
            IntLit 31

The idea now is to insert a `%` at each level, except in front of the `"name"`
and `"age"` in our case, the first elements in colon expressions.

{% highlight nimrod %}
proc toJson(x: PNimrodNode): PNimrodNode {.compiletime.} =
  case x.kind
  of nnkBracket: # Corresponds to Bracket in dumpTree
    result = newNimNode(nnkBracket)
    for i in 0 .. <x.len:
      result.add(toJson(x[i])) # Recurse to add %

  of nnkTableConstr: # nnk stands for Nim node kind
    result = newNimNode(nnkTableConstr)
    for i in 0 .. <x.len:
      assert x[i].kind == nnkExprColonExpr
      result.add(newNimNode(nnkExprColonExpr)
        .add(x[i][0])           # First element: no %
        .add(toJson(x[i][1]))): # Second element: Recurse to add %

  else:
    result = x # End of recursion

  result = result.prefix("%") # Surround this level with %
{% endhighlight %}

And that's it! Now our `%*` works just as we want it to. If we did anything
wrong, we can modify the macro to check the actual code it produces:

{% highlight nimrod %}
macro `%*`*(x: expr): expr =
  result = toJson(x)
  echo result.repr # Print code representation of AST
{% endhighlight %}

This prints:

    % [% {"name": % "John", "age": % 30}, % {"name": % "Susan", "age": % 31}]

Perfect! This macro we just developed landed in Nim's [json
module](http://nim-lang.org/docs/json.html) already.

### Enum Parsing optimization

With enums we can create new types that contain ordered values, just like this:

{% highlight nimrod %}
  type Fruit = enum Apple, Banana, Cherry
{% endhighlight %}

Strings can be parsed to an enum using `parseEnum` from [strutils](http://nim-lang.org/docs/strutils.html):

{% highlight nimrod %}
  let fruit = parseEnum[Fruit]("cherry")
{% endhighlight %}

If we do this a lot, we notice that it's kind of slow though:

{% highlight nimrod %}
for i in 1 .. 10_000_000:
  var select = parseEnum[Fruit]("cherry")
  doAssert select == Cherry
{% endhighlight %}

This takes 2.2 seconds on my machine. Let's look at the definition of
`parseEnum` to find out why:

{% highlight nimrod %}
proc parseEnum*[T: enum](s: string): T =
  ## Parses an enum ``T``.
  ##
  ## Raises ``ValueError`` for an invalid value in `s`. The
  ## comparison is done in a style insensitive way.
  for e in low(T)..high(T):
    if cmpIgnoreStyle(s, $e) == 0:
      return e
  raise newException(ValueError, "invalid enum value: " & s)
{% endhighlight %}

We can see the problem already. We iterate through all the values inside the
enum type, from `low(T)` to `high(T)`. Then `$e` creates a string of each enum
value, which is quite expensive. Since we already know the type of the enum at
compile time, we could create the strings at compile time as well.

Again, let's think about what we want the result to look like before writing
the macro. Basically what we want to do is unroll the for loop at compile time:

{% highlight nimrod %}
if cmpIgnoreStyle(s, "Apple") == 0: return Apple
if cmpIgnoreStyle(s, "Banana") == 0: return Banana
if cmpIgnoreStyle(s, "Cherry") == 0: return Cherry
raise newException(ValueError, "invalid enum value: " & s)
{% endhighlight %}

Now we can create the proc. Other than in the last example we won't create the
AST manually this time. Instead we use `parseStmt` to create a statement AST
from a string containing Nim code. An equivalent `parseExpr` for expressions
exists as well. Here's how the final proc with a macro inside looks:

{% highlight nimrod %}
proc parseEnum*[T: enum](s: string): T =
  macro m: stmt =
    result = newStmtList()
    for e in T: result.add parseStmt(
      "if cmpIgnoreStyle(s, \"$1\") == 0: return $1".format(e))

    result.add parseStmt(
      "raise newException(ValueError, \"invalid enum value: \"&s)")

    #echo result.repr # To make sure we get what we want

  m() # Actually invoke the macro to insert the statements here
{% endhighlight %}

Running the same code with our new implementation of parseEnum takes 0.5
seconds now, about 4 times faster than before. Great!

### HTML DSL

We can use Nim's templates and macros to create domain specific languages
(DSL) that are translated into Nim code at compiletime. Nim's syntax is quite
flexible, so this is a powerful tool. As an example we build a simple HTML DSL.

The goal is to be able to write this:

{% highlight nimrod %}
proc page(title, content: string) {.htmlTemplate.} =
  html:
    head:
      title: title
    body:
      h1: title
      p: "Default Content"
      p: content

echo page("My own website", "My extra content")
{% endhighlight %}

And thus print the following HTML:

{% highlight html %}
<html>
  <head>
    <title>
      My own website
    </title>
  </head>
  <body>
    <h1>
      My own website
    </h1>
    <p>
      Default Content
    </p>
    <p>
      My extra content
    </p>
  </body>
</html>
{% endhighlight %}

For convenience we want to use the `htmlTemplate` macro as a pragma, annotated
as `{.htmlTemplate.}`. Instead we could also write it in this way:

{% highlight nimrod %}
htmlTemplate:
  proc page(title, content: string) =
    html:
      head:
        title: title
      body:
        h1: title
        p: "Default Content"
        p: content
{% endhighlight %}

The `htmlTemplate` macro shall transform the `page` proc, adding a `string`
return type and creating a new body out of the DSL definition, into this:

{% highlight nimrod %}
proc page(title, content: string): string =
  result = ""
  result.add "<html>\n"
  ...
  result.add "</html>\n"
{% endhighlight %}

Looks simple enough, here's how the macro works:

{% highlight nimrod %}
macro htmlTemplate(procDef: expr): stmt =
  procDef.expectKind nnkProcDef

  # Same name as specified
  let name = procDef[0]

  # Return type: string
  var params = @[newIdentNode("string")]
  # Same parameters as specified
  for i in 1..<procDef[3].len:
    params.add procDef[3][i]

  var body = newStmtList()
  # result = ""
  body.add newAssignment(newIdentNode("result"),
    newStrLitNode(""))
  # Recurse over DSL definition
  body.add htmlInner(procDef[6])

  # Return a new proc
  result = newStmtList(newProc(name, params, body))
{% endhighlight %}

The real magic of recursively handling the HTML tags happens in `htmlInner` of
course, a compiletime proc that calls itself recursively to iterate over the
body definition:

{% highlight nimrod %}
template write(arg: expr) =
  result.add newCall("add", newIdentNode("result"), arg)

template writeLit(args: varargs[string, `$`]) =
  write newStrLitNode(args.join)

proc htmlInner(x: NimNode, indent = 0): NimNode
              {.compiletime.} =
  x.expectKind nnkStmtList
  result = newStmtList()
  let spaces = repeat(' ', indent)
  for y in x:
    case y.kind
    of nnkCall:
      y.expectLen 2
      let tag = y[0]
      tag.expectKind nnkIdent
      writeLit spaces, "<", tag, ">\n"
      # Recurse over child
      result.add htmlInner(y[1], indent + 2)
      writeLit spaces, "</", tag, ">\n"
    else:
      writeLit spaces
      write y
      writeLit "\n"
{% endhighlight %}

We can check that we get the expected output by adding a simple `echo
result.repr` at the end of `htmlTemplate`:

{% highlight nimrod %}
proc page(title, content: string): string =
  result = ""
  add(result, "<html>\x0A")
  add(result, "  <head>\x0A")
  ...
  add(result, "</html>\x0A")
{% endhighlight %}

Where `\x0A` is just the newline character. Looks good and the output works!

[emerald](http://flyx.github.io/emerald/) is a much more complete HTML DSL that
works in a similar manner. A simpler HTML generator is included in the
standard library in the [htmlgen](http://nim-lang.org/docs/htmlgen.html)
module.

## Conclusion

I hope you enjoyed this trip through Nim's metaprogramming capabilities. Always
remember: With great power comes great responsibility, so use the least
powerful construct that does the job. This reduces complexity and makes it
easier to understand the code and keep it maintainable.

For further information and reference see:

- [Nim Tutorial (Part 2)](http://nim-lang.org/docs/tut2.html#templates)
- [Nim Manual](http://nim-lang.org/docs/manual.html#templates)
- [Macros Module](http://nim-lang.org/docs/macros.html)

Discuss on [Hacker News](https://news.ycombinator.com/item?id=11851234) and [r/programming](https://www.reddit.com/r/programming/comments/4mpuh7/introduction_to_metaprogramming_in_nim/).
