# Linux command-line survival guide
A probably-opinionated list of things I've found useful in 20 years of using Linux.


## Linux documentation
+ `man ls`: show manpage for `ls`
+ `info ls`: show infopage for `ls` (like `man`, but different)
+ `apropos 'kernel module'`: list manpages with "kernel module" in the description
+ [Manpage sections](https://www.commandlinux.com/man-pages-sections)
  + `man 1 stat`: show manpage for command-line tool `stat`
  + `man 2 stat`: show manpage for kernel API call `stat`
  + `man 3 stat`: show manpage `libc` API call `stat`
  + `man 4 sd`: show manpage for devices like `/dev/sda`
  + `man 5 resolv.conf`: show manpage for `/etc/resolv.conf` format
  + `man 7 tcp`: manpage for TCP protocol and APIs
  + `man 8 wg`: manpage for WireGuard config tool


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


## Remoting
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
  + `barrier`: used to be `synergy`


### Window-level remoting
+ `xpra`: like `tmux` but for windows (fast, uses adaptive video codecs)
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
Compressors:

+ `zstd`
+ `gzip` (`pigz` for multicore): LZ77 + Huffman
+ `bzip2` (`pbzip2` for multicore):
+ `xz`
+ `lzma`
+ `zstd`
+ `lzo`
+ `lz4`
+ `ppmd`
+ `zpaq`
+ `compress`

Archivers (file-level access):

+ `tar`
+ `zip` + `unzip`
+ `7zr` (`apt install p7zip`)
+ `rar` + `unrar`
+ `ar`

Random-access compression using filesystems:

+ `btrfs` with file compression (fastest, worst compression)
+ `squashfs` (best compresison ratio, single-threaded decompression unless you have a custom kernel)
+ `cramfs`
+ FUSE `archivemount` (slow, not really random-access)


### Moving files over the network
+ Via SSH
  + `rsync`
  + `scp`
  + `sftp`
  + `tar -c ... | gzip | ssh user@host tar -xz`
  + FUSE: `sshfs`
+ Via S3/cloud storage protocols
  + `rclone` (multi-backend adapter + `rsync` for many cloud storage providers)
  + `aws` (via `awscli`)
  + ... (dedicated packages for other cloud providers)
+ Serving via HTTP, fetching via HTTP
  + `python3 -m http.server`: serve current directory on port 8080
  + `curl -sSL https://url`: fetch URL and write to stdout/terminal
  + `wget https://url`: fetch URL and save to file
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
+ `mtr`: `traceroute`
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
+ `iw`: manage wifi adapters


### Traffic shaping / traffic control
+ `ip route change default via 192.168.0.1 realm 2`: assign a realm to all traffic sent through gateway
+ `tc`: manipulate traffic control rules ([here's a guide](https://tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/))

`tc` is complex and powerful; here's my setup for rate-limiting traffic I try to upload so the modem doesn't saturate the uplink:

```sh
# identify this traffic by putting it into a realm, which tc can refer to
ip route change default via 192.168.0.1 realm 2

# set up tc traffic classes
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
  + `w`: show CPU by user/login session
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


### Process inspection/debugging
+ `strace ls`: show all system calls made when running `ls`
+ `gdb ls`: debug `ls` (catch segfaults, set breakpoints, etc)
+ `gdb -p $pid`: debug running process `$pid`


### Hardware / I2C / SMBus / IPMI
See [this ArchWiki page for more tools](https://wiki.archlinux.org/title/Lm_sensors).

+ `sensors` (`apt install lm-sensors`): sensor inspection
+ `ipmitool sdr list`: sensor inspection for IPMI-equipped machines (servers)
+ `/sys/class/...`: system devices grouped by type
+ `acpi`: battery and power supply status (for laptops)


### Troubleshooting
+ `dmesg`: low-level kernel events, hardware stuff
+ `tail -f /var/log/syslog`: system events + daemons
+ `journalctl -xe`: show recent systemd service failures
+ `systemctl --status-all`: show all systemd services


### RAID monitoring
+ `cat /proc/mdstat`
+ `mdadm --monitor /dev/md0`


## Filesystems and block devices
### Mounting block devices and image files
+ `mount` with no arguments: show all mounted filesystems
+ `mount -o loop file /mount/point`: mount a file as a block device
+ `mount /dev/sda1 /mount/point`
+ `umount /mount/point`
+ `lsof /mount/point`: list processes with open files under `/mount/point` (open files prevent a volume from being unmounted)


### Block devices
+ Locations
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
  + `nfs`
+ Filesystem checking and repair
  + `fsck`, for instance `fsck /dev/sda1`


### FUSE: userspace filesystems
+ `archivemount`: view archive entries as files
+ `encfs`: per-file on-the-fly encryption (not maximally secure, but probably fine for many use cases)
  + `encfs --reverse`: encrypted view of regular files


### Software RAID
+ `mdadm --assemble --scan`: usually the best option
+ `mdadm --assemble /dev/md0 /dev/sda /dev/sdb ...`: manually specify devices


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
### Parsing data
+ `jq`: encode/decode/transform JSON
+ `wc`: count words/lines/characters
+ `cut`: select fields from TSV/CSV
+ `sed`: transform text with regex (`sed -r` for more regex support)
+ `awk`: general text processing (`mawk` for JIT performance)
+ `grep`: select lines by regex (`egrep` for more regex support)


### Rearranging data
+ `sort`: disk-backed mergesort with tempfile compression (can sort data larger than your free disk space)
+ `tsort`: topological sort, topsort


### Piping data
+ `cat`: read data from file(s), copy stdin to stdout
+ `pv`: like `cat`, but with progress reporting
+ `pv -L 100`: like `cat`, but rate-limit data to 100 bytes/second


### Profiling shell commands
+ `time <shell command>`: show user, sys, realtime
+ `pv file | ...`: watch progress + throughput while processing a file


### Plotting
+ `gnuplot`: scriptable online/offline graphing tool
+ `labplot`: interactive graphing


### Calculators and math
+ `factor`: factor integers quickly (surprisingly useful)
+ `units`: unit conversions, physics, real-world math (e.g. `units -t '500GB/4Mbps' hours`)
+ `bc`: command-line calculator
+ `maxima`, `axiom`, `pari`, `singular`: computer algebra systems
+ `octave`: Matlab-style environment


## Media
### Audio
+ Audio playback
  + `mplayer`: general-purpose command-line audio player
  + `aplay`/`pacat`: play WAV files or raw sample data
  + `audacious`: graphical audio player (like WinAmp)
  + `pianobar`: Pandora streaming from command-line (ad-free)
+ Audio processing
  + `sox`: audio-specific command-line processor
  + `ffmpeg`: general media-format filtering/conversion
  + `audacity`: graphical sound editor


### Pictures/images



### Video
+ Video recording/downloading/capture
  + `ffmpeg -f x11grab -i :0.0`: record X11 display `:0.0` as video
  + `ffmpeg -f v4l2 -i /dev/video0`: stream video from local camera device
  + `youtube-dl`: download video from youtube/vimeo/dailymotion/etc
+ Video playback
  + `vlc`: plays basically everything
  + `mplayer`: plays video, including with ASCII art if you're over SSH
+ Video processing
  + `ffmpeg`: video encoding/decoding/transcoding
  + `blender`: CGI, 3D modeling, rendering, compositing, production-quality video editing (a handful to learn)
