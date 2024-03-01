- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# GPIO peripheral RFC

## Summary
[summary]: #summary

Add a SoC peripheral to control GPIO pins.

## Motivation
[motivation]: #motivation

[GPIOs](https://en.wikipedia.org/wiki/General-purpose_input/output) are useful for a wide range of scenarios, such as driving external circuitry or acting as fallback for unimplemented/misbehaving peripherals in early iterations of a design.

Amaranth SoC seems like an appropriate place for a GPIO peripheral, which depends on features that are already provided by the library. Due to its relative simplicity, it is also a good candidate for using the recent CSR register API in realistic conditions.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Usage

```python3
from amaranth import *
from amaranth.lib import wiring
from amaranth.lib.wiring import connect

from amaranth_soc import csr
from amaranth_soc import gpio


class MySoC(wiring.Component):
    def elaborate(self, platform):
        m = Module()

        # ...

        # Use a GPIO peripheral to control four LEDs:

        m.submodules.led_gpio = led_gpio = gpio.Peripheral(pin_count=4, addr_width=8, data_width=8)
        for n in range(4):
            connect(m, led_gpio.pins[n], platform.request("led", n, dir="io"))

        # Add the peripheral to a CSR bus decoder:

        m.submodules.csr_decoder = csr_decoder = csr.Decoder(addr_width=31, data_width=8)

        csr_decoder.add(led_gpio.bus, addr=0x1000)

        # ...

        return m
```

### Overview

The following figure is a simplified diagram of the peripheral. CSR registers are on the left-hand side, a single pin is on the right side:

<img src="./0049-soc-gpio-peripheral/overview.svg">

### Registers

#### Mode (read/write)

<img src="./0049-soc-gpio-peripheral/reg-mode.svg"
     alt="bf([
         {name: 'pin_0', bits: 2, attr: 'RW'},
         {name: 'pin_1', bits: 2, attr: 'RW'},
         {name: 'pin_2', bits: 2, attr: 'RW'},
         {name: 'pin_3', bits: 2, attr: 'RW'},
     ], {bits: 8})">

Each `Mode.pin_x` field can hold the following values:

```python3
class Mode(enum.Enum, shape=unsigned(2)):
    INPUT_ONLY = 0b00
    PUSH_PULL  = 0b10
    OPEN_DRAIN = 0b01
```

- If `Mode.pin_x` is `INPUT_ONLY`, then `pins[x].oe = 0`.
- If `Mode.pin_x` is `PUSH_PULL`, then `pins[x].oe = 1` and `pins[x].o = Output.pin_x`.
- If `Mode.pin_x` is `OPEN_DRAIN`, then `pins[x].oe = ~Output.pin_x` and `pins[x].o = 0`.

#### Input (read-only)

<img src="./0049-soc-gpio-peripheral/reg-input.svg"
     alt="bf([
         {name: 'pin_0', bits: 1, attr: 'R'},
         {name: 'pin_1', bits: 1, attr: 'R'},
         {name: 'pin_2', bits: 1, attr: 'R'},
         {name: 'pin_3', bits: 1, attr: 'R'},
         {bits: 4, attr: 'ResR0W0'},
     ], {bits: 8})">

Each `Input.pin_x` field holds the last value of `pins[x].i` sampled on a clock cycle.

#### Output (read/write)

<img src="./0049-soc-gpio-peripheral/reg-output.svg"
     alt="bf([
         {name: 'pin_0', bits: 1, attr: 'RW'},
         {name: 'pin_1', bits: 1, attr: 'RW'},
         {name: 'pin_2', bits: 1, attr: 'RW'},
         {name: 'pin_3', bits: 1, attr: 'RW'},
         {bits: 4, attr: 'ResR0W0'},
     ], {bits: 8})">

Each `Output.pin_x` field holds the next value of `pins[x].o` in `PUSH_PULL` mode.

#### SetClr (write-only)

<img src="./0049-soc-gpio-peripheral/reg-setclr.svg"
     alt="bf([
         {name: 'set_0', bits: 1, attr: 'W'},
         {name: 'set_1', bits: 1, attr: 'W'},
         {name: 'set_2', bits: 1, attr: 'W'},
         {name: 'set_3', bits: 1, attr: 'W'},
         {bits: 4, attr: 'ResR0W0'},
         {name: 'clr_0', bits: 1, attr: 'W'},
         {name: 'clr_1', bits: 1, attr: 'W'},
         {name: 'clr_2', bits: 1, attr: 'W'},
         {name: 'clr_3', bits: 1, attr: 'W'},
         {bits: 4, attr: 'ResR0W0'},
     ], {bits: 8, lanes: 2})">

- Writing `1` to an `SetClr.set_x` field sets `Output.pin_x`.
- Writing `1` to an `SetClr.clr_x` field clears `Output.pin_x`.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `amaranth_soc.gpio.PinSignature`

The `gpio.PinSignature` class is a `wiring.Signature` describing the interface between the GPIO peripheral and a single pin.

The members of a `gpio.PinSignature` are defined as follows:

```python3
{
    "i":  In(unsigned(1)),
    "o":  Out(unsigned(1)),
    "oe": Out(unsigned(1)),
}
```

### `amaranth_soc.gpio.Peripheral`

The `gpio.Peripheral` class is a `wiring.Component` implementing a GPIO controller, with:
- a `.__init__(self, *, pin_count, addr_width, data_width, name=None, input_stages=2)` constructor, where:
  * `pin_count` is a non-negative integer.
  * `input_stages` is `0`, `1` or `2`.
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "bus": In(csr.Signature(addr_width, data_width)),
    "pins": Out(gpio.PinSignature()).array(pin_count),
}
```

- a `.elaborate(self, platform)` method, that connects each pin in `self.pins` to its associated fields in the registers exposed by `self.bus`.

## Drawbacks
[drawbacks]: #drawbacks

While existing implementations (such as STM32 GPIOs) have features like pin multiplexing and configurable pull-up/down resistors, in the proposed design, those would have to be implemented in a separate component.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposed design moves platform-specific details outside of its scope, which:
- reduces the amount of non-portable code to maintain, while allowing implementation freedom for users needing it.
- avoids introducing dependencies on upstream APIs that are deprecated or expected to evolve soon (such as `amaranth.build`).

As an alternative:
- do not host any peripheral in amaranth-soc and always develop them downstream.
- include a pin multiplexer inside the GPIO peripheral.

## Prior art
[prior-art]: #prior-art

While they can be found in most microcontollers, the design of GPIOs in STM32 has inspired part of this RFC.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- ~~Should we support synchronizing a pin input on falling edges of the clock ?~~ Users can synchronize pin inputs on falling edges by instantiating a `gpio.Peripheral` with `input_stages=0`, and providing their own synchronization mechanism.

## Future possibilities
[future-possibilities]: #future-possibilities

- Implement a pin multiplexer peripheral, that can be composed with this one to allow reusing other pins of a SoC as GPIOs.
- Add support for interrupts.
