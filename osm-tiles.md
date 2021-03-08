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
OSM's XML is encoded in big blocks of similarly-typed tags; that is, we have all the nodes followed by all the ways followed by all the relations. This means it takes quite some time before we get to ways or relations -- so let's go ahead and unpack those into their own files so we can iterate more quickly.

```sh
$ export NI_PIPELINE_IO_SIZE=1048576; \
  ni planet R, \
     =[R'/<way.*<\/way>/'           S32R='</way>'      z4\>ways] \
     =[R'/<relation.*<\/relation>/' S32R='</relation>' z4\>relations] \
     R'/<node.*\/>/' \
     S32[R'/<node.*?\/>/' \
         p'my ($lat, $lon) = (/lat="([^"]+)"/, /lon="([^"]+)"/);
           r /id="([^"]+)"/, ghe $lat, $lon, -60'] \
     ^{row/sort-buffer=262144M row/sort-parallel=64 row/sort-compress=lz4} \
     o bf^QQ \>nodes-packed.QQ
```


## Tiles with dereferenced nodes
Let's reuse the memory mapping across threads and do binary-search lookups to fetch each node's coordinates. Since `nodes-packed.QQ` comes from a spinning disk, we need to read it through up front to populate virtual memory pages efficiently. If we let them get paged in by the binary search, we'll get horrible performance because our IO will be mostly random.

```sh
$ sudo apt install libsys-mmap-perl
$ mkdir -p tiles
$ ni ways R,R'/<way.*<\/way>/' \
     S64[R='</way>' \
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
     ^{row/sort-buffer=262144M row/sort-parallel=64 row/sort-compress=lz4} \
     g W\> e'xargs -P64 -n1 xz -9e'
```


## Relations
Relations can refer to ways, nodes, and other relations. Let's start by building a graph of inter-relation dependencies and a mapping from ways to geo-tiles. Then we can shard relations into the geo-tiles that contain their contents.

```sh
$ ni tiles fx64[%f : i%f \<D:id p'r a, "%f" =~ /\/(\w+)\.xz/' z\>] \<# \
     ^{row/sort-buffer=65536M row/sort-parallel=64 row/sort-compress=lz4} \
     o p'r a, join",", b_ rea' z\>wayid-tile

$ ni relations \
     S64p'r je {(a.b) =~ /(\w+)="([^"]*)"/g},
            je {map /<tag k="([^"]+)" v="([^"]*)"/g, FR 1},
            grep s/.*<member type="(\w+)" ref="([^"]+)".*/$1:$2/, FR 1' \
     z\>relation-refs
```

What's an upper bound for the number of tiles in which a way will appear?

```sh
$ ni wayid-tile S64p'/,/ ? r a, length(b =~ s/[^,]//gr) : ()' OB
162812399       255
187764116       69
253106786       62
253106778       47
60185202        45
253106770       43
377962040       42
232928715       41
354228054       38
...

$ ni wayid-tile S64p'/,/ ? r length(b =~ s/[^,]//gr) : ()' ,l2qoc
1348869 0
18197   1
5389    2
434     3
168     4
35      5
3       6
1       8
```
