# Using ni to visualize audio files
I wrote the [binary interfacing for
ni](https://github.com/spencertipping/ni/blob/develop/doc/binary.md#binary-decoding)
with the intention of doing signal processing, but until recently I haven't
used it much. This weekend I decided to try some spectral visualizations for
audio. Here's Bonobo's Black Sands album, log-frequency FFT at every
quarter-second:

![image](http://storage7.static.itmages.com/i/17/0506/h_1494093989_2534502_b5faae72de.jpeg)

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
![image]()

### U2: Mysterious Ways
![image](http://storage1.static.itmages.com/i/17/0506/h_1494102140_4736862_0dd2d4a590.jpeg)

### Penguin Cafe Orchestra: Perpetuum Mobile
![image](http://storage2.static.itmages.com/i/17/0506/h_1494102699_1853566_e0df5678fe.jpeg)

### 