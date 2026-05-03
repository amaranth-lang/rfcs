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

        uart = uart.Peripheral(addr_width=8, data_width=8, symbol_shape=unsigned(8),
                               phy_config_shape=unsigned(16), phy_config_init=uart_divisor)

        m.submodules.uart = uart

        # Instantiate and connect the UART PHYs:

        uart_pins = platform.request("uart", 0)

        uart_phy_rx = AsyncSerialRX(uart_divisor, divisor_bits=16, pins=uart_pins)
        uart_phy_tx = AsyncSerialTX(uart_divisor, divisor_bits=16, pins=uart_pins)

        m.submodules.uart_phy_rx = ResetInserter(uart.rx.rst)(uart_phy_rx)
        m.submodules.uart_phy_tx = ResetInserter(uart.tx.rst)(uart_phy_tx)

        m.d.comb += [
            uart_phy_rx.divisor.eq(uart.rx.phy_config),

            uart.rx.symbols.payload.eq(uart_phy_rx.data),
            uart.rx.symbols.valid.eq(uart_phy_rx.rdy),
            uart_phy_rx.ack.eq(uart.rx.symbols.ready),

            uart.rx.overflow.eq(uart_phy_rx.err.overflow),
            uart.rx.error.eq(uart_phy_rx.err.frame),
        ]

        m.d.comb += [
            uart_phy_tx.divisor.eq(uart.tx.phy_config),

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

##### Config (read/write)

<img src="./0060-soc-uart-peripheral/reg-config.svg"
     alt="bf([
         { name: 'enable', bits: 1, attr: 'RW' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 8})">

`Config.enable` is initialized to 0 on reset.

- If `Config.enable` is 0, the receiver PHY should be held in reset state.
- If `Config.enable` is 1, the receiver PHY should operate normally.

##### PhyConfig (read/write)

<img src="./0060-soc-uart-peripheral/reg-phy_config.svg"
     alt="bf([
         {name: 'phy_config', bits: 16, attr: 'RW'},
     ], {bits: 16})">

The `PhyConfig` register exposes an implementation-specific mechanism to configure the receiver PHY, such as its baudrate. Its shape is given by the `phy_config_shape` parameter (`unsigned(16)` in the above example).

An implementation may choose to not use the `PhyConfig` register and configure its PHY through unspecified means.

- If `Config.enable` is 0, `PhyConfig` is read/write.
- If `Config.enable` is 1, `PhyConfig` is read-only.

`PhyConfig` is initialized to `phy_config_init` on reset.

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

`Status.overflow` and `Status.error` are not reset when writing 0 to `Config.enable`.

##### Data (read-only)

<img src="./0060-soc-uart-peripheral/reg-rx-data.svg"
     alt="bf([
         {name: 'data', bits: 8, attr: 'R'},
     ], {bits: 8})">

The `Data` register can be read to consume symbols from the receive buffer. Its shape is given by the `symbol_shape` parameter (`unsigned(8)` in the above example).

- If `Config.enable` is 0 or `Status.ready` is 0:
  * reading from `Data` has no side-effect and returns an unspecified value.
- If `Config.enable` is 1 and `Status.ready` is 1:
  * reading from `Data` consumes one symbol from the receive buffer and returns it.

#### Transmitter

##### Config (read/write)

<img src="./0060-soc-uart-peripheral/reg-config.svg"
     alt="bf([
         { name: 'enable', bits: 1, attr: 'RW' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 1})">

`Config.enable` is initialized to 0 on reset.

- If `Config.enable` is 0, the transmitter PHY should be held in reset state.
- If `Config.enable` is 1, the transmitter PHY should operate normally.

##### PhyConfig (read/write)

The `PhyConfig` register exposes an implementation-specific mechanism to configure the transmitter PHY, such as its baudrate. Its shape is given by the `phy_config_shape` parameter (`unsigned(16)` in the above example).

An implementation may choose to not use the `PhyConfig` register and configure its PHY through unspecified means.

- If `Config.enable` is 0, `PhyConfig` is read/write.
- If `Config.enable` is 1, `PhyConfig` is read-only.

`PhyConfig` is initialized to `phy_config_init` on reset.

##### Status (read-only)

<img src="./0060-soc-uart-peripheral/reg-tx-status.svg"
     alt="bf([
         { name: 'ready', bits: 1, attr: 'R' },
         { bits: 7, attr: 'ResR0W0' },
     ], {bits: 8})">

- `Status.ready` indicates that the transmit buffer has available space for at least one character.

##### Data (write-only)

<img src="./0060-soc-uart-peripheral/reg-tx-data.svg"
     alt="bf([
         {name: 'data', bits: 8, attr: 'W'},
     ], {bits: 8})">

The `Data` register can be written to append symbols to the transmit buffer. Its shape is given by the `symbol_shape` parameter (`unsigned(8)` in the above example).

- If `Config.enable` is 0 or `Status.ready` is 0:
  * writing to `Data` has no side-effect.
- If `Config.enable` is 1 and `Status.ready` is 1:
  * writing to `Data` adds one symbol to the transmit buffer.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `amaranth_soc.uart.RxPhySignature`

The `uart.RxPhySignature` class is a `wiring.Signature` describing the interface between the UART peripheral and its receiver PHY, with:
- a `.__init__(self, phy_config_shape, symbol_shape)` constructor, where `phy_config_shape` and `symbol_shape` are shape-like objects.

Its members are defined as follows:

```python3
{
    "rst":      Out(1),
    "config":   Out(phy_config_shape),
    "symbols":  In(stream.Signature(symbol_shape)),
    "overflow": In(1),
    "error":    In(1),
}
```

- The `rst` port is driven to 1 if `Config.enable` is 0, and 0 if `Config.enable` is 1.
- The `config` remains constant if `rst` is 0.
- The `overflow` port is pulsed for one clock cycle if a symbol was received before the previous one is acknowledged (i.e. before `symbols.ready` is high).
- The `error` port is pulsed for one clock cycle in case of an unspecified error, specific to the PHY implementation.

### `amaranth_soc.uart.TxPhySignature`

The `uart.TxPhySignature` class is a `wiring.Signature` describing the interface between the UART peripheral and its transmitter PHY, with:
- a `.__init__(self, phy_config_shape, symbol_shape)` constructor, where `phy_config_shape` and `symbol_shape` are shape-like objects.

Its members are defined as follows:

```python3
{
    "rst":     Out(1),
    "config":  Out(phy_config_shape),
    "symbols": Out(stream.Signature(symbol_shape)),
}
```

- The `rst` port is driven to 1 if `Config.enable` is 0, and 0 if `Config.enable` is 1.
- The `config` remains constant if `rst` is 0.

### `amaranth_soc.uart.RxPeripheral`

The `uart.RxPeripheral` class is a `wiring.Component` implementing the receiver of an UART peripheral, with:
- a `.__init__(self, *, addr_width, data_width=8, name=None, phy_config_shape=unsigned(16), phy_config_init=0, symbol_shape=unsigned(8))` constructor, where:
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`.
  * `phy_config_shape` is the shape of the single-field `PhyConfig` register.
  * `phy_config_init` is the initial value of the single-field `PhyConfig` register.
  * `symbol_shape` is the shape of the single-field `Data` register.
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "phy":     Out(RxPhySignature(phy_config_shape, symbol_shape)),
}
```

### `amaranth_soc.uart.TxPeripheral`

The `uart.TxPeripheral` class is a `wiring.Component` implementing the transmitter of an UART peripheral, with:
- a `.__init__(self, *, addr_width, data_width=8, name=None, phy_config_shape=unsigned(16), phy_config_init=0, symbol_shape=unsigned(8))` constructor, where:
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`.
  * `phy_config_shape` is the shape of the single-field `PhyConfig` register.
  * `phy_config_init` is the initial value of the single-field `PhyConfig` register.
  * `symbol_shape` is the shape of the single-field `Data` register.
- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "phy":     Out(TxPhySignature(phy_config_shape, symbol_shape)),
}
```

### `amaranth_soc.uart.Peripheral`

The `uart.Peripheral` class is a `wiring.Component` implementing an UART peripheral, with:
- a `.__init__(self, *, addr_width, data_width=8, name=None, phy_config_shape=unsigned(16), phy_config_init=0, symbol_shape=unsigned(8))` constructor, where:
  * `addr_width`, `data_width` and `name` are passed to a `csr.Builder`. `addr_width` must be at least 1. The peripheral address space is split in two, with the lower half occupied by a `RxPeripheral` and the upper by a `TxPeripheral`.
  * `phy_config_shape` and `phy_config_init` are the shape and initial value of the `PhyConfig` registers of `RxPeripheral` and `TxPeripheral`.
  * `symbol_shape` is the shape of the `Data` registers of `RxPeripheral` and `TxPeripheral`.

- a `.signature` property, that returns a `wiring.Signature` with the following members:

```python3
{
    "csr_bus": In(csr.Signature(addr_width, data_width)),
    "rx":      Out(RxPhySignature(phy_config_shape, symbol_shape)),
    "tx":      Out(TxPhySignature(phy_config_shape, symbol_shape)),
}
```

## Drawbacks
[drawbacks]: #drawbacks

- This design decouples the UART peripheral from its PHY, which must be provided by the user.
- An `uart.Peripheral` has separate `PhyConfig` registers for its receiver and transmitter, despite using common values in most cases due to their symmetry.
- A `PhyConfig` register has a single field whose shape is user-provided, even though it may contain multiple values. Amaranth SoC will have to take this into account when generating a BSP.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This design is intended to be minimal yet useful for the most common use-cases (i.e. 8-N-1).
- Decoupling the peripheral from the PHY allows flexibility in implementations. For example, it is easy to add FIFOs between the PHYs and the peripheral.
- A standalone `RxPeripheral` or `TxPeripheral` can be instantiated.
- The choice of a parameterized shape for the `PhyConfig` register facilitates interoperability with PHY implementations. Some may not rely on this register and configure themselves through alternate means (or not at all).

- As an alternative:
  * implement the PHY in the peripheral itself, and expose pin interfaces in a similar manner as the GPIO peripheral of [RFC 49](https://amaranth-lang.org/rfcs/0049-soc-gpio-peripheral.html).
  * do not allow `PhyConfig` to be parameterized and provide its layout ourselves.

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
