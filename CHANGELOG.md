# Changelog

## 3.0.0

* Handles the new events:
    * Type=52 (0x34) - Block Data Chunk ACK.
    * Type=53 (0x35) - Block Data Chunk NACK.
    * Type=54 (0x36) - Bootloader CHECK.

## 2.0.0

  * [VSCP-framework repository](https://github.com/BlueAndi/vscp-framework) was restructured and the generic bootloader moved to this dedicated repository.
  * ```dataNum``` variable in the VSCP message (from VSCP-framework) was renamed to ```dataSize```. Therefore the source code was updated accordingly.
