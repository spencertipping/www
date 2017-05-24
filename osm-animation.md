# Animating OpenStreetMap history
## Step 1: Preprocessing
Starting with the [OSM planet file](http://planet.openstreetmap.org/), the
first thing to do is transcode to LZ4. Bzip2 isn't a bad compression format,
but the decompressor is only 5% the throughput of LZ4 and that's a process
bottleneck. While we're at it, let's build the sorted node table. Checkpoints
are useful here; if anything is wrong with the parser, we don't want to have to
restart the full download or recompress.

(By the way, I highly recommend that you install `pbzip2` for this; it's still
worth recompressing as LZ4 to save overall CPU time, but `pbzip2` will
distribute the load across all CPUs and in this case gets about 250MB/s for the
initial LZ4 transcoding, vs 20MB/s with regular `bzip2 -dc`. ni uses `pbzip2`
automatically if you've got it installed.)

```sh
$ ni http://planet.openstreetmap.org/planet/planet-latest.osm.bz2 \
     z4:osm-planet.lz4 S12r/\<node/F:\"fHDFN z4:nodes-p gz4\>nodes-p-by-time
```

### How the parser works
Because OSM's export format is so regular, we can simplify the design by
minimizing the amount of context required for a parse.

- `r/\<node/`: select just the lines containing opening `<node ...>` tags.
- `F:\"`: column-split on double quotes, which separate field values from XML
  syntax.
- `fHDFN`: select the columns corresponding to timestamp, latitude, longitude,
  and username, respectively.

We can safely scale this out because we later sort the nodes by timestamp.
Here's what `nodes-p` looks like:

```sh
$ ni nodes-p r10
2014-06-26T15:50:35Z    51.5161619      -0.0815187      ecatmur
2014-03-20T21:50:01Z    51.5176625      -0.0708957      yourealwaysbe
2014-03-23T17:45:26Z    51.5171361      -0.0714455      jfrankcom
2016-09-28T21:01:54Z    51.5173495      -0.0667203      harg
2013-02-12T18:01:35Z    51.5176847      -0.0656604      Amaroussi
2013-02-12T17:55:21Z    51.5181549      -0.0639747      Amaroussi
2008-11-26T23:02:08Z    51.5171563      -0.0634871      randomjunk
2015-05-02T20:08:15Z    51.5169859      -0.0654873      markbegbie
2008-11-26T23:02:09Z    51.5170374      -0.0644735      randomjunk
2017-04-19T14:20:57Z    51.5128129      -0.0597911      peregrination
```

## Step 2: Generating animation frames
The general idea here is to use two ni operators, `G` for `gnuplot` and `GF`
for `ffmpeg`, to convert a partitioned stream into JPEG images and stream those
into a single AVI file. Conceptually the process works like this:

```sh
$ for f in frame-data/*; do             # NB: ni emulates this loop
    gnuplot -e '<stuff to plot a frame>'
  done | ffmpeg -f image2pipe -i - -vcodec mjpeg output.avi
```

`ni GF[...]` == `ffmpeg -f image2pipe -i - -vcodec mjpeg ...`, and `GA'...'`
expands into the `for` loop above, except that `gnuplot` will get each frame
input on stdin rather than from a file. That means no individual frames are
written to disk; we can start with the compressed sorted node list and end up
with a compressed AVI result.

### Test run
Let's start just with London to test render settings. The initial extraction
was still running when I wrote the test below, so I'm re-extracting from a
previous copy of `osm-planet.lz4`.

```sh
$ ni osm-planet.lz4 \
     S24r/\<node/F/[\"T]/fHFDrp'within -2, 2, b and within 50, 52, c' \
     rE7gz4:london GA'
       set terminal jpeg background "#000000" size 1600,900;
       unset tics; unset border;
       plot [-2:2] [50:52]
         "< ni london S24rp\"a lt q{" . KEY . "}\" fBC </dev/null" with dots lc "#f8cccccc",
         "-" with dots lc "#80ff4400"' \
     GF[-y -qscale 5 london.avi]
```

[london.avi on Youtube](https://www.youtube.com/watch?v=KByWykeUsZQ)

Halfway decent for a first attempt. A few things could be better of course:
ideally we'd have new points fade into the map rather than flash and disappear.
We should also fix up the aspect ratio, and from a processing perspective we
need something more efficient than doing the full scan per frame for "stuff in
the past."

### Compositing to the rescue
There's a case to be made for transforming the data itself to make each frame
the way we want it, but to get things like accumulation and fading it's way
easier to use image compositing with a reduction loop. Here's the idea:

```
start with empty reduction frame
for each group of data:
  gnuplot a delta frame
  composite the delta frame into the reduction frame
  emit the new reduction frame
```

This requires ImageMagick and [a couple of new ni
operators](https://github.com/spencertipping/ni/blob/develop/doc/image.md).

Now we can add a running blend and blur:

```sh
$ ni london GA'
       set terminal png background "#000000" size 3200,1800;
       unset tics; unset border;
       plot [-2:2] [50:52] "-" with dots lc "#c0c4d0ff"' \
     IC: [-blur 1x1 - -compose blend -define compose:args=100,98 -composite] \
         [-resize 1600x900] \
     IJ GF[-y -qscale 5 london.avi]
```

[On youtube](https://youtu.be/EfkcGxCU48M)

### Full-planet render
Exactly like before, compositing at twice the render resolution.

```sh
$ ni nodes-p-by-time p'r a =~ /^([^T]+)/, c, b' GA'
       set terminal png background "#000000" size 3840,2160;
       unset tics; unset border;
       plot [-180:180] [-90:90] "-" with dots lc "#c0c4d0ff"' \
     IC: [-blur 1x1 - -compose blend -define compose:args=100,98 -composite] \
         [-resize 1920x1080] \
     IJ GF[-y -qscale 5 world.avi]
```

[On youtube](https://youtu.be/_GSai4UWhFU)
