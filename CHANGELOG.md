# Changelog

## Unreleased

- `echo_width_us`, the awaited-edge form: an `async fn` over any
  `GpioInAsync` and `Timer` that parks the task on Echo's rising and
  falling edges and answers the pulse in microseconds, 38 ms at most. It is
  declared `@tier(rt)`, since the package's embedded tier refuses an
  `async fn`. It needs a toolchain in which a generic `async fn` that
  awaits builds, which the novo-lang repository fixes after 0.24.0.

## 0.1.0

The first release: a driver for the HC-SR04 ultrasonic ranging module on
two GPIO pins and a timer.

- `HcSr04`, the type a program builds from the board's GPIO and timer
  handles and the numbers of the Trig and Echo pins, implementing
  devices-nv's `Rangefinder`. The value keeps no state, so the trait's
  methods take a plain `self`: `range_start` answers 0 and `range_read`
  fires one measurement, times its echo and answers the distance in
  millimetres with its status.
- The measurement as functions over any `GpioOut` and `Timer`: the trigger
  pulse of 12 µs, the Echo pulse timed by polling the pin and the
  microsecond tick, the 38 ms cap that means no target, the 5 ms wait for
  Echo to rise, and the wait for Echo to be low before a trigger.
- The 60 ms measurement cycle, waited out at the end of each measurement
  from the tick of its trigger; `measure_fast` leaves it to the caller.
- The conversion of a pulse to millimetres at a speed of sound that is a
  parameter, 343 m/s by default, and `with_temperature_milli_c`, which takes
  the speed from the air's temperature (331.3 m/s plus 0.606 m/s a degree).
- Statuses: `RangeValid` from 2 cm to 4 m, `RangeOutOfRange` outside it,
  `RangeNoTarget` for a pulse of 38 ms or more, `RangeFault` for an Echo
  line that is high and stays high, and no reading for a module that does
  not raise Echo.
- The device program `examples/range_reader.nv`, a distance reader. Built
  for the nRF52840-DK it is 12,324 B of flash and for the nRF54LM20-DK
  13,620 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  module.
- The specification read, "Ultrasonic Ranging Module HC-SR04" from
  ElecFreaks, carries no document number and no revision. It does not give
  the 38 ms no-echo pulse; that figure is from Zephyr's driver and the
  modules' behaviour.
- Not implemented: input capture on a hardware timer, which embedded-hal-nv
  has no trait for.
- `devices-nv` is a path dependency (`../devices-nv`, 0.3.3), which has the
  `Rangefinder` trait and is not on the registry yet. `embedded-hal-nv` is
  the registry's 0.3.2, because a path copy of it conflicts with the
  registry requirement of devices-nv.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module
  also checks at `rt` and `system`.
