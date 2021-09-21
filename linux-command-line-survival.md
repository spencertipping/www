# Linux command-line survival guide
A probably-opinionated list of things I've found useful in 20 years of using Linux.

`ni` is something I wrote which you can get [here](https://github.com/spencertipping/ni). I tend to use it a lot for sysops and data work, and I've mentioned it inline with other commands when it's especially useful for something. (It's got a steep learning curve, so I consider it a last resort for a page like this.)


## Linux documentation
+ `man ls`: show manpage for `ls`
+ `info ls`: show infopage for `ls` (like `man`, but different)
+ `apropos 'kernel module'`: list manpages with "kernel module" in the description
+ [Manpage sections](https://www.commandlinux.com/man-pages-sections)
  + `man 1 stat`: show manpage for command-line tool `stat`
  + `man 2 stat`: show manpage for kernel API call `stat`
  + `man 3 stat`: show manpage `libc` API call `stat`
  + `man 3p stat`: show POSIX definition for `stat`
  + `man 4 sd`: show manpage for devices like `/dev/sda`
  + `man 5 resolv.conf`: show manpage for `/etc/resolv.conf` format
  + `man 7 tcp`: manpage for TCP protocol and APIs
  + `man 8 wg`: manpage for WireGuard config tool
+ To list all your manpages: `ls /usr/share/man/man1` etc (one directory per section)
  + ...and to find more: `apt list *manpages*`


## Shell scripting survival
I'll try to put together a short version at some point, but for now here's the [bash scripting cheatsheet](https://devhints.io/bash), which covers a lot of ground.


## Terminal and serial IO
### Pagers
+ Ideal solutions
  + `less`
  + `more`
+ Ad-hoc
  + `command > file & tail -f file`
  + `command | while : ; do head -n10; read line <&2; done`
  + `command | vim -` (reads the full input before showing you anything)


### Persistence
+ `tmux` (see [bashrc-tmux](https://github.com/spencertipping/bashrc-tmux) for auto-tmux-over-SSH)
+ `screen`
+ `byobu`: a frontend for `tmux` (previously for `screen`)
+ `reptyr`: move a running program from one TTY to another (useful if you started something outside of `screen` and want to migrate it)


### Serial IO
+ `screen`, e.g. `screen -F /dev/tty0`
+ `minicom`
+ `stty` to set serial-line parameters if you then want to use normal IO, e.g. with `cat`


### Terminal device control
+ `stty`: set terminal parameters, e.g. echo, no-echo, char/line buffering
+ `reset`: fix a borked terminal


## Graphical remoting
### Whole-screen remoting
+ Remote control
  + `x11vnc`: serve X11 display over VNC
  + `xtightvncviewer`: view VNC connection
    + **Recommended:** `xtightvncviewer -via user@host` for SSH-encrypted connection
  + `remmina`: RDP client (connect to Windows desktop)
+ Remote viewing
  + `x11vnc -viewonly`
  + **Poor man's solution:** `ssh user@machine ffmpeg -f x11grab -i $DISPLAY -f flv - | ffplay -`
+ Control nearby computers
  + `barrier`: used to be `synergy`, soft KVM switch: move mouse onto adjacent computer desktops


### Window-level remoting
+ `xpra`: like `tmux` but for windows (fast, uses adaptive video codecs)
  + See [bashrc-tmux](https://github.com/spencertipping/bashrc-tmux) some automation to help with `xpra` server-side
+ `ssh -X`: native X11 remoting over SSH tunnel (slow)


## Working with files
### Measuring and profiling disk usage
+ `df -h`: show free space on all filesystems
+ `baobab` (graphical)
+ `du -s` (command-line)
  + **Idiom:** `du | sort -rn | less`: measure all subdirectories, show largest ones first


### `.deb` packages
+ `apt install X`
+ `apt remove X`
+ `apt autoremove X`: remove `X`, then remove unneeded dependencies
+ `apt list *x*`: list packages matching glob `*x*`
+ `dpkg -S upnpc`: show packages that provide files called `upnpc`


### File basics
+ File metadata
  + `stat`, `ls -l`: show file metadata (size, owner, link count, etc)
  + `file`: detect file type from contents (find magic numbers)
  + `filefrag`: show fragmentation of a file (number of extents)
  + `cloc`: count lines of code within a repo/subdirectory/etc
+ Finding files
  + `tree`: multi-level `ls`
  + `find`: depth-first search for files by attributes
  + `grep`: depth-first search for files by contents (also consider `ack` or `ag`)
  + `locate`: indexed search for files by name (`apt install mlocate`)


### Compression and archiving
Stream compressors:

+ Small-window LZ family (dictionary compression, medium speed)
  + `gzip` (`pigz` for multicore)
  + `lzo`
  + `lz4`
  + `compress`
+ Large-window LZ (better+slower compression, more memory usage, fast decompression)
  + `zstd`
  + `xz`/`lzma`
+ Statistical/esoteric (slow compressors)
  + `bzip2` (`pbzip2` for multicore)
  + `ppmd`
  + `zpaq`

File archivers:

+ `tar`: serial archive of files, commonly run through a stream compressor, e.g. `.tar.gz` or `.tar.xz`
+ `ar`: manage `.a` archives, commonly used for system libraries
+ `zip` + `unzip`: DEFLATE + archiving
+ `7zr` (`apt install p7zip`): LZMA + archiving
+ `rar` + `unrar`: PPM-style compression + archiving

Random-access compression using filesystems:

+ `btrfs` with file compression (fastest, worst compression)
+ `squashfs` (best compression ratio, single-threaded decompression unless you have a custom kernel -- `grep SQUASHFS /boot/config-$(uname -r)` to check)
+ `cramfs`
+ FUSE `archivemount` (read-only, slow, file-level rather than block-level access)


### Accessing compressed files
+ `file` to detect filetype
+ `ni` will autodetect and decompress things, including as streams
  + `ni file.gz` -- same as `zcat file.gz`
  + `cat file.gz | ni`
  + `cat file.lz4 | ni`
  + `ni zip://file.zip` will show you zip entries without extracting
  + `ni zipentry://file.zip:foo` will stream out a single entry
  + `ni tar://file.tar.gz`: same thing for tar
  + `ni 7z://file.7z`
  + `ni ar://libfoo.a`


### Moving files over the network
+ Via SSH
  + `rsync`
  + `scp`
  + `sftp`
  + `tar -c ... | gzip | ssh user@host tar -xz`
  + FUSE: `sshfs`
+ Via S3/cloud storage protocols
  + `rclone` (multi-backend adapter + `rsync` for many cloud storage providers)
  + `aws` (`apt install awscli`)
  + ... (dedicated packages for other cloud providers)
+ Serving via HTTP, fetching via HTTP
  + `python3 -m http.server`: serve current directory on port 8080
  + `curl -sSL https://url`: fetch URL and write to stdout/terminal
  + `wget https://url`: fetch URL and save to file
+ Direct socket IO (unencrypted but very fast)
  + `nc -l -p 8000 < file`: send `file` to whoever connects to TCP port 8000
  + `nc host 8000 > local`: connect to host:8000 and save results into `local`
+ BitTorrent
  + `transmission-cli` (uses transmission daemon)
  + `rtorrent` (frontend-only)


## Containers and virtualization
### Kubernetes: multi-machine container orchestration
+ [Rancher k3s](https://k3s.io)
+ [microk8s](https://ubuntu.com/tutorials/getting-started-with-kubernetes-ha#4-create-a-microk8s-multinode-cluster)
+ many others...


### Single-machine containers
+ Kubernetes (`snap install --classic microk8s`)
  + `microk8s`
+ Build and run Docker images (`apt install docker.io`)
  + `docker`
  + `docker-compose`
  + `rkt` + `acbuild`
+ Containerized app formats (alternative package formats)
  + `snap`
  + `flatpak`
+ Containerize existing apps for security
  + `firejail`
  + `bubblewrap`


### Single-machine virtualization
+ VirtualBox (graphical, easy to use)
+ `multipass` (`snap install multipass`)
+ `lxd` (`snap install lxd`)
+ `qemu-system-x86_64`


## Networking
+ `/etc/resolv.conf`: DNS server configuration (may be owned by systemd; see [AskUbuntu thread on disabling it](https://askubuntu.com/questions/907246/how-to-disable-systemd-resolved-in-ubuntu))
+ `/etc/hosts`: static host ↔ IP mappings
+ `/etc/nsswitch.conf`: meta-config that dictates how various types of names are resolved


### VPN and port forwarding
+ Wireguard: simple, secure (invisible), and fast: I highly recommend it
+ OpenVPN
+ `sshuttle` for TCP/UDP insta-VPN (forward a subnet with or without DNS)
+ `ssh -L 5000:localhost:6000 user@host`: forward `localhost:5000` to `host:6000` over SSH tunnel


### Discovery and diagnostic tools
+ `ping`
+ `mtr`: enhanced `traceroute`
+ `dig`, `nslookup`: issue DNS queries
+ `fping`: ping many hosts quickly (find reachable machines)
+ `nmap`: port scan
+ `avahi-browse`: show mDNS/Bonjour devices (autodiscover printers, etc)
+ `wireshark`, `tcpdump`: packet sniffers
+ `iptables -L -v`: local packet processing rules with per-table traffic counters (so you can find out if your firewall is eating traffic)


### Traffic monitoring
+ `nethogs`: like `top`, but for network traffic
+ `ss`: show active connections
+ `netstat`: show listening ports


### Manual link/address management
+ `ip addr add 10.4.0.0/24 dev wls2`: add address with subnet to `wls2`
+ `ip link set enp3s0f0 up`: manually activate link (e.g. for fiber)
+ `dhclient wls2`: request DHCP configuration on `wls2`


### Wi-Fi
+ `iw`: manage wifi adapters
+ `wpa_supplicant`, `wpa_passphrase`: handle WPA2 authentication
+ `wicd`: simple wifi management daemon
+ `NetworkManager`: more complicated (and builtin) wifi management
  + `nmcli` for command-line access, although IIRC it's incomplete

For example, here's how I connect to wifi:

```sh
# show SSIDs
iw wlp2s0 scan | grep -i ssid: | cut -d: -f2

# if passwordless
ip link set wlp2s0 up
iw wlp2s0 connect "$ssid"
dhclient wlp2s0

# if using WPA2
ip link set wlp2s0 up
echo "$password" | wpa_passphrase "$ssid" > /etc/wpa_supplicant.conf
wpa_supplicant -B -D wext -i wlp2s0 -c /etc/wpa_supplicant.conf
iw wlp2s0 scan
dhclient wlp2s0
```


### Traffic shaping / traffic control
+ `ip route change default via 192.168.0.1 realm 2`: assign a realm to a route (in this case, to all outbound gateway traffic)
+ `tc`: manipulate traffic control rules ([here's a guide](https://tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/))

`tc` is complex and powerful; here's my setup for rate-limiting traffic I try to upload so the modem doesn't saturate the uplink:

```sh
# identify this traffic by putting it into a realm, which tc can refer to
ip route change default via 192.168.0.1 realm 2

# set up tc traffic classes with rate limits
tc qdisc add dev enp2s0 root handle 1: htb default 12
tc class add dev enp2s0 parent 1:  classid 1:1  htb rate 1gbit ceil 1gbit
tc class add dev enp2s0 parent 1:1 classid 1:10 htb rate 5mbit ceil 5mbit

# assign realm-2 traffic to the 5mbit class
tc filter add dev enp2s0 route to 2 classid 1:10

# set queueing and prioritization parameters
tc qdisc add dev enp2s0 parent 1:10 handle 20: sfq perturb 10
```


### Network switching
+ `arp`: show/change ARP entries
+ `brctl`: create/delete/modify bridge devices (`apt install bridge-utils`)


### Network routing
+ `ip route list`: show routing table
+ `ip route add default via 192.168.0.1`: set gateway
+ `ip route add 10.4.0.0/16 dev wlp2s0`: specify device for outbound traffic
+ `upnpc -a 192.168.0.100 22 2222 TCP`: use uPnP to forward WAN port 2222 to 192.168.0.100:22 (`apt install miniupnpc`)
+ `iptables -t FORWARD`: kernel-level gateway behavior


### Network/bash adapters
+ `nc`: socket ↔ stdio
+ `socat`: like `nc`, but also supports UNIX domain sockets
+ `curl`: up/download http[s] resources


## System monitoring / process management
+ `conky`: customizable system stats on your desktop
+ `ssh -X user@host conky`: a janky way to have multiple systems on your desktop


### Resource usage
+ Local resources
  + `htop`: show memory/CPU per process
  + `atop`: show system bottlenecks (very useful: disk IO, network, memory, CPU, etc -- whichever is most saturated)
  + `iotop`: IO by process
  + `nethogs`: network by process
  + `powertop`: system power consumption and optimizations
  + `w`: show cumulative CPU by user/login session
+ Containerized resources
  + `kubectl top pod`: CPU/memory for all pods
  + `kubectl top node`: CPU/memory for all nodes
  + `docker stats`
  + `firejail --top`


### Process management
+ `pgrep sshd`: find all `sshd` process IDs (see `man pgrep` for many filtering options)
+ `pkill rsync`: send `SIGTERM` to all `rsync` processes
+ `pkill -STOP rsync`: temporarily stop all `rsync` processes
+ `pkill -CONT rsync`: un-stop all `rsync` processes
+ `kill $pid`: send `SIGTERM` to specific PID
+ `reptyr $pid`: move process with ID `$pid` to this terminal
+ `watch command...`: run `command` every 2s, showing output


### Process inspection/debugging
+ `strace ls`: show all system calls made when running `ls`
+ `gdb ls`: debug `ls` (catch segfaults, set breakpoints, etc)
+ `gdb -p $pid`: debug running process `$pid`
+ `radare2`: binary reverse engineering tool


### Hardware / I2C / SMBus / IPMI
See [this ArchWiki page for more tools](https://wiki.archlinux.org/title/Lm_sensors).

+ `lshw`: probe for and show all hardware
+ `acpi`: battery and power supply status (for laptops)
+ `sensors` (`apt install lm-sensors`): sensor inspection
+ `/sys/class/...`: system devices grouped by type
+ `ipmitool sdr list`: sensor inspection for IPMI-equipped machines (servers)


### Troubleshooting
+ `dmesg`: low-level kernel events, hardware stuff
+ `tail -f /var/log/syslog`: system events + daemons
+ `tail -f /var/log/auth.log`: attempted and successful logins
+ `journalctl -xe`: show recent systemd service failures
+ `systemctl --status-all`: show all systemd services


### RAID monitoring
+ `cat /proc/mdstat`
+ `mdadm --monitor /dev/md0`


## Filesystems and block devices
If you don't use a custom `cd` for bash, I wrote a [cd script](https://github.com/spencertipping/cd) that does some useful things like automounting filesystems. So I say `cd machine:/tmp` to FUSE-mount its `/tmp` dir and cd there; then `cd`-ing out will unmount and clean everything. (It supports a lot of other stuff like `cd /dev/sda1`, `cd enc:foo`, etc.)


### Mounting block devices and image files
+ `mount` with no arguments: show all mounted filesystems
+ `mount -o loop file /mount/point`: mount a file as a block device
+ `mount /dev/sda1 /mount/point`
+ `umount /mount/point`
+ `lsof /mount/point`: list processes with open files under `/mount/point` (open files prevent a volume from being unmounted)
+ `mount -t tmpfs -o size=1024m ramdisk /tmp/foo`: create a 1GiB memory-resident filesystem


### Block devices
+ Locations (see section `4` of `man` for docs on these)
  + `/dev/mapper/vgx-y`: LVM2 volume `y` from volume group `x`
  + `/dev/sda`: SATA/SAS disk
  + `/dev/sda1`: SATA/SAS disk partition 1
  + `/dev/nvme0n1`: NVMe SSD
  + `/dev/nvme0n1p1`: NVMe SSD partition 1
+ Performance testing
  + `hdparm -t /dev/sda`: measure unbuffered read performance
  + `hdparm -T /dev/sda`: measure cached read performance
  + `command1 | pv | command2`: measure data throughput
+ Forwarding a block device over the network
  + `nbd` [network block device](https://medium.com/@aysadx/linux-nbd-introduction-to-linux-network-block-devices-143365f1901b)
+ Filesystem checking and repair
  + `fsck`, for instance `fsck /dev/sda1`


### Partitions / boot sector
+ `parted`
+ `gparted` (graphical)
+ `fdisk`
+ `hdparm -z /dev/sda`: rescan `/dev/sda` for partitions (e.g. to repopulate `/dev/sda1`, `/dev/sda2`, etc)


### Kernel-hosted filesystems
+ Regular filesystems
  + `ext2`, `ext3`, and `ext4`
  + `xfs`
  + `jfs` (obsolete)
  + `reiserfs` (obsolete)
+ Copy-on-write (COW) filesystems
  + `btrfs`
  + `zfs`
+ Network filesystems
  + `nfs` (`apt install nfs-kernel-server`, `apt install nfs-common`)
+ Stacking/unions
  + `overlayfs`: union filesystem (simpler, mainline, used by Docker)
  + `aufs`: union filesystem (more powerful/advanced, not mainline)


### FUSE: userspace filesystems
+ `archivemount`: view archive entries as files
+ `encfs`: per-file on-the-fly encryption (not maximally secure, but probably fine for many use cases)
  + `encfs --reverse`: encrypted view of regular files
+ `exfat-fuse`: FAT interop
+ `fusermount -u /path`: unmount FUSE filesystem
+ [And many more](https://awesomeopensource.com/projects/fuse-filesystem)


### Software RAID, LVM, `dm_crypt`, cryptsetup
+ `mdadm --assemble --scan`: usually the best option
+ `mdadm --assemble /dev/md0 /dev/sda /dev/sdb ...`: manually specify devices
+ LVM
  + `pvs`: show physical volumes
  + `lvs`: show logical volumes
  + `pv*`, `lv*`: more LVM commands
+ Cryptsetup/LUKS (whole-disk encryption)
  + `cryptsetup` (has `luks` commands for passphrase/key management)
  + `luksformat`


### Data recovery
+ High-level
  + `testdisk`: discover partitions, filesystems, recover data
  + `recoverjpeg`: automatically recover JPEG files from a disk image
+ Low-level
  + `dd`: read/write block devices
  + `ddrescue`: read and bisect on error
  + `pv -EE`: read and ignore errors
  + `grep -abo`: print binary offset of regex within data (for example: `grep -abo VimCrypt disk-image` to find VIM-encrypted file offsets; then `dd skip=N` to recover each)


## Data processing
**Pro tip:** `bash` supports process redirection, which turns shell outputs into filenames that can be consumed by other programs. For example:

```sh
# strategy using tempfiles
$ gunzip file1.gz
$ gunzip file2.gz
$ sort file1 > file1.sorted
$ sort file2 > file2.sorted
$ join file1.sorted file2.sorted

# same thing, but no tempfiles (output is streamed as join reads it)
$ join <(zcat file1.gz | sort) <(zcat file2.gz | sort)
```

This works anywhere the program doesn't need the filesize or random access -- i.e. it can accept a FIFO. Same thing for files to write: `tee >(gzip > file)` for example.


### Parsing data
+ `jq`: encode/decode/transform JSON
+ `wc`: count words/lines/characters
+ `cut`: select fields from TSV/CSV
+ `sed`: transform text with regex (`sed -r` for more regex support)
+ `awk`: general text processing (`mawk` for JIT performance)
+ `grep`: select lines by regex (`egrep` for more regex support)
+ `csvkit` for small data
+ `ni` is extremely useful for this but it's a lot to learn


### Parallel processing
+ `xargs -P`
+ `parallel` (`apt install parallel`)
+ `ni`
  + `ni S` for parallel mappers on line boundaries
  + `ni S\>` for sharded outputs
  + `ni \*\>` for parallel processing
  + `ni fx[]` for `xargs` streaming


### Rearranging data
+ `sort`: disk-backed mergesort with tempfile compression (can sort data larger than your free disk space)
+ `tsort`: topological sort, topsort
+ `ni`
  + `ni g`: `sort`
  + `ni o`: `sort -n`
  + `ni O`: `sort -rn`
  + `ni gg`: sort within grouped data


### Joins and set operations
+ `join`: incrementally join sorted streams
+ `comm`: set operations on sorted streams
+ `paste`: zip files together
+ `ni`
  + `ni j` for streaming/sorted joins
  + `ni J` for in-memory joins


### Piping data
+ `cat`: read data from file(s), copy stdin to stdout
+ `pv`: like `cat`, but with progress reporting
+ `pv -L 100`: like `cat`, but rate-limit data to 100 bytes/second
+ `head -c1000`: pipe first 1000 bytes, then exit
+ `tail -c1000`: pipe last 1000 bytes
+ `tee`: duplicate a stream into a file
  + `tee >(wc -l)`: ...or another command, using `>()` process redirection


### Profiling shell commands
+ `time <shell command>`: show user, sys, realtime
+ `pv file | ...`: watch progress + throughput while processing a file


### Plotting
+ `gnuplot`: scriptable online/offline graphing tool (works well with `ffmpeg -f image2pipe` if you want to animate graphs, for example)
+ `labplot`: interactive graphing


### Calculators and math
+ `factor`: factor arbitrarily large integers quickly (surprisingly useful; it's sort of like a "where did this number come from" tool)
+ `units`: unit conversions, physics, real-world math (e.g. `units -t '500GB/4Mbps' hours`)
+ `bc`: command-line calculator
+ `maxima`, `axiom`, `pari`, `singular`: computer algebra systems
+ `octave`: Matlab-style environment
+ `ni`: one-liners in a number of languages
  + `ni 1p'3 + 4'`: perl
  + `ni 1y'3 + 4'`: python
  + `ni 1m'3 + 4'`: ruby
  + `ni 1l'(+ 3 4)'`: SBCL
  + `ni 1js'3 + 4'`: nodeJS
  + `ni 1hs'main = putStrLn (show (3 + 4))'`: Haskell, if you really want this
  + `ni 1c++$'#include <stdio.h>\nint main(){printf("%d\\n", 3 + 4);}': C++

If you want to do matrix math, the fastest thing I've found is numpy. `octave` is good too but doesn't always use multicore ops and sometimes uses more memory. `ni` can stream data into numpy in binary using the `N` operator, or you can use `by''`.


## Media
Short version: `ffmpeg` and `convert` can do almost everything in a scripted way. `ffmpeg`'s grammar is basically `ffmpeg [-f format -i input] transformers... [-f format output]`. `convert` is stack-based but often works in a similar way. Its command-line grammar is more powerful than `ffmpeg`'s.


### Audio
+ Audio playback
  + `mplayer`: general-purpose command-line audio player
  + `aplay`/`pacat`: play WAV files or raw sample data
  + `arecord`/`parec`: record WAV audio
  + `audacious`: graphical audio player (like WinAmp)
  + `pianobar`: Pandora streaming from command-line (ad-free)
+ Audio processing
  + `sox`: audio-specific command-line processor
  + `ffmpeg`: general media-format filtering/conversion
  + `audacity`: graphical sound editor


### Pictures/images
+ `ffmpeg` with `image2pipe` format: frames to/from video
+ `convert`: command-line image processing/manipulation/conversion (`apt install imagemagick`)
+ `dcraw`: raw image decoding (e.g. from DSLR cameras)
+ `feh`: image preview, including from stdin (`feh -`)
+ `gimp`: photoshop-style image editing
+ `ni` has some operators for repeated `convert` calls, and for image/video conversions with `ffmpeg`


### Video
+ Video recording/downloading/capture
  + `ffmpeg -f x11grab -i :0.0`: record X11 display `:0.0` as video
  + `ffmpeg -f v4l2 -i /dev/video0`: stream video from local camera device
  + `youtube-dl`: download video from youtube/vimeo/dailymotion/etc
  + `ni` has shorthands for things like this
+ Video playback
  + `vlc`: plays basically everything
  + `mplayer`: plays video, including with ASCII art if you're over SSH
  + `ffplay`: less-interactive playback, e.g. from video data over stdin (ships with `ffmpeg`)
+ Video processing
  + `ffmpeg`: video encoding/decoding/transcoding
  + `blender`: CGI, 3D modeling, rendering, compositing, production-quality video editing (a handful to learn)
