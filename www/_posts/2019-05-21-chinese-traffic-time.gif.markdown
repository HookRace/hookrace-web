---
layout: post
title: "Chinese Traffic to time.gif"
tags: Haskell Programming
permalink: /blog/chinese-traffic-time.gif/
---

Nearly two years ago [I posted](/blog/time.gif/) this endless GIF that always shows the current time in UTC:

<!--more-->
![time.gif (reload if it fails)](/time.gif)

Now looking at my [GoAccess](https://goaccess.io/) dashboard I can see that it is picking up in popularity rather suddenly:

![GoAccess excerpt](/public/chinese-traffic/goaccess.png)

But strangely I can't find anything about time.gif being linked on the web. So this might just be an attempted Denial of Service (DoS) attack? At least that would be something I am familiar with from the [DDNet](https://ddnet.tw/) direction, but it's certainly strange on HookRace. But instead of simply shutting down time.gif I decided to try to find out who is accessing it and whether I can keep the server up.

Let's look into the [nginx](https://www.nginx.com/) logs, since I use nginx to proxy the requests to the Haskell program. There I see about 40 new requests per second looking like this:

```
hookrace.net 123.185.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 3335 "XXX" "Mozilla/5.0 (Linux; Android 8.1.0; V1818A Build/OPM1.171019.026; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/66.0.3359.126 MQQBrowser/6.2 TBS/044681 Mobile Safari/537.36 MMWEBID/XXX MicroMessenger/7.0.4.1420(0x2700043A) Process/tools NetType/WIFI Language/zh_CN" 8.055
hookrace.net 111.62.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 32061 "XXX" "Mozilla/5.0 (Linux; Android 5.1; OPPO A59s Build/LMY47I; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/66.0.3359.126 MQQBrowser/6.2 TBS/044704 Mobile Safari/537.36 MMWEBID/XXX MicroMessenger/7.0.4.1420(0x2700043B) Process/tools NetType/WIFI Language/zh_CN" 89.256
hookrace.net 111.29.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 543830 "XXX" "Mozilla/5.0 (Linux; Android 7.1.1; OPPO R11 Build/NMF26X; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/66.0.3359.126 MQQBrowser/6.2 TBS/044704 Mobile Safari/537.36 MMWEBID/XXX MicroMessenger/7.0.4.1420(0x2700043A) Process/tools NetType/WIFI Language/zh_CN" 1540.238
hookrace.net 112.2.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 172102 "XXX" "Mozilla/5.0 (Linux; Android 8.1.0; V1816A Build/OPM1.171019.011; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/66.0.3359.126 MQQBrowser/6.2 TBS/044611 Mobile Safari/537.36 MMWEBID/XXX MicroMessenger/7.0.4.1420(0x2700043A) Process/tools NetType/WIFI Language/zh_CN" 492.600
hookrace.net 123.13.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 1275 "XXX" "Mozilla/5.0 (Linux; Android 9; LYA-AL00 Build/HUAWEILYA-AL00L; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/66.0.3359.126 MQQBrowser/6.2 TBS/044704 Mobile Safari/537.36 MMWEBID/XXX MicroMessenger/7.0.4.1420(0x2700043A) Process/tools NetType/WIFI Language/zh_CN" 1.888
hookrace.net 117.91.XXX.XXX - - [21/May/2019:21:21:27 +0200] "GET /time.gif HTTP/2.0" 200 4684 "XXX" "Mozilla/5.0 (iPhone; CPU iPhone OS 12_1_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/16D57 MicroMessenger/7.0.3(0x17000321) NetType/WIFI Language/zh_CN" 12.123
```

I checked a few IP addresses and they were all in mobile networks, not data centers. The user agent containing MicroMessenger and MQQBrowser indicates that the source of the traffic are WeChat and/or QQ, popular chinese chat apps.

## Quantifying the Traffic

For reference, the system I'm running on is a simple [Debian](https://www.debian.org/) based VPS with 2 threads and 2 GB of RAM that also functions as the main server for [DDNet's website](https://ddnet.tw/), database and my HookRace blog.

I already had to do some scaling when posting the [initial blog post](/blog/time.gif/) on [Hacker News](https://news.ycombinator.com/item?id=14996715), optimizing the Haskell application itself to use LZW encoding in the GIF frames, to properly clean up connections to prevent any memory leaks and disable buffering in nginx's config.

But the current level of traffic is on a different scale with 2.4 million hits on time.gif in the last 23 hours (30 hits per second) resulting in 113 GB of data being transferred. And many of those connections don't finish quickly, instead they linger for seconds, minutes or even hours.

Using `lsof -i | grep Time | wc -l` I can see that there are about 6000 people downloading the GIF at peak times, causing up to 30 Mbit/s of outgoing traffic with 7000 packets/second incoming and the same number outgoing. The [DDNet server statistics](https://ddnet.tw/stats/server/) lets me monitor this nicely ([related blog article](/blog/server-statistics/)):

Network
![Network Traffic](/public/chinese-traffic/ddnet-network.png)
Packets
![Network Packets](/public/chinese-traffic/ddnet-packets.png)
CPU
![CPU Usage](/public/chinese-traffic/ddnet-cpu.png)

## Keeping Up with the Traffic

Regenerating the [ranks pages of DDNet](https://ddnet.tw/ranks/) usually causes the main CPU load on the server, which can be seen in the above CPU graph as spikes. This task is already set to only run when the server is below a specified load, so that more essential tasks have priority.

The first new problem was nginx running into a limit of 768 worker\_connections:

```
2019/05/20 20:41:30 [alert] 761#761: *3828093 768 worker_connections are not enough while connecting to upstream, client: 49.114.XXX.XXX, server: hookrace.net, request: "GET /time.gif HTTP/2.0", upstream: "http://127.0.0.1:5002/", host: "hookrace.net", referrer: "XXX"
```

Luckily that is easily fixed in `/etc/nginx/nginx.conf` by increasing the number of worker\_connections to keep alive, each of which is handling one of the long-lasting time.gif requests:

{% highlight nginx %}
events {
  worker_connections 20000;
}
{% endhighlight nginx %}

and `systemctl reload nginx`. No downtime required since nginx will start new worker processes to handle new requests while keeping the old ones alive for a time to keep handling existing connections.

Unfortunately that fix only lasted a few hours until the next problem appeared:

```
2019/05/20 23:09:21 [alert] 15188#15188: *4041619 socket() failed (24: Too many open files) while connecting to upstream, client: 27.207.XXX.XXX, server: hookrace.net, request: "GET /time.gif HTTP/2.0", upstream: "http://127.0.0.1:5002/", host: "hookrace.net", referrer: "XXX"
```

Increasing the limits in `/etc/security/limits.conf` for the nginx user fixes this:

```
#<domain>      <type>  <item>         <value>
nginx          soft    nofile         1048576
nginx          hard    nofile         1048576
```

The value of `1048576` is chosen since it's the value set in `sysctl fs.file-max` and it should be good enough for now.

Next I noticed that the server was running out of memory with both the Haskell application and nginx having to keep track of so many connections at once. For now I increased the swap size on the fly to keep some less commonly used stuff there using `dd if=/dev/zero of=/var/swap bs=1M count=5000 && mkswap /var/swap && swapon /var/swap`.

When running out of memory I noticed that Python's msgpack implementation [fails quite confusingly](https://github.com/msgpack/msgpack-python/issues/239) when it runs OOM. So I had to add some fixes to the code creating the [DDNet ranks pages](https://ddnet.tw/ranks/) to handle this possibility.

The Linux Kernel's TCP buffers ran out of memory next, complaining in dmesg:

```
[1638211.984805] TCP: out of memory -- consider tuning tcp_mem
```

So I increased them with a `net.ipv4.tcp_mem = 116730 155640 233460` in `/etc/sysctl.conf` and reloaded it with `sysctl -p`.

A limitation of my current approach is the number of ports nginx can open to proxy to the Haskell application. If that gets blown I'll have to communicate to the application differently or simply redirect to the application directly instead of proxying it. That would also reduce the CPU load significantly, cutting out nginx which happens to be much more expensive than the Haskell application, probably because it's also handling TLS.

## Final Words

While it was fun to keep time.gif running in the face of this amount of traffic, I still haven't answered the final question of where this traffic is coming from. It might be that lots of Chinese happen to be spreading time.gif on WeChat and QQ, but for that the traffic looks a bit too sterile. Has anyone seen similar traffic patterns and might know if they are real or some kind of botnet? Maybe someone has embedded traffic.gif on some WeChat-specific page. If anyone has a clue please drop me an email at [dennis@felsin9.de](mailto:dennis@felsin9.de).

The best guess so far is by Kolen:

> Hi,
>
> I read your post and this is just my guess:
>
> Chinese “WhatsApp” type of communication culture is very strange. They
> are like spam emails in the old days. Often time some one might create
> posts in picture, eg include warm message such as reminding each other to
> put on more clothes as the weather is getting cold, etc.
>
> My guess is then someone read your trick and thought that it is a good
> idea to create some picture that always show the current time. Eg to
> remind others that it is time to sleep or something.
>
> And like email spams in the old days, people can share things like this
> crazily, often by older people who don’t know much about technology.
> (Just like how some tweet can go viral, messages like this could also go
> viral within there networks. And unfortunately the viral thing we often
> received are really rubbish like.)
> yours,
>
> Kolen

Discuss on [Hacker News](https://news.ycombinator.com/item?id=19978393).
