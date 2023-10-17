# Dplug Faust Example

![Screenshot](./screenshot.png)

This is an example plugin using Dplug with a Faust backend.  It is a stereo reverb plugin using the Freeverb demo from the Faust library.

# How to Build
Install the ldc2 D compiler.

Clone [Dplug](https://github.com/AuburnSounds/Dplug)
```
git clone https://github.com/AuburnSounds/Dplug.git
```

Build `dplug-build`
```
# From Dplug repository
cd tools/dplug-build
dub --compiler ldc2
```

Add `dplug-build` to your path variable.

Clone this repo.
```
git clone https://github.com/ctrecordings/dplug-faust-example.git
```

Run `dplug-build`
```
dplug-build -c LV2 --compiler ldc2
```

You should now see the built LV2 plugin in `builds/`

You can copy this plugin to the folder on you PC that holds LV2 plugins.  

If you are licensed to build VST2 or VST3 plug-ins you may also target those formats.  See Dplug's documentation for more info on this.


# How it Works

The faust DSP code is written in `dsp/freeverb.dsp`.  Currently only one faust module can be supported in a single project.

In `dub.json` we specify a pre-build command to compile the faust code to Dplug format
```
"preBuildCommands": [
    "faust dsp/freeverb.dsp -lang dlang -a dplug.d -cn Freeverb -vec -o src/freeverb.d"
]
```

The faust architecture file for Dplug creates a complete gui-less Dplug plugin which has it's own entrypoints. We need to override these entrypoints so there is a version identifier provided to do this.
In `dub.json` we add the `faustoverride` version.

```
"versions": ["futureMouseDrag", "faustoverride"],
```

Now we are able to import and inherit from the generated Dplug client.

```
final class ExampleClient : FaustClient
```

As well as define our own entry points

```
mixin(pluginEntryPoints!ExampleClient);
```

## Linking params

When we specify our params enum, it is important that they are ordered to match what faust generates.

We can look at the `buildUserInterface` method of our generated `freeverb.d` to see the order.

Some code removed for conciseness.
```D
	void buildUserInterface(UI* uiInterface) nothrow @nogc {
        ...
		uiInterface.addVerticalSlider("Damp", &fVslider0, cast(FAUSTFLOAT)0.5, cast(FAUSTFLOAT)0.0, cast(FAUSTFLOAT)1.0, cast(FAUSTFLOAT)0.025);
        ...
		uiInterface.addVerticalSlider("RoomSize", &fVslider2, cast(FAUSTFLOAT)0.5, cast(FAUSTFLOAT)0.0, cast(FAUSTFLOAT)1.0, cast(FAUSTFLOAT)0.025);
        ...
		uiInterface.addVerticalSlider("Stereo Spread", &fVslider3, cast(FAUSTFLOAT)0.5, cast(FAUSTFLOAT)0.0, cast(FAUSTFLOAT)1.0, cast(FAUSTFLOAT)0.01);
        ...
		uiInterface.addVerticalSlider("Wet", &fVslider1, cast(FAUSTFLOAT)0.3333, cast(FAUSTFLOAT)0.0, cast(FAUSTFLOAT)1.0, cast(FAUSTFLOAT)0.025);
        ...
	}
```

Alternatively we can add metadata to our controls by naming them like `[paramIndex] Control Name`. For example: `[3] Wet`. Although this example does not leverage this since it's using the freeverb demo.

So to match this we need to specify our params enum as follows:
```D
enum : int
{
    paramDamp,
    paramRoomSize,
    paramStereoSpread,
    paramWet
}
```

The last step is to override `processAudio` so we can update our UI if needed.

```D
override void processAudio(const(float*)[] inputs, float*[]outputs, int frames, TimeInfo info)
{
    assert(frames <= 512);
    super.processAudio(inputs, outputs, frames, info);

    if (ExampleGUI gui = cast(ExampleGUI) graphicsAcquire())
    {
        // update gui here.
        graphicsRelease();
    }
}
```
