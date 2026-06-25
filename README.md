# Small-Shell-for-the-TMS9900

![Platform](https://img.shields.io/badge/CPU-TMS9900-blue)
![OS](https://img.shields.io/badge/OS-CP/M--Like-green)
![Language](https://img.shields.io/badge/Language-Assembly-orange)
![Status](https://img.shields.io/badge/Status-Retro--Project-lightgrey)

---

# TMS99105 SBC Shell — SHELLV57

## Overview

The shell is the command processor for the TMS99105 SBC. It runs in protected memory at `0xD000`, accepts commands at the `%` prompt, and manages program loading into the paged memory system.

Version 5.7 adds full `.EXE` paged chain format support alongside the existing `.COM` flat binary loader. The overlay manager (`OVLMGR`) is now a permanent OS service resident in common memory at `0x0A00`, booted by the shell at startup.

---

## Memory Layout

```
0x0000–0x003F  Interrupt vectors
0x0040–0x007F  XOP vectors
0x0080–0x00AF  OS vectors (SHELL_PTR, BDOS_PTR, stack ptr)
0x00B0–0x012F  Interrupt workspace (8 × 16)
0x0100–0x011F  BDOS XOP handler (permanent resident)
0x0130–0x022F  XOP workspace (16 × 16)
0x0230–0x024F  Shell workspace (R0–R15)
0x0250–0x027F  Command line buffer
0x0280–0x02BF  Common FCB
0x02A0–0x02FF  Page variables / overlay manager tables
0x0300–0x04FF  Loader area (COM or EXE loader, swapped per launch)
0x0500–0x08FF  EXE staging buffer (2 × 512 bytes)
0x0900–0x09FF  Free
0x0A00–0x0A5F  OVLMGR (permanent resident, booted by shell)
0x0A60–0x0FFF  Free (stack grows down from 0x1000)
0x1000–0xCFFF  TPA — paged program area / free memory
0xD000         SHELL
0xE400         BDOS
0xF000         DISC_MONITOR
```

### Key Vectors

```
0x0080  BLWP @SHELL_PTR  — return to shell
0x0084  BLWP @BDOS_PTR   — BDOS entry
```

---

## Page Variables (0x02A0–0x02FF)

Overlay manager vectors and tables, populated at shell startup and by `LOADERCODE_EXE`:

```
0x02A0  OVL_PRINT_VEC    Application print callback
0x02A2  OVL_INIT_VEC     Address of OVLMGR_INIT (at 0x0A00)
0x02A4  OVL_SWAP_VEC     Address of OVLMGR     (at 0x0A04)
0x02A6  OVL_COUNT        Number of overlays registered by loader
0x02A8  OVL_CURR_ID      Currently mapped overlay ID (0 = none)
0x02AA  OVL_CURR_SEG     Currently mapped segment
0x02AC  OVL_PAGE_TAB     16 words — physical page per OVLID 1–16
0x02CC  OVL_SEG_TAB      16 words — virtual segment per OVLID 1–16
```

`OVL_INIT_VEC` and `OVL_SWAP_VEC` hold the actual addresses of the OVLMGR routines. Applications call them indirectly:

```asm
MOV  @OVL_INIT_VEC,R0
BL   *R0                 ; call OVLMGR_INIT

LI   R1,OVLID_1
MOV  @OVL_SWAP_VEC,R0
BL   *R0                 ; call OVLMGR with R1 = overlay ID
```

---

## Command Dispatch

1. Shell reads input into command line buffer (`0x0250`)
2. `PARSENAME` fills the FCB with the command name and extension
3. `ICMD` scans `ICLIST` for a matching internal command
4. If not found, `LOOK` searches disk trying extensions in order: `COM`, `EXE`, `BAS`, `BAT`
5. `SET_FTY` derives the file type from the FCB extension
6. `LOADRUN` dispatches to `GOPROG` (COM) or `GOPROG_EXE` (EXE)
7. On return via `BLWP @0x0080`, returns to `%` prompt

### File Type Detection

`SET_FTY` walks `EXT_TAB` comparing the FCB extension against known types:

| Type code | Extension | Loader |
|-----------|-----------|--------|
| 1 | `.COM` | `GOPROG` → `LOADERCODE` |
| 2 | `.EXE` | `GOPROG_EXE` → `LOADERCODE_EXE` |
| 3 | `.BAS` | — |
| 4 | `.BAT` | — |

---

## Internal Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `DIR` | `%DIR` | List all files on disk |
| `SAVE` | `%SAVE n filename` | Save n sectors of memory to file |
| `ERA` | `%ERA filename` | Erase a file |
| `TYPE` | `%TYPE filename` | Display file contents |
| `LOAD` | `%LOAD filename` | Load overlay into paged memory |

---

## Program Loading

### COM — Flat Binary

A raw binary with no header. Loaded directly to TPA (`0x1000`) in segment 0 and executed. `LOADERCODE` is copied to `0x0300` by `GOPROG` before each launch.

### EXE — Paged Chain Format

Produced by `link99` with `-P#` page tags. `LOADERCODE_EXE` is copied to `0x0300` by `GOPROG_EXE` before each launch.

Each block in the chain:

```
next_offset   word    byte distance to next block (0 = last)
page          word    physical page (0 = common/flat)
start         word    virtual load address
size          word    data byte count
[data bytes]
```

Load sequence:
1. Two sectors read into staging (`0x0500`–`0x08FF`) upfront — avoids mid-block reloads for typical EXE files
2. For each block: if paged, call `MAP_SET` and register in `OVL_PAGE_TAB`/`OVL_SEG_TAB`; enable PSEL; copy data to virtual address
3. `OVL_COUNT` incremented for each paged block — OVLID assigned by load order
4. After last block (`next_offset=0`), disable PSEL and launch from first block's `start` address

### EXE Build Command

```
link99 -O1000 -M ovltest.EXE ovltest.R99 -P2 ovla.R99 -P3 ovlb.R99
```

- `-O1000` sets base module load address to TPA (`0x1000`)
- Overlay modules use `AORG 2000H` in source; `-P#` assigns physical page
- OVLMGR is **not** linked with the application — it is permanently resident at `0x0A00`

---

## LOAD Command (manual overlay loading)

`%LOAD filename.COM` loads a single-block chain format overlay **without executing it**:

- Programs the mapper for the overlay's page
- Copies code to the virtual address
- Returns to shell prompt

Supports multi-block overlays and sector reload for large files. Used alongside the manual COM overlay workflow where overlays are pre-loaded before running the base program.

---

## Overlay Manager (OVLMGR)

`OVLMGR` is a permanent OS service copied from the shell binary to `0x0A00` at startup. It is always visible in common memory regardless of PSEL state.

### Shell Boot Sequence

1. `COPY_OVLMGR` copies `OVLMGR_CODE` to `0x0A00`
2. Installs vector addresses at `OVL_INIT_VEC` (`0x02A2`) and `OVL_SWAP_VEC` (`0x02A4`)
3. Clears `OVL_COUNT`, `OVL_CURR_ID`, `OVL_CURR_SEG`

### OVLMGR_INIT (via OVL_INIT_VEC)

Called once per program at startup:
- Clears `OVL_CURR_ID` and `OVL_CURR_SEG`
- Enables PSEL

### OVLMGR (via OVL_SWAP_VEC)

Called with `R1 = OVLID` to swap an overlay into segment 2:
- If `R1 == OVL_CURR_ID`: no-op (already loaded)
- Otherwise: disables PSEL, programs mapper (`OVL_PAGE_TAB[OVLID-1]` → `OVL_SEG_TAB[OVLID-1]`), re-enables PSEL
- Modifies `R0`, `R3`, `R9`, `R12`; preserves all others

### OVLID Assignment

OVLIDs are assigned by load order in the EXE file — first paged block = OVLID 1, second = OVLID 2 etc. No hardcoded page numbers anywhere.

---

## Mapper Programming

### MAP_SET

Programs one 6116 mapper register via CRU:

```asm
; R9 = segment (0–15), R0 = physical page (0–15)
MAP_SET:
    LI   R12,MAP_WIN_BASE    ; 0x80C0
    SLA  R9,1                ; segment × 2 = CRU bit offset
    A    R9,R12
    SLA  R0,8                ; page to high byte (LDCR BYTEWIDE uses high byte)
    LDCR R0,BYTEWIDE
    RT
```

### MAP_INIT

Clears all 16 mapper registers to page 0. Called at shell entry and on program return.

### PSEL

Controlled via XOP 2. With PSEL enabled, the 6116 selects the mapped physical page. Disabled forces all segments to page 0 (common).

```asm
LI  R9,PSEL_EN  :  PSEL R9    ; enable
CLR R9          :  PSEL R9    ; disable
```

---

## Interrupt Handling

Seven interrupt vectors at `0x00B0` are initialised to `INT_SAFE` (`RTWP`) at boot.

Before launching any program, `INT2_HANDLER` is installed at `0x000A`. On a privilege violation or illegal opcode it dumps:

```
INT2 PC:xxxx WP:xxxx ST:xxxx
MAP:ssss ssss ... (all 16 mapper register values)
```

---

## Staging Addresses (EQUs)

All staging addresses are named constants — no hardcoded values in loader code:

```asm
EXE_STG:      EQU  0500H    ; EXE loader staging base (sector 1)
EXE_STG2:     EQU  0700H    ; EXE loader second sector
EXE_STG_END:  EQU  0900H    ; EXE loader staging end (2 × 512 bytes)
STAGING:      EQU  0500H    ; COM loader staging base
STAGING_END:  EQU  STAGING+512
```

To extend EXE staging to 3 sectors: change `EXE_STG_END` to `0B00H` and add a third `RDSEQ`.

---

## Version History

| Version | Changes |
|---------|---------|
| 4.0 | Initial TMS9900 port of Small/Shell |
| 4.6 | Internal commands: DIR, SAVE, ERA |
| 5.4 | Paged memory loader; transparent GAL mapping |
| 5.5 | MAP_SET/MAP_INIT; multi-sector loading; INT2 debug handler |
| 5.6 | DOLOAD command; overlay manager variables; interrupt vector init |
| 5.7 | EXE paged chain format support; SET_FTY file type detection; DOLOAD rewritten for chain format; LOADERCODE_EXE with two-sector staging; OVLMGR permanent resident at 0x0A00; OVL_PAGE_TAB/OVL_SEG_TAB populated dynamically by loader; all staging addresses via named EQUs |
