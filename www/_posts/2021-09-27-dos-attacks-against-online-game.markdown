---
layout: post
title: "DoS Attacks against my Online Game"
tags: DDNet
permalink: /blog/dos-attacks-against-online-game/
---

[DDraceNetwork](https://ddnet.tw/) is an [open source](https://github.com/ddnet/ddnet) online game I've been running since 2013 with a community of volunteers. The game is available for free, I'm hosting servers for it in many countries [around the world](https://ddnet.tw/status/) so that we have trusted [official ranks](https://ddnet.tw/ranks/). The servers are paid for by [donations](https://ddnet.tw/funding/), which I stop collecting once the cost of the servers for the current year is covered. I wrote about DDNet in a [previous post in 2016](/blog/ddnet-evolution-architecture-technology/) and also a bit about the [game history in 2013](https://forum.ddnet.tw/viewtopic.php?t=1824).

Pretty much since the beginning we have been suffering from DoS attacks against the servers. Since the [Steam release](https://store.steampowered.com/app/412220/DDraceNetwork/) in 2020 the player number has increased significantly, so that we have [about 1300 players](https://ddnet.tw/stats/) playing on average.

<!--more-->

<a href="https://ddnet.tw/stats/"><img alt="Points earned per Day by Server Type" src="/public/points-earned.svg" style="width: 100%;"></a>

Based on the rising player number the urge to deal with the DoS problem is larger than ever. A few months ago for example we organized a [Tournament](https://ddnet.tw/tournaments/56/) for everyone in the community to participate. Unfortunately the event was the continuous target of DoS attacks for multiple hours, with players fleeing from one server to the next, trying to find a safe refuge. These [bandwidth graphs](https://ddnet.tw/stats/server/) for our servers in Netherlands and Germany respectively illustrate the problem:

![Netherlands](/public/nld.ddnet.tw-net-1d.png)
![Germany](/public/ger2.ddnet.tw-net-1d.png)

Since DDNet only runs online and you feel every small network latency it is predestined as a target for DoS attacks. Outside of attacks our servers don't need to be particularly powerful for regular gameplay, 1 CPU core can support 150 players concurrently, memory consumption is minimal. So the total we are paying for our ~23 servers around the world is only [~2300 € for the year 2021](https://ddnet.tw/funding/).

## What we have tried

The kinds of attacks we receive are varied. There are generic reflection and amplification attacks, as well as spoofed game specific attacks of considerable strength. Some hosters have thrown us out following attacks impacting their other customers, and most hosters will blackhole our IP address for multiple hours before it gets to that.

There is even a [Wireshark dissector](https://github.com/heinrich5991/libtw2/tree/master/wireshark-dissector) for DDNet/Teeworlds available, written in Rust:

![Wireshark Dissector Screenshot](/public/dissector.png)

Some components of our system were relatively easy to protect: The database server has been moved to a secret IP address, and the game servers will store ranks in a local SQLite file if the main MySQL-based database is not available.

The web server hosting [DDNet.tw](https://ddnet.tw/) and other HTTPS-based components like the map downloader and updater are now reachable through Cloudflare only, and have received no attacks since then.

For the individual server infos the client currently has to communicate with each game server by UDP, thus revealing its own IP address without having connected to a server. Since one of the known attackers is running their own DDNet server, they can use this method to collect legitimate player IP addresses and spoof them in their attacks. To prevent this heinrich5991 implemented a centrally provided [HTTPS-based server info](https://github.com/ddnet/ddnet/pull/3772/). This means that every player only has to access the [JSON file](https://master1.ddnet.tw/ddnet/15/servers.json) containing all server information, instead of previously receiving this information via UDP from each server directly.

To prevent our own servers being used for amplification attacks we limit the server info packets per IP address in our [server setup script](https://github.com/ddnet/ddnet-scripts/blob/892c22672d93c1ea78e24ef81fa09ff60d28fdcc/ddnet-setup.sh#L29-L38):

{% highlight bash %}
iptables -t raw -A PREROUTING -p udp -j NOTRACK
iptables -t raw -A OUTPUT -p udp -j NOTRACK
iptables -N serverinfo
iptables -A INPUT -p udp -m u32 --u32 "38=0x67696533" -j serverinfo
iptables -A INPUT -p udp -m u32 --u32 "38=0x66737464" -j serverinfo
iptables -A serverinfo -m hashlimit --hashlimit-above 1000/s --hashlimit-burst 2500 --hashlimit-mode dstport --hashlimit-name si_dstport -j DROP
iptables -A serverinfo -m hashlimit --hashlimit-above 20/s --hashlimit-burst 100 --hashlimit-mode srcip --hashlimit-name si_srcip -j DROP
iptables-save > /etc/iptables.up.rules
{% endhighlight %}

Unfortunately even sending a few packets per second is considered a lot by some people, so with the spoofed server info requests coming in, we get a lot of complaints for the "port scanning" that we seem to be doing. I have to react quickly to those abuse mails in order to prevent our servers from being shut down, having to explain the situation and blocking any traffic from the IP range in question. An example mail:

> On 2021-04-19T14:58+0200, abuse@hetzner.com wrote:
> > On 18 Apr 12:03, abuse-out@X.com wrote:
> > > The address 49.12.97.180 from Your network tried to log in to
> > > our network using Port [...]
> > > 
> > > Below You will find a listing of the dates and
> > > times the incidents occured as well as the attacked IP-Addresses.
> > > This is a matter of concern for us and continued tries might result in
> > > legal action. If the machine was victim to a hack take it offline, repair
> > > the damage and use better protection next time.
> > > The times included are in Central European Time (CET).
> > > 
> > > [...]
> > >
> > > Regards,
> > > 
> > > X
>
> > Dear Mr Dennis Felsing,
> >
> > We have received information regarding spam and/or abuse from abuse-out@X.com
> > Please take all necessary measures to avoid this in the future.
> >
> > We also request that you send a statement within 24 hours to us and to the person who filed the complaint. This
> response should contain information about how this could have happened and what you intend to do about it.
> >
> > How to proceed:
> > - Solve the issue
> > - Send us a statement by using the following link: https://abuse.hetzner.com/statements/?token=X
> > - Send a response by email to the person who filed the complaint
> >
> > The statement will be checked by a staff member who will then coordinate any further proceedings. If you fail to
> comply within the stated deadline, the IP may be blocked.
> >
> > Important note:
> > When replying to us, please leave the abuse ID [AbuseID:X] unchanged in the subject line.
> >
> > Kind regards
> >
> > X
>
> Hi Mr. X,
> 
> The server in question (49.12.97.180) only responds to incoming requests. We received an incoming DoS attack with
> spoofed IP addresses, so the server replied to those. We already limit the number of packets per server that we respond
> to. For the future I have blocked all packets from X/24 of reaching the server, so the server also won't send
> any packets to them.
> 
> Best regards
> 
> Dennis Felsing

When players get a timeout ingame, they can reconnect later and keep playing from where they were before. We tried detecting a DoS attack and prolonging the timeout protection to [1 hour](https://github.com/ddnet/ddnet/issues/3780), but this lead to the problem of servers being full of characters having a timeout, making it impossible for new players to join, even those trying to reclaim their timed out character. Additionally teams can use the save functionality to save their progress and continue it at any later time, or even on another server.

This doesn't help when a VPS gets nullrouted for hours or even days at a time due to attacks. Many hosters offer "DDoS protection" for their servers, but so far none worked well. Some block all UDP traffic during an attack, which is clearly suboptimal for a UDP-based game. Others try to train their DDoS detection a bit to detect legitimate traffic, but some players and some packets of all players will still be considered as part of the attack and get blocked. The attackers can even use the DDoS protection and spoof the collected IP addresses of real players to get specific player IP addresses blocked. As a reaction of that I then have to discuss with the hoster for a long time that the players probably are not part of a botnet, but being spoofed.

Instead of cheap VPS servers we have tried getting dedicated servers at larger European hosters like OVH, Hetzner, ihor and NFOrce. The idea is that we have exclusive resources, so the chances of us impacting other customers is lower, and thus we won't get nullrouted so easily. Largely this works, but the available network bandwidth (usually 1-10 Gbit/s) as well as CPU usage become the limit. Each individual game server uses a single thread to handle all network packets, so it is still relatively easy and cheap for attackers to overload.

Cloud-based servers at AWS or Google Cloud perform well, even during attacks, but the bandwidth cost is exorbitant. We could consider using them for short periods during tournaments instead of permanently though.

## What we can still try

So far we have only tried 10 Gbit/s dedicated servers. Maybe the reason that the Cloud providers function so well during DoS attacks is that the actual servers the virtual machines run on have even larger bandwidth available. So we could still try to go to 20-25 Gbit/s for some dedicated servers, but only few cheap hosters offer this and what they charge for it is a lot for us. An example would be an additional 50 € booking fee and 100 € more per month for 20 Gbit/s at [NFOrce](https://www.nforce.com/)

The network handling of the game servers can be switched to run multithreaded, so that we can at least handle the packets of unconnected players faster. This doesn't help with the really cheap servers, and the attacker could just increase the attack strength beyond what we can handle with our usual 2-4 available cores. This already happens when multiple game servers are attacked at once, so it seems like it might not help much.

Instead of improving the game servers, we could have a Linux kernel based firewall using Express Data Path (XDP) for higher performance. Another server hoster called noby had success with this approach, but for us this hasn't helped much yet, probably since we are still out of bandwidth and CPU.

So the next logical step would be not to have this kind of firewall on our rented virtual machines, but as part of the dedicated DoS solutions that the hosters are providing. Unfortunately so far no hoster was willing to add it, except for a large fee. Hosting a more popular online game would make this easier, maybe we should just imitate another game's network packets to benefit from the improved DoS protection support.

Instead of another UDP-based game's network packets we also have a way to communicate using Websockets instead. Potentially Cloudflare or other web-oriented DoS protections could protect that well. Unfortunately Websockets are TCP-based and thus not perfectly suited for a fast-paced game where it makes no sense to resend out-of-date state updates from server to client.

Of course there are proper solutions that offer DoS protection for UDP-based games, such as Cloudflare Gaming or Akamai Gaming, but their costs start at thousands of € per month. Thus we clearly can't afford them.

Instead of technical solutions, we could try working with the police. We know the real name of the main DoS attacker, but there is no way to link him to the attacks from the data we have. Even then, this might not help much. I have previously reported a DoS attacker to the police, since he was kind enough to send a message claiming an attack as his own from his real phone number. But since the attacker was a minor and we are a free online game and thus have no measurable economic damage, the public prosecutor left it at a sternly worded warning.

## The Unknown

I wrote most of this post back in May when we were facing large-scale attacks the last time. Since then not much happened, so I didn't want to bring any unwanted attention to our situation. Now in September the attacks seem to be starting again with a long downtime on all our servers today, so I decided to finish this up and post it anyway. Let's hope that this brings more positive attention than negative. After all, I just showed exactly how good of a DoS target we are.

If you have any suggestions, or dealt with a similar problem before, we'd be interested to hear from you. You can reach me on [dennis@felsin9.de](mailto:dennis@felsin9.de) as well as on the [DDNet Discord](https://ddnet.tw/discord) as deen#5910 or on IRC (deen in #ddnet on Quakenet). I hope that there is something trivial we are missing.

Discussion on [Hacker News](https://news.ycombinator.com/item?id=28675094).
