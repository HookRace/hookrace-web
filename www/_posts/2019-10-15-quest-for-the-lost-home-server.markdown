---
layout: post
title: "Quest for the Lost Home Server"
tags: Programming
permalink: /blog/quest-for-the-lost-home-server/
---

Today I lost access to my home server. As I described in [a previous post](/blog/linux-desktop-setup/) I depend heavily on the server to fetch my emails, as a file server, to synchronize files, for newsbeuter and irssi sessions and many other things. As no one was going to be in proximity of the server for the next few hours, my goal for today was to solve the problem remotely.

<!--more-->

The symptom was that my SSH connection attempts failed. The server also didn't respond to pings. As the server is sitting at home behind a regular DSL connection it uses a dynamic IP address that is shuffled every 24 hours. So my first hunch was that the daily reconnect might just have happened at a different time today and I gave the server some time to broadcast its new IP address to my domain registrar.

After about an hour I still couldn't connect, so I started investigating. Maybe the API of the domain registrar changed (as happened before) or my script failed for another reason? I know that my home server does nightly backups of the other servers I run. So I connected to one of them and checked the journalctl log. To my surprise no connection from the server happened last night. My worst fear was that the server was hanging due to a hardware problem, as I ran into [similar problems](https://bugzilla.kernel.org/show_bug.cgi?id=109051) with this Intel Bay Trail CPU before. (The issue seems to be that Intel underdesigned the power delivery on those systems, which the graphics driver is trying to [work around](https://cgit.freedesktop.org/drm-intel/commit/?id=a75d035fedbdecf83f86767aa2e4d05c8c4ffd95).)

But I wasn't ready to give up yet, so I tried to think of any other activities the server would do that might leave a trace. I found out that my email hoster doesn't provide easy access to the IP addresses that access the mailbox. I couldn't think of any other traces at the time, but next time I might check if my IP address is visible on some IRC server.

As a last resort I remembered that my ISP usually gives me quite similar IP addresses, so I used whois on yesterday's IP address to see how large the subnet is. I got back a subnet of only 2¹⁴ addresses, which seemed reasonable enough to scan. I hope the ISP is not too mad at me for such a small port scan, but to reduce the impact I also told nmap to only scan the specific SSH port that I use.

After about 5 minutes I had found a single server responding on my custom SSH port, which was indeed my home server. After logging in and checking out the system I noticed the (usual) problem of cronie being unable to spawn new processes after a glibc update. So I simply restarted cronie to fix the problem.

I still wish that Arch Linux will one day tell me which running executables to restart after a system upgrade, as SLES does, to prevent problems like this in the future. I think the general approach is just to check which process references deleted shared libraries. Something like this, [similar to here](https://locallost.net/?p=233):

{% highlight bash %}
ps axh -o pid | while read PID; do grep ".so" /proc/$PID/maps | grep "(deleted)" && echo "$PID" && sed -e "s/\x00/ /g" < /proc/$PID/cmdline && echo "\n"; done
{% endhighlight %}
