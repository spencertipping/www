# Random project: custom wallpapers from handwritten notes
I tend to have a bunch of loose pages of notes lying around, and a few days ago
Joyce scanned them in so I could store digital copies. Some fairly aimless
scripting later, I ended up with a bunch of generated backgrounds (thousands,
but these are some of the good ones):

![image](http://spencertipping.com/note-wallpaper-1.png)

![image](http://spencertipping.com/note-wallpaper-2.png)

![image](http://spencertipping.com/note-wallpaper-3.png)

![image](http://spencertipping.com/note-wallpaper-4.png)

![image](http://spencertipping.com/note-wallpaper-5.png)

## Extracting note images
Joyce used a ScanSnap, which is a two-sided sheet feed scanner that produces
PDFs. The first step was to pull the individual pages out using ImageMagick,
inverting each in the process:

```sh
$ convert -density 600 -trim input.pdf \
          -negate \
          -quality 95 output-%03d.jpg
```

My notes are awkward to work with digitally because I don't use the whole page
in a linear way. So each output file looks something like this:

![image](http://spencertipping.com/note-wallpaper-extracted-1.jpg)

I opened a bunch of them in the GIMP and selected/rotated/cropped to get
correctly-oriented note snippets like this:

![image](http://spencertipping.com/note-wallpaper-extracted-2.png)

The last change was to convert black to alpha and blur the images a bit.
Blurring hides the fact that the scanner dropped the images to monochrome, which
loses some detail for handwriting.

```sh
$ convert output-001.jpg -fuzz 10% -transparent black -gaussian-blur 10x2 \
          output-transparent-001.png
```

You might not be able to see this in github, but if you open the image
separately you can see white text against the browser's default PNG background:

![image](http://spencertipping.com/note-wallpaper-extracted-3.png)

I ended up with 126 of these from 104 scanned images (some of which were blank;
I'd guess there were about 80 actual pages of stuff).

## Compositing the background
Almost everything about these is random: the background photo, which note images
go where, the opacity, blurring, etc. It was a lot easier to automate this and
throw away 95% of the outputs than it would have been to lay it out by hand.

The basic idea is to composite the transparent images onto a darkened photo
using the `plus` blend mode. This uses a `convert` sub-context:

```sh
# composite a single note onto the background
$ convert photo.jpg \
          -resize 3840x2160 \
          -evaluate multiply 0.5 \
          \( note.png -resize 30% -geometry +400+600 \) \
          -compose plus -composite \
          +repage output.png
```

ImageMagick is concatenative, so if we want more than one note we can just add
more arguments, in this case `\( ... \) -compose plus -composite`. The generator
script generates the command-line arguments by repeatedly adding stuff to a bash
array.

## Parameter space
At first I just randomly generated each parameter on an individual basis, but
this tended to produce a bunch of wallpapers that looked more or less the same;
if you generate enough identically-distributed random numbers, you'll converge
to the mean. To avoid this, I started by generating some higher-order parameters
up front:

```sh
#!/bin/bash
# Usage: ./wallpaper > out.png

# Distribution parameters
opacity_floor=$(( 30 + $RANDOM % 50 ))
size_floor=$(( 4 + $RANDOM % 4 ))
size_scale=$(( 8 + $RANDOM % 16 ))

x_quantum=$(( 1 << $RANDOM % 8 ))
y_quantum=$(( 1 << $RANDOM % 8 ))

max_blur=$(( $RANDOM % 24 + 1 ))

declare -a composite_args
for f in `ls transparent-pngs/* | shuf | ni r.$(( 10 + $RANDOM % 39 ))`; do
  # single-image compositing code...
done

convert `find photos -name '*.jpg' | shuf | head -n1` \
        -resize 3840x2160 \
        -evaluate multiply 0.$(( 10 + $RANDOM % 80 )) \
        "${composite_args[@]}" +repage png:-
```

Just a heads-up about the `ni r.$((stuff))` in there: this is a quick way to
randomly select some fraction of input rows. I did it this way because I wanted
the process to scale up with the number of images available, which in hindsight
isn't a great idea. A better approach would be to replace this with just
`head -n $(( 20 + $RANDOM % 60 ))` or similar; then you've got a more stable
control over the density of notes you're mixing in.

## Single-image compositing
The contents of the for-loop generate a bunch of variates from the distributions
we set up in the header. Each image has a few variables:

- XY position, quantized to (`x_quantum`, `y_quantum`)
- Orientation
- Opacity, at least `opacity_floor`
- Size as a percentage of the original 600DPI, between `size_floor` and
  `size_floor + size_scale`
- Blur sigma, up to `max_blur` (which can be 0)

Here's the code:

```sh
# bash uses int-only math, so /q*q is a way to quantize something
px=$(( $RANDOM % 3600 / xq * xq ))
py=$(( $RANDOM % 3600 / yq * yq ))

opacity=0.$(( opacity_floor + $RANDOM % (90 - opacity_floor) ))
size=$(( size_floor + $RANDOM % size_scale ))
blur=0x$(( $RANDOM % max_blur ))

# this is a hacky way to get a biased distribution (in hindsight I should have
# parameterized it, but this configuration seemed reliably inoffensive)
case $(( $RANDOM % 64 )) in
1|2) rotation=90 ;;
3)   rotation=-90 ;;
4|5) rotation=180 ;;
*)   rotation=0 ;;
esac

composite_args+=( \
  \( $f -resize $size% \
        -rotate $rotation \
        -geometry +$px+$py \
        -channel a -evaluate multiply $opacity +channel \
        -bordercolor none -border 100x100 \
        -gaussian-blur $blur \) \
  -compose plus -composite )
```

It's important to expand the virtual canvas of images before you blur them;
otherwise you get blurs that run off the edges and create sharp lines.

Also note the hack to get floating-point values; it's totally possible to use
integers to do this as long as they're all in the same order of magnitude.

## Rendering a bunch of these
This is a pretty CPU-bound process for a number of reasons, but not all of it is
parallelized by default. This means there's some speedup to be had by using
`xargs`:

```sh
$ seq 1000 | xargs -P12 -I{} sh -c 'echo {}; ./wallpaper > results/{}.png'
```

Each wallpaper takes a few minutes of CPU time on the server.

## Flying blind: examples of fail
Throwing random parameters at a problem like this is a high-recall,
low-precision strategy, which is ok because it's quick to go through the images
and throw out the ones I don't like (or more accurately, rescue the few that I
do). Here are some that didn't make the cut:

![image](http://spencertipping.com/note-wallpaper-bogus-1.png)

![image](http://spencertipping.com/note-wallpaper-bogus-2.png)

![image](http://spencertipping.com/note-wallpaper-bogus-3.png)

![image](http://spencertipping.com/note-wallpaper-bogus-4.png)
