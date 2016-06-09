---
layout: post
title: "DDNet Evolution, Architecture & Technology"
tags: DDNet
permalink: /blog/ddnet-evolution-architecture-technology/
---

It's been roughly 3 years since [DDraceNetwork](https://ddnet.tw/) (DDNet)
started in the summer of 2013. Last year I wrote a non-technical [History of
DDNet](https://forum.ddnet.tw/viewtopic.php?f=3&t=1824). Today in this post we
will dive into the technical side of what makes DDNet run.

For the uninitiated, DDNet is an [open-source
modification](https://github.com/ddnet/ddnet) of
[Teeworlds](https://www.teeworlds.com/), a retro multiplayer shooter. DDNet is
a game in which you, instead of killing each other, cooperate with each other
to work your way through challenging maps, trying to beat the map or get a
better time than other teams.

<!--more-->
<p>
  <div class="video-container">
    <iframe src="https://www.youtube.com/embed/xwyk4hPZM1g" frameborder="0" allowfullscreen></iframe>
  </div>
</p>

What we offer for DDNet is the [client and server
software](https://ddnet.tw/downloads/) as well as official servers that run all
around the world. We have an international community of players trying to beat
the latest maps and records, and dozens of mappers who send in their newest
creations for our consideration. A new map is [released every few
days](https://ddnet.tw/releases/) and occasionally
[tournaments](https://ddnet.tw/tournaments/) happen in which the best compete
against each other on brand-new maps.

At the time of writing there are 968 people playing Teeworlds, 678 of which are
on a server running the DDNet mod, and 430 on the official DDNet servers. So
this project is not especially big and not many people stumble upon DDNet. Our
player base consists in a large part of old Teeworlds players and their friends
who heard about us by word of mouth.

Nevertheless I hope that the technical challenges we faced and face as well as
our solutions are interesting. Let's start with an overview of what we have:

## Overview & Financing

DDNet runs official servers in Germany, Russia, USA, Canada, Chile, Brazil,
China, South Africa and Iran.

![DDNet Locations](/public/locations.png)

Since DDNet is totally free, has no advertisements and no in-game purchases, it
offers no stream of revenue and is solely [funded by donations, donated servers
and my own money](https://ddnet.tw/funding/). So one of the goals is to keep
costs down: On average we pay 10 € per month for each location.

Let's see what we can offer with this to our thousands of players ([detailed
statistics](https://ddnet.tw/stats/)) and how DDNet evolved to keep with
increasing number of players, maps and the product of both, ranks.

[![Number of finishes](/public/stats-finishes.png)](https://ddnet.tw/funding/)
[![Number of players](/public/stats-players.png)](https://ddnet.tw/funding/)

## Servers & Locations

We exclusively use cheap virtual private servers (VPS). They offer enough
performance for us and are significantly cheaper than dedicated servers. To
have a bit stronger guarantees on performance I prefer KVM and XEN over OpenVZ,
but there are good and bad hosters for each. In the end you always have to try
out a hoster for a few days up to a month to find out if it is suitable for
hosting official DDNet servers. The most important criteria for us are:

- Good and consistent CPU performance: should do about 400 players per CPU core
- Low network latency to our players in the region: DDNet is only enjoyable
  with a low and stable ping
- 0.5 to 1 GB of RAM is enough for <del>everyone</del> most locations: running
  about 20 game servers per location

When searching for hosters in a country you should prefer to look in the
language of the country instead of English. These tend to be cheaper and less
overcrowded by international buyers. Be prepared to use a translator and have
trouble with the support, but after some time all these hoster websites start
to look the same to you, no matter if they're in English, German, French,
Spanish or Farsi.

When in doubt, ask locals what hosters they use or recommend, or check the
whois entries for servers in the region that seem to run well. For example the
`whois 31.186.251.128` entry tells you that [DDNet.tw](https://ddnet.tw/) (and
this blog) are hosted at Nuclear Fallout Enterprise, connected by the InterNAP
Network:

    % This is the RIPE Database query service.
    % The objects are in RPSL format.
    %
    % The RIPE Database is subject to Terms and Conditions.
    % See http://www.ripe.net/db/support/db-terms-conditions.pdf
    
    % Note: this output has been filtered.
    %       To receive output for a database update, use the "-B" flag.
    
    % Information related to '31.186.250.0 - 31.186.251.255'
    
    % Abuse contact for '31.186.250.0 - 31.186.251.255' is 'noc@internap.com'
    
    inetnum:        31.186.250.0 - 31.186.251.255
    netname:        INAP-LON-nuclearfallout-31-186-250-0
    descr:          Nuclear Fallout Enterprise
    country:        DE
    admin-c:        INAP-RIPE
    tech-c:         INAP-RIPE
    status:         ASSIGNED PA
    mnt-by:         INAP-MAINT-RIPE
    created:        2013-08-08T19:09:47Z
    last-modified:  2014-03-06T23:07:27Z
    source:         RIPE # Filtered
    
    person:         InterNAP Network Operator
    address:        InterNAP Network Services
    address:        Two Union Square
    address:        601 Union Street Suite 1000
    address:        Seattle, WA 98101 USA
    phone:          +1 206 441 8800
    fax-no:         +1 206 256 9580
    nic-hdl:        INAP-RIPE
    mnt-by:         INAP-MAINT-RIPE
    created:        2002-03-04T12:57:06Z
    last-modified:  2002-03-04T12:57:06Z
    source:         RIPE # Filtered

    % This query was served by the RIPE Database Query Service version 1.87.3 (DB-2)

Unfortunately the story doesn't end here for us. Since DDNet is a multiplayer
open-source game we seem to attract the kind of people who know how to find a
DDoS (distributed denial of service) booter. This resulted in us being kicked
out at many hosters since they either do not have any (D)DoS protection or it
is not sufficient against some attacks, which then also affect other customers
of theirs.

In the end we settled on [NFOservers](https://www.nfoservers.com/) for our main
servers, While they do not advertise any DDoS protection, they turned out to be
the only one of dozens of hosters I tried that can withstand most of the
attacks we get regularly. I'm of course talking about cheap VPSes, I hear that
good DDoS protection exists for a few hundred euros per month, which is far
outside of our reach. Unfortunately NFOservers is only available in USA and
Germany.

Here's what the attacks in a regular 2 week period looks like in our most
popular location, Europe (GER server):

<p>
  <div style="height: 20em; overflow: scroll;">
    <a href="/public/ddos.png"><img src="/public/ddos.png" alt="GER DDoS attacks"></a>
  </div>
</p>

In Russia (Moscow) we had ok experiences with [reg.com](https://www.reg.com/),
at least the ping for local players is lower than in any other location.

Recently we got a server hosted in Iran again, which has always been a
difficult country to host a server in, for matters of cost as well as their
preference of an intranet over the internet.

In Chile we're now with [zGlobalHost](http://www.zglobalhost.com/), one of the
few hosters with unlimited bandwidth and by far the best I found for just 9 €
per month. Their server room looks very cozy:

![ZGlobalHost data center](/public/zglobalhost.jpg)

In Chile we also had this [bizarre
story](https://www.facebook.com/zglobalhost/) back in January, after the entire
data center of ZGlobalHost went down:

- January 26, 16:04 (local time Chile)  
  Our provider GTD has trouble again and is still not giving estimated time of
  repair. We will inform you with more news as soon as possible.

- January 26, 18:06  
  Our provider informs us that in the next 30 minutes everything should be
  operational again.

- January 26, 19:09  
  GTD reports that there are two fiber cuts, one in Pudahuel and another in the
  Eastern Zone. They are certifying the fibers and lifting services one by one.

- January 26, 22:34  
  Be informed that when we are back online and GTD reports that it will provide
  further details about the incident as it was just reported that the fiber cut
  was due to vandalism, the streets in Providencia are now reported to be safe
  for repair works again.  
  We do not have backup lines. It will at least take another hour but
  repairs should not take longer than until this morning. You will be
  mailed a report instantly when the servers are back online and tomorrow
  GTD will send a report with details.

- January 26, 22:36  
  The new details we have say that there were 4 cuts today, which is why
  there was such a long delay:  
  Locations: Pudahuel, Providencia, La Florida, Macul  
  Causes for each cut:
  Fire, Vandalism, Highway Accident, Material fatigue in electric pole cut it down  
  All ISPs in various areas have been affected, particularly fiber optic, both
  business and home addresses.

- January 27, 03:00  
  Service is fully operational again since 02:30 AM.

To make sure that our servers keep running well, we [monitor
them](https://ddnet.tw/status/) and record the [server
statistics](https://ddnet.tw/stats/server/). When an unexpected event happens,
like a server downtime or high network traffic, a notification is automatically
sent. You can read more about this system in [a separate post from last
month](https://hookrace.net/blog/server-statistics/).

## Software Platform

As the operating system our servers run Linux, preferably
[Debian](https://www.debian.org/), but also some instances of Ubuntu and
CentOS. In areas like South Africa, South America and Iran bandwidth can be
exorbitantly expensive, so when you get a donated server in one of those
locations you can not afford to be picky.

For the databases we use [MariaDB](https://mariadb.org/), the drop-in MySQL
fork started by the original MySQL developers after Oracle acquired it. We use
this database to record our maps, saved games and the ranks of all players
worldwide and keep these synchronized around the world.

Initially we started with a single VPS and a database running directly on it.
As we grew to add more locations around the world we used a circular
master-slave replication. With time some servers turned out to be less stable
than others, so they were taken out of the loop, their game servers
communicating with nearby MariaDB servers instead:

<img class="halfimg" src="/public/ddnet-replication-1.png" alt="Old MariaDB Replication"><img class="halfimg" src="/public/ddnet-replication-2.png" alt="New MariaDB Replication">

Now we're switching away from this circular master-slave replication to having
a small number of master databases which are replicated by slaves at each
location. This has the advantage that database reads happen locally, so getting
your own rank is always fast. Meanwhile new ranks are sent to a master server
when possible or otherwise stored in a local file to be added later. This is
necessary since our cheap servers are not always connected to the entire
internet, sometimes they are only available from inside their own country.

The DDNet client and server, being based on Teeworlds, are also written in
C/C++ (also known as C with classes). Luckily their performance was good from
the start, but a few optimizations still helped with keeping the load low when
running up to 60 game servers on a single VPS, for example:

- Reduce the number of syscalls by caching the value of `gettimeofday()` until
  a new tick happens or network packet comes in
- Do not execute the main game loop when the server is currently empty, as
  nothing will happen with no players anyway
- Use CPU (`nice`) and IO (`ionice`) priorities to prevent background tasks
  (lower priority) to get in the way of game server processes (high priority)
- Each VPS compiles its own server binary tuned for the specific target
  architecture
- Prefer to download map files over HTTP from our web server instead of the
  built-in UDP protocol that can be shaky and slow. This also reduces the
  network traffic on game servers, keeping network jitter lower.
- Instead of downloading the entire archive on a new client release, update
  only the changed files

Generally the idea is to wait until you notice that some resource is being used
too much, benchmark and analyze what the reason for it is, and only then
optimize this.

## Website

Back when DDNet started in 2013 the website ran on the same VPS as our game
servers. Later we switched to hosting the website on a separate VPS, in part to
improve the DDoS protection by having mostly UDP on game servers and mostly TCP
on the web server, but also to make sure that the website traffic and CPU usage
do not impact the game servers.

This has become even more important since the map downloads are now also
happening over HTTP from the web server instead of going over UDP from the game
servers. Automatically updating the list of available maps is done with a small
Nim script that I already presented in [a previous
article](/blog/what-makes-nim-practical/#use-as-a-scripting-language):

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

Most of the [DDNet.tw](https://ddnet.tw/) website is [statically built by
jekyll](https://github.com/ddnet/ddnet-web) and automatically deploying using
GitHub's webhooks. That's a simple solution and requires very few resources.

Pages about [player status](https://ddnet.tw/status/),
[ranks](https://ddnet.tw/ranks/), [map releases](https://ddnet.tw/releases/)
and [mappers](https://ddnet.tw/mappers/) are also statically built, but by
[Python
scripts](https://github.com/ddnet/ddnet-scripts/tree/master/servers/scripts)
directly on the server. This also makes sense since these pages are expensive
to build and are requested more often than they have to be generated.

We also used to generate the [player pages](https://ddnet.tw/players/) in the
same way, but since we have about 90,000 ranked players so far this would've
become a bit too expensive. Instead we run a small uWSGI Python server to
dynamically generate them now from the data kept in memory. Right now it takes
about 40-80 ms to generate a single player page, by far fast enough for us. If
the computation time goes up or we suddenly become much more popular caching
could be implemented.

We used to package a growing selection of skins with the DDNet client, but
maintaining this became too cumbersome and increased memory usage for the
client. Now instead the website offers a [database of
skins](https://ddnet.tw/skins/) where you can select individual skins or skin
packs.

## Software Releases

DDNet has been in active development for the last 3 years, with our first
client & server software release back in [October 9,
2013](https://github.com/ddnet/ddnet/releases/tag/1.18).

Initially I used to build the releases manually using virtual machines for
Windows and Linux, later also Mac OS X. After some time this got tedious of
course, so I wrote an [automated build
script](https://github.com/ddnet/ddnet-scripts/blob/master/ddnet-release.sh).
It uses a Debian chroot for Linux building, MinGW for Windows and a QEMU VM for
Mac OS X. I set up the Mac OS X QEMU image using [a continuously updated
guide](http://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/) and it works just
fine. I connect to the guest OS through a simple ssh connection that is
forwarded to the virtual machine:

{% highlight bash %}
# Start the Mac OS X VM
qemu-system-x86_64 [...] \
  -netdev user,id=hub0port0,hostfwd=tcp::10022-:22 \
  -device e1000-82545em,netdev=hub0port0,id=mac_vnet0 \
  &>/dev/null &

# Wait until VM is booted up
while ! ssh -p 10022 -o ConnectTimeout=10 localhost exit; do true; done

# Run build script
ssh -p 10022 localhost "# Run build script for Mac OS X"
{% endhighlight %}

Building for Android is also possible but disabled right now because we
still have to port the Android version to SDL2, and DDNet is not really
playable on Android anyway since a mouse is the most important tool in the
game.

When you think about it, it's pretty amazing that you can easily cross-compile
on a single machine for all of these targets:

- Windows x86, x86_64
- Linux x86, x86_64
- Mac OS X
- Android armeabi, armeabi-v7a, mips, x86

The automated builds happen on my cheap ASRock Q1900-ITX home sever (which was
also used to [stream gameplay from DDNet servers](/blog/ddnet-live/)) and
still only take a few minutes to complete:

![Home server](/public/q1900-itx.jpg)

Finally when all goes well you get a small summary of the build times that
looks like this:

    Preparation:      10 s
    Linux x86_64:     62 s
    Linux x86:        64 s
    Windows x86_64:  106 s
    Windows x86:      84 s

For the automated builds we use bundled static libraries, while people who
build our software locally probably prefer to use their system libraries. Using
`pkg-config` we detect whether libraries are available at compile-time and
otherwise use the static ones if they are available. This behaviour can be
controlled manually as well:

{% highlight bash %}
# Analog to ./configure
bam config curl.use_pkgconfig=false \
           opus.use_pkgconfig=false \
           opusfile.use_pkgconfig=false \
           ogg.use_pkgconfig=false
# Analog to make
bam release
{% endhighlight %}

Even a [JavaScript port](http://teewebs.net/) of DDNet client is available for
playing without any installation, compiled with Emscripten. For this purpose
the official servers also accept WebSocket connections additionally to UDP.

New releases are added to the [website](https://ddnet.tw/downloads/) with a
nice changelog. But most existing players get their update notification in the
client directly and run the updater from there, only downloading the files that
actually changed instead of the entire new release.

## Map Testing & Releases

The best part about DDNet is its active community. New maps for our servers are
regularly sent in to be tested and later released. Initially the testing
process was implemented as [Trac](https://trac.edgewall.org/) installation. But
it turned out that the non-technical people did not appreciate it much, so we
switched to a [regular phpBB forum](https://forum.ddnet.tw/) for testing and
communication.

New maps can be uploaded by our testers to the test servers and then tested
there. The map upload happens with an upload script to the web server. The test
servers around the world use sshfs to access these maps.

Once a new map has been cleared by the testers and all bugs fixed by the
mapper, it can be
[released](https://github.com/ddnet/ddnet-scripts/blob/master/map-release) on
the proper server. The preparation for this happens on the web server and is
then synchronized to the other servers using an internal git repository.

The new map is also announced on the [recent map releases
page](https://ddnet.tw/releases/) as well as its feed. Players on the official
servers are informed about the map release using a broadcast message.

It's possible to take a look at our maps using a WebGL renderer that supports
parts of the map format, but no animations for example: [Lonely
map](https://ddnet.tw/maps/?map=Lonely)

When a popular new map is released, it's possible to have hundreds of players
taking up the challenge of playing the new map at once, either in small teams
or in big groups.

Finally every day the newly released maps are added to our [public maps
repository](https://github.com/ddnet/ddnet-maps),
which can be used to easily run a clone of the official DDNet servers at home or
on your own server. Just download the repository, add the DDNet-Server binary
to the same directory and run it. A simple shell script suffices to update this
repository:

{% highlight bash %}
#!/usr/bin/env zsh
mkdir -p /home/teeworlds/ddnet-maps
cd /home/teeworlds/ddnet-maps
rm -rf types

echo 'add_path $USERDIR\nadd_path $CURRENTDIR' > storage.cfg

for i in `cat ../servers/all-types`; do 
  TYPEDIR=types/${i:l}
  echo "add_path $TYPEDIR" >> storage.cfg
  mkdir -p $TYPEDIR/maps
  grep "|" ../servers/$TYPEDIR/maps | cut -d"|" -f2 | while read j; do 
    cp -- "../servers/maps/$j.map" $TYPEDIR/maps
    cp ../servers/$TYPEDIR/flexvotes.cfg $TYPEDIR
    grep -v "flexname.cfg" ../servers/$TYPEDIR/flexreset.cfg > $TYPEDIR/flexreset.cfg
    tail -n +5 ../servers/$TYPEDIR/votes.cfg > $TYPEDIR/votes.cfg
  done
done

git add * &>/dev/null
git commit -a -m "daily update" &>/dev/null
git push &>/dev/null
{% endhighlight %}

## Server Types

In the start DDNet hosted only new maps categorized by their difficulty:

- [Novice](https://ddnet.tw/ranks/novice/)
- [Moderate](https://ddnet.tw/ranks/moderate/)
- [Brutal](https://ddnet.tw/ranks/brutal/)

Later we also added categories for maps from old servers:

- [Oldschool](https://ddnet.tw/ranks/oldschool/): Ancient maps
- [DDmaX](https://ddnet.tw/ranks/ddmax/): Old DDRace server before DDNet

As well as categories for different kinds of single player maps:

- [Solo](https://ddnet.tw/ranks/solo/)
- [Race](https://ddnet.tw/ranks/race/): Classic mod with fewer features,
  converted maps to new DDNet format
- [Dummy](https://ddnet.tw/ranks/dummy/): Maps where you play with another
  player that is unable to move

The idea is that, since we are the major hoster in many countries, many of
these good maps would be forgotten forever if we don't host them. At the same
time we try to motivate people to make creative new maps, having worked
together with mappers to add new features that they require.

While the DDNet servers are the main focus, we also use our servers for hosting
a few other Teeworlds mods since they use very few resources and the servers
are running already anyway.

## Tournaments

When a well-known mapper makes a special map, they can choose to have their map
played at an official DDNet [tournament](https://ddnet.tw/tournaments/). For
this it is necessary to keep the map under wraps, showing it only to a few
select testers. This ensures that none of the players at the tournament will be
familiar with the map already, which would give them an unfair advantage.

Usually tournaments are done on Sunday, 20:00 Central European Time since most
of our player base is located in Europe. You can read about the work that goes
into [preparing a
tournament](https://github.com/ddnet/ddnet-scripts/blob/master/tournament-preparation).
But the really stressful part is actually holding the tournament.

Since it runs at the same time on all (currently 9-10) servers worldwide, you
need to control them all at once. If the tournament map has not been tested
well enough it might need to be quickly fixed and reloaded during the
tournament. Sometimes problems occur on a single server. Unfortunately
tournaments also attract DDoS attacks.

Here you can see the last tournament's top run, for which the players had 2
hours to compete:

<p>
  <div class="video-container">
    <iframe src="https://www.youtube.com/embed/wzfSDzgP6mA" frameborder="0" allowfullscreen></iframe>
  </div>
</p>

Some of our mappers even created [fancy ingame loading
screens](https://forum.ddnet.tw/viewtopic.php?p=1560#p1560) for the start of
the tournament:

<p>
  <div class="video-container">
    <iframe src="https://www.youtube.com/embed/_OEgiJFKtQw" frameborder="0" allowfullscreen></iframe>
  </div>
</p>

## Client & Server Development

Development of the client and server software is often challenging because of
backwards compatibility:

1. People with regular Teeworlds client should still be able to play on DDNet
2. People with old DDNet client (or custom client) should still be able to play
3. Old ranks would be invalidated by slight fixes/changes in game physics

Nevertheless we managed to put in many new features, for example:

- 64 player servers instead of just 16 players:  
  Players with vanilla Teeworlds client only see the 15 closest players on a server.
- Protect players from connection timeouts:  
  The timeout protection works by the client having a unique timeout code that
  it sends to the server whenever it connects. This tells the server if the
  player has had a timeout before and they can get back their ingame character.
- Allow saving game in team and continuing at a later time:  
  The save games are stored in our MariaDB database, but can only be loaded at
  a single location. Otherwise players would have the ability to load a save
  game multiple times, which is not what we want.
- Save the ranks of teams and not just single players
- SDL2 client instead of old SDL1.2
- New physics: Jetpack, extended teleporters, walljump, tune zones
- Highly compressed Opus sounds in maps, it's amazing how much quality you get
  with small bit rates
- In-client updater using HTTPS and a [simple JSON
  file](http://update2.ddnet.tw/update.json) for control
- Dummy feature to control two ingame characters at once:  
  At first this was meant to help map testers to be able to test maps easier,
  without requiring a second player. But it's also very popular with regular
  players as well as mappers who come up with interesting maps where only one
  player has all the control.
- ...

Particularly important for the automation of the official DDNet servers is our
FIFO console system. Every server listens for commands from a FIFO file. This
enables us to automate the map releases, server restarts and broadcasts among
other things. To broadcast a message about a new map release to all servers,
you can just do this:

{% highlight bash %}
echo 'broadcast "New map Uniswim by Im corneum just released on Solo server"' > servers/*.fifo
{% endhighlight %}

We have a [small Python
script](https://github.com/ddnet/ddnet-scripts/blob/master/servers/bc.py) that
can do this automatically and also broadcast in a [custom big ASCII art
font](https://github.com/ddnet/ddnet-scripts/blob/master/servers/scripts/asciiart):

![ASCII DDNET](/public/ascii-ddnet.png)

Using the FIFO system at multiple locations at once is achieved using Cluster SSH.

We also offer a list of official DDNet servers [in
JSON](http://update2.ddnet.tw/ddnet-servers.json) which is used to inform the
client of the official servers as well as internally to generate the [status
page](https://ddnet.tw/status/).

Unfortunately attacks with spoofed IP addresses are quite common for us. So we
have added a spoofing protection using tokens to prevent attackers from filling
up servers with players with spoofed IPs.

We have no automated tests for the client and server, the community quickly
catches new bugs though. At least we use [automated Circle
CI](https://circleci.com/gh/ddnet/ddnet) building to make sure new changes
still compile fine.

## Communication

Ingame communication is a bit tricky since the game is built on simplicity and
we prefer not to have accounts. Introducing accounts this late into existance
would give another vector of abuse by giving attackers the chance to register
other people's names. So you can never be sure who you're actually talking to.
Thanks to some Social Engineering this has been a common source of leaked
moderator passwords.

So for official communications the [DDNet forum](https://forum.ddnet.tw/) is
used instead. We tried to use [Let's Encrypt](https://letsencrypt.org/) for
SSL/TLS certificates for the website, but it turned out that Windows XP is [not
supported](https://github.com/certbot/certbot/issues/1660) and many of our
players have old systems still running Windows XP. I guess that's in the nature
of running one of the few actively developed online games that work perfectly
fine on 15 year old machines. Instead we're now using
[StartSSL](https://www.startssl.com/), which seems to work fine, except that
wildcards would cost money. Once Let's Encrypt works fine on Windows XP we
might be able to switch back.

Otherwise technically inclined people mostly discuss on IRC
([#ddnet](irc://irc.quakenet.org/ddnet) on QuakeNet,
[WebChat](http://webchat.quakenet.org/?channels=ddnet&uio=d4),
[Logs](https://ddnet.tw/irclogs/)) and Skype. Most non-technical discussions
happen via Skype.

## Future Directions

The main goal is to keep DDNet running, well maintained, and keep a steady
stream of new maps from the community.

We submitted DDNet to [Steam
Greenlight](http://steamcommunity.com/sharedfiles/filedetails/?l=german&id=506147661)
and have been greenlit since then, which means we could release it on Steam
now. This would be an opportunity for us to reach more potential players,
especially those who have never played Teeworlds before. The main problem right
now is that DDNet is a difficult game to get started with, since it requires
practice, fine control and patience. Right now work is [being
done](https://forum.ddnet.tw/viewtopic.php?t=3771) to create a tutorial for new
players and generally improve the ease of getting started.
