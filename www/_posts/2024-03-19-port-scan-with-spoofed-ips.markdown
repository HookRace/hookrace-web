---
layout: post
title: "Port Scan with Spoofed IP Addresses"
tags: DDNet
permalink: /blog/port-scan-with-spoofed-ip-addresses/
---

Or how to make naïve hosters shut down your victim's server.
<!--more-->

Do you want to bring down your victim's game servers? Do they host their game servers on cheap VPSes at hosters who are not that experienced with networking? Is running an actual DoS against the server too expensive and illegal for you?

Just grab a server for yourself from a questionable hoster with IP address spoofing allowed on their network. Next run a port scan using `nmap` ([man page](https://linux.die.net/man/1/nmap)) with a spoofed source address (`-S` or `-D` for multiple decoys) using your victim's IP address(es). As the target of the port scan choose a university with a cybersecurity department that likes to send security incident reports to abuse mail addresses. To the target university of the port scan it will look like the scan came from your victim, since you are spoofing their IP address, so they will send a serious sounding email to the victim's hoster:

[![Security incident report](/public/security-incident-report.png)](https://reports.csirt.muni.cz/getReport.php?uuid=66250B82-E48F-11EE-AA82-0983312413ED)

As a consequence your victim's VPS will most likely be shut down for a few hours or even days. To prevent this your victim will try to block all of their incoming and outgoing TCP traffic on their VPS, and also block the reporting university's IP range entirely:

```
$ iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
DROP       all  --  147.251.0.0/16       anywhere
DROP       tcp  --  anywhere             anywhere             tcp ctstate NEW multiport dports  !27685,6546,ssh

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       all  --  anywhere             147.251.0.0/16
```

And yet you can rerun your port scan against the university's network every day, which will trigger automatic abuse mails to the victim's hoster, who will still trust these abuse mails more than the victim, since the hoster is unlikely to be able to monitor the network traffic of the VPS to check if those packets actually originated there. At some point the hoster will be so annoyed that they simply terminate the VPS and throw out the customer, victory!

Congratulations, you have brought down some small-fish game servers in the [DDNet](https://ddnet.org/) community and made them lose the remaining money they had already put into the hoster! Now you can collect the disgruntled players on your own servers, post your freedom-fighting messages on Discord, and feel proud of having some hard-earned power!

In the next post we will continue in this vein and show you the equally satisfying techniques to steal candy from a baby.

P.S.: If anyone has better suggestions than manually sending the same explanations and this link to my server hosters every time they get such an abuse report, please tell me.
