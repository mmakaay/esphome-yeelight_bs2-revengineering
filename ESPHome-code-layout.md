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

My assumption here is that the transitions happens at the wrong level.
The ESPHome light framework handles transitioning between different settings
by linearly transitioning the individual light properties over time. As I
found by doing measurements, the original firmware of the lamp uses linear
transitioning at the level of the GPIO duty cycle outputs. So put
differently: ESPHome transitions over the input values, whereas the device
transitions over the output values.

This would not be a problem, when a clean relation between the input and
output values would exist. However, this is not the case.

In the GPIO measurements.xlsx spreadsheet, you can find a lot of
measurements that I took, to see how the original device firmware drives the
LED circuitry. As you can see there, the required GPIO output levels are
very irregular. Quite different from the levels that are applied by the
default ESPHome code.

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
- disable the transitioning time when appropriate.

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
         | light::LightState |----> various helper classes, e.g.
         +-------------------+      light:LightTransformer
                                    light::LightColorValues
                                    light::LightCall

```

All that the `light::LightOutput` implementation does, is receive some input
in form of a `light::LightState` object, and apply that to the physical
light(s). This means that the light output has no handles whatsoever to
implement the requirements from above.

## Access the LightTransformer?

There is another light component, that also requires more control over the
transitioning: `esphome::light::AddressableLight`. The implementation of
this class accesses `state->transformer_` to get access to the required end
state of an ongoing transition.

However, this field is classified as protected and there is no getter method
to access its contents.

The reason that the addressable light is able to access the transformer, is
that it is specifically registered as a `friend class AddressableLight`
in the LightState code. This means that the esphome code does recognize
that a use case does exist for accessing the transformer, but that the
transformer was not exposed in such way that custom components can make use
of it.

## Access the LightState->remote_values?

One property that I can access from within my light output code, is the
`remote_values` field of the light state. That might provide some information that can be used to detect a transition.
I did some investigation in this, and found that this was not feasible.

- For handling a transition, we would need to recognize that a transition is
  in progress. Unfortunately, the only transition that I can recognize is
  when turning on or off the light. For those transitions, the state field
  will contain a fractional number between 0 and 1. When The light is on and
  a new color is selected, this state will always be 1 during the
  transition.

- Even when there is a way to detect an ongoing transition, the
  remote_values would not provide enough information to handle things
  correctly. For example when flashing the light, the remote_values only
  contain the target light color for the flash. There is no indication that
  this is supposed to be a flash and that the light should return to its
  original state after the flash. It basically contains the same information
  as when transitioning to a new color.

My conclusion on this one is that I cannot use it either.

## Maybe create a LightState subclass?

Since the LightState class has such a central role, maybe it is a good idea
to create a subclass of it, specifically for the Yeelight BS2 integration.
The main thing to investigate here, is whether or not it is feasible to
generate the required code from the component Python code generation layer.

Looking at the configuration input for `def to_code(config)` in my light
code, the following properties can be found (non-interesing data stripped):

- `id` = ID<type=light::LightState, ...>
- `output_id` = ID<type=yeelight::YeelightBS2LightOutput, ...>

If I am able to use a different class for `id`, then things might work.
Let's do a quick test, to see if I can get the code generation to work
for this idea, by ading the following construct to my `light.py` code:

```python
yeelight_ns = cg.esphome_ns.namespace("yeelight") 
bs2_ns = yeelight_ns.namespace("bs2")
BS2LightState = bs2_ns.class_("BS2LightState", cg.Nameable, cg.Component)

CONFIG_SCHEMA = light.RGB_LIGHT_SCHEMA.extend(
    {
            cv.GenerateID(): cv.declare_id(BS2LightState),
            // ...
    }
)
```

Now when I try to compile the firmware, I get a hopeful error message:

```
src/main.cpp:21:11: error: 'BS2LightOutput' in namespace 'esphome::yeelight'
does not name a type
```

The generated `main.cpp` code now contains the following snippets:

```c++
yeelight::BS2LightOutput *yeelight_bs2lightoutput;
yeelight::bs2::BS2LightState *yeelight_bs2_bs2lightstate;

// ...

  yeelight_bs2lightoutput = new yeelight::BS2LightOutput();
  yeelight_bs2_bs2lightstate = new yeelight::bs2::BS2LightState(
      "Bedside Lamp Office RGBW Light", yeelight_bs2lightoutput);
  App.register_light(yeelight_bs2_bs2lightstate);
  App.register_component(yeelight_bs2_bs2lightstate);                                                                                                        
```

Wonderful! So this allows me to now override the behavior of the LightState
class.

## Yeah but, no but, yeah but ...

Next hurdle: I can have my own derived version of the LightState class. But
looking at the LightState code, there’s virtually nothing (pun intended) in
there that I can use for implementing the required behavior. There are only
a few overridden methods from the Component class, but those don’t provide
any useful hooks to do my thing.

I cannot simply hide the base class methods by re-implementing them, because
the LightState is used in polymorph calls. Therefore, my re-implemented
methods would not be visible to the calling code.

Next idea then: maybe I can add some accessors to my derived LightState
class, to make it possible for my LightOutput impementation to get access to
the now missing data.

## Yeah but, ...

I can add extra methods to the derived class, but my LightOutput class is
unable to access them. The derived LightState class will call the method
`write_state(light::LightState *state)`, in which polymorphism rules will
hide the custom methods from the derived class.

It is literally running in cirlces here. With the LightOutput being passed
to the LightState constuctor and the LightState being passed to the
LightOutput in the write_state() method. That squishes the options for
extending the default LightState class.

## Are the responsibilities correctly assigned in ESPHome?

I am getting the strong feeling that the responsibilities in the ESPHome
light classes aren't as clean as they could be. What I want is my
LightOutput to be responsible for translating a requested light state or
transition into the actual GPIO output levels. However, the light classes
assume that transitions will work correctly, when transitioning over the
light state properties. My light output class is never in the lead here, and
can only react to light state changes.

## Okay, one more try

One possibility is to define an extra interface on my custom LightState
class, that is used for expsing the required data. The LightOutput class can
get a method to store my custom LightState as a pointer to this extra
interface. This method can be called from the customer LightState's
constructor.

It does not feel really clean, but this is a technical possibility to get
the plumbing going. The concept has been implemented and can be checked
out in my github repo:

- [a LightStateDataExposer interface class](https://github.com/mmakaay/esphome-yeelight_bs2/blob/92d935b0b47ba4a4457235acf28a37d3e03ee8be/light_output.h#L31)
- [a setter](https://github.com/mmakaay/esphome-yeelight_bs2/blob/92d935b0b47ba4a4457235acf28a37d3e03ee8be/light_output.h#L78)
  to pass a LightStateDataExposer to the LightOutput class
- [use of the LightStateDataExposer](https://github.com/mmakaay/esphome-yeelight_bs2/blob/92d935b0b47ba4a4457235acf28a37d3e03ee8be/light_output.h#L82)
  in the `write_state` method
- [the LightState implementation](https://github.com/mmakaay/esphome-yeelight_bs2/blob/92d935b0b47ba4a4457235acf28a37d3e03ee8be/light_output.h#L241)
  which also implements the LightStateDataExposer interface

This has been validated to work. I can access transitioning settings.

## Great, so now it's full steam ahead?

That remains to be seen. The next hurdle will be exposing enough data
and interpreting those data correctly. One issue that I already see coming
up, is that there's knowledge bound to the LightTransformer class that is
handling a transformation:

- The flash transformer will set the color of the light to a different
  color for a short period of time. In the data, the flash color will
  be in the current settings and the original color to return to will be in
  the target settings.
 
- The transition transformer will gradually change the color of the light
  from the original settings to the target settings. In the data, the
  current color will be in the current settings and the target color will be
  in the target settings.

When only looking from the outside at the data, these two cases look the
same and it is not clear whether gradual transitioning is done or a flash.
One could do some heuristics on the transformation over time. When the
current data do not change while the transformation is running, it likely is
a flash. One could also check if the transformer class is a flash or a
transitioning.

Both don't feel right. When the ESPHome light implementation gets a new
transformer implementation, my LightOutput would have to be updated to
recognize the new transformer logic. As a fan of the SOLID principles, that
does not give me warm fuzzy feelings.

## Hrm, again, a dead end...

I decided to go for a setup in which I would do some type checking to find
out what kind of transition is being handled. I can't get it to work
though. Using dynamic_cast is prohibited by the `-fno-rtti` compile flag.
I tried using some polymorphic dispatch calls to work out the active
tranformer type, but I can only see `std::unique_ptr<LightTransformer>` and
not the type of the derived class from my code.

All that is left at this point, is looking at the behavior of the active
tranformer, which brings me to the heuristic path. I don't like where this
is going. Way too many hoops to get where I need to be.

