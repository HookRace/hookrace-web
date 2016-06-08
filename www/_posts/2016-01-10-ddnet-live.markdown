---
layout: post
title:  "DDNet Live: Twitch spectates an online game"
tags: DDNet
permalink: /blog/ddnet-live/
---

Last night I had an idea and implemented it, soo let's see what will happen.
But first, the idea:

> Have a livestream [1] of [DDNet](https://ddnet.tw/) running non-stop [2] that
> always shows some interesting [3] players on the server.

The resulting livestream is running on [Twitch](https://www.twitch.tv/ddnetlive).
All the scripts are on [Github](https://github.com/ddnet/ddnet-live)

<!--more-->
## 1. Livestream

It's surprisingly simple to livestream from Linux to Twitch. Only FFmpeg is needed:

{% highlight bash %}
#!/bin/sh

INRES="1280x720" # input resolution
FPS="30" # target FPS
GOP="60" # i-frame interval, should be double of FPS, 
GOPMIN="30" # min i-frame interval, should be equal to fps, 
THREADS="2"
CBR="2000k" # constant bitrate (should be between 1000k - 3000k)
QUALITY="ultrafast"  # one of the many FFMPEG preset
AUDIO_SRATE="44100"
AUDIO_CHANNELS="2" # 1 for mono output, 2 for stereo
AUDIO_ERATE="96k" # audio encoding rate
SERVER="live-fra" #  http://bashtech.net/twitch/ingest.php for list
source ./secret.sh

ffmpeg -v 0 -f x11grab -s "$INRES" -r "$FPS" -i :0.0 \
  -f alsa -i pulse -f flv -ac $AUDIO_CHANNELS \
  -b:a $AUDIO_ERATE -ar $AUDIO_SRATE \
  -vcodec libx264 -g $GOP -keyint_min $GOPMIN -b:v $CBR \
  -minrate $CBR -maxrate $CBR -vf "format=yuv420p"\
  -preset $QUALITY -acodec libmp3lame -threads $THREADS \
  -strict normal -bufsize $CBR \
  "rtmp://$SERVER.twitch.tv/app/$STREAM_KEY"
{% endhighlight %}

This works pretty well, but takes quite a bit of CPU on my old computer. So
instead I wanted to run it on my new server with a Haswell-era J1900 CPU.

Possible ways to improve performance:

- Use an OpenGL recorder instead of x11grab
- Get Intel Quick Sync Video working for hardware H264 encoding

## 2. Running Non-Stop

My server is a small and cheap ASRock Q1900-ITX:

![Q1900-ITX](/public/q1900-itx.jpg)

The nice part is that it's passively cooled and quite power efficient, drawing
only 5 Watts in idle. This machine has been running as my home server for quite
some time, but barely gets any action. Let's change that!

The X server starts without problems even without any monitors attached, the
only thing that's left to do is increasing the framebuffer size so that our
game can run in it:

{% highlight bash %}
$ cat ddnet.sh
#!/bin/sh

pulseaudio --start
xrandr --fb 1280x720 # Adjust framebuffer
cp settings_ddnet.cfg ~/.teeworlds/ # Restore backup
cd ddnet && ./DDNet

$ xinit ddnet.sh
{% endhighlight %}

We don't even need a window manager. After all we just run a single window in
exactly the resolution of the framebuffer.

The FFmpeg recording still works in exactly the same way. Unfortunately I
couldn't get H264 hardware encoding to work with the J1900 CPU. [Related bug
reports](https://github.com/shenhailuanma/qsv-ffmpeg-codec/issues/3) make me
believe it just doesn't work on these cheaper Intel CPUs.

## 3. <s>Artificial Intelligence</s> Twitch Control

Now that we have the game running and are streaming it to Twitch, we need to
control it somehow. My goal was to find an approach that always shows some
interesting players in action, so that you could watch the stream all day and
enjoy it.

But finding a reasonable way to do this seems too complicated and I didn't look
forward to hacking the DDNet C/C++ source code, so instead I opted to utilize
an existing system in DDNet: FIFO command input!

DDNet servers and clients can be remote controlled through a FIFO file. This is
very useful to send the same commands to dozens of servers at once. But for the
client its use was pretty limited, until now!

Instead of thinking of an algorithm to find interesting players, why not let
the Twitch viewers themselves control who they want to watch through the chat?

I did not modify the DDNet source code in any way and instead wrote a small
[Nim](http://nim-lang.org/) script based on the [IRC
module](https://github.com/nim-lang/irc) to connect to Twitch's IRC server and
forward the commands to the FIFO:

{% highlight nimrod %}
import irc, strutils, secret

const forbiddenCommands = @["exec", "quit", "exit", "disconnect"]

var
  client = newIrc("irc.twitch.tv", nick = "ddnetlive",
    serverPass = serverPass, joinChans = @["#ddnetlive"])
  fifo = open("input.fifo", fmWrite)

proc handle(nick, cmd: string) =
  for f in forbiddenCommands:
    if cmd.contains(f):
      return

  echo nick, ": ", cmd
  stdout.flushFile()

  fifo.write(cmd)
  fifo.write("\n")
  fifo.flushFile()

client.connect()
var event: TIrcEvent

while true:
  if client.poll(event):
    case event.typ
    of EvConnected:
      discard
    of EvDisconnected, EvTimeout:
      client.reconnect()
    of EvMsg:
      case event.cmd
      of MPrivMsg:
        handle(event.nick, event.params[^1])
      else:
        discard
{% endhighlight %}

Right now there is really not much limitation to what you can do. Only a
handful of commands are explicitly blocked, otherwise the client can be
controlled freely and every single chat message is sent to the client. The
[client commands](https://ddnet.tw/settingscommands/#client-commands) and
[client settings](https://ddnet.tw/settingscommands/#client-settings) list the
available commands and settings. [Chat
commands](https://ddnet.tw/settingscommands/#chat-commands) can be sent to the
server as well through the `say` command. Some examples:

{% highlight bash %}
connect ger.ddnet.tw:8303
team -1 # Join spectators
spectate_next # Spectate the next player
spectate 0 # Spectate player with ID 0
say Hi from twitch.tv/ddnetlive # Write chat messages
player_name DDNetLive # Change name
{% endhighlight %}

Let's see how this goes. Luckily the Nim script runs independently from the
server, so I will be able to make changes to it on the fly. Just be nice and
don't cause any trouble, thanks.

Feel free to [head over to Twitch](https://twitch.tv/ddnetlive), watch the
action and control the server using the Twitch chat. Twitch has some delay, so
it takes a few seconds for you to see your command executed.
