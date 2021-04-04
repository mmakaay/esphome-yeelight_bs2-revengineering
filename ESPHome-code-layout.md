# ESPHome code layout

## Purpose

This document was written to gather some insights about the way in which to
layout the ESPHome code for controlling the Yeelight Bedside Lamp 2. 

The need for these insight has arisen from the fact that driving the RGB and
white light LEDs is not at all like driving regular RGBWW LED lights.

## What's the problem?

I did write some code that could successfully drive the LED circuitry to get
the desired light colors. However, when using transitions when switching to
a different color, things did not always look good.

For example, when transitioning from warm to cold light, the original device
shows a smooth transition, whereas my implementation would show a transition
including the white light zone in the middle, causing the transition to be a
bit flashy bright in the middle.

## How come?

My assumption here was that the transitions happened at the wrong level.
The ESPHome light framework handles transitioning between different settings
by linearly transitioning the individual light properties over time.  As I
found by doing measurements, the original firmware of the lamp uses linear
transitioning at the level of the GPIO duty cycle outputs.  So put
differently: ESPHome transitions over the input values, whereas the device
transitions over the output values.

This would not be a problem, when a clean relation between the input and
output values would exist. However, this is not the case.

In the [GPIO measurements.xlsx](GPIO measurements.xlsx) spreadsheet, you can
find a lot of measurements that I took, to see how the original device
firmware drives the LED circuitry. As you can see there, the required GPIO
output levels are very irregular. Quite different from the levels that are
applied by the default ESPHome code.

## Anything else?

Another thing that I ran into during implementation, is that I would like to
have more control over transitions. For example, when switching from night
light mode to RGB color mode, no transitioning is required. This switch can
be immediate. However, I have no control over this from the light output
implementation.

## How to fix?

Based on the above information, I figured that my light implementation would
have to handle transitioning differently. The requirements here would be to:

- handle light setting changes by taking the current GPIO output levels,
  finding the required new GPIO output levels and transitioning linearly
  between those two;
- disable transitioning when appropriate.

Unfortunately, this is not an easy thing to implement. Reason for this, is the
structure of the ESPHome light code, which uses dependency injection to stitch
some objects together.

The regular way to implement some sort of light, is to implement a subclass
of the `light::LightOutput` class. An instance of this class is injected
into a `light::LightState` instance. This light state is the class that is
responsible for driving the transitions (making use of a few other classes).

In a dependency graph, this looks somewhat like this:

```
     +---------------------------+
     | Yeelight BS2 Light Output |
     +-------------+-------------+
                   |
         +---------v----------+
         | light::LightOutput |
         +---------^----------+
                   |
         +---------+---------+
         | light::LightState |----> various helper classes
         +-------------------+
```

All that the `light::LightOutput` implementation does, is receive some input
from the `light::LightState` and apply that to the physical light(s). This
means that the light output has no handles whatsoever to implement the
requirements from above.


