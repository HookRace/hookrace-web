---
layout: post
title:  "Nimrod Adventures"
categories: nimrod
permalink: /blog/nimrod-adventures/
---

To learn some Nimrod I held a talk about it at the [GPN14](https://entropia.de/GPN14) (in German, [Slides](http://felsin9.de/nnis/nimrod/nimrod-gpn14.pdf)).

Afterwards I started solving [Rosetta Code](http://rosettacode.org/wiki/Rosetta_Code) tasks in [Nimrod](http://rosettacode.org/wiki/Category:Nimrod) to get a better feel for the language and standard library. That also made me discover some [rough edges](https://github.com/Araq/Nimrod/issues/created_by/def-?page=1&state=open) in the language, but luckily the community, albeit small, is active and competent. All the small code pieces I wrote are now on [Github](https://github.com/search?q=user%3Adef-+nimrod) too.

What I noticed is that most problems are as easy to solve as in Python, but much more performant. I'm now more confident that this language is the right choice for writing HookRace in.

Some highlights:

## [Insertion Sort](http://rosettacode.org/wiki/Sorting_algorithms/Insertion_sort#Nimrod)
Once in Python:
{% highlight python %}
def insertion_sort(l):
    for i in xrange(1, len(l)):
        j = i-1 
        key = l[i]
        while (l[j] > key) and (j >= 0):
           l[j+1] = l[j]
           j -= 1
        l[j+1] = key
{% endhighlight %}

and in Nimrod:
{% highlight nimrod %}
proc insertSort[T](a: var openarray[T]) =
  for i in 1 .. <a.len:
    let value = a[i]
    var j = i
    while j > 0 and value < a[j-1]:
      a[j] = a[j-1]
      dec j
    a[j] = value
{% endhighlight %}

## [Rank languages by popularity](http://rosettacode.org/wiki/Rosetta_Code/Rank_languages_by_popularity#Nimrod)
The standard library is extremely useful, providing an HTTP client, JSON parser, regular expression, string utils and algorithm, which I use to create a ranking of the popularity of languages on Rosetta Code:
{% highlight nimrod %}
import httpclient, json, re, strutils, algorithm, future

const
  langSite = "http://www.rosettacode.org/w/api.php?action=query&list=categorymembers&cmtitle=Category:Programming_Languages&cmlimit=500&format=json"
  catSize = "http://www.rosettacode.org/w/index.php?title=Special:Categories&limit=5000"
let regex = re"title=""Category:(.*?)"">.+?</a>.*\((.*) members\)"

var langs = newSeq[string]()
for l in parseJson(getContent(langSite))["query"]["categorymembers"]:
  langs.add(l["title"].str.split("Category:")[1])

var ranks = newSeq[tuple[lang: string, count: int]]()
for line in getContent(catSize).findAll(regex):
  let lang = line.replacef(regex, "$1")
  if lang in langs:
    let count = parseInt(line.replacef(regex, "$2").strip())
    ranks.add((lang, count))

ranks.sort((x, y) => cmp[int](y.count, x.count))
for i, l in ranks:
  echo align($(i+1), 3), align($l.count, 5), " - ", l.lang
{% endhighlight %}

## [Calling a C function](http://rosettacode.org/wiki/Call_a_foreign-language_function#Nimrod)
{% highlight nimrod %}
proc printf(formatstr: cstring) {.header: "<stdio.h>", varargs.}

var x = "foo"
printf("Hello %d %s!\n", 12, x)
{% endhighlight %}

## [Wrapping a shared C library](http://rosettacode.org/wiki/Call_a_function_in_a_shared_library#Nimrod)
{% highlight nimrod %}
proc openimage(s: cstring): cint {.importc, dynlib: "imglib.so".}

echo openimage("foo")
{% endhighlight %}

## [Quine](http://rosettacode.org/wiki/Quine#Nimrod)
A quine is a program that prints its own source code. This Nimrod program does that once at compiletime and once at runtime:
{% highlight nimrod %}
const x = "const x = |const y = x[0..9]&34.chr&x&34.chr&10.chr&x[11..100]&10.chr&x[102..115]&10.chr&x[117 .. -1]|static: echo y|echo y"
const y = x[0..9]&34.chr&x&34.chr&10.chr&x[11..100]&10.chr&x[102..115]&10.chr&x[117 .. -1]
static: echo y
echo y
{% endhighlight %}
