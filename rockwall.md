# Building the living room bouldering wall
A bouldering wall is basically a floor that runs vertically, and ends up
getting framed the same way. I didn't have a background in construction, but I
did have the [Metolius
guide](http://www.metoliusclimbing.com/pdf/How-to-Build-a-Home-Climbing-Wall.pdf)
and [an accomplice](http://joycetipping.com). I sketched up a rough model of
the wall in Blender, along with some relevant house structure from the
architectural drawings:

![image](http://storage5.static.itmages.com/i/17/0511/h_1494499689_7545312_d4f02ecb3f.jpeg)

## Basic design constraints + framing
The project wasn't entirely straightforward for a few reasons:

1. I didn't want to remove any drywall; the bouldering wall needed to be
   removable, modulo a few holes.
2. There's a baseboard heater running the length of the main wall, so the
   entire structure is hanging rather than supported from the floor.
3. The house is split-level and the main wall is backed by concrete and/or
   interior wiring below 54" (so all supporting hardware needs to be above this
   level).
4. The bouldering wall has a ceiling-mounted section, but I don't want to hang
   weight on the ceiling; instead, it's transferred down the diagonal.

I started out by installing a ridiculously overengineered 2x6 frame over the
studs in the wall. Each 2x6 has three lag bolts in the top half, and the bottom
half hangs. (This is structurally sound due to the tensile strength of wood
parallel to the grain.)

The baseboard heater is 3kW and has aluminum fins spaced about 1/4" apart over
a span of 13'. Conservatively estimating each fin at 2 square inches:

```sh
$ units -t '3 kW / (2 in^2 / 0.25 in * 13 ft)' 'kW/m^2'
3.725969
```

So no more than 4x full-sun heat density, which should be fine (i.e. won't
ignite anything by convection).

![image](http://spencertipping.com/rockwall-frame.jpg)
![image](http://storage3.static.itmages.com/i/17/0511/h_1494501139_7811429_cf77052439.jpeg)

The ceiling section is also lagged into the structure, which is 12" OC 2x12 and
supports the flat roof directly. Ultimately, though, the load gets transferred
down the diagonal after about 1/4" of deflection.

![image](http://spencertipping.com/rockwall-frame3.jpg)

Here's the top receiver for the diagonals. I had tried assembling the five 2x6
sections as a single piece, but it was too heavy to hold against the ceiling
while drilling the bolt holes. Ended up doing it piece by piece and assembling
once the ceiling boards were in place.

![image](http://spencertipping.com/rockwall-frame5.jpg)
![image](http://spencertipping.com/rockwall-frame4.jpg)

Installed diagonals, which interestingly aren't fastened in any way. The joist
hangers and routed slots in the main 2x6s hold them in place, and I figured any
fasteners would only weaken the structure. I didn't take any pictures of the
bottom diagonal receivers -- but they're routed 3/4" into the main boards so
the diagonals rest on the wood grain; the joist hangers on the bottom don't
take much load. (Also, 2x6 for the diagonal framing was way overkill in
hindsight.)

![image](http://spencertipping.com/rockwall-frame6.jpg)
![image](http://spencertipping.com/rockwall-frame7.jpg)

## Paneling
It's common to use 3/4" plywood for the climbing surface, but that stuff is
expensive and I wanted to try using OSB instead. This made the project about
30% cheaper and the extra weight didn't seem to cause any problems. It also has
TruFlor logos all over it, which matches Joyce's interior decor aesthetic.

![image](http://spencertipping.com/rockwall-panel1.jpg)

3/4" OSB is also quite strong under bending, as I found out when load testing:

![image](http://spencertipping.com/rockwall-panel2.jpg)

Getting the holds mounted was straightforward enough. The first step was to
chalkline a grid onto each panel:

![image](http://spencertipping.com/rockwall-panel3.jpg)

Then I drilled out the grid points with small pilot holes. It's important to
make the T-nut holes properly perpendicular to the surface, which even at 3/4"
isn't guaranteed with a hand drill. I ended up using a plunge router to finish
each hole, then conscripted an offspring for the hard part:

![image](http://spencertipping.com/rockwall-panel4.jpg)

Two-dude final load test:

![image](http://spencertipping.com/rockwall-panel5.jpg)

Yep, totally solid. (And very safe.)

![image](http://spencertipping.com/safetyscrew.jpg)
