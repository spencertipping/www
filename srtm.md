# Visualizing global elevation
SRTM tiles are encoded in a very simple format; for the SRTM1 (one arc-second
per sample) dataset, this is just a 3601x3601 list of big-endian signed short
integers. Here's what four of them look like:

![image](http://storage5.static.itmages.com/i/17/0517/h_1495052873_3796138_5ed0950cf4.jpeg)

## Visualizing a single tile
[ni](https://github.com/spencertipping/ni) provides the `bp` operator to enter a
binary Perl context. We can use that to unpack the binary files into rows of
heights in TSV:

```sh
$ ni srtm1/N35W106.hgt bp'r rp"n3601"' Y r10
0       0       1821
0       1       1819
0       2       1816
0       3       1811
0       4       1806
0       5       1803
0       6       1804
0       7       1806
0       8       1806
0       9       1805
```

Breaking this down:

- `bp'...'` evaluate `...` as perl within a binary reader context
  - `rp"3601"`: binary-read the pack string `n3601` (3601 unsigned big-endian
    shorts), advance the cursor, and return the array of results
  - `r @ints`: write a TSV row of the ints
- `Y`: convert a dense matrix to a sparse one

If you load this up in `ni --js`, you'll get this after some viewport scaling:

![image](http://storage8.static.itmages.com/i/18/0107/h_1515294960_3694975_44a0e8ba27.png)

The challenging part from here is making the coordinates consistent if we want
to look at multiple tiles.

## Combining everything
Rather than combining stuff, we just need to convert each tile into a sparse
form that includes absolute lat/lng coordinates for each height sample. We can
use `f[]` to drop the tile name (which contains its base coordinates) into a
pipeline.

```sh
$ ni srtm1 f[%x : i%x] r10
srtm1/N00E006.hgt
srtm1/N00E009.hgt
srtm1/N00E010.hgt
srtm1/N00E011.hgt
srtm1/N00E012.hgt
srtm1/N00E013.hgt
srtm1/N00E014.hgt
srtm1/N00E015.hgt
srtm1/N00E016.hgt
srtm1/N00E017.hgt
```

Before we get into processing stuff, it's worth designing an output format that
isn't going to be horribly inefficient. For example, we could emit (lat, lng,
elevation) tuples -- but even in binary that's going to be 5x larger than the
original data. A better approach is to simply build up tables of (lat, lng,
3600-elevations), where the elevations proceed eastwards as they do in the
original data. Queries are relatively straightforward and the size doesn't
increase much at all since we're still emitting binary.

```sh
$ ni srtm1 f[%x : i%x \< \
       bp'^{($lats, $lat, $lngs, $lng) = q{%x} =~ /([NS])(\d+)([WE])(\d+)/;
            $lat *= -1 if $lats eq "S";
            $lng *= -1 if $lngs eq "W"}
          wp "ffn3600", $lat - bi/2/(3601**2), $lng, rp"n3601"'] \
     z4\>srtm1.ffn3600
```

Now let's visualize the whole globe. `ni --js` can hold about 5M points in
memory, so let's figure out a reasonable scaling factor:

```sh
$ units -t '5million/(3600*3600*360*180)' 1/1000
0.0059537418
```

This is enough of a reduction that preprocessing makes sense. Here's the basic
idea:

- `bp'r rp"ffn3600"'` to unpack the format into a single row of `lat lng pts...`
  - Actually, anytime the `pack` template is fixed-length, `bp'r rp"..."'` is
    faster written as `bf'...'` because ni can precompute the binary length of
    each data item and load multiple at once.
- `YC` to sparsify the heights; now we have `lat lng row col height`
- `p'r a, b+d/3600, e'` to get correct `lat lng height`

We can scale `YCp...` because each row out of `bf` is independent. I'm also
going to export 1/1000th of the data instead of the ridiculously small fraction
we had above. I'll also encode this as `ffn` binary again to save space. It's ok
(and necessary) to use full coordinates per point because we're working with a
sparse representation.

I'm also going to use gzip instead of lz4 here. LZ4 generally gets its advantage
from repeated pieces of data, whereas gzip also includes a Huffman stage that
should get a bit of leverage from the common-ranged values (maybe from the
sign/exponent bits in the floats).

The other thing is that ni's `bf` unpacker can't saturate LZ4's output speed,
nor even gzip's as far as I know.

```sh
$ ni srtm1.ffn3600 bfffn3600 S24YCr.001p'r a, b+d/3600, e' \
     p'wp "ffn", F_' z\>srtm1.ffnsample
```

Now we can visualize that result:

![image](http://storage9.static.itmages.com/i/18/0108/h_1515427638_8884282_a0c1c4b469.png)
