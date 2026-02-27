# Small-Shell-for-the-TMS9900

![Platform](https://img.shields.io/badge/CPU-TMS9900-blue)
![OS](https://img.shields.io/badge/OS-CP/M--Like-green)
![Language](https://img.shields.io/badge/Language-Assembly-orange)
![Status](https://img.shields.io/badge/Status-Retro--Project-lightgrey)

------------------------------------------------------------------------

## Table of Contents

-   [Overview](#overview)
-   [Historical Background](#historical-background)
-   [TMS9900 Version](#tms9900-version)
-   [Shell Prompt](#shell-prompt)
-   [Commands](#commands)
    -   [%DIR](#dir)
    -   [%ERA](#era-filename)
    -   [%SAVE](#save-n-filename--hexloadaddress)
-   [Executing Programs](#executing-programs)
-   [Load Without Executing](#load-without-executing)
-   [Running in Alternate Memory
    Segments](#running-in-alternate-memory-segments)

------------------------------------------------------------------------

## Overview

**Small-Shell-for-the-TMS9900** is a compact command processor for DOS
on a TMS9900 CP/M-like system.

It provides a simple and efficient user interface for interacting with
the operating system and managing disk-based programs.

------------------------------------------------------------------------

## Historical Background

This project is based on James Hendrix's 8080 Small VM/Shell, originally
written to complement North Star DOS.

It was featured in a 1982 article in Volume 7, Number 63 of *Dr. Dobb's
Journal*.\
The original article provides a full description of the procedural
command structure.

Archived article:
https://ia600109.us.archive.org/17/items/dr_dobbs_journal_vol_07_201803/dr_dobbs_journal_vol_07.pdf

------------------------------------------------------------------------

## TMS9900 Version

This version:

-   Preserves the original structure and design philosophy
-   Has been rewritten for a **TMS9900 CP/M-like system**
-   Adds CP/M-style disk commands:
    -   `DIR`
    -   `SAVE`
    -   `ERA`

------------------------------------------------------------------------

## Shell Prompt

The Shell prompt character is:

    %

------------------------------------------------------------------------

## Commands

### `%DIR`

Lists the disk directory.

Example:

    %DIR

------------------------------------------------------------------------

### `%ERA FileName`

Erases a file from the disk directory.

Example:

    %ERA TEST.TXT

------------------------------------------------------------------------

### `%SAVE N FileName -HexLoadAddress`

Saves memory contents (typically an executable) to disk.

-   `N` = number of 512-byte blocks
-   Total bytes written = `N × 512`
-   `HexLoadAddress` = starting address (hexadecimal)

Example:

    %SAVE 6 XMODEM -0100

This writes 3 KB (6 × 512 bytes) starting at address `0100H` to disk as
`XMODEM`.

------------------------------------------------------------------------

## Executing Programs

After saving, execute a program by typing its name:

    %XMODEM FileName

This example runs `XMODEM`, receives a file, and saves it to disk as
`FileName`.

------------------------------------------------------------------------

## Load Without Executing

To load a program into memory without executing it, prefix the filename
with a period:

    %.XMODEM

This loads the file into memory and returns to the Shell.\
Useful for inspection or patching.

------------------------------------------------------------------------

## Running in Alternate Memory Segments

By default, programs execute in segment 0.

To run in another segment (e.g., segment 1):

    FileName -1

When the Shell detects `-1`, it:

1.  Creates a loader in common memory
2.  Moves the object file into the specified segment
3.  Transfers control for execution

------------------------------------------------------------------------


