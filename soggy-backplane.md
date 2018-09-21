# Recovering a soggy SAS backplane
Although great from a space-efficiency point of view, the laundry-room data
center has the unfortunate downside of mixing plumbing and electronics. I became
aware of one such mishap when four of the 12 RAID-6 drives failed, resulting in
an unusable array and a server that wouldn't stop beeping. So I powered it down,
removed the drives, and did some investigating.

## Initial state: server works, four drive slots don't
At first I thought the water might have just bridged some pins or something, so
after waiting a bit I plugged the drives back in and powered it up. The drives
themselves seemed fine; I could switch them around and the error lights didn't
follow them. Instead, error states were fixed to four RAID slots: three on the
left, and the bottom-most bay on the second from the left. This was pretty
consistent over several reboots, including a day later.

## Opening up the server
I started by removing the drives and fans:

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_101533.jpg)

Through the plastic the backplane looked pretty good. I could see a little bit
of water corrosion but nothing too major:

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_101632.jpg)

The board came out through the front of the server once I had removed the
plastic hardware to hold the drive trays in place.

## Cleaning the board
![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_102111.jpg)

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_103623.jpg)

You're supposed to put circuitboards in the dishwasher to clean them, but Joyce
was running some dishes and I didn't want to wait. So I ran it under a hot
shower -- hot because the water softener applies only to the hot line, and I
figured fewer minerals == better outcome. This was just a pre-clean though,
mostly to get the worst of the dust off.

Next up, Serious Power Tools: isopropyl alcohol and q-tips.

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_104801.jpg)

It was reassuring knowing that I was using the _ultimate_ beauty tool. I suspect
had I gone with the penultimate beauty tool, the outcome would have been
measurably worse.

### Dissolving the corrosive salt (warning: unhappy hardware)
I found out while reading about this that water forms a corrosive salt on
electronics. It slowly dissolves existing conductors and, I assume, is somewhat
conductive itself -- so it's pretty bad stuff, and you need something like
isopropyl alcohol to get rid of it. Here's the board under the microscope before
cleaning:

![image](http://spencertipping.com/soggy-backplane-images/my_photo-4.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-6.jpg)

I had to do some scrubbing on a few of these chips. Reassembling the server to
test took probably ten minutes anyway, so it was worth spending the time to make
sure everything got cleaned up correctly. I think I spent about an hour going
over all of the components with a microscope and applying ultimate
beautification.

Before/after:

![image](http://spencertipping.com/soggy-backplane-images/my_photo-67.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-68.jpg)

One interesting thing about these photos is that the five pins on the bottom
side of U3 are all commoned together, yet there was still a lot of corrosion.
This surprised me because I had assumed the corrosion came from current flowing
through stuff -- but it seems like it's just a result of water sitting on the
pins.

### An hour later: signs of life
Mostly-full reassembly, power up, and...

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_122122.jpg)

Not bad! Three of the four faults cleared. I took everything back apart and
scrubbed some more. I also got a meter and tested continuity to make sure there
weren't any unexpected bridges.

![image](http://spencertipping.com/soggy-backplane-images/my_photo-170.jpg)

There wasn't a smoking gun anywhere, but I made sure all of the components were
properly soaked this time around.

![image](http://spencertipping.com/soggy-backplane-images/my_photo-183.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-184.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-185.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-186.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-187.jpg)

I suspect it may have been this transistor, which had a bit of salt on the two
connections:

![image](http://spencertipping.com/soggy-backplane-images/my_photo-191.jpg)

![image](http://spencertipping.com/soggy-backplane-images/my_photo-192.jpg)

## Everything works
Everything back together again, power up, and all 12 drives were back online and
functioning. Next up was the RAID recovery:

```sh
# I should have run this; instead I did something worse
$ sudo mdadm --assemble --run --force --update=resync /dev/md0 /dev/sd[a-l]
```

Before I re-racked the machine, I wanted to apply some waterproofing to the
case. The water had gotten in through the seam in the metalwork, so some duct
tape was in order:

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_135850.jpg)

![image](http://spencertipping.com/soggy-backplane-images/IMG_20180921_140049.jpg)

...and that's it! I really didn't expect to ever see the machine function again,
but it turned out to be a pretty easy fix.
