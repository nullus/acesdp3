# ACES Display P3

## Contents

* [What?](#what)
* [Why?](#why)
    * [A curious problem](#why-problem)
    * [The obsession](#why-obsession)
    * [Discovery](#why-discovery)
* [How?](#how)
    * [Install the OCIO configuration as a Houdini package](#how-install)
    * [The gory details](#how-details)

## What? {#what}

### Handcrafted minimal ACES OCIO config for Houdini

It contains the absolute minimal configuration required to transform views in
Houdini for a Display P3 colour space, which is default on macOS. sRGB is
included, and is the default on other systems.

By design this is an _incomplete_ colour management config, but Houdini is
very light on colour management (almost colour space agnostic), and this
seemed like a reasonable place to start.

There are many other interesting things that can be done with
OpenColorIO (OCIO), and this config could be used in packages other than
Houdini, but you're on your own with that for now. Have fun!

## Why? {#why}

### A curious problem {#why-problem}

A while ago I was presented with a problem: on macOS Houdini Render View
didn't match render output when viewed through Quick Look, or Preview.

The difference was reasonably subtle, but colours in the render output
appeared slightly desaturated. It was particularly noticable with shades of
green.

![Side-by-side view of unmananged Render View and render output, showing differences in colour](images/Render%20view%20unmanaged%20vs%20render%20unmanaged.png)

At the time I didn't have a good explanation of what was happening, and since
I didn't require any rigorous colour matching I left it at that.

### The obsession {#why-obsession}

Recently I've had need for colour matching, on a project using ACEScg for
colour management. So I went off to find an [OCIO config](https://github.com/colour-science/OpenColorIO-Configs).

I didn't know at the time that SideFX Labs bundled a minimal ACES 1.2 config,
but it didn't turn out to be any more helpful.

Surely with colour management the situtation will be greatly improved...

![Side-by-side view of mananged Render View and render output, showing extreme differences in colour](images/Render%20view%20managed%20vs%20render%20managed.png)

Not so much.

Not only that, but reference images I was using as a background showed
over-saturated colours, bringing yellows out of beige!

What followed was a maddening journey, trying every conceivable OCIO View,
various arrangements of OCIO Transform nodes, and numerous fruitless searches.

### Discovery {#why-discovery}

I knew I'd reached the limits of what could be done with this OCIO config,
but also that there _was_ a reasonable solution. So the next steps were:
1. Figure out how to write my own OCIO config; and
2. Understand what colour space I was targetting.

It turns out step 1 was much easier than I thought. The [documentation](https://opencolorio.readthedocs.io/en/latest/guides/authoring/authoring.html)
is reasonably good, and there is a great Python package [Colour](https://github.com/colour-science/colour)
also with [good documentation](https://colour.readthedocs.io/en/v0.4.1/) that
can generate the required transformation matrices, and LUTs.

The intention was always that people could build their own OCIO configs as
required!

I'm embarrased that I hadn't already realised what I discovered at step 2.
[Display P3](https://developer.apple.com/videos/play/wwdc2017/821/) was
announced during WWDC17, and I probably even flipped through the slide deck
at some stage back then. "Wide gamut," eh, whatever...

It's a combination of DCI-P3 primaries (wider gamut), D65 illuminant, and sRGB
transfer function. The green primary being a significant difference—now the
washed out/oversaturated green makes sense.

## How? {#how}

### Install the OCIO configuration as a Houdini package {#how-install}

1. Clone this repository
2. Copy the contents of the [packages](packages/) directory into a local
   Houdini packages directory. I used `~/Library/Preferences/houdini/19.5/packages`,
   but other Houdini packages paths will work. Don't copy the packages directory
   itself, you should end up with `~/Library/Preferences/houdini/19.5/packages/acesdp3.json`
   etc.
3. In Houdini Scene View, activate the Correction Toolbar in Viewport options.
   It should look like:
   ![Correction toolbar showing Display P3 (as display), and Native (as view)](images/Correction%20toolbar%20acesdp3.png)

Now, after all that effort, surely the colours match?

![Side-by-side view of mananged Render View using my OCIO config and render output, still showing differences in colour](images/Render%20view%20managed%20acesdp3%20vs%20render%20managed.png)

No, actually, we're not quite there yet. However I had noticed that info in
Finder reported the image colour profile as 'sRGB IEC61966-2.1 Linear', being
some sort of sRGB without the transfer for displaying linear images without an
explicit colour profile. So, after a bit more digging I found a tool to tag
EXR files with chromaticities (primaries in CIE XYZ colour space):

```
exrstdattr -chromaticities 0.713 0.293 0.165 0.830 0.128 0.044 0.32168 0.33767 "Render managed.exr" "Render managed ACEScg primaries.exr"
```

And finally, we have a match!

![Side-by-side view of mananged Render View using my OCIO config and render output, with matching colours](images/Render%20view%20managed%20acesdp3%20vs%20render%20tagged.png)

This tool is part of [OpenEXR](https://www.openexr.com/). There is also a tool
called `exr2aces` which you should avoid, as it applies the ACES2065-1
primaries to the output image.

Finder now shows 'AppleEXR RGB linear' as the colour profile, and the colours
are a good match. This step shouldn't be necessary if your next stage treats
EXR input as ACEScg by default, but I didn't want to leave that stone
unturned.

There is a slight seam in the side-by-side, where it looks like I've lost some
precision making the comparison images. Note that all the images used here have
been convered to sRGB for display on the web, so the differences aren't as
noticable as you'd see on a Display P3 monitor. I've used EXR exclusively for
render output, and I don't have a good solution for what happens to other
"non-linear" formats (i.e. 8-bit PNG), but it's not what you want to be using
with ACES anyway.

### The gory details {#how-details}

With a suitable Python environment you can `pip install colour-science` which
gives you the `colour` module.

From Python you now have access to colour space definitions, and conversion
methods:

```python

import colour

print(colour.RGB_COLOURSPACES['Display P3'])
```

```
Display P3
----------

Primaries          : [[ 0.68   0.32 ]
[ 0.265  0.69 ]
[ 0.15   0.06 ]]
Whitepoint         : [ 0.3127  0.329 ]
Whitepoint Name    : D65
Encoding CCTF      : <function eotf_inverse_sRGB at 0x11bdd4a60>
    Decoding CCTF      : <function eotf_sRGB at 0x11bdd4af0>
        NPM                : [[  4.86570949e-01   2.65667693e-01   1.98217285e-01]
        [  2.28974564e-01   6.91738522e-01   7.92869141e-02]
        [ -3.97207552e-17   4.51133819e-02   1.04394437e+00]]
        NPM -1             : [[ 2.49349691 -0.93138362 -0.40271078]
        [-0.82948897  1.76266406  0.02362469]
        [ 0.03584583 -0.07617239  0.95688452]]
        Derived NPM        : [[  4.86570949e-01   2.65667693e-01   1.98217285e-01]
        [  2.28974564e-01   6.91738522e-01   7.92869141e-02]
        [ -3.97207552e-17   4.51133819e-02   1.04394437e+00]]
        Derived NPM -1     : [[ 2.49349691 -0.93138362 -0.40271078]
        [-0.82948897  1.76266406  0.02362469]
        [ 0.03584583 -0.07617239  0.95688452]]
        Use Derived NPM    : False
        Use Derived NPM -1 : False
```

To generate a matrix for conversion from ACES to Display P3:

```python

import colour
import numpy

# Start with the 4x4 identity matrix (diagonal 1s)
aces2065_to_p3 = numpy.identity(4)
# Slice in the 3x3 RGB transformation matrix
aces2065_to_p3[:3, :3] = colour.matrix_RGB_to_RGB(colour.RGB_COLOURSPACES['ACES2065-1'], colour.RGB_COLOURSPACES['Display P3'])
# Present it as a comma-separated list of 16 coefficients
print(", ".join(str(i) for i in aces2065_to_p3.reshape(16)))
```

Gives you matrix coefficients:

```
2.02528732757, -0.691962172309, -0.333325155452, 0.0, -0.182621506073, 1.28661600582, -0.103994499686, 0.0, 0.0085845525082, -0.054816290055, 1.04623173759, 0.0, 0.0, 0.0, 0.0, 1.0
```

Which go directly into an OCIO TransformMatrix:

```yaml

- !<ColorSpace>
    name: DisplayP3
    bitdepth: 32f
    from_reference: !<GroupTransform>
        children:
            - !<MatrixTransform> {matrix: [2.02528732757, -0.691962172309, -0.333325155452, 0.0, -0.182621506073, 1.28661600582, -0.103994499686, 0.0, 0.0085845525082, -0.054816290055, 1.04623173759, 0.0, 0.0, 0.0, 0.0, 1.0]}
            - !<FileTransform> {src: linear_to_sRGB.spi1d, interpolation: linear}
```

Additionally, we need a transfer function (the FileTransform above), because
Display P3 is not linear. It uses the same curve as sRGB, which is close to,
but subtly different from "gamma 2.2". That's why we need to use a LUT:

```python

import colour
import numpy

# Generate a linear sequence between [0, 1], covert it using the sRGB transfer
# function, and create a 1-D LUT from those values
lut = colour.LUT1D(colour.RGB_COLOURSPACES['sRGB'].cctf_encoding(numpy.linspace(0., 1., 4096)))
# Write the LUT to a file
colour.write_LUT(lut, "linear_to_sRGB.spi1d")
```
