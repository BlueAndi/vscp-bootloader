# VSCP L1 Bootloader

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](http://choosealicense.com/licenses/mit/)
[![Repo Status](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![Release](https://img.shields.io/github/release/BlueAndi/vscp-bootloader.svg)](https://github.com/BlueAndi/vscp-bootloader/releases)

A generic VSCP L1 bootloader, which supports the [standard bootloader algorithm](#vscp-standard-boot-loader-algorithm). It provides the possibility to program a new firmware image to a VSCP node.

* [VSCP L1 Bootloader](#vscp-l1-bootloader)
* [VSCP](#vscp)
* [VSCP Standard Bootloader Algorithm](#vscp-standard-bootloader-algorithm)
  * [Description](#description)
  * [Facts](#facts)
* [FAQ](#faq)
  * [How to integrate it?](#how-to-integrate-it)
  * [How can a application be programmed via bootloader?](#how-can-a-application-be-programmed-via-bootloader)
  * [Why does the node sends a new node online event with 0xFE?](#why-does-the-node-sends-a-new-node-online-event-with-0xfe)
* [Issues, Ideas And Bugs](#issues-ideas-and-bugs)
* [License](#license)
* [Contribution](#contribution)

# VSCP

<img src="https://github.com/grodansparadis/vscp_logo/raw/master/vscp_logo.jpg" width="200px" alt="Logo" />

The Very Simple Control Protocol (VSCP), an open and free protocol for IoT/m2m automation tasks.

More information can be found on the main site http://www.vscp.org

# VSCP Standard Bootloader Algorithm

![VSCP standard bootloader algorithm](https://github.com/BlueAndi/vscp-bootloader/blob/master/doc/vscp_std_bootloader_algorithm.png)

[More detail](https://github.com/BlueAndi/vscp-bootloader/blob/master/doc/vscp_std_bootloader_algorithm_detail.png)

## Description

The bootloader algorithm follows the standard VSCP bootloader algorithm. The following
description contains more details about how it proceeds.

1. Determine whether
    a) the user force bootloader mode,
    b) an valid application is present and shall be started,
    c) the application jumped to the bootloader,
    d) no application is present.

2. Conditions for case a):
    - If the segment initialization button is pressed during power-up, stay in the bootloader.
    Go to step 6.

3. Conditions for case b):
    - The segment initialization button is released.
    - The boot flag in the persistent memory shall contain 0xAA for destination application.
    If all conditions met, jump to the application otherwise go to step 6.

4. Conditions for case c):
    - The hardware jumper is not set.
    - The boot flag in the persistent memory shall contain 0xBB for destination bootloader.
    If all conditions met, go to step 9.

5. Conditions for case d):
    - The segment initialization button is released.
    - The boot flag in the persistent memory shall contain anything else than 0xAA and 0xBB.
    If all conditions met, go to step 6.

6. Send a "new node online/probe" event, so that the programming tool knows the node is available now.
    Node id 0xFE is used!

7. If any "ACK probe" event is received, the node shall go to sleep mode, because the
    nickname is already used.

8. Wait (infinite) for the "enter bootloader" event. Only if the received nickname
    and GUID are equal to the internal ones of the node, validate the proposed programming
    algorithm.
    If the algorithm is supported, continue with next step, otherwise a
    "NACK enter bootloader" event will be sent.

9. Because the application already received the "enter bootloader" event and validated its
    parameters, a "ACK enter bootloader" event shall be sent back now in the bootloader.
    Send the flash block size (= flash page size) and the number of blocks as parameters.

10. Wait (infinite) for the "start block data transfer" event. If the proposed memory type
    is correct acknowledge it via "ACK start block data transfer" event otherwise via
    "NACK start block data transfer" event.

11. Now several "block data" events will be received and acknowledged with "ACK block chunk" events,
    until the complete block is transfered.
    Every received block data shall be written to RAM buffer. After the last data is transfered,
    a CRC16-CCITT shall be calculated over the whole block. And an "ACK block data" event shall
    be sent with the calculated CRC as parameter.

12. If a complete block is received, wait for a "program block" event. This is the
    trigger to delete a flash page and write the block RAM buffer to the flash page.
    If the block is successful written, send an "ACK program block" event to confirm it.
    Otherwise send an "NAK program block" event.
    Note that before writing the first block to flash, write 0xFF to boot flag in the persistent memory. 
    This shall be done here, because after this, there is no way back.

13. The next block shall be handled (go to step 10), except an "drop nickname/reset device" event
    is received. In this case the block programming operation shall be stopped.

14. Validate the application in the flash.
    If the application is valid, a "ACK activate new image" event shall be sent, the boot flag
    in the persistent memory shall be set to 0xAA and reboot the node.
    Otherwise "NAK activate new image" event shall be sent and the node shall go to sleep.

Notes:
* Unused bytes in a flash block shall be 0xFF.
* The host shall send always an image in the size of the whole application section.
* Because the AVR has flash pages, but the VSCP has blocks, both are considered as equal.
* If the programming procedure is aborted by "drop nickname/reset device" event
  the node reboots.

Error codes:
* 0 - Programming algorithm not supported
* 1 - Memory type not supported
* 2 - Bad block number
* 3 - Invalid message

## Facts

* The bootloader supports only programming the program space right now.
* The bootloader can only handle complete blocks. If the last block is not completly received from a programmer, the bootloader will infinite wait for the rest!
  In other words, the size of the new application shall be always a multiple of the block size.

# FAQ

## How to integrate it?

Include the following files to your project:
* /src/crc16ccitt.h
* /src/crc16ccitt.c
* /src/vscp\_bootloader.h
* /src/vscp\_bootloader.c

Additional include the following files to your project from the [vscp-framework](https://github.com/BlueAndi/vscp-framework):
* /src/vscp\_class\_l1.h
* /src/vscp\_types.h
* /src/vscp\_type_protocol.h

Copy the templates to your project and adapt them to your needs:
* /templates/vscp\_bl\_adapter.h
* /templates/vscp\_bl\_adapter.c
* /templates/vscp\_platform.h
* /templates/vscp\_tp\adapter.h
* /templates/vscp\_tp\adapter.c

Call in your main routine the
* vscp\_bootloader\_init() and
* vscp\_bootloader\_run()

Note that the vscp\_bootloader\_run() will never return!

## How can a application be programmed via bootloader?

* For the command line interface: [VSCP L1 programmer](https://github.com/BlueAndi/vscp-cli-tools).
* With a graphical user interface: [vscp-works-qt](https://github.com/grodansparadis/vscp-works-qt).

## Why does the node sends a new node online event with 0xFE?

If the bootloader starts, it try to figure out whether a valid application is programmed or not.
In this case it seems there is no valid application and therefore most probably an invalid persistent memory.
Now the bootloader simulates a new node with nickname 0xFE, so that you are able to program a new application via programmer.

If you think the application should be valid, because you programmed it just a minute ago via debugger ... can it be that
you forgot to program a valid persistent memory? ;-)

# Issues, Ideas And Bugs

If you have further ideas or you found some bugs, great! Create a [issue](https://github.com/BlueAndi/vscp-bootloader/issues) or if you are able and willing to fix it by yourself, clone the repository and create a pull request.

# License
The whole source code is published under the [MIT license](http://choosealicense.com/licenses/mit/).

# Contribution
Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, shall be licensed as above, without any
additional terms or conditions.
