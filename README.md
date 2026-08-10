# ogol-pico2

[Ogol](https://github.com/sysl-lang/ogol) on a Raspberry Pi Pico 2 W, over the USB serial port — the
same program as [the hosted console](https://github.com/sysl-lang/ogol-host), differing only in where
its bytes come from.

```
Ogol on a Raspberry Pi Pico 2 W. Type a line.

ogol> print 3 + 4
7
ogol> set a 5
ogol> print a
5
```

Echo, backspace, `←→`, Home, End, Delete, `Ctrl-A/E/B/F/U/K` and a 64-line history on the arrows —
none of which the port supplies, because a USB CDC connection has no line discipline. All of it is
`sysl.term.edit`, which is why this program is short.

## What is actually here

Almost nothing, and that is the point:

| | |
|---|---|
| the language | [`sh.sysl.ogol`](https://github.com/sysl-lang/ogol), a coordinate — including `session`, the read-run-print loop |
| the board | [`sh.sysl.pico2`](https://github.com/sysl-lang/pico2), a coordinate — `console_in()` and `console_out()` are the port as a `Reader` and a `Writer` |
| the editor | `sysl.term.edit`, in sysl's standard library |
| here | which streams to build, and a banner |

**There is no C in this project at all.** sysl exports `main`, the SDK's `crt0` branches straight into
it, and the archive in `add_executable`'s source list is what satisfies CMake's "a target must have
sources" rule with no translation unit to infer a language from.

## Building

Needs **sysl 0.0.38 or newer**, the [pico-sdk](https://github.com/raspberrypi/pico-sdk), `picotool`,
and Arm's toolchain from the **`gcc-arm-embedded` cask** — Homebrew's `arm-none-eabi-gcc` *formula*
ships no newlib and cannot link the SDK's own boot stage.

```
PICO_SDK_PATH=/path/to/pico-sdk cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
picotool load -f build/sysl_ogol.uf2
```

Then `screen /dev/cu.usbmodem2101 115200`.

`-f` reflashes a running board with no button, because firmware built with `pico_enable_stdio_usb`
carries the USB reset interface. A board that has never been flashed, or one whose port is held by
another program, needs BOOTSEL: unplug, hold the button, plug in, release. It mounts as
`/Volumes/RP2350`.

## Licence

ISC.
