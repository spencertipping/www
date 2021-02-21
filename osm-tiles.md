# OSM tiling
Just some notes I keep around for new OSM downloads.

```sh
# Note for future reference: bz2 is expensive to decompress even with pbzip2;
# 177MB/s requires 800-1600% CPU. It would be better to transcode to lz4 here:
#
# ni planet-210208.osm.bz2 z4\>planet

$ ln -s planet-210208.osm.bz2 planet
```


## Nodes, ways, and relations
```sh
$ ni planet S32r/\<node/p'r /id="([^"]+)"/, /lat="([^"]+)"/, /lon="([^"]+)"/' \
            S64p'r a, ghe b, c, -60' \
            ^{row/sort-buffer=262144M row/sort-parallel=64} \
            o p'wp"QQ", a, b' \>nodes-packed.QQ

$ ni planet e[egrep -v '<node|</?changeset'] \
            e[perl -ne 'print if /<way/../<\/way/'] z4\>ways

$ ni planet e[egrep -v '<node|</?changeset'] \
            e[perl -ne 'print if /<relation/../<\/relation/'] z4\>relations
```


## Tiles with dereferenced nodes
Let's reuse the memory mapping across threads and do binary-search lookups to
fetch each node's coordinates. Since `nodes-packed.QQ` comes from a spinning
disk, we need to read it through up front to populate virtual memory pages
efficiently. If we let them get paged in by the binary search, we'll get
horrible performance because our IO will be mostly random.

```sh
$ sudo apt install libsys-mmap-perl
$ mkdir -p tiles
$ ni ways e'tr "\n" " "' \
     e[ perl -e 'for (my $s = ""; read STDIN, $s, 1<<20, length $s;)
                 { print $1, "\n" if $s =~ s/(<way.*<\/way>)// }' ] \
     S64[p'split /<\/way>/' \
         p'my ($way) = (my $l = $_) =~ /<way ((?:\w+="[^"]*"\s*)*)/;
           my $attrs = json_encode {$way =~ /(\w+)="([^"]*)"/g};
           my $tags  = json_encode {$l =~ /tag k="([^"]*)" v="([^"]*)"/g};
           r $attrs, $tags, /nd ref="(\d+)"/g' \
         p'^{ use Sys::Mmap;
              open my $fh, "< nodes-packed.QQ";
              mmap $nodes, 0, PROT_READ, MAP_SHARED, $fh;
              my $x;
              $x = substr $nodes, $_ << 20, 1048576 for 0..length($nodes) >> 20 }
           r a, b, map bsflookup($nodes, "Q", 16, $_, "x8Q"), FR 2' \
         p'r "tiles/$_", F_ for uniq map gb3($_ >> 45, 15), FR 2'] \
     ^{row/sort-buffer=262144M row/sort-compress=pigz row/sort-parallel=64} \
     g W\> e'xargs -P64 -n1 xz -9e'
```


## Relations
**TODO**
