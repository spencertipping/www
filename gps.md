# GPS accuracy
An experiment from around 2015 while I was working with GPS data at Factual.

I work with a lot of GPS data at my day job, and I always wondered how accurate
it was in practice. Devices will report things like "within 4-8 meters," but in
many ways that's an ideal case; what about noise or reflected signals? So I
started doing a bunch of experiments to figure it out.

## First experiment: overnight home location recording
I installed a [GPS
logger](https://play.google.com/store/apps/details?id=com.mendhak.gpslogger&hl=en)
on my phone and used it to record my phone's position every second as it
charged overnight. Due to the lack of major tectonic activity in Albuquerque,
the phone remained stationary during this recording.

I started with the GPX trace from the phone and converted it to TSV of (lat,
lng, timestamp, source):

```sh
$ sudo cpan install Date::Parse
$ nfu --run 'use Date::Parse' *.gpx \
      -m 'row %0 =~ /^.trkpt.*lat="([^"]+).*lon="([^"]+)".*<time>(.*)<\/time>.*<src>(.*)<\/src>/, $ARGV =~ s/\..*$//r' \
      -k1m 'row %0, %1, str2time(%2), %3, %4' -o2 --pipe uniq \
  | gzip > trace.gz
```

I used Google Maps' satellite view to find the coordinates of the phone inside
(to within a meter or so):

![phone location](https://tipping.haus/gpsa-home-charging-location.png)

### Initial results
Here are the traced data points sourced from the GPS. The plus-sign on the map
is where the phone was physically located (35.194193, -106.481589):

```sh
$ nfu trace.gz -k '%3 eq "gps" && %4 eq "20150616"' \
      -nm 'row %0 % 2 ? 35.194193 + rand() * 0.00005 - 0.000025 : 35.194193,
               %0 % 2 ? -106.481589 : -106.481589 + rand() * 0.00005 - 0.000025,
               %1,
               %2' \
      -m 'row(%1, %0), row(%3, %2)' -p %d
```

![phone gps](https://tipping.haus/gpsa-home-tracemap.png)

The most distant point was 35.5m away:

```sh
$ nfu trace.gz -k '%3 eq "gps" && %4 eq "20150616"' \
               -m 'edist(%0, %1, 35.194193, -106.481589)' -OT1
0.0355021736061369      # km
```

Some weird things about this data:

1. It's really biased; almost all of the error is directed southeast.
2. Android seems like it's inferring motion, probably from random GPS jitter.
   (**NOTE:** I found out later that Android doesn't do this at all; these are
   proper GPS hardware signals with no motion inference. This means that GPS is
   very low-variance; the error is all bias.)

Here's a map of the location deltas (by ascending time):

```sh
$ nfu trace.gz -k '%3 eq "gps" && %4 eq "20150616"' -f10S01D1p %d
```

![gps deltas](https://tipping.haus/gpsa-home-deltas.png)

Here's the absolute error rate (x axis = meters, y axis is frequency):

```sh
$ nfu trace.gz -k '%3 eq "gps" && %4 eq "20150616"' \
      -m 'edist(%0, %1, 35.194193, -106.481589) * 1000' \
      -q0.1ocf10p %l
```

![error rate](https://tipping.haus/gpsa-home-error-rate.png)

So a typical error amount is 20 meters, quite a lot more than the phone's
estimated 4-8 meter accuracy. It's clear there's some kind of interference
going on, so I set up another log from the other end of the house to see
whether the errors were consistent.

### Trying again on the other side of the house
Just in case.

![west location](https://tipping.haus/gpsa-home-location2.png)

```sh
$ nfu trace2.gz -k '%3 eq "gps"' \
      -nm 'row %0 % 2 ? 35.194130 + rand() * 0.00005 - 0.000025 : 35.194130,
               %0 % 2 ? -106.481832 :  -106.481832 + rand() * 0.00005 - 0.000025,
               %1,
               %2' \
      -m 'row(%1, %0), row(%3, %2)' -p %d
```

![west errors](https://tipping.haus/gpsa-west-errors.png)

Looks surprisingly similar. Here's the delta map:

```sh
$ nfu trace2.gz -k '%3 eq "gps"' -f10S01D1p %d
```

![west deltas](https://tipping.haus/gpsa-west-deltas.png)

Ok, so the error is not being caused by the house itself (and probably also not
by sporadic ionospheric interference, since the logs were from two separate
nights); the most likely explanation is some type of environmental
interference.

## Satellite visibility experiment
I live in the foothills, so we've probably got all kinds of interference from
terrain. This comes in two forms: sometimes the satellite signals will just be
blocked, and sometimes they'll be reflected. Reflected interference is harder
to analyze, so I decided to start by just mapping out the horizon angles and
correlating the moments of error with satellites' horizon intersections.

### Measuring satellite visibility
GPS satellites have 12-hour orbits, which matters because it means that any
signal interference pattern will be time-dependent. I live in the foothills, so
the horizon isn't flat; in order to know which satellites were visible at any
moment in time, we need to digitize the horizon line.

#### Camera-mount transit
I built a horizontal transit to hold the phone and took a 360-degree video,
marking every 15-degree increment. The front of the transit has a ruler spaced
32 inches from the phone, so some simple trig will establish baseline angles. I
used the tower as the zero-degree point because I could later find its exact
coordinates on a satellite map (in practice it's 112 degrees from north, an
adjustment factor I apply just before saving the horizon profile).

![camera transit](https://tipping.haus/gpsa-camera-transit.jpg)

![tower coordinates](https://tipping.haus/gpsa-survey-tower.png)

Vertically calibrated with incredible precision...

![camera vertical](https://tipping.haus/gpsa-camera-transit-vertical.jpg)

Here are some examples of the measurement photos, going clockwise from the
tower:

![measurement 0](https://tipping.haus/gpsa-transit-snap0.png)
![measurement 1](https://tipping.haus/gpsa-transit-snap1.png)
![measurement 2](https://tipping.haus/gpsa-transit-snap2.png)
![measurement 3](https://tipping.haus/gpsa-transit-snap3.png)

My hand is in some of them because I was indicating the horizontal angles in
the original video. The first thing to do is load one up in Octave and
experiment with some edge detection strategies until we get one that highlights
the terrain properly. I started by converting to grayscale and using default
edge detection:

```matlab
$ sudo apt-get install octave-image
$ octave
> i = imread("gpsa-measurement-0.png")
> imshow(edge(rgb2gray(i)))
```

![edges](https://tipping.haus/gpsa-edge1.png)

Nowhere near good enough for automatic line extraction, so I did some
experimentation and ended up with this:

```matlab
> [dx, dy] = gradient(rgb2hsv(i)(:,:,3))        % 3 = value channel
> imshow(abs(dy) * 5)                           % 5 just helps visibility
```

![better edges](https://tipping.haus/gpsa-edge2.png)

Once the camera was facing away from the sun, though, the value channel became
insufficient:

![blurry edge](https://tipping.haus/gpsa-edge3.png)

I fixed it by using the hue instead:

```matlab
> [dx, dy] = gradient(rgb2hsv(i)(:,:,1))        % 1 = hue
> imshow(abs(dy) * 5)
```

![better edge](https://tipping.haus/gpsa-edge4.png)

#### Lens distortion
As a quick aside, before we rely too much on any line tracing from the camera
we need to invert any distortion caused by the lens; typically this will either
be pincushion or barrel distortion. Either of these will deform a grid, so I
found one online and took a picture, then copied cells to other locations to
check for consistency:

![lens distortion test](https://tipping.haus/gpsa-distortion-test.jpg)

This is really good; there's a little bit of pincushion distortion towards the
middle and some barrel distortion at the corners, but for the most part sizes
and lines are preserved (there's probably some software correction happening).
Other sources of error will probably dominate here.

#### Line tracing
After more experimentation than I care to admit, I ended up with a fairly
straightforward line tracing function that operates on the signed gradient
`dy`. The idea here is to track the maximum value in each column of the image,
expecting vertical deviations with Gaussian-distributed probabilities. The core
of the algorithm looks like this:

```matlab
[rs cs] = size(m);
for i = 1:cs
  [y, yi]   = max(m(:, i) .* normpdf(1:rs, yi, sigma));
  result(i) = yi;
endfor
```

The full source for the line-tracing utility is in the attached
`trace-profile`. Most of the images came out perfectly (we only need stuff to
the right of the ruler):

```sh
# if invoked with a number, trace-profile debugs just that image
$ octave --persist trace-profile 1
```

![good alignment](https://tipping.haus/gpsa-good-alignment.png)

Only one of them didn't; the pattern of rocks created a high-contrast path that
diverted the measurement.

![bogus alignment](https://tipping.haus/gpsa-bogus-alignment.png)

Fixing it was just a matter of convolving with a narrow Gaussian; this removed
enough detail that the tracer took the upper path.

```matlab
@(i) conv2(rgb2hsv(i)(:,:,3), normpdf(1:10, 5, 2))(:,5:end-5);
```

![fixed alignment](https://tipping.haus/gpsa-fixed-alignment.png)

#### Alignment
Each image is a plane (as opposed to the spherical horizon profile we're
ultimately modeling), so any affine transformation will introduce
perspective-projection distortion since the planes have different angles. The
way to fix this is to transform the images into cylindrical coordinates; that
way all shifts are in terms of angles and are uniform.

The 16" ruler measures 670 pixels and is 32" away from the camera, so the view
plane is 25.8" tall by 45.85" wide at 32". Solving for pixels gives 2680.

```sh
$ nfu profile.gz \
      -m 'my $z0 = 2680;
          my $r  = dist(%1 - 960, $z0) / $z0;
          row %0, %1 / $r, %2' \
      --partition %0 \
        [ --octave "xs = [xs(1:1750,1), \
                          (1:1750)', \
                          interp1(xs(:,2), xs(:,3), 1:1750, \"spline\")']" ] \
      -o01 \
  | gzip > cylindrical.gz
```

Now let's grab everything to the right of the ruler (about X = 1000) in each
image. This will result in a bunch of curves, each the same size and valid at
every point.

```sh
$ nfu cylindrical.gz -k '%1 >= 1000' \
                     --partition %0 'nfu -f12 | gzip > right-{}.gz'
```

We have three degrees of freedom due to mechanical imprecision in the transit:

![dof diagram](https://tipping.haus/gpsa-dof-diagram.png)

Since the curves overlap, the goal is to solve for these variables by
minimizing error. The first step is to find the overlap amount by generating
micro-adjustments from 380, a reasonable baseline. I'm doing this by taking
cumulative sums of dy² and measuring the average X distance between those
curves.

```sh
$ nfu cylindrical.gz -m 'row %0, "right-" . %0              . ".gz",
                                 "right-" . (%0 + 15) % 360 . ".gz"' \
      -ocf1.k -e%2 \
      -m "row %0, %1, %2, 380 + int
            qx[nfu %1 -S1D1m 'row \%0, \%1 ** 2' \
                      -i0 @[ %2 -m 'row \%0 + 750 - 380, \%1' \
                                -S1D1m 'row \%0, \%1 ** 2' -s1 ] \
                      -s1 \
                      --octave \"xs = [interp1(xs(:,2), xs(:,1), (0:1000))', \
                                       interp1(xs(:,3), xs(:,1), (0:1000))']\" \
                      -nK 'grep /NA/, \@_' -m '\%2 - \%1' -a0T+1]" \
  | gzip > dx.gz

# check the alignment:
$ nfu dx.gz \
      -nm "map row(\$_),
               rl qq[sh:nfu %2 -m 'row \%0 + 750 * %0 - 750, \%1' \
                     -I0 @[ %3 -m 'row \%0 + 750 * %0 -  %4, \%1' ]]" \
      -m 'map row(%0, -$_), @_[1..$#_]' -p %d
```

![valigned curves](https://tipping.haus/gpsa-valigned-curves.png)

Looks good. Now let's use those overlaps to solve for dy and dθ. This is a
little tricky because if we're not careful we'll infer some crazy angles for
stuff. To prevent bias in dθ, we first solve for dys such that ∑dy = 0.

```sh
$ nfu dx.gz \
      -m "map row(%0, \$_),
              rl qq[sh:nfu %1 -m 'row \%0 + 750 * %0 / 15 - 750, \%1' \
                    -i0 @[ %2 -m 'row \%0 + 750 * %0 / 15 -  %3, \%1' ]]" \
      --partition %0 [ \
        -f123o --octave "dy = mean(xs(:,3) - xs(:,2)); \
                         dx = xs(end, 1)   - xs(1,1); \
                         s1 = xs(end, 3) - xs(end, 2); \
                         s0 = xs(1,   3) - xs(1,   2); \
                         xs = [xs(1,1), \
                               mean(xs(:,1)), \
                               mean(xs(:,3)), \
                               dy, \
                               (s1 - s0) / dx]" \
        -m 'row {}, @_' ] \
      -m 'row @_[0..4], (rect_polar %5, 1)[1]' \
      -i0 @[ dx.gz -f032 ] -o \
  | gzip > solved-initial.gz

$ nfu solved-initial.gz \
      -m "%4 -= `nfu solved-initial.gz -f4a0T+1`;
          %5 -= `nfu solved-initial.gz -f5a0T+1`;
          row @_" \
      -s46a5,0 \
      -nm 'map row(@_, $_), rl %8' \
      -m  'my ($r, $h) = rect_polar %9 - (%3 - %2) - 1000, $_[10] - %4;
           my ($x, $y) = polar_rect $r, $h + %6;
           row %1, $x + 750 * %0 - %7, $y + %4 - %5' \
  | gzip > horizon.gz

# check alignment again:
$ nfu horizon.gz -m 'row %1, -%2' -p %d
```

![aligned curves](https://tipping.haus/gpsa-aligned-curves.png)

This really isn't bad considering how egregious the setup was. We can finalize
the profile by inserting the run of zero data between 45 and 165 degrees,
vertically averaging, and converting the pixel angles to real degrees. The cell
camera is level at 7/8" on the ruler, which appears at pixel 655 from the top
of the image.

```sh
# how many pixels wide is the horizon image?
$ nfu dx.gz -f3m '750 - %0' -a0T+1m '%0 / 15 * 360'
8766
```

The "0" measurement image was aligned to pixel 5638 against the 345 image. 960
pixels across the image marks the zero point, so we now adjust 40 pixels to the
left to correct for the 1000 floor we used earlier. (We couldn't have used 960
for image processing because the ruler isn't perfectly vertical and we need
some margin.) While we're at it, we account for the 112-degree angle from due
north to the tower I used for zero.

```sh
$ nfu horizon.gz \
      -m 'row((%1 + (%0 >= 165 ? 8766 - 5638 : 0) + 40
              + 112 / 360 * 8766) % 8766,
              max 0, (rect_polar 655 - %2, 2680 / 2)[1])' \
      -q1oA 'row $_, mean @{%1}' \
      --octave "xs = interp1(xs(:,1), xs(:,2), (1:8765)')" \
      -nm 'row %0 / 8766 * 360, %1' \
  | gzip > horizon-final.gz
```

![final horizon](https://tipping.haus/gpsa-horizon-final.png)

Awesome. Now it's time to get satellite positions for each moment in time and
figure out when each one was visible.

### A quick aside: orbital geometry and coordinate systems
When satellites are launched, their orbits are specified with respect to the
celestial sphere rather than to Earth's surface (the obvious exception being
geosynchronous orbits). As a result, there are a few different coordinate
systems people use to refer to things in space:

- [ECI](https://en.wikipedia.org/wiki/Earth_Centered_Inertial) (Earth-centered
  inertial): The X axis is aligned to the celestial sphere, the Z axis to
  Earth's rotational axis. As a result, the X-Y plane contains the equatorial
  ring. Orbital models like [SGP4](https://en.wikipedia.org/wiki/SGP4) return
  coordinates in ECI.
- [ECEF](https://en.wikipedia.org/wiki/ECEF) (Earth-centered, earth-fixed): This
  is just like ECI, but the X and Y axes are defined with respect to points on
  the earth's surface. Converting from ECI to ECEF involves calculating the
  earth's celestial rotation, called
  [sidereal time](https://en.wikipedia.org/wiki/Sidereal_time), which differs
  from solar time because during one solar day the earth progresses by almost
  one degree in its orbit around the sun.
- [Azimuth/elevation/distance](https://en.wikipedia.org/wiki/Azimuth) (look
  angles): Ground-referenced coordinates from a given surface location on Earth.
  This is what we need to figure out whether the horizon we built in the last
  post obscures a satellite at any given moment.

If you look at some of the
[ISS time-lapse videos](http://eol.jsc.nasa.gov/BeyondThePhotography/CrewEarthObservationsVideos/Africa.htm),
particularly the ones at night, Earth appears to be rotating independently of
the celestial sphere because while the ISS orbit would look like a flat ring in
ECI, it would look quite curved in ECEF.

### Generating the satellite positions
I downloaded the [TLE orbit
descriptions](https://en.wikipedia.org/wiki/Two-line_element_set) for the GPS
satellites from [NORAD](http://www.celestrak.com/NORAD/elements/gps-ops.txt),
then used [satellite.js](https://github.com/shashwatak/satellite-js) to generate
azimuth/elevation/distance for each second the phone was recording.

```sh
$ sudo apt-get install nodejs
$ sudo npm install satellite.js
$ curl http://www.celestrak.com/NORAD/elements/gps-ops.txt \
  | sed -r '/^GPS/ {s/^/"/;  s/\s*$/":/}
            /^1/   {s/^/["/; s/.$/",/}
            /^2/   {s/^/"/;  s/.$/"],/}
            1      {i\
{
            }
            $      {s/,$//; a\
}
            }' \
  > gps-tle.json
```

Now let's write up a quick driver script:

```js
// satellite-locations.js
var fs        = require('fs');
var satellite = require('satellite.js').satellite;
var tau       = Math.PI * 2;
var tles      = JSON.parse(fs.readFileSync('gps-tle.json', 'utf8'));
var sats      = {};

var observer  = {
  latitude:    35.194193 / 360 * tau,  // radians
  longitude: -106.481589 / 360 * tau,  // radians
  height:       2.0                    // km above sea level
};

for (var k in tles) sats[k] = satellite.twoline2satrec(tles[k][0], tles[k][1]);
for (var t = +process.argv[2]; t <= +process.argv[3]; ++t) {
  var d = new Date(t * 1000);
  for (var k in sats) {
    var pv = satellite.propagate(sats[k],
              d.getUTCFullYear(), d.getUTCMonth() + 1, d.getUTCDate(),
              d.getUTCHours(),    d.getUTCMinutes(),   d.getUTCSeconds());
    var gmst = satellite.gstimeFromDate(
                d.getUTCFullYear(), d.getUTCMonth() + 1, d.getUTCDate(),
                d.getUTCHours(),    d.getUTCMinutes(),   d.getUTCSeconds());

    var ecef = satellite.eciToEcf(pv.position, gmst);
    var look = satellite.ecfToLookAngles(observer, ecef);
    var tsv  = [k, t,
                look.azimuth * 360 / tau, look.elevation * 360 / tau,
                                          look.rangeSat,
                pv.position.x, pv.position.y, pv.position.z,
                ecef.x,        ecef.y,        ecef.z].join("\t");

    // Blocking writes are important; otherwise we'll run out of memory if the
    // downstream process is slower than we are (and in this case, it is).
    fs.writeSync(1, tsv + "\n");
  }
}
```

The output format here is TSV like this, where `LOOK`, `ECI`, and `ECEF` each
comprise three columns:

```
sat_name  timestamp  LOOK  ECI  ECEF

LOOK = azimuth_degrees  elevation_degrees  distance_km
ECI  = x  y  z
ECEF = x  y  z
```

We don't really need ECI or ECEF for this analysis, but I wanted to export them
for debugging, just in case.

Let's generate satellite locations for times when the phone was recording:

```sh
$ nfu trace.gz -f2o --duplicate ^T1 ^T+1 -o
1433801944              # minimum
1435108548              # maximum
$ nodejs satellite-locations.js 1433801944 1435108548 \
  | gzip > satellite-angles.gz
```

Let's take a quick look at some satellite positions in ECI:

```sh
$ nfu satellite-angles.gz -E100f567 --splot %d
```

![orbit diagram](https://tipping.haus/gpsa-orbit-diagram.png)

Here's what they look like in ECEF:

```sh
$ nfu satellite-angles.gz -E100f8. --splot %d
```

![orbit diagram](https://tipping.haus/gpsa-orbit-diagram-ecef.png)

### Horizon and visibility
Right now we have a bunch of azimuth+elevation coordinates, but some of these
fall below the horizon. In that case the GPS signal would be unavailable. We can
filter out those points using the horizon profile:

```sh
$ nfu --run '/(\S+)\t(\S+)/ and $::h{$1} = $2
             for rl
               q{sh:nfu horizon-final.gz -q0.1oA "row %0[0], mean @{%1}"}' \
      satellite-angles.gz -q2,0.1k '%3 >= $::h{%2}' \
  | gzip > visible-angles.gz
```

Checking the ECEF to make sure it looks reasonable (we should see a partial
hemisphere):

```sh
$ nfu visible-angles.gz --sample 0.01 -f8. > visible-hemisphere
$ gnuplot <<'EOF'
set terminal gif animate delay 5 optimize size 856,720
set output "visible-hemisphere.gif"
do for [i = 0:360] {
  set view 75,i
  splot "visible-hemisphere" with dots title "" lc "red"
}
EOF
```

![](https://tipping.haus/gpsa-visible-hemisphere.gif)

### Correlating satellite visibility with error
The first thing to do here is to join one of the GPS traces to the visible
satellite list by timestamp:

```sh
$ nfu gpsdata/20150616trace.xz \
      -k '%3 eq "gps"' -i2 @[ visible-angles.gz -f10. ] \
  | gzip > 20150616-satellite.gz
```

Now we can construct a preliminary time-series plot of satellite visibility
windows and absolute error, which I've annotated with some vertical lines. Time
is on the X axis. The bottom half is error over time, and the top half contains
one horizontal line per satellite -- shaded in moments when that satellite is
visible.

```sh
$ nfu 20150616-satellite.gz \
      -z4m 'row(%0, 1000 * edist %1, %2, 35.194193, -106.481589),
            row(%0, 30 + %4)' -p %d
```

![satellite error
correlation](https://tipping.haus/gpsa-satellite-error-correlation.png)

This doesn't line up particularly well, so let's see whether it could be a
byproduct of measurement errors. The gnuplot image above represents the trace
across 1254 horizontal pixels.

```sh
$ nfu 20150616-satellite.gz -f0 --duplicate ^T1 ^T+1 -oSD1
30576               # total duration of the trace (seconds)

$ nfu 20150616-satellite.gz -f406zK0f12S01D1m 'row %0, abs(%1)' \
                            -s01T+1m 'row %1 / %0'
0.00414757596657057 # average d(|elevation|)/dt for visible arc of sat 0

# How many degrees of elevation error are implied by each horizontal pixel of
# difference? (1254 is unitless because GNU units doesn't know what pixels are.)
$ units -t '0.00414 deg/s * 30576s / 1254' deg
0.10094469
```

Given how the horizon was measured we might be off by as much as three degrees
vertically, which means 30 pixels here. Most of the alignment errors are much
less than that, so I think we might be ok. Some of the errors might also be
caused by reflective interference while the satellite remains visible; let's
look at that next.

### Correlating satellite positions with error
Although there are some hills near the house, none obscure a substantial
fraction of the sky by area. This means that the GPS receiver probably has
enough satellites to get a fairly good signal the whole time; errors wouldn't be
caused by
[dilution of precision](https://en.wikipedia.org/wiki/Dilution_of_precision_%28GPS%29)
as much as they would by
[multipath artifacts](https://en.wikipedia.org/wiki/Error_analysis_for_the_Global_Positioning_System#Multipath_effects).
This also makes sense given that (1) the phone thinks the accuracy is higher
than it is, and (2) phones tend to have inexpensive, low-power receivers that
would be less likely to do effective multipath filtering.

If we model it as a ray tracing problem, multipath effects should correspond to
satellite sky positions with consistently large ground errors. Here's a graph of
ground error (color and distance: blue is more, yellow is less) for each
satellite position:

```sh
# Visualize the horizon profile so we can more easily tell which way is north
# (the mountains are to the east)
$ nfu horizon-final.gz \
      -m 'my ($x, $y) = polar_rect 1, %0;
          my ($r, $z) = polar_rect 1, 90 - %1;
          $. % 10 ? row $x * $r, $y * $r, $z
                  : map row($x * $r, $y * $r, $_ / 40), 0..int($z * 40)' \
  > horizon-trace

$ nfu 20150616-satellite.gz \
      -f5612m 'row %0, %1, rect_polar %2 - 35.194193, %3 - -106.481589' \
      -m 'my ($z, $r) = polar_rect 0.3 + 2e3*%2, %1;
          my ($x, $y) = polar_rect $r, %0;
          row $x, $y, $z, 0.00035 - %2' \
      --append @[ horizon-trace -m 'row @_, 0' ] \
  > 20150616-satellite-error

$ gnuplot <<'EOF'
unset colorbox
unset border
unset tics
set terminal gif animate delay 5 optimize size 856,480
set output "20150616-error-animation.gif"
do for [i = 0:360] {
  set view 55,i
  splot "20150616-satellite-error" with dots lc palette title ""
}
EOF
```

![image](https://tipping.haus/gpsa-error-animation.gif)

**TODO:** Be a little more resourceful here; given the nature/direction of the
multipath error, I think we can still do the terrain surface mapping stuff.

Nothing very informative here either. Although some satellite positions
correspond to lower ground error, because the satellites are locked relative to
one another we can't determine which one is responsible for the multipath
problems from this data alone. All we really know is that the ground errors do
in fact correspond to satellite locations, so they're inherently predictable
(though possibly very chaotic).

## Experiment to nowhere: multipath tracing
**NOTE:** This was a fun but misguided detour. Doing this properly would
involve a much more complete 3D model of the surrounding terrain, but I haven't
gotten a quadcopter to build it yet. A lot of the video processing stuff would
apply in either case, though, particularly on an unstable device like a
quadcopter.

This is where things start to get challenging. In order to predict how
multipath interaction is going to work, we need to first construct a 3D model
of the landscape. To do this, I used a rooftop crane I had made earlier:

![roof crane](https://tipping.haus/gpsa-parallax-device.jpg)

The idea here is to attach the camera 9ft away from the hinge point and sweep
it across a circular path. The hinges are precise enough that the resulting
images should be mostly coplanar, and since we know the radius we can recover
the horizontal displacement by comparing angles. Here are some stills from the
resulting video:

![beginning of arc](https://tipping.haus/gpsa-parallax-1.png)
![middle of arc](https://tipping.haus/gpsa-parallax-4.png)
![end of arc](https://tipping.haus/gpsa-parallax-7.png)

One problem with these pictures, or really with the video, is that my cell
phone has a cheap camera that captures frames using a rolling shutter. This
means that any vibration will result in distortion of the image, since each row
of pixels is offset in time. Luckily we can fix the issue by disassembling the
video into its constituent frames and reverse time-shifting each row of pixels
by its offset.

### Rolling shutter correction
Mathematically speaking, if the shutter rolls over a time window of `Δt` then
the i<sup>th</sup> row of pixels from the top is displaced by `Δt * i / h`. The
rolling shutter in my camera might not be perfectly uniform with respect to the
video framerate, so I recorded a board spinning on a drill to check. I was
willing to assume a constant rotational speed for the board because it's
lightweight and mostly balanced (which means the primary force acting on it
should be air resistance). Here's a frame from the video:

![drill frame](https://tipping.haus/gpsa-drill-frame.png)

The rolling shutter produces the curved distortion here. Here's the next frame:

![drill frame 2](https://tipping.haus/gpsa-drill-frame2.png)

I need to solve for `Δt` as a fraction of the framerate. A simple strategy is
to take all measurements in terms of rotation of the board. I can't measure it
using the screw head because air resistance might cause some slip; instead I'll
take a segment of video where the camera is reasonably still and
Fourier-transform a rectangle around the drill light. The result will be a mode
at twice the rotational frequency of the drill.

```sh
$ mkdir drill-frames
$ avconv -ss 9 -i drill-video.mp4 -t 10 -f image2 drill-frames/%03d.jpg
$ nfu n:1 --octave "for i = 1:150
                      img      = imread(sprintf(\"drill-frames/%03d.jpg\", i));
                      means(i) = mean(mean(img(594:680, 906:1012, 2))');
                    endfor;
                    xs = abs(fft(means'))" \
          -p %i
```

![drill fft](https://tipping.haus/gpsa-drill-fft.png)

The mode is 62 over 150 frames, so the board's speed is 74.4 degrees per frame.
Now we just need to measure the rotation between two shutter positions:

![drill angle
measurement](https://tipping.haus/gpsa-drill-angle-measurement.png)

So in 1010 vertical pixels the board has rotated about 40 degrees, which gives
a total of 44.772 degrees of rotation within each frame. This means the shutter
takes 0.602 frames to hit the bottom of the image, and there's a 0.398-frame
delay before the next scan starts. Now we know the time offset of each row of
pixels. Unlike above, I need to extract the parallax frames using a lossless
image encoding.

```sh
$ mkdir parallax-frames
$ avconv -ss 129 -i parallax-video.mp4 -t 17 -f image2 parallax-frames/%04d.png
```

There are 510 images, each of which requires 1920 x 1080 x 3 bytes = 6.22MB, so
I'd have to close some Chrome tabs to load them all into memory at once. Rather
than do that, a more feasible strategy is to use nfu to rearrange the rows
among Octave subprocesses, each of which handles either one row across all
images, or one image. The intermediate sorts make sure `--partition`
subprocesses aren't restarted due to discontinuous data.

```sh
$ mkdir parallax-fixed
$ nfu sh:'ls parallax-frames' -om 'row %0 =~ /^0*(\d+)/' \
      --partition %0 [ \
        --octave "i  = double(imread(sprintf(\"parallax-frames/%04d.png\", {})));
                  id = [(1:1080)', ones(1080, 1) * {}];
                  xs = [ones(1080, 1) * 1, id, i(:,:,1);
                        ones(1080, 1) * 2, id, i(:,:,2);
                        ones(1080, 1) * 3, id, i(:,:,3)];" ] \
      -o01 --partition "%0,%1" [ \
        -o2 \
        --octave "dt = 0.602 * (xs(1,2) - 1) / 1080.0;
                  xs = [xs(:,1:3), interp1(xs(:,3) + dt,
                                           xs(:,4:end),
                                           xs(:,3),
                                           \"spline\")]" ] \
      -o2 --partition %2 [ \
        --octave "for c = 1:3
                    rs = find(xs(:,1) == c);
                    i(xs(rs,2), :, c) = uint8(xs(rs, 4:end));
                  endfor
                  imwrite(i, sprintf(\"parallax-fixed/%04d.png\", xs(1,3)));
                  xs = []" \
        -R /dev/null ]
```

### 3D reconstruction
The basic idea at this point is that we have a moving vantage point from which
coplanar images are provided. We know the exact coordinates of each frame with
minimal effort because they all lie along the same arc and are rotated
proportionally to the arc distance. There are some challenges, however:

1. The rolling shutter correction wasn't perfect, probably in part because the
   source frames were motion-compressed by the video encoder. This means
   there's still some distortion.
2. The frames are not truly coplanar, though they're probably close.
3. Any given frame may have some degree of directional error within the arc
   path due to vibrations in the rotating arm or shifting of the base.
4. The hill we're digitizing has trees in front of it.

The most immediate challenge is to recover the rotational angle of each image,
which we can do using [phase
correlation](https://en.wikipedia.org/wiki/Phase_correlation) in cropped
[log-polar space](https://en.wikipedia.org/wiki/Log-polar_coordinates). I use
the value channel because video codecs tend to preserve its high-frequency
signal (in hindsight I should have also done the rolling-shutter interpolation
in HSV terms for the same reason).

#### Log-polar transformation
We transform an image into log-polar space by resampling it radially, which is
easy in Octave if we use `interp2`. Here's an example of what this does to the
first video frame:

```matlab
$ octave
> function r = lp(i)
    [h, w, c] = size(i);
    thetas    = linspace(0, 2*pi, h)(1:end-1);
    rhos      = log(1:h);
    rhos     *= h/2.0 / max(rhos);
    [tt, rr]  = meshgrid(thetas, rhos);
    [xs, ys]  = pol2cart(tt, rr);
    for c = 1:3
      r(:,:,c) = interp2(i(:,:,c), xs + w/2.0, ys + h/2.0);
    endfor
  endfunction
> imshow(lp(imread("parallax-fixed/0002.png")))
```

![logpolar example](https://tipping.haus/gpsa-logpolar-example.png)

Log-polar space is useful because like polar coordinates, an image rotation
becomes a log-polar translation. This makes it possible to use FFT-based phase
correlation to recover the rotational delta, which we do in batch here:

```sh
$ mkdir parallax-logpolar
$ nfu n:1 \
      --octave "function r = lp(i)
                  [h, w, c] = size(i);
                  thetas    = linspace(0, 2*pi, h)(1:end-1);
                  rhos      = log(1:h);
                  rhos     *= h/2.0 / max(rhos);
                  [tt, rr]  = meshgrid(thetas, rhos);
                  [xs, ys]  = pol2cart(tt, rr);
                  for c = 1:3
                    r(:,:,c) = interp2(i(:,:,c), xs + w/2.0, ys + h/2.0);
                  endfor
                endfunction
                base_i = lp(imread('parallax-fixed/0002.png'));
                base_f = fft(rgb2hsv(base_i)(1:10,:,3));
                for i = 3:510
                  alt_i    = lp(imread(sprintf('parallax-fixed/%04d.png', i)));
                  alt_f    = conj(fft(rgb2hsv(alt_i)(1:10,:,3)));
                  p        = base_f .* alt_f;
                  r        = ifft(p ./ abs(p));
                  [m, mi]  = max(r(:));
                  [mt, mr] = ind2sub(size(r), mi);
                  ys(i,:)  = [mt, mr];
                  imwrite(alt_i, sprintf('parallax-logpolar/%04d.png', i));
                endfor
                xs = ys;" \
  | gzip > parallax-lp-shifts.gz
```

**TODO:** finish this

## Experiment, possibly to somewhere: using OSM data
At this point I'm confident that a significant amount of GPS error comes from
landscape features, even in places without a lot of buildings nearby. Since the
ultimate goal is to end up with a way to correct for this error, we need to
either predict the error from known location features or use an empirical model
to measure it. OpenStreetMaps gives us a way to do the latter (and possibly the
former too).

OSM provides both a world map and user-submitted GPX traces. The GPX traces are
timestamped, which is crucial here: we can look for systematic divergence based
on satellite positions. We need to have more than one GPX trace for a given OSM
feature because
[OSM uses GPS to correct their maps](https://www.mapbox.com/blog/openstreetmap-gps-layer/#Misaligned.and.missing.roads).
We'll observe errors only when we have multiple GPS datapoints representing the
same physical location.

### Preparing the OSM data
I downloaded the [OSM planet XML file](http://planet.openstreetmap.org/), which
contains a world map encoded using nodes and ways. A node is a physical location
and a way is a series of these locations; some ways are roads, and that's what
we need here. The XML looks like this:

```xml
...
 <node id="1" lat="48.5669850" lon="13.4465242" timestamp="2015-05-24T21:25:26Z" version="13" changeset="31430396" user="Peda" uid="13010">
 <node id="10" lat="51.3806887" lon="9.3601583" timestamp="2014-08-01T14:23:42Z" version="8" changeset="24482318" user="dhuyp" uid="1779584"/>
 <node id="11" lat="4.0838053" lon="73.5122471" timestamp="2010-04-02T19:47:35Z" version="5" changeset="4307385" user="mabapla" uid="24748">
 <node id="12" lat="51.3400541" lon="9.4818121" timestamp="2014-02-20T15:19:56Z" version="5" changeset="20677113" user="emilde" uid="250214"/>
...
 <way id="41" timestamp="2014-04-02T17:09:47Z" version="7" changeset="21461939" user="blackadder" uid="735">
  <nd ref="200541"/>
  <nd ref="2715159905"/>
  <nd ref="200575"/>
  <nd ref="180180789"/>
  <nd ref="200576"/>
  <tag k="name" v="Rowan Road"/>
  <tag k="is_in" v="Sutton Coldfield"/>
  <tag k="highway" v="residential"/>
  <tag k="incline" v="-6.9%"/>
  <tag k="abutters" v="residential"/>
  <tag k="maxspeed" v="30 mph"/>
  <tag k="postal_code" v="B72"/>
 </way>
...
```

We're interested in the geometry of ways so we can look for GPX traces that
appear to be following them. This means we need to remove the way/node
indirection and index by geohash tile.

#### Map data conversion
The XML is in a huge bzip2 file, which is just about the worst thing for data
processing because `bzip2 -dc` is single-threaded and CPU-bound. I converted the
file to lzo, which removes that bottleneck:

```sh
$ pv osm-planet*.bz2 | bzip2 -dc | lzop > osm-planet.lzo
```

Once that was done I extracted the node list into a TSV. Normally I'd use nfu
for this, but a hand-written shell pipeline is faster and not much more verbose
(if I were redoing this today I'd use
[ni](https://github.com/spencertipping/ni), which is much faster than nfu and
will soon support XML destructuring):

```sh
# nfu osm-planet.lzo -k '%0 =~ /<node/' \
#                    -m 'row %0 =~ /id="(\d+)"/,
#                            %0 =~ /lat="([-0-9.]+)/,
#                            %0 =~ /lon="([-0-9.]+)/' \
# | gzip > nodes.gz

$ pv osm-planet.lzo | lzop -dc \
  | grep '\<node' \
  | perl -ne 'print join("\t", /id="(\d+)"/,
                               /lat="([-0-9.]+)/,
                               /lon="([-0-9.]+)/), "\n"' \
  | gzip > nodes.gz
```

`osm-planet.lzo` contains changesets, nodes, ways, and other stuff, in that
order. There are enough nodes that it takes a lot of time to get to ways, so I
extracted the ways verbatim so I could iterate on a parser quickly:

```sh
$ pv osm-planet.lzo | lzop -dc \
  | egrep -v '<node' \
  | perl -ne 'print if /<way/../<\/way/' \
  | gzip > ways-xml.gz
```

Before I get into how the parser works, let's talk about how the node/way
dereferencing process is implemented.

#### Dereferencing nodes in ways
We've got a node TSV in this form:

```
node_id  latitude  longitude
```

Each way will have an ID, multiple node IDs, and some tags indicating what it is
(and whether we're interested in it). This analysis is about roads, so we want
things tagged with `highway`. We also need to preserve the node ordering.

```sh
$ export NFU_SORT_COMPRESS=lzop
$ export NFU_SORT_BUFFER=4096M

$ nfu ways-xml.gz -m '$::id = $1, $::road = 0 if /<way id="(\d+)"/;
                      push @::nodes, $1 if /<nd ref="(\d+)"/;
                      $::road = 1 if /<tag k="highway"/;
                      if ($::road && /<\/way/) {
                        print join("\t", $::id, $_, $::nodes[$_]), "\n"
                          for 0..$#::nodes;
                        @::nodes = ();
                      }
                      ()' \
                  -i2 nodes.gz \
  | gzip > joined-ways.gz
```

Now we need to group by way ID. This is just a sort operation, and nfu
introduces more overhead than we want by looping over lines (`cut` and `sort`
use all kinds of optimizations to avoid this); since we're running on 24
processors, nfu's `-f1.` operation becomes a bottleneck to the high-throughput
`sort` process. We can work around this by removing perl line-loops from the
equation entirely and using `cut` instead, for a total of about 20x better
performance:

```sh
# nfu joined-ways.gz -f1.o01 | gzip > assembled-ways.gz
$ zcat joined-ways.gz | pv | cut -f2-5 \
  | sort --parallel=24 -S 128G -g --compress-program=lzop \
         -g -k 1,1 -k 2,2 \
  | gzip > assembled-ways.gz
```

#### GPX data conversion
The GPX traces are provided in a `.tar.xz` archive, so we need to extract that.
The contents are organized like this:

```sh
$ find gpx-planet-2013-04-09
...
gpx-planet-2013-04-09/identifiable/000/942
gpx-planet-2013-04-09/identifiable/000/942/000942334.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942628.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942269.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942270.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942267.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942610.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942316.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942266.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942322.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942652.gpx
gpx-planet-2013-04-09/identifiable/000/942/000942361.gpx
...
```

These files are highly redundant and small, so I compressed each `942`-level
directory using xzip (in hindsight not the best option; `xz -9e` is really slow
on the compression side and I switched to gzip, as shown here, halfway
through):

```sh
$ cd gpx*/identifiable/000
$ ls | grep -v z$ | xargs -P4 -I{} sh -c 'cat {}/* | gzip > {}.gz && rm -r {}'
```

The data in the gpx files looks like this, where `identifiable` and `trackable`
contain timestamps:

```xml
<gpx xmlns="http://www.topografix.com/GPX/1/0" version="1.0" creator="OSM gpx_dump.py">
  <trk>
    <name>Track 0</name>
    <number>0</number>
    <trkseg>
      <trkpt lat="54.4794300" lon="-6.2304850">
        <ele>137.80</ele>
        <time>2006-01-01T22:22:59Z</time>
      </trkpt>
      <trkpt lat="54.4794300" lon="-6.2304800">
        <ele>137.80</ele>
        <time>2006-01-01T22:23:00Z</time>
      </trkpt>
      <trkpt lat="54.4794300" lon="-6.2304750">
        <ele>137.80</ele>
        <time>2006-01-01T22:23:01Z</time>
      </trkpt>
      ...
```

We're interested in contiguous travel, delineated here by `<trkseg>` elements.
The simplest representation that captures this is a TSV like this:

```
segment_id  lat  lng  time
```

Parsing it out using some super-janky stateful logic:

```sh
$ nfu --run '$::id = $::lat = $::lng = 0' \
      identifiable/000/* \
      -m '++$::id if /<trkseg/;
          ($::lat, $::lng) = ($1, $2) if /lat="([-0-9.]+)" lon="([-0-9.]+)"/;
          return row $::id, $::lat, $::lng, $1 if /<time>([^<]+)</;
          ()' \
      --pipe q[ \
        ruby -rdate \
          -ne 'id, lat, lng, t = $_.split /\t/
               puts [id, lat, lng, DateTime.parse(t).to_time.to_i].join("\t")' ] \
  | gzip > identifiable-000.gz

# same thing for identifiable/001, trackable/000, and trackable/001
```

### Sharding
Right now we have enormous lists of spatially-unordered worldwide data. The next
thing to do is to shard it into geo-tiles. Before we do that, though, we need to
measure the distribution of way lengths to see how big the tiles should be.
Let's start by getting the maximum distance between the first and any other
point for each way:

```sh
$ nfu assembled-ways.gz \
      -A 'max map edist(${%2}[0], ${%3}[0], ${%2}[$_], ${%3}[$_]),
                  1..$#{%2}' \
      -T100000 \
      -q0.01ocf10p %l
```

![image](https://cloud.githubusercontent.com/assets/2106256/13753059/ac83f634-e9ce-11e5-8934-031126e34f34.png)

Awesome, looks like most ways are about 50m long. Let's check the GPX traces:

```sh
$ nfu trackable-000.gz \
      -A 'max map edist(${%1}[0], ${%2}[0], ${%1}[$_], ${%2}[$_]),
                  1..$#{%1}' \
      -T10000 \
      -q0.01ocf10p %l
```

![image](https://cloud.githubusercontent.com/assets/2106256/13753213/4eec5470-e9cf-11e5-81a7-7ad67c12af8a.png)

Looks great. We can index into 1km² tiles and have about 5% redundancy. A
geohash-6 has ±610m of error, which is about 1.5km²; let's use that. (**Note:**
this turns out to be wrong; I went with gh4s for reasons described in the next
section.)

#### Geohash-4 tiling
Let's take each way/trace and collapse it, then generate a list of GH6s per
object. We can shard in place using `--partition`. Because nfu's partitioning
will close and reopen a given shard, we're relying on the convenient property
that the concatenation of gzip files is the same as the gzip of their
concatenation.

Before we commit to the process, let's just confirm that we're not getting very
much tile duplication:

```sh
$ nfu assembled-ways.gz \
      -A 'scalar uniq map ghe(${%2}[$_], ${%3}[$_], 6), 0..$#{%2}' \
      -T100000 \
      -oc
60334   1
25849   2
6307    3
2198    4
1083    5
694     6
469     7
350     8
265     9
255     10
177     11
139     12
131     13
100     14
87      15
70      16
57      17
65      18
58      19
61      20
...
```

We're actually getting about 3x, which is horrible. Let's back out until we can
get this down to about 1.1, in this case gh4.

```sh
$ mkdir ways-shards
$ unset NFU_ALWAYS_VERBOSE
$ export NFU_PARALLELISM=512
$ pv assembled-ways.gz | zcat \
  | nfu -k23A 'my @ghs    = uniq map ghe(${%2}[$_], ${%3}[$_], 4), 0..$#{%2};
               my $points = join "\t", map "${%2}[$_],${%3}[$_]", 0..$#{%2};
               print "$_\t${%0}[0]\t$points\n" for @ghs; ()' \
        --partition 'substr %0, 0, 2' 'gzip >> ways-shards/{}.gz' \
  > /dev/null
```

We can do the same thing with traces:

```sh
$ mv identifiable-000.gz i000.gz
$ mv identifiable-001.gz i001.gz
$ mv trackable-000.gz t000.gz
$ mv trackable-001.gz t001.gz

$ mkdir traces-{i000,i001,t000,t001}-shards
$ unset NFU_ALWAYS_VERBOSE
$ export NFU_PARALLELISM=512
$ for f in i000 i001 t000 t001; do
    nfu $f.gz -A 'my $n1 = $#{%0};
                  return () unless ${%3}[$n1] > ${%3}[0];
                  my @ghs    = uniq map ghe(${%1}[$_], ${%2}[$_], 4), 0..$n1;
                  my $points = join "\t", map "${%1}[$_],${%2}[$_],${%3}[$_]",
                                              0..$n1;
                  print "$_\t${%0}[0]\t$points\n" for @ghs; ()' \
        --partition 'substr %0, 0, 2' "gzip >> traces-$f-shards/{}.gz" \
      > /dev/null &
  done
```

#### Trace/way resolution
This is where things get fun. Let's find a densely-populated GH4, like `dr5r`
for Manhattan:

```sh
$ nfu ways-shards/dr.gz -k '%0 eq "dr5r"' -f2.m @_ -F , --sample 0.1 -f10p %d
```

![image](https://cloud.githubusercontent.com/assets/2106256/13782803/e090b464-ea85-11e5-9aa7-fc792f4823a1.png)

Um... crap. That's not well-sharded data at all. There must be something
horribly wrong with my ways-sharder. Let's quickly verify that each way at least
contains one point within its shard:

```sh
$ pv ways-shards/dr.gz | zgrep ^dr5r | gzip > dr5r.gz
$ nfu dr5r.gz -m 'my ($gh4, $way) = (%0, %1);
                  my @latlngs     = @_[2..$#_];
                  my @shards      = map ghe(split(/,/), 4), @latlngs;
                  my $correct     = grep $_ eq $gh4, @shards;
                  my %incorrect   = frequencies grep $_ ne $gh4, @shards;
                  row $gh4, $way, $correct, %incorrect' \
  | gzip > dr5r-errors.gz

$ nfu dr5r-errors.gz -k '%1 == 0' | wc -l
0
```

Interesting, so our sharder isn't completely wrong anyway; it seems to be doing
its job well enough. Let's look at the way data now, plotting vectors between
waypoints for any gh4 with "incorrect" entries:

```sh
$ nfu dr5r-errors.gz -k3f1i0 @[ dr5r.gz -f1. ] \
      -m 'my @latlngs = @_[1..$#_];
          map {row $latlngs[$_ - 1] =~ y/,/\t/r,
                   $latlngs[$_]     =~ y/,/\t/r} 1..$#latlngs' \
      -m 'row %0, %1, %2 - %0, %3 - %1' \
      -f1032p 'with vectors lc rgb "#f0203050"'
```

![gnuplot window id 0 on rorschach2_740](https://cloud.githubusercontent.com/assets/2106256/13787097/c04e396c-ea97-11e5-9da3-c95d9aaf10d8.png)
![gnuplot window id 0 on rorschach2_741](https://cloud.githubusercontent.com/assets/2106256/13787103/c61bbfcc-ea97-11e5-98a8-ab6adc133725.png)
![gnuplot window id 0 on rorschach2_742](https://cloud.githubusercontent.com/assets/2106256/13787108/c8d5866c-ea97-11e5-95e4-a8f7e5bd5dde.png)
![gnuplot window id 0 on rorschach2_743](https://cloud.githubusercontent.com/assets/2106256/13787110/caa2df9e-ea97-11e5-9f82-a5904f3b7d40.png)

So it looks like most of these ways ultimately tie back to Manhattan, but for
some reason join up with paths all over the world. I doubt this is really how
the OSM ways work, so let's make a rule that no two waypoints can be more than
~5km apart (if they are, we'll split the way):

```sh
$ nfu dr5r-errors.gz -k3f1i0 @[ dr5r.gz -f1. ] \
      -m 'my @latlngs = @_[1..$#_];
          map {row $latlngs[$_ - 1] =~ y/,/\t/r,
                   $latlngs[$_]     =~ y/,/\t/r} 1..$#latlngs' \
      -m 'row %0, %1, %2 - %0, %3 - %1' \
      -k 'dist(%2, %3) < 0.05' \
      -f1032p 'with vectors lc rgb "#f0203050"'
```

![gnuplot window id 0 on rorschach2_749](https://cloud.githubusercontent.com/assets/2106256/13788696/12972412-ea9e-11e5-9228-7303ec7d6cfc.png)

This looks a lot more reasonable.

#### Trace/way resolution, for real this time
Ok, for now ignoring the worldwide-way problem, let's take a look at some
Manhattan GPX traces overlaid onto OSM content:

```sh
$ nfu trace*shards/dr.gz -k '%0 eq "dr5r"' -f2.m @_ -F , \
      -f10m 'row @_, "NA", "NA"' \
      --append @[ dr5r.gz -f2.m @_ -F , -f10m 'row "NA", "NA", @_' ] \
      --mplot '%u1:2%d%t"GPX" lc rgb "#80ffa040";
               %u3:4%d%t"OSM" lc rgb "#c0203050"'
```

![gnuplot window id 0 on rorschach2_745](https://cloud.githubusercontent.com/assets/2106256/13788093/c3c16c3c-ea9b-11e5-98dd-1308a6c68430.png)
![gnuplot window id 0 on rorschach2_746](https://cloud.githubusercontent.com/assets/2106256/13788094/c3c283ec-ea9b-11e5-82e8-903dd021a7fd.png)
![gnuplot window id 0 on rorschach2_747](https://cloud.githubusercontent.com/assets/2106256/13788095/c3c3dcb0-ea9b-11e5-9f52-3b80eb5cf229.png)
![gnuplot window id 0 on rorschach2_748](https://cloud.githubusercontent.com/assets/2106256/13788096/c3cc0fde-ea9b-11e5-8504-283763f723e1.png)

And that's exactly what we're looking for: path deflection due to the urban
canyon problem. Let's study this particular case in more detail, this time
plotting vectors:

```sh
$ nfu trace*shards/dr.gz -k '%0 eq "dr5r"' \
      -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , -f0134 \
      -m 'row %1, %0, %3 - %1, %2 - %0, ("NA") x 4' -k 'dist(%2, %3) < 0.05' \
      --append @[ dr5r.gz \
        -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , \
        -m 'row(("NA") x 4, %1, %0, %3 - %1, %2 - %0)' \
        -k 'dist(%6, %7) < 0.05' ] \
      --mplot '%u1:2:3:4 with vectors %t"GPX" lc rgb "#80ffa040";
               %u5:6:7:8 with vectors %t"OSM" lc rgb "#c0203050"'
```

![gnuplot window id 0 on rorschach2_750](https://cloud.githubusercontent.com/assets/2106256/13791521/c9d43bfe-eaaa-11e5-8ab7-4b43fa801133.png)
![gnuplot window id 0 on rorschach2_751](https://cloud.githubusercontent.com/assets/2106256/13791522/c9eb4dc6-eaaa-11e5-9ac3-b40c5bc7033e.png)
![gnuplot window id 0 on rorschach2_752](https://cloud.githubusercontent.com/assets/2106256/13791523/c9ef9930-eaaa-11e5-9dda-d6138874bb14.png)
![gnuplot window id 0 on rorschach2_753](https://cloud.githubusercontent.com/assets/2106256/13791524/c9ef9a7a-eaaa-11e5-9cf2-961b59bd872b.png)
![gnuplot window id 0 on rorschach2_754](https://cloud.githubusercontent.com/assets/2106256/13791525/c9f0bf2c-eaaa-11e5-9d79-e265db995ea4.png)
![gnuplot window id 0 on rorschach2_755](https://cloud.githubusercontent.com/assets/2106256/13791527/c9f92aea-eaaa-11e5-9bf7-4be52ca37e82.png)
![gnuplot window id 0 on rorschach2_756](https://cloud.githubusercontent.com/assets/2106256/13791526/c9f91fd2-eaaa-11e5-8609-d163f6e22a07.png)
![gnuplot window id 0 on rorschach2_757](https://cloud.githubusercontent.com/assets/2106256/13791528/c9ffbce8-eaaa-11e5-8e1e-a71af612bb1c.png)
![gnuplot window id 0 on rorschach2_758](https://cloud.githubusercontent.com/assets/2106256/13791529/c9ffd732-eaaa-11e5-9078-84ab38f86d0a.png)
![gnuplot window id 0 on rorschach2_759](https://cloud.githubusercontent.com/assets/2106256/13791530/ca005b3a-eaaa-11e5-9614-38e096eab7bd.png)
![gnuplot window id 0 on rorschach2_760](https://cloud.githubusercontent.com/assets/2106256/13791531/ca021614-eaaa-11e5-9a36-38c9bd9ee738.png)

Ok, this just looks too cool not to make a wallpaper out of it.

```sh
$ wallpaper() {
    [[ -e /tmp/$1-gpx ]] \
    || nfu trace*shards/${1:0:2}.gz -k "%0 eq '$1'" \
           -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , -f0134 \
           -m 'row %1, %0, %3 - %1, %2 - %0' -k 'dist(%2, %3) < 0.05' \
       > /tmp/$1-gpx

    [[ -e /tmp/$1-osm ]] \
    || nfu ways-shards/${1:0:2}.gz -k "%0 eq '$1'" \
           -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , \
           -m 'row %1, %0, %3 - %1, %2 - %0' \
           -k 'dist(%2, %3) < 0.05' \
       > /tmp/$1-osm

    gnuplot5 <<EOF
    set terminal pngcairo enhanced background rgb "#040404" size 3840,2160
    set output "$1.png"
    unset border
    plot [$3-0.25:$3+0.25] [$2-0.140625:$2+0.140625] \
        "/tmp/$1-gpx" with vectors title "" lw 1 lc rgb "#a0ff9020", \
        "/tmp/$1-osm" with vectors title "" lw 1 lc rgb "#d0e0e4e8"
EOF
  }

$ wallpaper dr5r -74.03 40.705
```

![dr5r](https://cloud.githubusercontent.com/assets/2106256/13794161/c18b6d5c-eab7-11e5-876c-0dbd341c1e13.png)

Anyway. There are a lot of places above where the GPX path clearly diverges from
the OSM one, particularly when the GPS is in a canyon. There's also some very
weird divergence along the Manhattan Bridge. It's especially weird given that
it's all in the same direction; the directional vectors themselves are parallel
with the Brooklyn Bridge. I think it's reasonable to treat this as an outlier
for now.

##### Apparent rate of travel
This figures a lot into our analysis, simply because we're interested in just a
narrow range of speeds: stuff between about 20 and 60mph. Dividing Euclidean
distance by time-deltas gives us ~100km/sec, which we want to convert to mph:

```sh
$ units
You have: 100km/sec
You want: mph
        * 223693.63

$ speedplot() {
    nfu trace*shards/${1:0:2}.gz -k "%0 eq '$1'" \
        -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , \
        -m 'row %1, %0, %4 - %1, %3 - %0, %5 - %2' \
        -k '%4 > 0 && dist(%2, %3) < 0.05' \
        -m 'row @_[0..3], dist(%2, %3) / %4 * 223693.63, ("NA") x 4' \
        -k '%4 < 80' \
        --append @[ ways-shards/${1:0:2}.gz -k "%0 eq '$1'" \
          -m 'map row(@_[$_ - 1, $_]), 3..$#_' -F , \
          -m 'row(("NA") x 5, %1, %0, %3 - %1, %2 - %0)' \
          -k 'dist(%7, %8) < 0.05' ] \
        --mplot '%u1:2:3:4:5 with vectors %t"GPX" lc palette;
                 %u6:7:8:9   with vectors %t"OSM" lc rgb "#f0202020"'
  }

$ speedplot dr5r
```

![gnuplot window id 0 on rorschach2_761](https://cloud.githubusercontent.com/assets/2106256/13795809/489b2370-eac0-11e5-92bb-82610f17a18f.png)
![gnuplot window id 0 on rorschach2_762](https://cloud.githubusercontent.com/assets/2106256/13795807/489ac290-eac0-11e5-96a3-b4d0224243c2.png)
![gnuplot window id 0 on rorschach2_763](https://cloud.githubusercontent.com/assets/2106256/13795808/489af5da-eac0-11e5-96b2-4fde03ba9b30.png)
![gnuplot window id 0 on rorschach2_764](https://cloud.githubusercontent.com/assets/2106256/13795806/48987a80-eac0-11e5-8028-94c42470ed28.png)
![gnuplot window id 0 on rorschach2_765](https://cloud.githubusercontent.com/assets/2106256/13795810/48aa157e-eac0-11e5-82b3-34abba907d9c.png)
![gnuplot window id 0 on rorschach2_766](https://cloud.githubusercontent.com/assets/2106256/13795811/48ae8960-eac0-11e5-9260-3ed11e032749.png)
![gnuplot window id 0 on rorschach2_767](https://cloud.githubusercontent.com/assets/2106256/13795812/48b37024-eac0-11e5-8466-5f48c0515a7b.png)
![gnuplot window id 0 on rorschach2_768](https://cloud.githubusercontent.com/assets/2106256/13795813/48b7bd46-eac0-11e5-8d30-d727154cfcf1.png)
![gnuplot window id 0 on rorschach2_769](https://cloud.githubusercontent.com/assets/2106256/13795814/48bb4d26-eac0-11e5-953f-a7c31a663e58.png)
![gnuplot window id 0 on rorschach2_770](https://cloud.githubusercontent.com/assets/2106256/13795815/48c00faa-eac0-11e5-99e0-d0566178b68f.png)
![gnuplot window id 0 on rorschach2_771](https://cloud.githubusercontent.com/assets/2106256/13795816/48c42f7c-eac0-11e5-8d5e-b903a57386ba.png)

This is interesting. Some of the divergent paths from earlier are clearly
low-speed (most likely walking), so we can't attribute this to GPS error.
Higher-speed paths are much more regular; by the time we get up to 40-50mph
they're good to a couple of meters.

The assumption is that all high-speed traffic follows roads of some kind, but
OSM doesn't give us exact road boundaries. Instead, it provides what I assume is
the center of each road. The way metadata also doesn't specify exactly how large
each road is, but it does classify them into different categories using the
`highway` attribute. Let's go ahead and extract those, sharded like the ways
themselves so we can pull it into memory by tile:

```sh
$ nfu ways-shards/*.gz -f01gcf120 | gzip > ways-by-tile.gz
$ nfu ways-xml.gz --fold '!/<\/way>/' \
      -m 'my @attrs;
          push @attrs, "$1=$2" while /<tag k="([^"]+)" v="([^"]+)"/g;
          row /<way id="(\d+)"/, @attrs' \
  | gzip > way-metadata.gz
$ mkdir way-metadata-shards
$ NFU_MAX_FILEHANDLES=1024 \
  nfu --run '/(^\d+)\t(\w{4})/ and ($::shards{$1} //= "") .= ":$2"
             for rl "sh:nfu ways-by-tile.gz -f10"' \
      way-metadata.gz \
      -m 'map row($_, @_), grep length, split /:/, $::shards{%0}' \
      --partition 'substr(%0, 0, 2)' 'gzip >> way-metadata-shards/{}.gz'
```

**TODO:** Finish this at some point

# Appendix: failed experiments
## Original coarse transit (too imprecise, and too much work)
I live near some mountains that are probably blocking the GPS signal by quite a
bit. For context, here's what it looks like from the roof facing NE, SE, SW,
and NW respectively:

![northeast](https://tipping.haus/gpsa-cw3.jpg)
![southeast](https://tipping.haus/gpsa-cw0.jpg)
![southwest](https://tipping.haus/gpsa-cw1.jpg)
![northwest](https://tipping.haus/gpsa-cw2.jpg)

The phone will tell you which GPS satellites it can see, but since they aren't
geosynchronous (and therefore change as the log is running) I needed to first
get a rough horizon profile to find out when each satellite would be blocked.
To do this, I built a simple surveying transit:

![surveying transit](https://tipping.haus/gpsa-transit-survey.jpg)

I didn't have a magnetic compass, so I calibrated the zero angle to something
that would be visible on a satellite map:

![zero calibration](https://tipping.haus/gpsa-transit-survey-zero.jpg)
![tower coordinates](https://tipping.haus/gpsa-survey-tower.png)

Then I measured the horizon angle every 15 degrees. The transit was on top of
the air conditioner at 35.194155, -106.481735, and the tower is at 35.191365,
-106.473310. I got the angle between them by constructing a third point due
south of one and due west of the other, then converting the third-point
distances to polar coordinates:

```sh
$ echo | nfu -m 'my ($y1, $x1) = (35.194155, -106.481735);
                 my ($y2, $x2) = (35.191365, -106.473310);
                 row rect_polar( edist($y2, $x1, $y2, $x2),
                                -edist($y2, $x1, $y1, $x1))'
0.826065086148791       112.058692110008
```

Adding in the 112-degree offset, the measurements were:

adjusted angle | horizon
---------------|--------
7              | 10
22             | 18
37             | 19
52             | 20
67             | 23
82             | 23
97             | 21
112            | 14
127            | 11
142            | 6
157            | 4
172            | 3
187            | 0
202            | 0
217            | 0
232            | 0
247            | 0
262            | 0
277            | 0
292            | 0
207            | 0
322            | 3
337            | 6
352            | 7
