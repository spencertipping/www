# Using ni to visualize audio files
**NOTE:** The image host I was using went down a while back; I need to
regenerate the screenshots. Sorry for the wall of text.

I wrote the [binary interfacing for
ni](https://github.com/spencertipping/ni/blob/develop/doc/binary.md#binary-decoding)
with the intention of doing signal processing, but until recently I haven't
used it much. This weekend I decided to try some spectral visualizations for
audio. Here's Bonobo's Black Sands album, log-frequency FFT at every
quarter-second:

![image](http://storage7.static.itmages.com/i/17/0506/h_1494093989_2534502_b5faae72de.jpeg)

## How to replicate this setup
1. Install [ni](https://github.com/spencertipping/ni). This is just one
   command, and ni has no dependencies.
2. Install Python+NumPy (`apt install python-numpy` on Ubuntu).
3. Install ffmpeg (`apt install ffmpeg` on Ubuntu).

I also use [youtube-dl](https://en.wikipedia.org/wiki/Youtube-dl) to get
compressed audio files.

Once you've installed that stuff, you should be able to run `ni --js` in a
terminal, pop open a browser to `localhost:8090`, and enter commands into the
top bar as shown in the screenshots. (You'll have to do some
panning/zooming/etc to get the view to line up correctly.)

## What's going on here
Let's start with the command line. I've got these basic steps:

- `/home/spencertipping/r/glacial/music/orig/bonobo-black-sands-2.ogg`: the
  compressed audio file
- `sr1[...]`: run `...` on the `r1` server using an SSH connection
  - `e[ffmpeg -i - -f wav -]`: decode ogg audio to wav
  - `bp'r rp"ss"'`: run perl in a binary context: extract two packed short ints
    per record and emit them as text
  - `r-100`: drop rows from the header, give or take
  - `pa+b`: add left+right channels together to get mono
  - `p'r $., pl(44100), rl(10000)'`: grab and emit windows of 44100 samples,
    and advance forward by 10000 samples per output row. Each output will be
    prefixed with the sample offset (`$.`, which is the input line number in
    Perl).
  - `S24[...]`: horizontally scale `...` by a factor of 24, since `r1` has 24
    processors
    - `NB'x = abs(fft.fft(x * sin(array(range(x.shape[1]))*pi / x.shape[1])**2))'`:
      windowed FFT of each row of samples; `sin(...)**2` is a Hann window
    - `p'r a, $F[$_], log $_ for 1..FM>>1'`: flatten the row of FFT outputs to
      a series of rows, each of the form `(time offset, amplitude,
      log(frequency))`. Take only the first half, since the second half is a
      mirror image.
    - `p'r a, (1-rand()*rand())*b/1000000, c for 1..b/1000000'`: a way to get
      the graphics to look better. We output multiple dots per FFT point,
      weighted by amplitude; this accentuates spikes.

Here's what it looks like to develop this pipeline:

### Step 1: audio samples
If you just have the audio samples and plot them in 2D, you'll see correlation
between left/right channels.

![image](http://storage1.static.itmages.com/i/17/0506/h_1494094637_5688458_19ffc0abf4.jpeg)

You can add the sample offset as the third dimension to see the waveform:

![image](http://storage7.static.itmages.com/i/17/0506/h_1494095964_3353417_8c4145a645.jpeg)

The intro also has some channel covariance caused by phase shifting:

![image](http://storage1.static.itmages.com/i/17/0506/h_1494096070_9303399_7b5c8b0c31.jpeg)

### Step 2: rows of sample windows
I've added some of the later commands to convert the data into something that
can be visualized.

At this point we have windows of audio in the time-domain. Windows overlap
about 80% with each other, and are just over one second long.

![image](http://storage6.static.itmages.com/i/17/0506/h_1494095296_4207275_53e85423cb.jpeg)

### Step 3: FFT with rectangular window
Each of the above sample windows gets individually Fourier-transformed.
Initially it doesn't look like much:

![image](http://storage8.static.itmages.com/i/17/0506/h_1494095502_8079697_8b07b8d8a1.jpeg)

To make this easier to parse, let's log-transform the frequency (that's how we
perceive pitch), chop off the FFT mirror image, and alias amplitude as color:

![image](http://storage1.static.itmages.com/i/17/0506/h_1494099405_2952072_cce25d4ea3.jpeg)

### Step 4: Highlighting peaks
We can't see much here because so many values are zero or near-zero. Let's use
`rp'b>1000000'` to remove small values:

![image](http://storage2.static.itmages.com/i/17/0506/h_1494099691_7978016_22145367fe.jpeg)

Now we need a way to make the peaks stand out. A simple strategy is to make
multiple copies of tall points, each jittered slightly downwards from the top.
I'm also going to divide each amplitude by 1000000 so the view axes are scaled
more evenly.

![image](http://storage3.static.itmages.com/i/17/0506/h_1494100141_7789332_d6217e7182.jpeg)

### Step 5: Hann windowing
Windowing helps focus the frequencies we're measuring, and it also narrows the
effective time range of each FFT row (which is good because they overlap). We
can generate a Hann window in numpy like this:

```py
# here, N is the window width in samples
hann = sin(array(range(N))*pi / N)**2
```

Here's what that window looks like:

![image](http://storage7.static.itmages.com/i/17/0506/h_1494100361_4240090_40f493b18a.jpeg)

And here's the log of the FFT:

![image](http://storage8.static.itmages.com/i/17/0506/h_1494100511_3997480_e5e7513490.jpeg)

For comparison, here's the log FFT of a rectangular window:

![image](http://storage9.static.itmages.com/i/17/0506/h_1494100691_3959704_e633564bcf.jpeg)

So when we multiply the signal by the Hann function, we'll eliminate a lot of
the edge artifact noise we'd have otherwise.

![image](http://storage8.static.itmages.com/i/17/0506/h_1494101036_9127080_fac73e0ffb.jpeg)

## Are different types of music visibly different?
### Bach Cello Suite
![image](http://storage7.static.itmages.com/i/17/0506/h_1494103922_2126006_26c18b70a0.jpeg)

Key change:

![image](http://storage5.static.itmages.com/i/17/0506/h_1494104437_7701830_0b1e84f207.jpeg)

Large-scale structure:

![image](http://storage6.static.itmages.com/i/17/0506/h_1494104630_8105947_a8effbe9aa.jpeg)

Harmonics during a scale (since the frequencies are log-transformed, they
appear to converge):

![image](http://storage7.static.itmages.com/i/17/0506/h_1494104708_2211608_86c9859802.jpeg)

### Beethoven's Ninth Symphony, mvmt 2
![image](http://storage2.static.itmages.com/i/17/0506/h_1494106709_4616716_ac13c20e92.jpeg)

### U2: Mysterious Ways
![image](http://storage1.static.itmages.com/i/17/0506/h_1494102140_4736862_0dd2d4a590.jpeg)

### Penguin Cafe Orchestra: Perpetuum Mobile
![image](http://storage2.static.itmages.com/i/17/0506/h_1494102699_1853566_e0df5678fe.jpeg)

### Molly Johnson: Must have Left My Heart
![image](http://storage5.static.itmages.com/i/17/0506/h_1494103523_9209194_fc8ba072d8.jpeg)

This one is kind of interesting because the bass drum occupies the same
frequency range as the electric bass line, but they appear distinct by looking
at timing:

![image](http://storage2.static.itmages.com/i/17/0506/h_1494103771_4048927_3049f89e53.jpeg)

### Adele: Cold Shoulder
![image](http://storage1.static.itmages.com/i/17/0506/h_1494106483_6619307_fb6a90cb83.jpeg)

### Norah Jones: Sunrise
![image](http://storage9.static.itmages.com/i/17/0506/h_1494108034_2373520_cdf5d62112.jpeg)

## What does compression look like?
I'm going to compress Cold Shoulder because it has frequencies at both
extremes.

### Original
![image](http://storage5.static.itmages.com/i/17/0506/h_1494108809_6176614_8717821472.jpeg)
![image](http://storage7.static.itmages.com/i/17/0506/h_1494108861_5969353_6542d2a754.jpeg)

### MP3, 128kbps
No noticeable differences:

![image](http://storage6.static.itmages.com/i/17/0506/h_1494109406_2061431_78d3ed9e9d.jpeg)
![image](http://storage7.static.itmages.com/i/17/0506/h_1494109449_8849695_b02d4e3ce2.jpeg)

### MP3, 64kbps
No visible differences here either, though 64kbps is definitely audible:

![image](http://storage9.static.itmages.com/i/17/0506/h_1494109941_7149809_9d52503577.jpeg)
![image](http://storage3.static.itmages.com/i/17/0506/h_1494109978_7560711_333a861524.jpeg)

### MP3, 32kbps
Definite differences here. Some high frequencies have disappeared, and the low
frequencies are now quantized:

![image](http://storage8.static.itmages.com/i/17/0506/h_1494110345_6954197_2dba09dfce.jpeg)
![image](http://storage1.static.itmages.com/i/17/0506/h_1494110398_7718929_02656309df.jpeg)
