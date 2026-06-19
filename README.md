# Small-Shell-for-the-TMS9900

![Platform](https://img.shields.io/badge/CPU-TMS9900-blue)
![OS](https://img.shields.io/badge/OS-CP/M--Like-green)
![Language](https://img.shields.io/badge/Language-Assembly-orange)
![Status](https://img.shields.io/badge/Status-Retro--Project-lightgrey)

------------------------------------------------------------------------

# TMS99105 SBC Shell — SHELLV56

## Overview

The shell is the command processor for the TMS99105 SBC. It runs in protected memory at `0xD000`, accepts commands at the `%` prompt, and manages program loading into the paged memory system.

---

## Memory Layout

```
0x0000-0x00AF  Interrupt vectors
0x00B0-0x012F  Interrupt workspace
0x0130-0x022F  XOP workspace
0x0230-0x024F  Shell workspace (R0-R15)
0x0250-0x027F  Command line buffer
0x0280-0x029B  FCB
0x02A0-0x02FF  Page variables / overlay manager
0x0300-0x04FF  LOADERCODE (runs from COMMON)
0x0500-0x06FF  STAGING buffer (512 bytes)
0x1000-0xBFFF  TPA — paged program area
0xD000         SHELL
0xE000         BDOS
0xF000         ROM / DISC_MONITOR
```

### Key Vectors (COMMON)

```
0x0080  BLWP @SHELL_PTR  — return to shell
0x0084  BLWP @BDOS_PTR   — BDOS entry
```

---

## Command Dispatch

1. Shell reads input into command line buffer (`0x0250`)
2. `PARSENAME` fills the FCB with the command name
3. `ICMD` scans `ICLIST` for a matching internal command
4. If not found, searches disk for `command.COM`
5. Loads and runs the program
6. On return via `BLWP @0x0080`, returns to `%` prompt

---

## Internal Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `DIR` | `%DIR` | List all files on disk |
| `SAVE` | `%SAVE n filename` | Save n sectors of memory to file |
| `ERA` | `%ERA filename` | Erase a file |
| `TYPE` | `%TYPE filename` | Display file (stub) |
| `LOAD` | `%LOAD filename.COM` | Load overlay into paged memory |

---

## Program Loading

### Flat Binary

A raw binary with no pagemap header. Loaded directly to `FLATBASE` (`0x1000`) in segment 1 page 0 and executed.

### Paged Binary

A `.COM` file with a pagemap header (sentinel `FFFF FFFF FFFF` at byte offset 6):

```
FFFF FFFF FFFF          opening sentinel
vstart vend  page       pagemap entry (virtual start, end, physical page)
FFFF FFFF FFFF          closing sentinel
size                    block size in bytes
[code bytes...]
```

The loader:
1. Parses pagemap entries
2. Programs the 6116 mapper via `MAP_SET` for each segment
3. Enables PSEL and copies code to the virtual address
4. Disables PSEL and launches the program

### LOAD Command (overlays)

`%LOAD filename.COM` loads a paged binary **without executing it**:
- Assigns the next overlay ID (`OVL_COUNT++`)
- Stores physical page in `OVL_PAGE_TAB[ID]`
- Stores virtual segment in `OVL_SEG_TAB[ID]`
- Prints `Loaded as overlay N`
- Returns to shell prompt

---

## Mapper Programming

### MAP_SET

Programs one 6116 mapper register via CRU:

```asm
; R9 = segment, R0 = physical page
MAP_SET:
    LI   R12,MAP_WIN_BASE    ; 0x80C0
    SLA  R9,1                ; segment * 2 = CRU offset
    A    R9,R12
    SLA  R0,8                ; page to HIGH BYTE (LDCR uses high byte)
    LDCR R0,BYTEWIDE         ; program register
    RT
```

### MAP_INIT

Clears all 16 mapper registers to page 0. Called at shell entry and before loading a new program.

### PSEL

PSEL is controlled via XOP 2. With PSEL enabled, SA0-SA3 from the 6116 select the mapped physical page. With PSEL disabled, SA0-SA3 are forced low — all segments map to page 0 (COMMON).

```asm
LI   R9,PSEL_EN    ; enable
PSEL R9

CLR  R9            ; disable
PSEL R9
```

---

## Overlay Manager Variables (0x02A0-0x02FF)

Populated by `%LOAD` and used by OVLMGR:

```
0x02A0  OVL_PRINT_VEC    Print callback address for overlays
0x02A2  OVL_COUNT        Number of overlays loaded
0x02A4  OVL_PAGE_TAB     16 words: physical page per overlay ID
0x02C4  OVL_SEG_TAB      16 words: virtual segment per overlay ID
0x02E4  OVL_CURR_ID      Currently mapped overlay ID
0x02E6  OVL_CURR_SEG     Currently mapped segment
```

---

## Interrupt Handling

Seven interrupt vectors at `0x00B0` are initialised at boot to `INT_SAFE` (`RTWP`) so unexpected interrupts are silently handled.

Before launching any program, `INT2_HANDLER` is installed at `0x000A`. On a privilege violation or illegal opcode (INT2 NMI), it dumps:

```
INT2 PC:xxxx WP:xxxx ST:xxxx
MAP:ssss ssss ... (all 16 mapper registers)
```

---

## Version History

| Version | Changes |
|---------|---------|
| 4.0 | Initial TMS9900 port of Small/Shell |
| 4.6 | Internal commands: DIR, SAVE, ERA |
| 5.4 | Paged memory loader; transparent GAL mapping |
| 5.5 | MAP_SET/MAP_INIT; multi-sector loading; INT2 debug handler |
| 5.6 | DOLOAD command; overlay manager variables; interrupt vector init |
