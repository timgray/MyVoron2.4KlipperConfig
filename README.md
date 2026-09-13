# My Voron 2.4 Klipper Configuration

This is the current configuration for my Voron 2.4 300mm.  I do not intend this configuration to be a generic "copy these files into your Voron and everything will magically work" setup.  If you do that there is a very good chance you are going to have a bad day.  Pins, MCU IDs, probe locations, motor directions, heater limits, offsets and a pile of other things are specific to MY printer.

This aims to be a working example of how I set up my 2.4, and more importantly, why I chose to do some of the things in this way.  This printer has evolved over time and recently I went through the configuration and cleaned up a lot of old macros, duplicate behavior and things that had accumulated over years of changing hardware.

There is some custom stuff here, but none of it is magic.  The goal was to make the printer predictable.  Home means home, QGL means QGL, pause actually pauses safely, filament changes do what I expect, and the LEDs/buttons give me useful information instead of just looking pretty.

Oh did I use AI to make this MD file?  You bet your ass,  I am not going to sit here typing and researching all this crap through months of config file changes and tuning just to make some Anti AI witch hunters happy.  If my use of AI bothers you, then this makes me very happy.  I love driving unreasonable people insane,  it's kind of a hobby of mine.

## Printer Hardware

The printer is a 300mm Voron 2.4 running Klipper.

The main controller is a BIGTREETECH Octopus with the normal CoreXY A/B motion and four independently driven Z motors.  X and Y are configured for 300mm of travel and usable Z is configured to 280mm.

The toolhead is a Stealthburner/Clockwork 2 setup using a BIGTREETECH EBB CAN board.  The EBB is connected over CAN bus and handles most of the toolhead hardware locally:

- Extruder stepper and TMC2209 driver
- Hotend heater and thermistor
- Part cooling fan
- Hotend fan with tach feedback
- EBB electronics cooling fan
- Inductive Z probe input
- Stealthburner RGBW LEDs
- ADXL345 accelerometer

The extruder is Clockwork 2 with the 50:10 gear ratio and 5mm Bondtech style drive gears.  Extruder rotation distance has been calibrated on this actual machine.  There is currently a small printer-side pressure advance value in `EBBCanBus.cfg`.  I normally tune filament-specific pressure advance in the slicer, so if you copy anything from this repo pay attention to that and do not accidentally apply pressure advance twice.

The Z probe is an Omron TL-Q5MC2 style inductive probe connected to the EBB CAN board.  It has a 25mm Y offset and is sampled three times using the median result.  This probe has proven extremely repeatable on this machine, down around a few thousandths of a millimeter during repeated testing, which is kind of ridiculous when you stop and think about what we are asking a little inductive sensor to do.

There is a Nevermore recirculating filter in the chamber, chamber lighting, a BTT Mini12864 display, and a separate RP2040 hotkey board with seven physical buttons and addressable LEDs.

Filament detection is a BIGTREETECH Smart Filament Sensor V2.0 using both of its functions.  One input is the normal filament-present switch and the second is the motion encoder.  BTT specifies 2.88mm for the V2.0 motion detection distance, so that value is intentional and was not changed to Klipper's more generic 7mm example value.

## File Layout

I deliberately split hardware into a few files but moved the actual printer behavior into one macro file.

`printer.cfg` is the main machine configuration.  Motion, Z drives, bed heater, probe, QGL, mesh, display, filament sensors and the normal printer-level configuration live here.

`EBBCanBus.cfg` contains the CAN toolhead MCU, extruder, hotend, toolhead fans, ADXL345 and resonance tester.

`sb_leds.cfg` contains the Stealthburner LED hardware definition.

`Nevermore.cfg` contains the Nevermore fan, delayed shutoff and its Mini12864 controls.

`hotkey.cfg` contains the RP2040 hotkey MCU, seven physical button inputs and the hotkey LED chain.

`macros.cfg` is intentionally the central location for behavior.  Print start/end, G32, pause/resume/cancel, filament loading, nozzle cleaning, input shaper calibration, hotkey actions, LED states and the custom display menus are all maintained there.  I had pieces of this spread through several files before and that eventually turns into "which PAUSE macro am I actually running?"  No thanks.

## PRINT_START

The print start sequence got a fair amount of attention because I wanted it deterministic instead of relying on slicer timing or a collection of unrelated start G-code.

The slicer calls:

```gcode
PRINT_START EXTRUDER=[nozzle_temperature_initial_layer] BED=[bed_temperature_initial_layer_single]
```

`PRINT_START` takes it from there.

First the BTT filament motion encoder is disabled.  The physical filament-present switch remains active.  I found that allowing the motion encoder to monitor all of the startup/purge extrusion could occasionally cause a false runout right as a print began.  The SFS V2.0 only has 2.88mm between expected motion transitions, so rather than making the sensor less sensitive I start its monitoring when the actual print is ready to begin.

The skew correction profile is loaded and both heaters are explicitly turned off before `G32`.

This part is intentional.  I want homing and QGL to happen under a repeatable condition.  The inductive probe is being used as a measuring instrument here, so changing heater state in the middle of establishing the machine geometry is exactly the sort of variable I do not want.

After G32 is complete the bed begins heating to the slicer's requested temperature and the hotend is brought to a 150C standby temperature at the same time.  The 150C standby is just free time.  The bed takes much longer to heat, so there is no reason to leave the hotend completely cold and then wait for it afterward.  At the same time I do not want it sitting at 250C for the entire bed warmup drooling filament all over itself.  Once the bed is ready the hotend makes the short trip from 150C to actual print temperature.

At full print temperature the nozzle is wiped on the rear brush and Z is homed again with a hot, clean nozzle.  QGL is NOT repeated hot.  The saved default bed mesh is then loaded, the Nevermore starts, and the nozzle is primed.

Only after the startup extrusion is complete is the SFS motion encoder enabled.  The slicer then starts the actual object.

It sounds like a lot written out, but from the printer's point of view it is just a very deliberate sequence where every step has a reason.

## G32, Homing and QGL

`G32` is my normal "make the machine geometrically ready" command.

It:

1. Homes the printer.
2. Wipes the nozzle.
3. Re-homes Z.
4. Turns on the nozzle LEDs.
5. Runs Quad Gantry Level only if Klipper does not already report QGL as applied.

That last part matters.  There is no reason to beat the gantry around repeatedly if it is already leveled and Klipper still knows that state is valid.

The QGL itself uses the four independently driven Z motors to establish the gantry plane.  I also tested back-to-back QGL runs while troubleshooting first-layer behavior.  The second run was essentially sitting on top of the first, which gave me confidence that the Z system is staying put and QGL is repeatable.

When the motors are released that geometry can no longer be trusted.  The front hotkey LEDs use Klipper's actual homed/QGL state instead of maintaining some separate pretend state in the macros.

## Nozzle Cleaning

There is a fixed brush at the rear of the machine and `WIPE_NOZZLE` moves across it in several directions.

The wipe position was physically tested on this printer.  It is not a generic Voron coordinate and absolutely should not be copied blindly to another machine unless you enjoy finding out exactly how strong your brush mount is.

The final print-start wipe is done at full nozzle temperature immediately before the final Z home.  That is important to me.  If I am going to establish Z from the nozzle, I want the nozzle hot and I want the end of it clean.

There is also a `SERVICE_PRINTHEAD` macro.  It gets the machine into a known state and brings the toolhead to the front center at a useful working height with the nozzle lights on.  This is one of those silly little macros that becomes REALLY convenient once you have it.

## Bed Mesh

The current mesh is 9x9 over the usable central area of the bed:

```text
mesh_min: 40,40
mesh_max: 260,260
probe_count: 9,9
zero_reference_position: 150,150
algorithm: bicubic
```

I moved from a 5x5 mesh to 9x9 because I wanted more local information about the bed while chasing first-layer variation.  The zero reference is the center of the bed, not the physical Z endstop location.

There are two mesh workflows.

`BED_MESH_CALIBRATE_SAVE` is the simple home/QGL, probe and save workflow.

`BED_MESH_CALIBRATE_TEMP` is the more deliberate temperature version.  It establishes the machine geometry cold first.  If a bed temperature is requested it then heats and optionally soaks the bed before probing.

There is also probe callback logic that can gate the bed heater off during an individual probe sample and restore it afterward.  That exists because an inductive probe is sensitive enough that I wanted the option to remove heater switching as a variable while collecting a temperature mesh.

The important part here is not "everyone should do this."  The important part is that I was troubleshooting a measurement problem and wanted to control the variables instead of just adding more mesh points and hoping.

## First Layer / Probe Investigation

A fair amount of the recent configuration work happened while chasing a first-layer problem where the rear of the bed appeared more squished than the front.

The probe itself tested VERY repeatably.  Ten-sample `PROBE_ACCURACY` tests at several Y positions were generally around 0.0005 to 0.0012mm standard deviation.

QGL was also repeatable and the toolhead had no obvious mechanical play.

Manual nozzle-to-bed measurements with the mesh explicitly cleared showed a physical front-to-rear difference that did not agree with what the inductive probe was reporting.  Rotating the removable spring steel plate 180 degrees did not make that difference follow the plate, which made the removable sheet a much less likely explanation.

That investigation is why some of the probe/mesh behavior in this config is intentionally conservative.  I am not trying to hide a mechanical problem with a giant mesh.  I want the probe, machine geometry and actual nozzle-to-bed relationship to agree with each other.

This is still something I am evaluating, so treat the current probe settings as MY working configuration, not a declaration that this is the One True Way to configure an Omron probe.

## Pause, Resume and Cancel

I replaced the collection of inherited pause/cancel behavior with macros that are explicit about what happens.

`PAUSE`:

- Saves the current print state.
- Disables both filament sensors so changing/removing filament does not recursively trigger another runout.
- Raises Z if there is room.
- Parks the toolhead at the front center.
- Keeps the current hotend target instead of immediately letting it go cold.
- Extends the idle timeout to 12 hours.
- Changes the physical Pause button LED to show that the machine is paused.

`RESUME`:

- Re-enables both filament sensors.
- Restores the normal idle timeout.
- Waits for the saved nozzle temperature if necessary.
- Returns from the park position.
- Primes 2.5mm of filament.
- Restores the original print position and resumes.

This was tested with the actual workflow I care about: Pause -> Unload -> Load -> Resume.

`CANCEL_PRINT` is separate from `PRINT_END`.  A cancelled print is not a successful print and I do not want those two paths pretending they are the same thing.  Cancel retracts when safe, turns off the heaters, parks when possible, clears mesh/skew state, updates LEDs, leaves the Nevermore running briefly, and then calls Klipper's base cancel behavior.

## Filament Load and Unload

`LOAD_FILAMENT` and `UNLOAD_FILAMENT` are built around a 250C filament-change temperature.

They remember the previous hotend target, heat to the change temperature, perform the load/unload sequence, then restore whatever temperature the printer had before the macro was called.

The LEDs change during the process and the printer beeps before/after the operation.  This is mostly quality-of-life stuff, but it also means the same physical buttons work whether I am standing at the printer or controlling it through Mainsail.

The load sequence feeds 130mm total with progressively slower motion near the nozzle.

The unload sequence retracts 120mm total.

Those distances are for this machine and filament path.

## BTT Smart Filament Sensor V2.0

This printer uses both outputs from the BTT SFS V2.0.

The normal switch tells Klipper whether filament is physically present.

The motion sensor tells Klipper whether filament is actually moving when the extruder believes it should be moving.  This can catch a jam or other condition where filament is technically still present but is no longer feeding.

The V2.0 uses:

```text
detection_length: 2.88
```

That number came from BTT's own V2.0 documentation.  The older V1.0 sensor used 7mm, which is where some confusing examples on the internet come from.

As mentioned above, the motion sensor is deliberately disabled during `PRINT_START` and enabled immediately before the actual print begins.  I would rather control WHEN the 2.88mm detector starts watching than make it less useful by simply increasing its detection distance.

## Input Shaper and ADXL345

The EBB CAN toolhead has an ADXL345 wired directly to it and Klipper's `resonance_tester` is configured.

I added two macros around this.

`CALIBRATE_INPUT_SHAPER` is the "just calibrate the machine" version.  It prepares the printer, runs the X and Y shaper calibration and saves the resulting configuration.

`TEST_RESONANCES_ALL` is the diagnostic version.  It collects the raw X and Y resonance tests without replacing the saved input-shaper settings.

I like having both because "I want Klipper to tune the shaper" and "I want to see if something mechanically changed" are two different jobs.

The current saved shaper values are machine calibration data.  Do not copy those numbers to another printer.  That completely misses the point of having an accelerometer.

## Skew Correction

The printer has a saved skew profile named `TimSkew`, and `PRINT_START` loads it automatically.

Again, that is measured calibration for THIS frame.  If you are building your own config from this repo, calibrate your own machine.

## Nevermore

The Nevermore is exposed as a generic fan.

`PRINT_START` turns it on at full speed and `PRINT_END` schedules it to continue running after the print so it can keep circulating/cleaning the chamber for a while.

The Mini12864 also has direct Nevermore on/off and speed controls, so I can change it without opening Mainsail.

## Stealthburner and Chamber LEDs

The LEDs are being used as status indicators, not just decoration.

The Stealthburner nozzle LEDs can be turned on for cleaning/service work and the logo/status LED changes during printer operations.

The chamber lighting also changes state during operations such as printing, cleaning and completion.

I intentionally consolidated the LED behavior into `macros.cfg`.  There used to be more little status macros scattered around and several of them were either duplicates or no longer used.  Fewer layers of indirection makes this MUCH easier to understand six months later when I have forgotten why Tim from six months ago thought something was clever.

## Physical Hotkey Board

There is a separate RP2040 board on USB running seven physical buttons with addressable LEDs.

Current button assignments are:

```text
B1 - G32 / QGL
B2 - Home
B3 - Service Printhead
B4 - Cancel Print
B5 - Pause
B6 - Unload Filament
B7 - Load Filament
```

B1 and B2 are state indicators as well as buttons.

B1 is red when QGL is not currently applied and green when Klipper says it is applied.

B2 is red when the printer is not homed and green when XYZ are homed.

B5 changes color while the printer is paused.

An earlier version periodically polled Klipper and refreshed the LEDs every couple seconds.  That caused annoying Busy/Standby flicker in Mainsail, so I removed the constant polling.  State is updated at meaningful points instead.

There are only seven physical buttons.  Old B8-B12 definitions were removed instead of leaving dead config around just because it once existed.

## Mini12864 Menus

I added a `Voron Tools` menu to the Mini12864 so the useful macros are available while standing at the printer.

The menu is organized instead of dumping everything at the top level:

```text
Voron Tools
  Printer
    Home + QGL
    Service Printhead
    Wipe Nozzle

  Filament
    Load Filament
    Unload Filament

  Calibration
    Bed Mesh - Cold
    Calibrate Shaper
    Test Resonances
```

Pause/Resume and Nevermore controls are also available through the display.

I deliberately put things like input-shaper calibration under a Calibration submenu.  Accidentally clicking a menu item should not immediately result in the printer enthusiastically shaking itself across the room.

## Exclude Object

`printer.cfg` includes:

```ini
[exclude_object]
```

OrcaSlicer emits `EXCLUDE_OBJECT_DEFINE` information when object exclusion is enabled.  Without Klipper's `exclude_object` module those commands just show up as unknown commands in the console.

Enabling it also gives Mainsail the ability to cancel one failed object on a multi-object plate instead of throwing away the entire print.

## PRINT_END

A successful print has its own cleanup path.

The macro waits for buffered motion, retracts filament, turns off the heaters, makes a safe move/park, shuts down the part fan, clears temporary mesh/skew state as appropriate, handles the LEDs and Nevermore post-print behavior, and eventually releases the motors.

Once the motors are released I no longer consider Home or QGL to be valid states.  The hotkey state update reflects Klipper's actual state so the physical buttons do not keep telling me the machine is homed/leveled after the motors have been turned off.

That sounds like a small detail, but it is exactly the sort of thing I want from physical status indicators.  If the light says the machine is ready, it should actually be ready.

## Things I Intentionally Removed

This config had accumulated a lot of history.

During the cleanup I removed duplicate or dead macros, old button definitions for hardware that does not exist, duplicate G32 definitions, stale status helpers, old purge-line code that was no longer being used, and pause/cancel macros that depended on pieces from other macro packages.

I also stopped actively including `mainsail.cfg` because I wanted one known implementation of Pause/Resume/Cancel instead of stacking my macros on top of another macro package and hoping rename chains all resolve the way I think they do.

The general rule became: if a macro is not used, remove it.  If two things do the same job, pick one.  If something depends on a mystery `_KM_*` helper from a package that is no longer actually installed, it definitely does not belong in the active printer configuration.

Boring config is good config.

## SAVE_CONFIG - Do Not Casually Delete This

The bottom of `printer.cfg` contains Klipper's `SAVE_CONFIG` generated data.

That is NOT junk.

It contains actual calibration results for this printer, including things such as Z/endstop calibration, PID values, input shaper, skew correction, probe calibration and saved bed meshes depending on what has been calibrated/saved.

During this cleanup I treated that block as critical data and preserved it between revisions.

If you use this repo as a reference, do not copy my `SAVE_CONFIG` values into your machine.  If you are modifying your own config, also do not casually delete your existing block because somebody on the internet posted a cleaner-looking `printer.cfg`.

Calibration is data.

## A Note About Copying This Configuration

Please use this as a reference, not firmware-by-ouija-board.

At minimum, YOUR machine will have different:

- MCU serial/USB identifiers
- CAN UUID
- Motor directions
- Endstop locations
- Probe offset and Z offset
- PID calibration
- Extruder rotation distance
- Bed mesh
- Input shaper frequencies
- Skew correction
- Brush location
- Possibly fan and heater pins
- Probably several things I forgot to put in this list

Some commands in these macros physically move the printer to coordinates that are safe on MY 300mm Voron.  Verify them before running them on anything else.

If you copy a service macro from a stranger on GitHub and drive your toolhead into a bucket of tools sitting in front of the printer, that is between you, Klipper and the bucket.

## Why I Did All This

Mostly because the printer worked, but the configuration had reached that stage where it worked because I knew all of its weird little behaviors.

That is fine right up until six months from now when I am the idiot trying to remember what I did.

I wanted the configuration to describe the printer I actually have today.  CAN toolhead, Stealthburner, Nevermore, physical hotkeys, Mini12864, BTT filament monitoring, nozzle brush, ADXL input-shaper calibration and the current print workflow.

More importantly I wanted the behavior to be obvious when reading the files.  A macro should tell you WHY it is doing something, not require archaeology through four include files to discover which version of `PAUSE` won.

This is still a machine I tinker with, so the config will continue changing.  That is sort of the point of building a Voron in the first place.

 