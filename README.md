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

### Or on the RISC-V cores

**The RP2350 has two personalities and this program runs on either.** The chip carries a pair of
Cortex-M33s *and* a pair of Hazard3 RISC-V cores, and which set boots is a build-time choice — so
this is one project with two configurations rather than two projects. The whole difference is
`PICO_PLATFORM`, which the `CMakeLists.txt` also reads to decide sysl's `--target`.

```
PICO_SDK_PATH=/path/to/pico-sdk cmake -B build-riscv -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DPICO_PLATFORM=rp2350-riscv -DPICO_TOOLCHAIN_PATH=/path/to/riscv-toolchain-15
cmake --build build-riscv
picotool load -f build-riscv/sysl_ogol.uf2
```

**It needs a RISC-V toolchain, and not any RISC-V toolchain.** The SDK looks for `riscv32-pico-elf`,
`riscv32-unknown-elf`, `riscv32-corev-elf` or `riscv-none-elf`; Arm's provides none of them. Use
Raspberry Pi's own — `riscv-toolchain-15-mac.zip` and its siblings, in the
[pico-sdk-tools](https://github.com/raspberrypi/pico-sdk-tools/releases) releases — which carries
newlib and an `rv32imac_…_zba_zbb_zbkb_zbs/ilp32` multilib for what Hazard3 actually is. Homebrew's
`riscv64-elf-gcc` is the wrong triple *and* ships no C library, which fails deep inside the SDK's own
boot stage rather than in anything you wrote.

Nothing in the program changes between the two. `picotool info` is what tells them apart: `family ID
'rp2350-riscv'` and `image type: RISC-V` against `'rp2350-arm-s'` and `ARM Secure`.

Then `screen /dev/cu.usbmodem2101 115200`.

`-f` reflashes a running board with no button, because firmware built with `pico_enable_stdio_usb`
carries the USB reset interface. A board that has never been flashed, or one whose port is held by
another program, needs BOOTSEL: unplug, hold the button, plug in, release. It mounts as
`/Volumes/RP2350`.

## What a refusal looks like, and what it costs

**ogol v0.3.1 points at the mistake.** The language's reader sits on `sh.sysl.parsing`, so every token
carries the span it was written at and the console draws the line with a caret under the word:

```
ogol> to twice :n
.... output n * *
error: '*' needs something in front of it
 --> <console>:2:12
  |
2 | output n * *
  |            ^
```

That is worth more here than at a desktop, because a held definition may be several lines long by the
time something in it is refused and a serial console has nothing to scroll back to.

**It costs 23,428 bytes of flash, measured rather than guessed at**: text went 183,796 → 207,224 on
sysl 0.0.79, which is 12.7%. The caret is placed by *display width* rather than by byte count, so a
line with a tab or a wide character in it still has the caret under the right thing — and that links
Unicode's width tables, which is nearly all of the increase. On a 4 MB part it is not a question;
it would be one on a chip with tens of kilobytes.

**A fault from *running* has no caret and that is deliberate**, not a gap: `that divides by zero` is a
complaint about what a value came to, and the tree it came from carries no positions. The reader knows
where; the evaluator does not.

## Licence

ISC.
