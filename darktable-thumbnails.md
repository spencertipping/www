# Darktable thumbnails
Not the world's most pressing problem, nor the most glamorous to address -- but here's the setup.

Joyce is a prolific photographer-of-offspring, so we've got a bunch of photos from various camera imports over the years. They live on a RAID array on one of the servers, and are accessible over high-speed NFS to the other servers. We used to use Lightroom to process them, but I prefer Darktable on a server and `xpra` for remote access. The main compute server has 512GB of memory, so thumbnails load instantly.

_However,_ getting Darktable to generate thumbnails quickly is an undertaking because `darktable-generate-cache` runs a serial loop over the images. Even if each image is processed on multiple CPUs, we lose a lot of time blocking on image IO. (And in practice, I see about 130% CPU from Darktable, a far cry from the 6400% we'd get with true parallelism on this system.)

So I want to leverage all 64 cores and get thumbnails, ideally without modifying Darktable or `darktable-generate-cache`.


## Reading the library database
`~/.config/darktable/library.db` is a SQLite3 database with a few useful tables. We're interested in `images` and `film_rolls`, which we can join to construct full image paths. We're basically doing the [same query `generate-cache` does](https://github.com/darktable-org/darktable/blob/1074671f5d42afda9b3eaa7af13c0ebe42f37746/src/generate-cache/main.c#L92), but adding the film roll path so we have absolute filenames.

```sh
$ ni sqliteq://.config/darktable/library.db:"select \
       images.id, film_rolls.folder || '/' || images.filename \
       from images \
       inner join film_rolls on images.film_id = film_rolls.id"
```

This gives us two columns, `id` and an absolute pathname to each image. Now let's figure out how to generate the thumbnails Darktable expects.


## Using `dcraw` and `convert` to produce thumbnails
Darktable has multiple thumbnail sizes, each stored in a separate directory. You can see the table [here](https://github.com/darktable-org/darktable/blob/1074671f5d42afda9b3eaa7af13c0ebe42f37746/src/common/mipmap_cache.c#L541):

```c
int32_t mipsizes[DT_MIPMAP_F][2] = {
  { 180, 110 },             // mip0 - ~1/2 size previous one
  { 360, 225 },             // mip1 - 1/2 size previous one
  { 720, 450 },             // mip2 - 1/2 size previous one
  { 1440, 900 },            // mip3 - covers 720p and 1366x768
  { 1920, 1200 },           // mip4 - covers 1080p and 1600x1200
  { 2560, 1600 },           // mip5 - covers 2560x1440
  { 4096, 2560 },           // mip6 - covers 4K and UHD
  { 5120, 3200 },           // mip7 - covers 5120x2880 panels
  { 999999999, 999999999 }, // mip8 - used for full preview at full size
};
```

I'm mostly interested in small sizes; zooming in isn't such a problem because fewer images are on-screen and the server has enough background threads to fill them in pretty quickly. So let's start with mip levels 0, 1, and 2. We'll do each in a separate loop.

```sh
$ sudo apt install -y dcraw feh imagemagick
$ dcraw -c 2019.12.23-canon/DCIM/100CANON/IMG_0596.CR2 | convert - -resize 180x110 jpg:- | feh -
```

Looks great, but it's also pretty slow:

```sh
$ time dcraw -c 2019.12.23-canon/DCIM/100CANON/IMG_0596.CR2 | convert - -resize 180x110 jpg:- > /dev/null

real    0m8.021s
user    0m7.491s
sys     0m0.646s
```

Hmm, that's much slower than what Darktable was giving us. I bet Darktable used the embedded previews instead of full decoding -- that's available via `dcraw`'s `-e` option.

```sh
$ time dcraw -e -c 2019.12.23-canon/DCIM/100CANON/IMG_0596.CR2 | convert - -resize 180x110 jpg:- > /dev/null

real    0m1.552s
user    0m1.453s
sys     0m0.117s
```

That's better. How about [dcraw-fast](https://github.com/ttyridal/dcraw-fast)? We have to build it:

```sh
$ sudo apt build-dep dcraw    # eh, should be good enough
$ git clone git://github.com/ttyridal/dcraw-fast \
  && cd dcraw-fast \
  && sed -ri 's/-Werror//g' Makefile \
  && make dcraw \
  && mv dcraw ~/bin/dcraw-fast \
  && cd .. \
  && rm -rf dcraw-fast

$ time dcraw-fast -e -c 2019.12.23-canon/DCIM/100CANON/IMG_0596.CR2 | convert - -resize 180x110 jpg:- > /dev/null

real    0m1.663s
user    0m1.577s
sys     0m0.102s
```

About the same, and perhaps a bit slower. We'll stick with stock `dcraw` for this one.


## `xargs -P` to scale out
OK, so the big loop is simple enough; we just go through the images looking for mips that don't exist yet and generate them. We could use ni's `fx64[]` to do parallel functions that take multiple arguments, but it has some overhead per iteration. It's faster to go straight to `sh -c`.

Before we get into it, let's just sanity-check our file extensions:

```sh
$ ni sqlitet://.config/darktable/library.db:images p'f =~ s/.*\.//r' gcO

10932   ARW
6486    CR2
2189    RW2
101     JPG
1       filename
```

Cool, we just need to exclude JPG since `dcraw` doesn't handle those.

```sh
$ ni sqliteq://.config/darktable/library.db:"select \
       images.id, film_rolls.folder || '/' || images.filename \
       from images \
       inner join film_rolls on images.film_id = film_rolls.id" \
     r-1 p'r "$ENV{HOME}/.cache/darktable/mipmaps-7fbc3e0d67c3651b33f1b2d5910d35ba5b2d5f6d.d/0/" . a . ".jpg", b' \
     rp'not -e a' rp'b =~ /(RW2|ARW|CR2)$/i' \
     F^::e[xargs -P64 -I{} \
           sh -c 'mf="{}"; dcraw -e -c "${mf#*:}" | convert - -resize 180x110 "${mf%:*}"'] \
     wcl
```

Awesome, preview images are loading! And this process is really fast; anywhere from 30 to 200 images/second (probably depending on the camera). Now let's do the other mip levels:

```sh
$ ni sqliteq://.config/darktable/library.db:"select \
       images.id, film_rolls.folder || '/' || images.filename \
       from images \
       inner join film_rolls on images.film_id = film_rolls.id" \
     r-1 p'r "$ENV{HOME}/.cache/darktable/mipmaps-7fbc3e0d67c3651b33f1b2d5910d35ba5b2d5f6d.d/1/" . a . ".jpg", b' \
     rp'not -e a' rp'b =~ /(RW2|ARW|CR2)$/i' \
     F^::e[xargs -P64 -I{} \
           sh -c 'mf="{}"; dcraw -e -c "${mf#*:}" | convert - -resize 360x225 "${mf%:*}"'] \
     wcl

$ ni sqliteq://.config/darktable/library.db:"select \
       images.id, film_rolls.folder || '/' || images.filename \
       from images \
       inner join film_rolls on images.film_id = film_rolls.id" \
     r-1 p'r "$ENV{HOME}/.cache/darktable/mipmaps-7fbc3e0d67c3651b33f1b2d5910d35ba5b2d5f6d.d/2/" . a . ".jpg", b' \
     rp'not -e a' rp'b =~ /(RW2|ARW|CR2)$/i' \
     F^::e[xargs -P64 -I{} \
           sh -c 'mf="{}"; dcraw -e -c "${mf#*:}" | convert - -resize 720x450 "${mf%:*}"'] \
     wcl
```


## Icing on the cake: preloading the cache into IO memory
I notice that Darktable doesn't instantly load the preview images when I pull it up. They're a lot faster than they were, but it's still doing IO against a spinning disk to pull them up. I've got enough memory on the server to contain the whole cache directory, so let's preload them to avoid this delay:

```sh
# before running Darktable:
$ find ~/.cache/darktable -type f | xargs cat > /dev/null
```

...actually that didn't do it; there's still a small lag. Oh well, worth a shot.
