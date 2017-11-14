# Random project: custom wallpapers from handwritten notes
I tend to have a bunch of loose pages of notes lying around, and a few days ago
Joyce scanned them in so I could store digital copies. Some fairly aimless
scripting later, I ended up with a bunch of generated backgrounds (thousands,
but these are some of the good ones):

![image](http://spencertipping.com/note-wallpaper-1.png)

![image](http://spencertipping.com/note-wallpaper-2.png)

![image](http://spencertipping.com/note-wallpaper-3.png)

## Extracting note images
Joyce used a ScanSnap, which is a two-sided sheet feed scanner that produces
PDFs. The first step was to pull the individual pages out using ImageMagick, and
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


