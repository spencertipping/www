# Open source infrastructure
I made a conscious decision to avoid private (paid) accounts on cloud services
for most things, which means that I run a lot of internal services, typically on
my home servers. I did this both because I'm cheap, and because it's an
adventure. (It isn't cheaper to self-host, but that's not going to stop me.)

**NOTE:** A lot of internal services won't use HTTPS, which means they're not
very secure over wifi. I use an OpenVPN bridge, described below, to provide
secure access to a wired-only network that hosts this stuff.

- [RocketChat (similar to Slack)](#rocketchat)
- [Git server](#hosting-git)
- [He's Dead Jim monitoring](#hes-dead-jim)
- [OpenVPN bridge](#openvpn-bridge)
- **TODO:** collaborative editor (haven't found one I like yet)

## Rocket.Chat
![image](http://storage4.static.itmages.com/i/18/0323/h_1521770288_5272460_4de2ffb5cd.png)

This is nothing short of amazing. A lot of companies use Slack, and this is an
open-source chat system that is convincingly similar, trivial to deploy, and
quite fast. Here's how I set up
[dev.spencertipping.com](https://dev.spencertipping.com), with HTTPS, on an EC2
micro instance:

```sh
$ sudo snap install rocketchat-server
$ sudo snap run rocketchat-server.initcaddy
$ sudo vim /var/snap/rocketchat-server/current/Caddyfile    # two line change
$ sudo systemctl restart snap.rocketchat-server.rocketchat-caddy
```

...and that's it -- one DNS record and it was done. Caddy, bundled with the
rocketchat server snap, sets up a free SSL certificate for any site with public
DNS, and it also handles transparent HTTP proxy. Those four commands above are
all I did on the EC2 instance, and it runs quite fast in about 500MB of memory.
(I tried it on a t2.nano, but it's just too much; t2.micro fits the bill.)

Rocket chat has support for webhooks, so I can have channels [like this
one](https://dev.spencertipping.com/channel/ni-ci) that integrate with Travis CI
builds and let me know when I've broken stuff.

I also run an instance on our internal network so Joyce and I can collaborate
more efficiently. The internal one has webhooks for Gitlab CI, which after a bit
of trivial customization works great.

Backing it up is trivial; here's the cronjob I use:

```sh
# on the rocketchat server
$ sudo snap run rocketchat-server.backupdb

# on the backup server
$ rsync -a rocketchat-server:/var/snap/rocketchat-server/current/backup.tgz \
           $backup_dir/`date -u +%Y.%m%d.%H00.tgz`
```

It seems to be fairly efficient; right now the latest backup is 10MB, and Joyce
and I have about 30 channels with some uploads and ~1k messages. I haven't yet
dug into the backup format, but at a glance it looks like a straight Mongo
export of BSON files.

### Deploying Hubot
Our internal rocket chat also has a Hubot on it; here's the deployment script I
use:

```bash
#!/bin/bash
# Runs a detached hubot instance.

modules=(
  hubot-pugme
  hubot-help
  hubot-seen
  hubot-links
  hubot-diagnostics
  hubot-bookmark
  hubot-maps
  hubot-reminders
  hubot-ambush
  hubot-calculator
  hubot-http-status
)

join_by() { local IFS="$1"; shift; echo "$*"; }

docker run --detach --name=rocket-hubot \
           -e ROCKETCHAT_URL=http://10.35.0.3:8380 \
           -e ROCKETCHAT_ROOM='' \
           -e LISTEN_ON_ALL_PUBLIC=true \
           -e ROCKETCHAT_USER=bot \
           -e ROCKETCHAT_PASSWORD=bot \
           -e ROCKETCHAT_AUTH=password \
           -e BOT_NAME=rocketbot \
           -e EXTERNAL_SCRIPTS=$(join_by , "${modules[@]}") \
           rocketchat/hubot-rocketchat
```

**TODO:** write some custom hubot extensions and then brag about them here

## Hosting Git
I've tried two options, [Gogs](https://gogs.io/) and
[Gitlab](https://about.gitlab.com/). Both are excellent:

- If you have a lot of memory (>8GB) and want CI and webhooks, use Gitlab
- If you want something really fast+lean, use Gogs

### Gitlab
![image](http://storage8.static.itmages.com/i/18/0322/h_1521731938_5668075_2dce88950a.png)

Gitlab is awesome. It comes with a bunch of machinery for user account
management, issue tracking, pretty much the works -- and one of my favorite
things, [built-in continuous
integration](https://about.gitlab.com/features/gitlab-ci-cd/). I had initially
assumed that the builtin CI system would be a stopgap for people who hadn't yet
upgraded to something like Travis or Circle, but it's actually quite nice; it's
built to support Docker out of the gate, and you can run multiple "runner"
containers on multiple machines to get build parallelism. Overall I prefer it to
Travis -- I haven't tried Circle CI yet.

#### Deploying it
I used the `gitlab/gitlab-ce:latest` public Docker image, then bound three ports
and a couple of mount points:

```sh
$ docker run --detach \
  --publish 8143:443 --publish 8180:8180 --publish 8122:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
```

This setup has a few minor issues, the worst being that Gitlab's clone URLs
aren't correct because it thinks it's running on the system SSH port. I have
multiple services running on the same physical machine, so I didn't want to have
something like Gitlab monopolizing the real port 22. The result is that I clone
repos like this:

```sh
$ git clone ssh://git@server:8122/spencertipping/reponame   # internal gitlab
$ git clone git@github.com:spencertipping/reponame          # ...vs github
```

Backing it up is also nontrivial due to the way it manages directory
permissions. I ended up with a root cron job that repermissions the files to
create a user-readable snapshot:

```sh
umask 077
{ ionice -c3 -n7 rsync -a /srv $backup_file \
    && date +%s > $backup_file/backup-date \
    && chown backup-user $backup_file -R; } 2>$backup_file.log
```

...so not great, but it does work. Overall, Gitlab is easily worth it.

### TODO: document Gogs
It's basically a single Docker container deployment as well, and it walks you
through the setup process if I remember correctly.

## He's Dead, Jim
![image](http://storage6.static.itmages.com/i/18/0323/h_1521771741_5605635_c6f05d26e2.png)

I didn't want to learn how to use a real monitoring system, and wanted something
dead easy to deploy. So I wrote up a really simple one called [He's Dead
Jim](https://github.com/spencertipping/hesdeadjim), which is basically a tiny
webserver written in Perl (which contains an HTML+JS app in a heredoc), and a
monitoring script that gets run every minute from a cronjob.

This really is the bare minimum for monitoring. I have to use Firefox because it
pops up desktop notifications on failure, and Chrome started blocking all
notifications from non-HTTPS pages. [Here's what the monitoring script looks
like](https://github.com/spencertipping/hesdeadjim/blob/master/deadjim).

One fun aspect of this: my modem has unreliable port forwarding, and I want to
monitor external access for SSH so I can get to my stuff when I'm away from
home. It turns out that bogus SSH login attempts to random IPs are so regular
that I can bind port 22 as usual, and just [assert that one has happened within
the last 15
minutes](https://github.com/spencertipping/hesdeadjim/blob/master/deadjim#L168).
So far it has never triggered a false positive.

**TODO:** finish the SSH auth log writeup

## OpenVPN bridge
**TODO:** document this better

I want uniform access to my internal servers, which all have `10.35.0.X`
addresses. More specifically, I want to be able to use that address range
whether I'm on the home network or somewhere else, and I want proper VPN-level
security over wifi. So here's what I ended up with.

There are two networks on my LAN, `192.168.0.X` for DHCP wifi (managed by the
router), and `10.35.X.X` as a wired-only network with no direct access. One
server has interfaces on both and acts as a gateway, but doesn't allow access
inwards.

So if I SSH into the server, I can see `10.35.X.X` from there, but there aren't
any public routes that would get you there. With this configuration alone, I
could use [sshuttle](https://github.com/apenwarr/sshuttle) to get to the
internal network -- and did for about a year before switching to OpenVPN.

Getting OpenVPN to act as a bridge is a little nontrivial, and the whole
`easy-rsa` stuff is a huge mess. I ended up dockerizing the OpenVPN server and
binding it to the host network, then wrote some scripts to autogenerate client
connection profiles in self-contained Perl bundles. Here's a basic sketch of
what this looks like:

### `Dockerfile`
```
FROM ubuntu

RUN apt-get -y update \
 && apt-get -y install openvpn easy-rsa bridge-utils \
 && make-cadir /etc/openvpn/easy-rsa

ADD openvpn-setup /root/
RUN /root/openvpn-setup

ADD openvpn-down.sh openvpn-up.sh server.conf client.template /etc/openvpn/
RUN chmod +x /etc/openvpn/*.sh

ADD host-keys hosts /root/
RUN /root/host-keys

EXPOSE 1194/udp

CMD cd /etc/openvpn && openvpn --config server.conf
```

### `server.conf`
```
mode server
tls-server

keepalive 10 120

tun-mtu  1378

local 192.168.0.72    # ip/hostname of server
port 1194             # default openvpn port
proto udp

cipher aes-256-cbc

dev tap0              # If you need multiple tap devices, add them here
script-security 2     # allow calling up.sh and down.sh
up   "/etc/openvpn/openvpn-up.sh   br0 tap0 1500"
down "/etc/openvpn/openvpn-down.sh br0 tap0"

persist-key
persist-tun
duplicate-cn

client-to-client

ca ca.crt
cert server.crt
key server.key        # This file should be kept secret
dh dh2048.pem
tls-auth ta.key 0     # This file is secret

# DHCP
ifconfig-pool-persist ipp.txt
server-bridge 10.35.1.1 255.255.0.0 10.35.1.2 10.35.1.254
max-clients 100
```

### `client.conf.template`
```
client

proto udp
remote rorschach 1194
tun-mtu  1378

cipher aes-256-cbc

ca ca.crt
cert CLIENT.crt
key CLIENT.key
tls-auth ta.key 1

dev tap
nobind
float
```

### `run` (to build the image + run the container)
```bash
#!/bin/bash

if [[ $1 == --debug ]]; then
  docker build .
  image=`docker build -q .`

  docker run --cap-add=NET_ADMIN --device=/dev/net/tun \
             --rm --network=host $image
else
  image=$(docker build -q .)
  container=$(docker run --cap-add=NET_ADMIN --device=/dev/net/tun \
                         --detach --restart=always --network=host \
                         --name openvpn $image)
  echo $container

  rm -rf clients
  mkdir clients
  cd clients
  for h in $(<../hosts); do
    docker cp $container:/etc/openvpn/clients/$h.tgz .
    tar -xzf $h.tgz
    sed -r "s/CLIENT/$h/g" ../client.template > $h.conf

    # Now bundle it all into a perl script that unpacks its arguments and
    # supports custom hostnames (e.g. for connecting over a public network).
    ../make-runner $h > $h
    chmod +x $h
  done
fi
```

### ...and my personal favorite, `make-runner`
```bash
#!/bin/bash
# Usage: ./make-runner host > runner
#        ./runner [server-hostname]
#
# Generates a self-contained openvpn connector.

cd "$(dirname "$0")"

host=$1

ta_key=`     perl -e 'print unpack "H*", join"", <>' < clients/ta.key`
ca_crt=`     perl -e 'print unpack "H*", join"", <>' < clients/ca.crt`
client_crt=` perl -e 'print unpack "H*", join"", <>' < clients/$host.crt`
client_key=` perl -e 'print unpack "H*", join"", <>' < clients/$host.key`
client_conf=`perl -e 'print unpack "H*", join"", <>' < clients/$host.conf`

cat <<EOF
#!/bin/bash
cd "\$(mktemp -d)" || exit \$?
chmod 0700 .

server=\${1:-rorschach}
perl -e 'print pack "H*", "$ta_key"' > ta.key
perl -e 'print pack "H*", "$ca_crt"' > ca.crt
perl -e 'print pack "H*", "$client_crt"' > $host.crt
perl -e 'print pack "H*", "$client_key"' > $host.key
perl -e 'print pack "H*", "$client_conf"' | sed s/rorschach/\$server/g > conf

sudo openvpn --config conf

rm -r "\$PWD"
EOF
```

This creates a script that bundles all of the openvpn configs into Perl string
constants, then unpacks them into a tempdir:

```bash
#!/bin/bash
cd "$(mktemp -d)" || exit $?
chmod 0700 .

server=${1:-rorschach}
perl -e 'print pack "H*", "<hex-digits>"' > ta.key
perl -e 'print pack "H*", "<hex-digits>"' > ca.crt
perl -e 'print pack "H*", "<hex-digits>"' > iniidae.crt
perl -e 'print pack "H*", "<hex-digits>"' > iniidae.key
perl -e 'print pack "H*", "<hex-digits>"' | sed s/rorschach/$server/g > conf

sudo openvpn --config conf

rm -r "$PWD"
```
