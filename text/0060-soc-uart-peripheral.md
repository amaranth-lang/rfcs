- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#60](https://github.com/amaranth-lang/rfcs/pull/60)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# UART peripheral RFC

## Summary
[summary]: #summary

Add a SoC peripheral for UART devices.

## Motivation
[motivation]: #motivation

An UART is a generally useful peripheral for serial communication between devices.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Usage

```python3
from amaranth import *
from amaranth.lib import wiring
from amaranth.lib.wiring import connect

from amaranth_stdio.serial import AsyncSerialRX, AsyncSerialTX

from amaranth_soc import csr
from amaranth_soc import uart


class MySoC(wiring.Component):
    def elaborate(self, platform):
        m = Module()

        # ...

        # Instantiate an UART peripheral:

        uart_divisor = int(platform.default_clk_frequency / 115200)

        m.submodules.uart = uart = uart.Peripheral(divisor_init=uart_divisor, addr_width=8, data_width=8)

        # Instantiate and connect the UART PHYs:

        uart_pins = platform.request("uart", 0)

        uart_phy_rx = AsyncSerialRX(uart_divisor, divisor_bits=16, pins=uart_pins)
        uart_phy_tx = AsyncSerialTX(uart_divisor, divisor_bits=16, pins=uart_pins)

        m.submodules.uart_phy_rx = ResetInserter(uart.rx.rst)(uart_phy_rx)
        m.submodules.uart_phy_tx = ResetInserter(uart.tx.rst)(uart_phy_tx)

        m.d.comb += [
            uart_phy_rx.divisor.eq(uart.rx.divisor),

            uart.rx.symbols.payload.eq(uart_phy_rx.data),
            uart.rx.symbols.valid.eq(uart_phy_rx.rdy),
            uart_phy_rx.ack.eq(uart.rx.symbols.ready),

            uart.rx.overflow.eq(uart_phy_rx.err.overflow),
            uart.rx.error.eq(uart_phy_rx.err.frame),
        ]

        m.d.comb += [
            uart_phy_tx.divisor.eq(uart.tx.divisor),

            uart_phy_tx.data.eq(uart.tx.symbols.payload),
            uart_phy_tx.ack.eq(uart.tx.symbols.valid),
            uart.tx.symbols.ready.eq(uart_phy_tx.rdy),
        ]

        # Add the UART peripheral to a CSR bus decoder:

        m.submodules.csr_decoder = csr_decoder = csr.Decoder(addr_width=31, data_width=8)

        csr_decoder.add(uart.csr_bus, addr=0x1000)

        # ...

        return m

```

### Registers

#### Receiver

##### Control (read/write)

<img src="./0060-soc-uart-peripheral/reg-control.svg"
     alt="bf([
         { name: 'enable', bits: 1, attr: 'RW' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 8})">

`Control.enable` is initialized to 0 on reset.

- If `Control.enable` is 0, the receiver PHY should be held in reset state.
- If `Control.enable` is 1, the receiver PHY should operate normally.

##### Status (read/write)

<img src="./0060-soc-uart-peripheral/reg-rx-status.svg"
     alt="bf([
         { name: 'ready',    bits: 1, attr: 'R' },
         { name: 'overflow', bits: 1, attr: 'RW1C' },
         { name: 'error',    bits: 1, attr: 'RW1C' },
         { bits: 5, attr: 'ResR0W0' },
     ], {bits: 8})">

- `Status.ready` indicates that the receive buffer contains at least one character.
- `Status.overflow` is set and latched if a new frame was received while the receive buffer is full.
- `Status.error` is set and latched if any implementation-specific error condition occured.

`Status.overflow` and `Status.error` are initialized to 0 on reset.

The value of the `Status` register is not affected by `Control.enable`.

##### Divisor (read/write)

<img src="./0060-soc-uart-peripheral/reg-divisor.svg"
     alt="bf([
         {name: 'divisor', bits: 16, attr: 'RW'},
     ], {bits: 16})">

The `Divisor` register exposes an implementation-specific mechanism to control the receiver baudrate.

For example, assuming a clock frequency of 100MHz, an implementation may configure its baudrate like so:

```python3
baudrate = int(100e6 / divisor)
```

An implementation may also choose to ignore the `Divisor` register and configure the baudrate through unspecified means.

- If `Control.enable` is 0, `Divisor` is read/write.
- If `Control.enable` is 1, `Divisor` is read-only.

`Divisor` is initialized to `divisor_init` on reset.

##### Data (read-only)

<img src="./0060-soc-uart-peripheral/reg-rx-data.svg"
     alt="bf([
         {name: 'data', bits: 8, attr: 'R'},
     ], {bits: 8})">

- If `Control.enable` is 0 or `Status.ready` is 0:
  * reading from `Data` has no side-effect and returns an unspecified value.
- If `Control.enable` is 1 and `Status.ready` is 1:
  * reading from `Data` consumes one symbol from the receive buffer and returns it.

#### Transmitter

##### Control (read/write)

<img src="./0060-soc-uart-peripheral/reg-control.svg"
     alt="bf([
         { name: 'enable', bits: 1, attr: 'RW' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 1})">

`Control.enable` is initialized to 0 on reset.

- If `Control.enable` is 0, the transmitter PHY should be held in reset state.
- If `Control.enable` is 1, the transmitter PHY should operate normally.

##### Status (read-only)

<img src="./0060-soc-uart-peripheral/reg-tx-status.svg"
     alt="bf([
         { name: 'ready', bits: 1, attr: 'R' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 8})">

- `Status.ready` indicates that the transmit buffer has available space for at least one character.

##### Divisor (read/write)

<img src="./0060-soc-uart-peripheral/reg-divisor.svg"
     alt="bf([
         {name: 'divisor', bits: 16, attr: 'RW'},
     ], {bits: 16})">

The `Divisor` register exposes an implementation-specific mechanism to control the transmitter baudrate.

For example, assuming a clock frequency of 100MHz, an implementation may configure its baudrate like so:

```python3
baudrate = int(100e6 / divisor)
```

An implementation may also choose to ignore the `Divisor` register and configure the baudrate through unspecified means.

- If `Control.enable` is 0, `Divisor` is read/write.
- If `Control.enable` is 1, `Divisor` is read-only.

`Divisor` is initialized to `divisor_init` on reset.

##### Data (write-only)

<img src="./0060-soc-uart-peripheral/reg-tx-data.svg"
     alt="bf([
         {name: 'data', bits: 8, attr: 'W'},
     ], {bits: 8})">

- If `Status.rdy` is 1, writing to `Data` adds one character to the transmit buffer.

- If `Control.enable` is 0 or `Status.ready` is 0:
  * writing to `Data` has no side-effect.
- If `Control.enable` is 1 and `Status.ready` is 1:
  * writing to `Data` adds one symbol to the transmit buffer.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `amaranth_soc.uart.ReceiverPHYSignature`

The `uart.ReceiverPHYSignature` class is a `wiring.Signature` describing the interface between the UART peripheral and its receiver PHY, with:
- a `.__init__(self, *, symbol_width)` constructor, where `symbol_width` is a positive integer.

Its members are defined as follows:

```python3
{
    "rst":      Out(1),
    "divisor":  Out(unsigned(16)),
    "symbols":  In(wiring.Signature({
                    "payload": Out(unsigned(symbol_width)),
                    "valid":   Out(1),
                    "ready":   In(1),
                })),
    "overflow": In(1),
    "error":    In(1),
}
```

The `rst` port is driven to 1 if `Control.enable` is 0, and 0 if `Control.enable` is 1.

### `amaranth_soc.uart.TransmitterPHYSignature`

The `uart.TransmitterSignature` class is a `wiring.Signature` describing the interface between the UART peripheral and its transmitter PHY, with:
- a `.__init__(self, *, symbol_width)` constructor, where `symbol_width` is a positive integer.

Its members are defined as follows:

```python3
{
    "rst":     Out(1),
    "divisor": Out(unsigned(16)),
    "symbols": Out(wiring.Signature({
                   "payload": Out(unsigned(symbol_width)),
                   "valid":   Out(1),
                   "ready":   In(1),
               })),
}
```

The `rst` port is driven to 1 if `Control.enable` is 0, and 0 if `Control.enable` is 1.

### `amaranth_soc.uart.ReceiverPeripheral`

The `uart.ReceiverPeripheral` class is a `wiring.Component` implementing the receiver of an UART peripheral, with:
- a `.__init__(self, *, divisor_init, addr_width, data_width=8, name=None, symbol_width=8)` constructor, where:
  * `divisor_init` is a positive integer used as initial values for `Divisor.divisor`.
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`.
  * `symbol_width` is a positive integer passed to `Data` and `ReceiverPHYSignature`.
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "phy":     Out(ReceiverPHYSignature(symbol_width)),
}
```

### `amaranth_soc.uart.TransmitterPeripheral`

The `uart.TransmitterPeripheral` class is a `wiring.Component` implementing the transmitter of an UART peripheral, with:
- a `.__init__(self, *, divisor_init, addr_width, data_width=8, name=None, symbol_width=8)` constructor, where:
  * `divisor_init` is a positive integer used as initial values for `Divisor.divisor`.
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`.
  * `symbol_width` is a positive integer passed to `Data` and `TransmitterPHYSignature`.
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "phy":     Out(TransmitterPHYSignature(symbol_width)),
}
```

### `amaranth_soc.uart.Peripheral`

The `uart.Peripheral` class is a `wiring.Component` implementing an UART peripheral, with:
- a `.__init__(self, *, divisor_init, addr_width, data_width=8, name=None, symbol_width=8)` constructor, where:
  * `divisor_init` is a positive integer used as initial values for `Divisor.divisor`.
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`. `addr_width` must be at least 1. The peripheral address space is split in two, with the lower half occupied by a `ReceiverPeripheral` and the upper by a `TransmitterPeripheral`.
  * `symbol_width` is a positive integer passed to `ReceiverPeripheral`, `TransmitterPeripheral`, `ReceiverPHYSignature` and `TransmitterPHYSignature`.

- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "rx":      Out(ReceiverPHYSignature(symbol_width)),
    "tx":      Out(TransmitterPHYSignature(symbol_width)),
}
```

## Drawbacks
[drawbacks]: #drawbacks

- This design decouples the UART peripheral from its PHY, which must be provided by the user.
- The receiver and transmitter have separate `Divider` registers, despite using identical values
  in most cases.
- Configuring the baudrate through the `Divisor` register requires knowledge of the clock frequency used by the peripheral.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This design is intended to be minimal yet useful for the most common use-cases (i.e. 8-N-1).
- Decoupling the peripheral from the PHY allows flexibility in implementations. For example, it is easy to add FIFOs between the PHYs and the peripheral.
- A standalone `ReceiverPeripheral` or `TransmitterPeripheral` can be instantiated.

- The choice of a 16-bit divisor with an (otherwise) unspecified encoding allows implementation freedom:
  * some may not care about a clock divisor at all (e.g. a behavioral model of an UART PHY, interfacing with a pseudo-TTY).
  * some may provide their own divisor encoding scheme (e.g. a 13-bit base value with a 3-bit scale, that can cover common frequency/baudrate combinations with a [<1% error rate](https://github.com/amaranth-lang/rfcs/files/14672989/baud.py.txt) (credit: [@whitequark](https://github.com/whitequark))).


- As an alternative:
  * implement the PHY in the peripheral itself, and expose pin interfaces in a similar manner as the GPIO peripheral of [RFC 49](https://amaranth-lang.org/rfcs/0049-soc-gpio-peripheral.html).

## Prior art
[prior-art]: #prior-art

UART peripherals are commonly found in microcontrollers.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

## Future possibilities
[future-possibilities]: #future-possibilities

- Add a separate 16550-compatible UART peripheral.
- Add a separate peripheral with additional features, such as:
  * parity
  * auto baudrate
  * oversampling
  * hardware flow control
  * interrupts
  * DMA
- Add support for interrupts to this peripheral.
