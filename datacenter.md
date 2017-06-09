# The cheap home datacenter
![image](http://storage9.static.itmages.com/i/17/0608/h_1496922906_2417586_86eea04b33.jpeg)

About a year ago I searched for "rackmount servers" on ebay, mostly out of
curiosity. I was surprised to find that you can get a used server for next to
nothing; in short order I found an HP unit with 12 Xeons, 144GB of memory, and
four 300GB disks for $500 + $70 shipping. Thus began the Cheap Home Datacenter,
which has grown a bit but is still providing all of my compute power for
various projects and heating our laundry room.

## Why run your own servers?
Two big reasons. First, the learning experience. If you rent an EC2 instance
(which I also do), Amazon takes care of sysops for you -- but where's the fun
in that? Running your own servers means you're thinking about amps, cooling,
cable organization, RAID controllers, network performance, and where to buy
hard drives.

Second, it might be cheaper than EC2 or similar even considering the cost of
power. It's almost certainly cheaper if you're transferring tons of data; most
ISPs cap or complain if you download more than a few hundred GB of data per
month. We pay about $0.17/kWh and the server's power supply is 750W, so that
comes up to:

```sh
$ units -t '750W * 0.17USD/(kW*hour)' USD/month         # worst case
93.136761
```

That sounds like a lot, but EC2 prices for similar hardware are much higher per
hour (there are cheaper ones down the page, but anything with >100GB of memory
is over $1/hour):

![image](http://storage9.static.itmages.com/i/17/0608/h_1496924691_3768431_b3128d7946.png)

If you ran the `r4.4xlarge`, the cheapest >100GB machine, for a month you'd
have a much higher bill:

```sh
$ units -t '1.064 USD/hour' USD/month
777.2354
```

The other thing is that the power supply is rated at 750W, but most of the time
the server isn't using that much. For example, I've got the CPUs pegged with a
load average of just over 300:

![image](http://storage4.static.itmages.com/i/17/0608/h_1496927204_3515236_ae20e029b5.png)

The overall system wattage, though, is quite low -- just 320W:

![image](http://storage1.static.itmages.com/i/17/0608/h_1496927337_6812679_620fcb85b1.png)

```sh
# revised power cost
$ units -t '320W * 0.17USD/(kW*hour)' USD/month
39.738351
```

## Practical considerations: amps, heat, and noise
The most inflexible of these is probably amps. I'm currently running two
servers and would like to add a third, but given that they're sharing a circuit
with a managed switch, a freezer, and a washing machine it's likely I'll
overload the breaker. Most breakers will give you a little time before cutting
the power -- useful for inductive loads like freezer compressors, which draw a
bunch of power at startup and then taper off. But if your breakers are fast you
have to be especially careful.

(A possibility I haven't tried yet is to run the servers from the 220 outlet
powering the dryer. That would double the wattage per amp, but might overload
the dryer circuit and would probably require some wiring changes.)

If you're just running one or two servers, though, you're unlikely to overload
stuff. Any power issues you run into are probably going to be caused by
[homemade chaos
monkeys](https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey).

![image](http://storage2.static.itmages.com/i/17/0608/h_1496928747_9496741_92706f5d35.jpeg)

### Dealing with heat
Whatever heat your server generates is probably fine if it has decent airflow.
Our laundry room is about 100F, higher than recommended for servers, and I've
never had a thermal shutoff. I have gone through a few iterations of airflow
management, though, because I've got the servers in a fairly constrained space.
The right answer would be to build a rack for them but I haven't gotten around
to it yet.

I found a hotspot, which unsurprisingly was directly above the servers, and
then put a furnace blower there. It blows the air across the room, which
increases the temperature of the ceiling -- and that's useful because heat
transfer is proportional to temperature difference x area. This causes the
overall room temperature to be lower and prevents heat build-up around the
intake vents.

![image](http://storage5.static.itmages.com/i/17/0608/h_1496929603_2204802_39c52450bd.jpeg)

Something I didn't expect is that once I added the blower, every adjacent room
also got warmer because interior walls aren't insulated. So our guest bath is
about 90F most of the time. (On the upside, this is good news for cooling
because it means more area per watt.)

### Dealing with noise
The main server is an HP DL380 G6, and it's mostly silent unless the fans are
running. Once they are, there are about ten different levels -- but the ones
near the top are serious business; I just measured the noise levels at 78dBA
using a free app that [might be reasonably
accurate](https://blogs.cdc.gov/niosh-science-blog/2014/04/09/sound-apps/).

So although it's not to the level of causing hearing loss, you don't want a
server running in your living room. Having it behind one closed door drops the
noise to ~48dB and behind two closed doors it's nearly inaudible at ~27dB.

## Procuring the hardware
There are a surprising number of rackmount servers on ebay all the time; I
probably spent a couple of weeks total intermittently scouting for my two
servers. The one I mentioned above is mostly for compute, and the second one is
for storage; I got it for $330 shipped and it came with 4x 3TB drives.

**Uh oh:** Literally just a few minutes ago the power flickered (first time
this year) and I lost one of the external RAID-5 drives -- a Seagate 4TB that
was less than a year old I think. Everything still works but I'm running on a
degraded array, so I might as well fix this by getting a proper storage server
and offloading the data. I'm gonna hijack the narrative since I'm not yet sure
what to do next.

Also, quick aside since my compute job got killed: the server uses 138W idle so
the baseline cost is $17.13/month.

### Recovery time: fixing the storage situation
Right now I've got the following:

- The compute server: 4x 2.5" 300GB drives
- The storage server: 4x 3.5" 3TB drives
- The (now degraded) RAID5 array: 3x (now 2x) 4TB USB drives

The two servers are linked by a bonded 2xGbit connection running NFS and gets
about 240MB/s. I would add more links, but the storage server just has two.

The RAID array was built from random disks we had lying around and holds
backups of things since it's about 10x slower than everything else. It's been
on my todo list to get a real storage server instead of using the MacGyver
solution, but until now I haven't had a good reason. Now any disk failure means
real data loss -- so time to head over to ebay.

#### The options
Storage servers tend to come in a few different configurations:

- Lots of 2.5" bays: cheaper and lower power
- Lots of 3.5" bays: expensive, and a lot of these have backplanes limited to
  2TB drives
- Not very many bays: like my current storage server -- the problem with these
  is that RAID-5 or RAID-6 is inefficient, and proportionally a lot of the
  power ends up going to the CPU and non-disk stuff.

Two big questions right now:

1. What's the total cost of ownership of a 2.5" drive vs a 3.5" drive?
2. How long does it take for the extra power of 2x 2TB 3.5" drives to catch up
   to the price premium of 1x 3TB or 1x 4TB?

Some basic price points for new hard drives (I assume used isn't worth the
reliability loss, though I have no data to back that up):

- 2TB 3.5" Hitachi = $50, ~8W (6W idle according to [this article](http://www.tomshardware.com/forum/267776-32-hard-drive-power-consumption))
- 3TB 3.5" Hitachi = $70, ~8W
- 4TB 3.5" Hitachi = $115, ~8W

- 2TB 2.5" Seagate = $76, ~4W
- 3TB 2.5" Seagate = $140, ~4W?
- 4TB 2.5" Seagate = $115, ~3.5W

The 2.5" drives tend to be for laptops, but they should work fine for servers.
The reviews are pretty sketchy, though:

![image](http://pix.toile-libre.org/upload/original/1496945842.png)

Before I dig around for a more reputable source of 2.5" drives, let's talk
about power cost for 3.5" units. Here are some configurations and costs,
assuming $0.17/kWh and a 2% annual drive failure rate:

- 24TB net RAID-6:  8x 3.5" 4TB = 64W  = $920 + ($7.94  power + $2.19 fail)/mo
- 24TB net RAID-6: 10x 3.5" 3TB = 80W  = $700 + ($9.93  power + $1.67 fail)/mo
- 24TB net RAID-6: 14x 3.5" 2TB = 112W = $700 + ($13.91 power + $1.67 fail)/mo

The 2TB configuration appears to make no sense since it carries a $2/TB/year
power cost, but a 2TB-limited 36 bay SuperMicro server is $435 shipped whereas
a similar unit with an unlimited backplane goes for about $1040. That $600
covers 12 years of the extra power at 24TB, ~6 years at 48TB.

**Update:** Just found a 12x3.5" 2TB server for $580 with free shipping --
that's cheaper than the baseline cost of the drives, although they're used and
some might not work. It's a decent deal whether or not the drives are in good
shape. ...so once that gets in, this writeup might change a bit.

### Ok, back to the hardware stuff: networking gear
You can't get 2x throughput on bonded links using an unmanaged gigabit switch,
and you probably don't want to live with 125MB/s NFS IO. To get 250MB/s, you
need either two unmanaged switches or one managed switch with vlan support.
[This
post](http://louwrentius.com/achieving-450-mbs-network-file-transfers-using-linux-bonding.html)
explains the technique involved.

I use a Dell PowerConnect 5324, and the going rate appears to be about $50 on
ebay. You'll also need a USB-serial connector to configure it; once that's
plugged in you should be able to run `screen /dev/ttyUSB0 9600` or similar to
log into the switch's console.

One of the nice things about having vlans is that you can create secure
networks within the same hardware switch; right now I've got a 10.x network for
internal stuff like NFS, and it's separated from the 192.168.X network shared
with the wifi. (I use [sshuttle](https://github.com/apenwarr/sshuttle) to
connect wifi clients to the internal network.)
