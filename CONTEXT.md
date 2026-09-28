# UV-K1 / UV-K5 V3 Custom Firmware

Custom open-source firmware for Quansheng UV-K1 and UV-K5 V3 handheld radios based on the Puya PY32F071 MCU.

## Memory & Architecture

**Internal Flash**:
The on-chip 128 KiB flash memory of the PY32F071 MCU, partitioned into factory bootloader (10 KiB) and internal application area (118 KiB / 120,832 bytes maximum).
_Avoid_: ROM, external flash

**External Flash**:
The 2 MiB SPI NOR flash chip (PY25Q16) storing configuration banks, calibration data, boot logo, multiboot firmware slots, and overlay apps.
_Avoid_: EEPROM (when referring to physical storage), internal flash

**Preset Max**:
The top-tier firmware build preset enabling all resident radio, tools, and UI features simultaneously within the internal flash footprint.
_Avoid_: All-in-one, Full build

**Resident Feature**:
A capability compiled directly into the internal application binary that executes from internal flash.
_Avoid_: Native app, built-in plugin

**Overlay App**:
A standalone application binary stored in external flash and executed inside a dedicated 4 KiB internal RAM workspace without modifying internal flash.
_Avoid_: Dynamic library, plugin

**Multiboot**:
The firmware subsystem capable of reprogramming the internal flash at runtime from slot images stored in external flash via a RAM-resident stub.
_Avoid_: Dual boot, bootloader update
