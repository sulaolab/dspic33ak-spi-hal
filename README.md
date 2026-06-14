# dspic33ak-spi-hal

Small, readable blocking SPI master HAL for Microchip dsPIC33AK devices.

This project is intended as a compact alternative to large generated driver code.
The goal is not to hide everything behind a framework, but to provide a simple
driver that is easy to read, test, modify, and adapt.

## Status

Current validation target:

* Device: dsPIC33AK512MPS512
* Compiler: XC-DSC
* DFP: Microchip dsPIC33AK-MP DFP 1.3.185 or compatible
* Tested SPI instance: SPI4
* Tested peripheral: SST26 external SPI NOR flash
* Mode / clock: SPI mode 0, 8-bit, 12.5 MHz SCK (PBCLK 100 MHz, BRG = 3)

Other SPI instances (SPI1..SPI3) are supported by the same code; the register
table adapts automatically to whichever instances the device header defines.
Hardware validation has been performed on SPI4.

Confirmed operations on the validation target:

* Blocking single-byte full-duplex transfer
* Blocking buffer transfer / write / read
* Shift-register drain before chip-select de-assert
* SST26 JEDEC ID read, status/config read
* SST26 chip erase, page program (~100 KB), and full read-back verify
* Extended (`_ex`) APIs returning explicit results with a polling timeout
* Status snapshot, last-result query, error clear

## Design policy

This driver is intentionally small.

* Normal API is blocking and simple.
* Extended (`_ex`) API adds explicit result codes and a polling timeout.
* No XC-DSC / DFP bitfield structures are exposed in the public API.
* Device-specific register symbols are isolated in a small pointer table.
* The SPI register table is built from `#if defined(SPInCON1)` presence tests,
  so it adapts to the device without device-name conditionals.
* This HAL owns only the SPI module registers. Chip-select, WP/RESET, PPS
  routing, and pin/GPIO setup belong to the board / device layer.
* This HAL is CMSIS-agnostic: no `ARM_DRIVER_*` / `ARM_SPI_*` symbols and no
  CMSIS-Driver headers. A CMSIS-Driver wrapper can map the result/status types
  onto the CMSIS API in a separate layer.

## Scope

In scope:

* Master mode, 8-bit, blocking polling
* SPI mode 0/1/2/3
* Baud rate (BRG) configuration
* SPI instance selection (SPI1 / SPI2 / SPI3 / SPI4 as present on the device)

Out of scope (not handled here):

* DMA / interrupt-driven transfer
* TDM frame / audio streaming (kept independent from any audio SPI driver)
* 16/24/32-bit transfer
* Multi-client bus arbitration / RTOS locking
* Chip-select / WP / RESET lines and PPS / GPIO pin setup (board/device layer)

## Files

```text
src/
  dspic33ak_spi.c
  dspic33ak_spi.h
  dspic33ak_spi_reg.h
```

`dspic33ak_spi_reg.h` is the only place that references the raw SPI SFRs
(`SPIxCON1` / `SPIxSTAT` / `SPIxBRG` / `SPIxBUF`) as bit masks; the driver body
drives any instance through a pointer table.

## Basic usage

```c
#include "dspic33ak_spi.h"

static dspic33ak_spi_handle_t s_spi;

void app_spi_init(void)
{
    const dspic33ak_spi_config_t cfg = {
        .instance          = DSPIC33AK_SPI_INST_4,
        .peripheralClockHz = 100000000u,   /* SPI peripheral clock (Hz) */
        .targetSckHz       = 12500000u,    /* desired SCK (Hz)          */
        .mode              = DSPIC33AK_SPI_MODE_0,
    };

    /* NOTE: PPS routing and the chip-select / WP / RESET GPIO must be set up by
     * the board/device layer before/around the bus is used. This HAL configures
     * only the SPI module registers. */
    (void)dspic33ak_spi_init(&s_spi, &cfg);
}
```

A simple device transaction (caller controls chip-select):

```c
my_cs_assert();
(void)dspic33ak_spi_transfer8(&s_spi, 0x9F);     /* command */
uint8_t b0 = dspic33ak_spi_transfer8(&s_spi, 0x00);
uint8_t b1 = dspic33ak_spi_transfer8(&s_spi, 0x00);
dspic33ak_spi_wait_done(&s_spi);                 /* shifter idle before CS high */
my_cs_deassert();
```

## Simple vs extended API

Simple blocking API (wait forever, `bool` / sentinel returns):

* `dspic33ak_spi_transfer8()`
* `dspic33ak_spi_transfer()`  — `tx` and/or `rx` may be NULL
* `dspic33ak_spi_write()`     — transmit, discard RX
* `dspic33ak_spi_read()`      — clock out a dummy byte, capture RX
* `dspic33ak_spi_wait_done()` — wait shifter idle, drain a pending overflow

Extended API (`_ex`): same operations, but they return `dspic33ak_spi_result_t`
and take a polling `timeoutCount` (`0` == wait forever, matching the simple API):

* `dspic33ak_spi_transfer8_ex()`
* `dspic33ak_spi_transfer_ex()`
* `dspic33ak_spi_write_ex()`   — `len > 0` with NULL `tx` returns `INVALID_ARG`
* `dspic33ak_spi_read_ex()`    — `len > 0` with NULL `rx` returns `INVALID_ARG`
* `dspic33ak_spi_wait_done_ex()`

The simple APIs are thin wrappers over the `_ex` APIs with `timeoutCount == 0`,
so both paths share one implementation.

State queries:

* `dspic33ak_spi_get_status()`      — read-only snapshot (no hardware access)
* `dspic33ak_spi_get_last_result()`
* `dspic33ak_spi_clear_error()`     — clear the sticky overflow flag and last result

## Result codes

```c
DSPIC33AK_SPI_RESULT_OK
DSPIC33AK_SPI_RESULT_ERROR
DSPIC33AK_SPI_RESULT_INVALID_ARG
DSPIC33AK_SPI_RESULT_NOT_INITIALIZED
DSPIC33AK_SPI_RESULT_UNSUPPORTED
DSPIC33AK_SPI_RESULT_BUSY
DSPIC33AK_SPI_RESULT_TIMEOUT
DSPIC33AK_SPI_RESULT_OVERFLOW
```

`dspic33ak_spi_init()` returns `false` (and records the result in the handle) on
a NULL config, an instance not present on the device, the SPI1 instance (treated
as reserved — see Notes), an out-of-range mode, or a zero clock value.

## Instance selection and SPI1

`dspic33ak_spi_init()` refuses `DSPIC33AK_SPI_INST_1` and returns
`DSPIC33AK_SPI_RESULT_UNSUPPORTED`. On the reference board SPI1 (and optionally
SPI2) is reserved for an audio/TDM transport, so the generic HAL never
re-initializes it. Adjust this guard for a board where SPI1 is free.

## Timeout cleanup

For the `_ex` APIs, `timeoutCount` is a bounded count of status-polling
iterations (`0` == wait forever). If the RX-ready wait times out after the byte
was written to `SPIxBUF`, the driver drains a late RX byte and clears a pending
overflow so the next transaction does not read stale data. The returned result
stays `TIMEOUT`; any overflow seen during cleanup is recorded in the handle.

## Device mapping

The SPI register table in `dspic33ak_spi.c` is the only place that references
`SPI1CON1` / `SPI2CON1` / `SPI3CON1` / `SPI4CON1`. Each row is emitted only when
the device header defines that instance's SFRs, so the table tracks the silicon
automatically and the driver body stays device-neutral.

## Notes

* `BRG = peripheralClockHz / (2 * targetSckHz) - 1`, clamped to the 13-bit field.
  The achieved SCK is reported back in `handle.actualSckHz`.
* SPI mode maps to `CKP = CPOL` and `CKE = !CPHA` on dsPIC33AK; the HAL does the
  mapping.
* `dspic33ak_spi_deinit()` clears the module ON bit (if initialized) and resets
  the handle state.
* This HAL is the SPI *bus* layer only; chip-select, WP/RESET, and PPS/pin setup
  are intentionally left to the board / device driver.
* This repository does not include Microchip DFP header files.

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
