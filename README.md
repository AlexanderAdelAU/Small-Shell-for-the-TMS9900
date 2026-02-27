# Small-Shell-for-the-TMS9900

A small **Shell program** acting as a command processor for the DOS.\
Its purpose is to provide a user interface that allows interaction with
the operating system.100

This project is based on James Hendrix's 8080 Small VM/Shell, originally
written to complement North Star DOS and featured in a 1982 article in
Volume 7, Number 63 of Dr. Dobb's Journal.

The original article (which provides a full description of the
procedural commands) can be found here:

https://ia600109.us.archive.org/17/items/dr_dobbs_journal_vol_07_201803/dr_dobbs_journal_vol_07.pdf

------------------------------------------------------------------------

## TMS9900 Version

This version retains the original structure but has been rewritten to
run on a **TMS9900 CP/M-like system**.

In addition to the original functionality, this Shell incorporates
additional CP/M-style commands such as:

-   `DIR`
-   `SAVE`
-   `ERA`

The Shell prompt is:

    %

------------------------------------------------------------------------

## Commands

### `%DIR`

Lists the disk directory.

------------------------------------------------------------------------

### `%ERA FileName`

Erases a file from the disk directory.

**Example:**

    %ERA TEST.TXT

Erases the file `TEST.TXT` from the disk.

------------------------------------------------------------------------

### `%SAVE N FileName -HexLoadAddress`

Saves the contents of memory (normally an executable) starting at
`<HexLoadAddress>` to disk.

-   `N` specifies how many blocks to write.
-   Each block = 512 bytes.
-   Total bytes written = `N × 512`.

**Example:**

    %SAVE 6 XMODEM -0100

Saves 3 KB (6 × 512 bytes) starting at address `0500H` to disk as
`XMODEM`.

------------------------------------------------------------------------

## Executing Programs

Once saved, a program can be executed by typing its name.

**Example:**

    %XMODEM FileName

Executes `XMODEM`, which receives a file and saves it to disk as
`FileName`.

------------------------------------------------------------------------

### Load Only (Do Not Execute)

Placing a full stop (`.`) in front of the filename loads the file into
memory and returns to the Shell:

    %.XMODEM

This is useful for inspecting or patching a disk file without executing
it.

------------------------------------------------------------------------

## Running in a Memory Segment Other Than 0

To run a program in a different memory segment (for example, segment 1),
specify the segment number using the `-` switch:

    FileName -1

When the Shell detects `-1`, it creates a loader in common memory.\
This loader moves the object file into the specified segment before
execution.
