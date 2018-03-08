# Fixing port forwarding on the ZyXEL C2100Z
CenturyLink's new DSL modem, while reliable enough, is in every other respect
predictably mediocre. It supports some nice stuff like port forwarding, but
forwarded ports randomly become inactive even across modem reboots. And given
that I like my SSH ports to be accessible all the time, I needed a fix.

There are a few obvious options:

1. Enable telnet configuration and issue setup commands from a cron job
2. Replay HTTP commands against the modem's webserver from a cron job
3. Disable NAT altogether and run my own DNS/DHCP/gateway server (which I really
   should do at some point in the near future)

I wasn't able to find any telnet commands to implement (1), and both (2) and (3)
sounded like more work than I wanted to do ... so what else?

## UPnP to the rescue
UPnP lets specific applications claim WAN ports and set up temporary forwarding
rules -- most BitTorrent clients do this, for example. And, of course, there's a
`upnpc` binary for Ubuntu:

```sh
$ sudo apt install miniupnpc
$ upnpc -a 192.168.0.72 22 22 TCP   # 192.168.0.72 is my server's static LAN IP
```

So ... how to overcome the modem's amnesic tendencies? `cron` to the rescue:

```crontab
* * * * * upnpc -a 192.168.0.72 22 22 TCP
```

## Monitoring
This one is easy because the external port is the standard TCP 22. If you run an
SSH server on the Internet, you'll see some number of (hopefully) failed login
attempts from random people every few minutes:

```sh
$ ni /var/log/auth.log r/Invalid/
Mar  8 19:53:14 rorschach sshd[23377]: Invalid user support from 103.89.90.32
Mar  8 19:53:21 rorschach sshd[23395]: Invalid user ubuntu from 54.218.95.167
Mar  8 19:55:14 rorschach sshd[23478]: Invalid user support from 103.207.39.16
Mar  8 19:55:15 rorschach sshd[23481]: Invalid user support from 163.172.192.9
Mar  8 19:55:25 rorschach sshd[23486]: Invalid user support from 103.207.36.36
Mar  8 19:55:26 rorschach sshd[23488]: Invalid user itsupport from 103.207.36.36
Mar  8 19:58:47 rorschach sshd[23668]: Invalid user support from 103.89.90.32
Mar  8 20:03:54 rorschach sshd[23807]: Invalid user support from 103.207.36.36
Mar  8 20:03:56 rorschach sshd[23809]: Invalid user itsupport from 103.207.36.36
Mar  8 20:04:06 rorschach sshd[23821]: Invalid user support from 212.83.179.97
Mar  8 20:04:24 rorschach sshd[23830]: Invalid user support from 103.89.90.32
Mar  8 20:05:15 rorschach sshd[24113]: Invalid user admin from 185.110.132.49
...
```

...and that's a monitoring setup, right there. I figured out that port
forwarding had failed when I noticed that there weren't any invalid records in
the auth logs.
