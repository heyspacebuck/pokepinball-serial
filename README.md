# Pokémon Pinball for Serial Peripherals

This is a fork of [the Pokémon Pinball disassembly by PRET](https://github.com/pret/pokepinball).

The original instructions for setting up the repository are at [**INSTALL.md**](INSTALL.md).

## From Rumble to Serial

In `home.asm`, `options_screen.asm`, and some of the files in `engine/pinball_game/object_collision/`, you can find lines like this:

```asm
ld a, $ff
ld [wRumblePattern], a
ld a, $60
ld [wRumbleDuration], a
```

The first value (`0xff` in this example) sets the intensity of the motor based on the number of high bits (so, 100% intensity in this example). The second value (`0x60`) sets the duration of the rumble, in frames (60<sub>16</sub> = 96<sub>10</sub>, so at 60 fps this would be 1.6 seconds of rumble).

Before each of these sets of rumble instructions, I have inserted my own instructions:

```asm
ld a, $f6
ld [rSB], a
ld a, $1
ld [rSC], a
ld a, $81
ld [rSC], a
```

Writing a value to the `rSB` register and then writing `0x81` to the `rSC` register causes the game to transmit the `rSB` value over the serial cable. Setting `rSC` to `0x01` before setting it to `0x81` doesn't *seem* necessary, but that's how the "real" link cable instructions do it so I tried to put that in when possible.

Within the game there are a few different rumble patterns used that work out to 25% intensity, 50% intensity, or 100% intensity. And they run for either 1, 3, 4, 8, 64, or 96 frames. So, whenever a rumble event occurs, these new lines send that info out as a single byte for the peripheral to decode. Is it possible to transmit more than a single byte? Probably. But I haven't tried yet.

## The Peripheral

When the game sends an instruction over serial, it puts out an 8 kHz clock signal. Data gets read on the rising clock edge. I've set up an Arduino Uno to read the data line on clock rising-edge interrupts, which I'll put in the repo unless I forget. Right now that Arduino doesn't *do* anything with the data it receives; the next plan is to drive a DC motor through a MOSFET, and later use it to control a Hitachi/Doxy magic wand.

## I Am Not Good at Assembly

When building `blue_stage_resolve_collision.asm`, the linker(?) writes one particular `jr` instruction that is out-of-bounds (it tries to jump 128 lines, but the only valid range is -128 to 127). Instead of looking up ways to address this, I just commented out the `ld a, $1` line. This might affect the rumble/serial when hitting the Pikachu on the Blue Table.

Every time I tried adding a serial-transmit instruction to `red_stage_resolve_collision.asm`, I received the error

```
ERROR: main.asm(81) -> engine/pinball_game/draw_sprites/draw_red_field_sprites.asm(569):
    Section 'bank5' grew too big (max size = 0x4000 bytes, reached 0x4002).
```

Instead of learning how banks work and stuff, I just commented out the original rumble instructions and it compiles now. So this means, if you somehow put this ROM onto a cartridge with a built-in vibrator, it won't vibrate on the Red Table. But the serial port will work.