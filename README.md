# hcsr04-nv

A driver for the HC-SR04 ultrasonic ranging module, specified in
"Ultrasonic Ranging Module HC-SR04" from ElecFreaks (the copy that could
be fetched, three pages, carries no document number and no revision). The
module has no bus: it is driven through two GPIO pins, a trigger and an
echo, and a timer. The driver implements the `Rangefinder` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the
distance in millimetres.

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this module, so the driver has run only against a double of the pins and
the timer, in the host suite. The device program below builds for the
nRF52840 DK and the nRF54LM20 DK and has not been flashed.

## The device program

`examples/range_reader.nv` reads the module ten times a second and prints
the distance over the board's UART:

```
range 1000 mm
no target
```

```novo norun:needs-pkg
use embedded_hal.{ BoardGpio, BoardTimer }
use dev_range.{ RangeReading, Rangefinder }
use hcsr04.{ HcSr04 }

const TRIG = 3
const ECHO = 4

// Writes `n`, 0 or more, in decimal.  Formatting into a string would
// allocate, which @tier(embedded) refuses, so each digit goes out as a
// byte.  An Int has at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + (n / div) % 10)
            div = div / 10

// Writes the line for a reading.
@tier(embedded)
fn report(r: RangeReading) [hw]
    match r.status
        RangeValid =>
            hal.uart.write("range ")
            digits(r.mm)
            hal.uart.write(" mm\n")
        RangeWeak => hal.uart.write("weak\n")
        RangeOutOfRange => hal.uart.write("out of range\n")
        RangeNoTarget => hal.uart.write("no target\n")
        RangeFault => hal.uart.write("echo line stuck high\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    let sensor = HcSr04.attach(BoardGpio { id: 0 }, BoardTimer { id: 0 }, TRIG, ECHO)
    sensor.begin()
    // The module is ready 5 ms after its supply; waiting is the program's.
    hal.timer.delay_ms(5)
    while true
        if sensor.range_start() < 0
            hal.uart.write("no answer\n")
        else
            match sensor.range_read()
                Some(r) => report(r)
                None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The program is `examples/range_reader.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs
the image and checks that the program reports that the module does not
answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 12,324 B | 1,616 B | 2,616 B |
| `nrf54lm20-dk` | 13,620 B | 1,922 B | 2,616 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena, 65,536 B on both boards, is the runtime's heap and is reserved
whether or not the program uses it; the driver allocates nothing in it.
The driver's whole state is a `HcSr04` value of 40 bytes in the program's
frame.

The wiring: Trig to pin 3 and Echo to pin 4 (P0.03 and P0.04 on the
nRF52840 DK, the header's A0 and A1), the module's VCC to 5 V and its GND
to the board's. The module drives Echo at 5 V, and the boards' pins take
at most VDD plus 0.3 V, so Echo goes through a divider (1 k and 2 k) or a
level shifter. Trig accepts 3.3 V. The sheet gives the working voltage as
5 V.

### The effects

Every function of the driver that reaches a pin or the timer declares
`[hw]`, and nothing else. The driver performs no input or output of its
own and allocates nothing. The module checks at every tier, `embedded`
included. A measurement spins the core for the length of the echo, up to
38 ms, and for the rest of the 60 ms cycle if it is started early.

## What the module is

An ultrasonic module sends a burst of sound and times the echo. The
HC-SR04 sends eight pulses at 40 kHz when its Trig pin is held high for at
least 10 µs, and holds its Echo pin high from the end of the burst until
the burst comes back. The width of the Echo pulse is the time the sound
took to go to the target and return, so the distance is half the width
times the speed of sound.

| Item | Value | Source |
|---|---|---|
| Range | 2 cm to 4 m | the sheet |
| Trigger | a TTL pulse of at least 10 µs | the sheet |
| Conversion | `µs / 58 = cm`, or `time x 340 m/s / 2` | the sheet |
| Measurement cycle | more than 60 ms | the sheet |
| Beam | 15 degrees | the sheet |
| Echo with no target | high for about 38 ms | Zephyr's driver and the modules' behaviour, not the sheet |

The driver takes the speed of sound as 343 m/s, the figure
`dev_range.range_of_echo_us` uses, which differs by 0.9 % from the
sheet's 340 m/s. The speed in air rises by about 0.6 m/s a degree, so a
program that knows the air's temperature calls
`with_temperature_milli_c`, which uses 331.3 m/s plus 0.606 m/s a degree.
That line is the usual linear fit, not the sheet's.

## What the package contains

| Module | What is in it |
|---|---|
| `hcsr04` | The type `HcSr04` that holds a board's pins, timer and speed of sound, the measurement as functions over any `GpioOut` and `Timer`, the conversions and the reading |

## How to choose an entry point

A program builds one `HcSr04` with `HcSr04.attach`. The value holds the
pins, the timer and the speed of sound, and no state that a measurement
changes, so the `Rangefinder` methods take a plain `self` and a function
written against the trait can call them on a `let`.

| Method | What it does |
|---|---|
| `attach(gpio, timer, trigger, echo)` | the module on two pins of the board's GPIO, at 343 m/s; nothing is sent |
| `begin()` | Trig an output driven low, Echo an input |
| `range_start()` | `Rangefinder`: 0; the module measures when it is read, so there is nothing to start |
| `range_read()` | `Rangefinder`: one measurement, padded to the 60 ms cycle, as a distance and a status; `None` when the module did not raise Echo |
| `range_max_mm()` | `Rangefinder`: 4000 |
| `measure_fast()` | one measurement that leaves the 60 ms cycle to the caller |
| `echo_us()` | one padded measurement as the Echo pulse in microseconds, or a negative code: -1 no Echo, -2 Echo stuck high |
| `with_speed_mm_s(v)`, `with_temperature_milli_c(t)` | a copy of the module that converts at another speed of sound, directly or from the air's temperature |
| `speed_mm_s()` | the speed in use |

A program that holds its own pins, or a test with a double of the pins and
the timer, calls the module's functions: `hcsr04.measure(g, t, trigger,
echo, pad)` answers the pulse in microseconds, and
`hcsr04.reading_of(pulse, speed)` turns it into a reading.

The status of a reading follows `devices-nv`:

| Pulse | Status | Distance |
|---|---|---|
| 20 mm to 4 m | `RangeValid` | the distance |
| under 20 mm or over 4 m | `RangeOutOfRange` | 0 |
| 38 ms or more | `RangeNoTarget` | 0 |
| Echo high before the trigger and not falling | `RangeFault` | 0 |
| Echo never rises | no reading (`None`) | |

## The rules a user needs

1. The module is ready 5 ms after its supply. The program waits; the
   driver does not.
2. `range_read` blocks. The echo has to be timed as it comes, so the call
   spins until the Echo pin falls, up to 38 ms, and then until the 60 ms
   cycle is over. `range_start` answers 0 for that reason, and
   `range_read_after` of the trait accepts any wait.
3. The driver waits out the 60 ms cycle at the end of each measurement,
   counted from the trigger, so a loop can read at once and no trigger
   meets the echo of the last. The value keeps no state, so it cannot wait
   at the start of the next measurement instead, which would not spend the
   time of a caller who is slower than the cycle. A caller who keeps its
   own spacing uses `measure_fast`.
4. The driver does not trigger while Echo is high. A pulse from the last
   measurement that has not ended is waited for, up to 50 ms; an Echo that
   stays high is a `RangeFault` and no trigger is sent. Zephyr's driver
   fires the trigger regardless.
5. A pulse of 38 ms or more is "no target", and the driver stops timing at
   38 ms. A target past 4 m answers a pulse under 38 ms and is
   `RangeOutOfRange`.
6. The edges are found by polling, so each is seen up to one poll late, a
   microsecond or so on a Cortex-M4 (0.17 mm). The two edges carry the same
   delay, which cancels in the width. A timer whose `ticks_us` is coarser
   than a microsecond gives a coarser distance: 100 µs is 17 mm.
7. A measurement is made with the interrupts as the program has them. An
   interrupt that runs longer than a few microseconds between the edges
   adds to the width. A program with busy interrupts times the echo with a
   capture timer, which `embedded-hal-nv` does not name.
8. Echo is a 5 V output. See the wiring above.
9. On a host, `BoardGpio` reads Echo at 0, so a measurement ends after
   5 ms with no module.

## What is not included

NOT IMPLEMENTED:

- The awaited-edge form. `GpioInAsync` of embedded-hal-nv parks a task
  until an edge, which would time the pulse without polling. A generic
  `async fn` that awaits a method of its bound does not build with the
  toolchain this was written with: the filing is
  `bugs/language/a-generic-async-fn-that-awaits-a-bounded-trait-method-fails-monomorphization.md`
  in the novo-lang repository. No board implements `GpioInAsync` either.
- The `RangefinderAsync` trait, for the same reason.
- Input capture on a hardware timer, which `embedded-hal-nv` has no trait
  for.
- The 3-pin variant of the module, with Trig and Echo on one pin.
- Averaging several pulses or rejecting outliers.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `Rangefinder` trait this driver implements.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `GpioOut` and `Timer` traits and the board handles.
- [vl53l0x-nv](https://novo-lang.org/packages/vl53l0x-nv) drives ST's
  VL53L0X time-of-flight sensor as the same `Rangefinder`.

## Tests

`novo test` runs the module over a double of the pins and the timer: a
clock that advances on every read of the tick, and an Echo pin that rises
after the trigger falls and stays high for a set time. The suite checks
the conversions against the sheet's `µs / 58 = cm`, the speed of sound at
several temperatures, every status of a reading at its limits, the
trigger pulse's length, the order in which the pins are driven, a pulse
timed to within the poll, the 38 ms cap, a module that does not answer, an
Echo that is stuck high and one that is still high from the last pulse,
the 60 ms cycle between two triggers with and without the padding, a
function over the trait called on a `let`, and the type over the host's
board handles. The suite covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their fixes and not ported, are the
module's specification above, Zephyr's `drivers/sensor/hc_sr04` and the
`hc-sr04` Rust crate of nordmoen. Each fix kept is named in
`src/hcsr04.nv` where it is kept.

## Licence

Apache-2.0.
