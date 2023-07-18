---
layout: post
title: "Update on DoS Attacks against our Online Game"
tags: DDNet
permalink: /blog/dos-attacks-update/
---

In my [previous post](/blog/dos-attacks-against-online-game/) 8 months ago I described how our [open source](https://github.com/ddnet/ddnet) online game [DDraceNetwork](https://ddnet.org/) has been suffering under DoS attacks for about 8 years, basically since its inception. Recently the attacks have gotten much worse, forcing us to work on further approaches. Since many players made suggestions recently, I'm writing this blog post to summarize what we are attempting and to ask for help again.

<!--more-->

![Netherlands](/public/nld.ddnet.tw-net-7d.png)
![Germany](/public/ger2.ddnet.tw-net-49d.png)

These traffic graphs are from two of the servers we are running, note the logarithmic x-axis. Each spike represents an incoming DoS attack, as you can see some of them last for nearly a day.

Recently the attacks have been relatively weak in terms of incoming bandwidth, using spoofed IP addresses imitating our UDP-based connection process.

At first the CPU gets overloaded since the server suddenly has to try and handle hundreds of thousands of connection attempts per second. Since our game servers are mostly running on cheap VPSes and each game server runs single-threaded, it is quite easy to overload a system in this manner.

## HTTPS-based Whitelist
To prevent spoofing we collect all players' IP addresses and whitelist those. Since we also develop the game client, we can modify the client to connect to a server via HTTPs for this whitelisting. The iptables rules and [ipset](https://ipset.netfilter.org/) setup on the game servers for this whitelist look something like this:

{% highlight bash %}
ipset create official iphash
ipset create whitelist-ip iphash
iptables -N serverinfo
iptables -A serverinfo -m hashlimit --hashlimit-above 40/s --hashlimit-mode dstport --hashlimit-name si_dstport -j DROP
iptables -N game
iptables -A game -m set --match-set official src -j ACCEPT
iptables -A game -m set --match-set whitelist-ip src -m u32 --u32 "38=0x67696533" -j serverinfo
iptables -A game -m set --match-set whitelist-ip src -m u32 --u32 "38=0x66737464" -j serverinfo
iptables -A game -m set --match-set whitelist-ip src -j ACCEPT
# Still allow non-whitelisted players when there is no attack
iptables -A game -m limit --limit 10000 -j ACCEPT
iptables -A game -j DROP
iptables -A INPUT -p udp -m udp --dport 8000:9000 -j game
{% endhighlight %}

To keep updating the whitelist we run a simple script:
{% highlight bash %}
#!/bin/sh
rm -f whitelist whitelist.old
touch whitelist.old
while true; do
  curl -o whitelist $WHITELIST_URL
  LC_ALL=C comm -23 whitelist whitelist.old | while read line; do
    ipset add whitelist-ip $line
  done
  mv whitelist whitelist.old
  sleep 20
done
{% endhighlight %}
About 20 seconds after starting DDNet client the players are now being whitelisted and can join servers which are under a spoofed DoS attack. We are employing this on the servers which are commonly attacked.

After a few days the ipset runs out of space unfortunately. Since we have ~25k unique players daily, the default maximum element size of 65536 of `ipset` is quite tight. To work around this we'll have to increase the limit or remove older entries automatically.

In order to make this work reliably we had to disable IPv6 for the HTTPS access, since our game servers are IPv4 only at the moment and collecting IPv6 addresses doesn't help.

An unfortunate realization was that some ISPs don't provide fixed IPv4 addresses anymore, but instead tunnel their native IPv6 traffic through different IPv4 addresses, based on protocol and port. So we get one IP address in the HTTPS-based whitelist, but another when connecting to the gameserver via UDP. At least one Israeli ISP is doing this.

## Dedicated Servers
For some of the servers this whitelisting is not enough, since the hosters' DoS protection kicks in and starts aggressively blocking UDP traffic. This includes legitimate traffic and results in all players timing out on the game servers. I have tried talking to a few of the hosters about this, but it is by design since they don't support our game protocol, and wish to protect their own infrastructure as well as other customers. At least they are not kicking us out for now, which has happened in the past with a few hosters after multiple DoS attacks against us impacting their other customers.

So for a few locations we have upgraded to dedicated servers, where the hoster cares less about us exhausting in the incoming network connection and our CPUs for hours at a time. Unfortunately it is difficult to find hosters that provide more than 1 Gbit/s network bandwidth and still fall into our budget of 100 € / month.

The attackers of course notice that these servers can't be downed as easily, so they switch to different kinds of attacks, exhausting our measly 2 Gbit/s network.

## Small Hosting Companies
Some small hosting companies in Europe offered to help us by providing a free server with custom DoS protection. We accepted two such offers, but didn't have much success with either of them.

For the first we never managed to get legitimate traffic to get through their firewall.

For the second everything seemed to work fine when there was no attack, but during DoS attacks some player packets would get misflagged all the time. Additionally the network stack had problems and caused our MariaDB-based SQL connection to time out. Unfortunately we didn't handle this error properly in our code and ended up losing legitimate player ranks.

In the end we didn't continue using either of these hosters, but spent a few hours trying to set up the firewall with them. I had similar experiences with every paid hoster which offered a custom DoS protection.

## DoS Protection Companies
There are of course also larger companies providing custom DoS protection for UDP-based applications. One of them offered to help us out for free with our problem, and I've been in touch with them since the last post. Unfortunately everything is moving very slowly, and many of my requests to move things forward over the weeks have not been answered, so I guess the fact that we are unable to pay thousands of € per month matters after all. So far no custom DoS protection is available here, but I'll keep pinging them every few weeks.

## Proxy Servers
Instead of exposing our real game servers directly to the players, for the future we are considering a [proxy server](https://github.com/ddnet/ddnet/pull/4791) that sits inbetween the game server and player. This would allow us to run multiple proxies for subsets of the players and attempt to isolate the impact the attacker can have.

This might also allow us to proxy Steam players' traffic through [Steam Datagram Relay](https://partner.steamgames.com/doc/features/multiplayer/steamdatagramrelay) (SDR), although more work would be required to implement that. Someone from Valve reached out to us after the previous post, and we might follow up on this once the proxy server implementation is ready. It is really cool that Valve is offering help here, even for a free game like ours.

We can't entirely switch to SDR through Steam's servers since we want DDNet to stay playable as an open source game. As a solution we can provide a regular open source proxy for open source players, and a separate SDR-based proxy for Steam players.

## IPv6 Servers
A longshot idea is using IPv6-only servers. We haven't tried this yet, maybe the IP spoofing capabilities of our attackers are lower on IPv6. This could also be combined well with the proxy servers by providing an additional ipv6-only proxy server.

Running all these proxy servers for each location will definitely add cost, but it's probably our best path forward right now. The setup would look something like this for each Server location:
![Graph](/public/proxy.png)
{% highlight dot %}
digraph D {
  node [shape="box"];
  "OS Player 1" -> "IPv4 Proxy" -> "Game Server";
  "OS Player 2" -> "IPv4 Proxy";
  "OS Player 3" -> "IPv6 Proxy" -> "Game Server";
  "OS Player 4" -> "IPv6 Proxy";
  "Steam Player 1" -> "SDR Proxy" -> "Game Server";
  "Steam Player 2" -> "SDR Proxy";
}
{% endhighlight %}

## Conclusion
If you have any suggestions, or dealt with a similar problem before, we'd be interested to hear from you. You can reach me on [dennis@felsing.org](mailto:dennis@felsing.org) as well as on the [DDNet Discord](https://ddnet.org/discord) as deen#5910 or on IRC (deen in #ddnet on Quakenet).
