# Small-Shell-for-the-TMS9900

![Platform](https://img.shields.io/badge/CPU-TMS9900-blue)
![OS](https://img.shields.io/badge/OS-CP/M--Like-green)
![Language](https://img.shields.io/badge/Language-Assembly-orange)
![Status](https://img.shields.io/badge/Status-Retro--Project-lightgrey)

---
# SHELL V6.3 — TMS99105 SBC

Command interpreter for the TMS99105 SBC V4 (paged memory), sitting between
the BDOS 5.9 filesystem and the user. Assembled with A99, loaded at `>C000`
as `SHELL.SYS`.

---

## What it does

- Prompts, reads a command line, and runs it — internal command, `.COM`,
  `.EXE`, or `.PRO` procedure file.
- Owns the low-memory interface that applications use: command line, FCB,
  sector buffer, free-memory and memory-limit cells.
- Sets up and tears down the paging state around every application launch.
- Interprets procedure files — batch scripts with parameters, comments,
  echo control and single-step.

---

## Memory map

| Address | Label | Purpose |
| --- | --- | --- |
| `>0080` | `SHELL_PTR` | `B @RGATE` — how an application returns |
| `>0084` | `BDOS_PTR` | BDOS entry gateway |
| `>00A0` | `CMDL_PTR` | pointer to the raw command line |
| `>00A2` | `CMDL_SIZE` | its length |
| `>00A4` | `BUFF_PTR` | pointer to the common sector buffer |
| `>00A6` | `FREEMEM` | first free byte above the loaded image |
| `>00AC` | `FCB_PTR` | pointer to the common FCB |
| `>00B0` | `MEMLIMIT` | top of usable memory for the application |
| `>0230` | `SH_WP` | shell workspace, R0–R15 |
| `>0250` | `SH_CMD` | command line, raw input string |
| `>0280` | `CM_FCB` | FCB, 36 bytes |
| `>02A4` | `CM_VARS` | bridge variables, pointers, status flags |
| `>0300` | `CM_BUF` | 512-byte sector buffer / LOADERCODE area |
| `>0FFE` | `STACKP` | stack, grows down |
| `>C000` | `SHELL` | the shell itself |

Paging (Rev 24Q hardware):

- Segment 0 (`>0000`–`>0FFF`) — COMMON, forced to physical page 0
- Segments 1–E (`>1000`–`>EFFF`) — PAGED, selected by the 6116 map registers
- Segment F (`>F000`–`>FFFF`) — ROM, forced to physical page 0

The 6116 mapper is programmed over the CRU with `LDCR`/`STCR`. The `PSEL`
XOPs enable and disable the GAL page-select output. Segments C and D run on
physical page 1 while an application executes, so the shell's own pages are
hidden from it and restored on return.

---

## Command lookup

A bare command name is resolved by walking `EXEC_TAB` in precedence order:

1. `.COM` — flat image, loaded at the directory entry's load address (FLA)
2. `.EXE` — paged image with an overlay chain
3. `.PRO` — procedure file, interpreted rather than loaded

An explicitly typed extension is used as typed. The file type comes from the
directory entry (`FTY`, FCB offset 11), not from the extension, so
`SEARCH1`/`FOPEN` decide what happens rather than the name.

Type codes match the BDOS: `SYS=0 COM=1 EXE=2 TXT=3 BAS=4 PRO=5`.

### Internal commands

| Command | Purpose |
| --- | --- |
| `DIR` | directory listing |
| `SAVE` | write memory to a file: `SAVE <sectors> <name> [-HHHH]` |
| `ERA` | erase a file |
| `TYPE` | list a text file |
| `LOAD` | load without running |

A leading `.` on any command line loads without running.

---

## Procedure files

A procedure is an ordinary text file whose directory type is `PRO`. It is
never loaded into memory — the shell reads it one 512-byte record at a time
and feeds each assembled line to the command dispatcher, exactly as if it had
been typed.

### Syntax

| Form | Meaning |
| --- | --- |
| `command args` | run it |
| `: text` | comment, discarded |
| `_ text` | echo the line, don't run it |
| `__` | wait: prompt, then `CR` continues, `^C` aborts, `^S` skips the next line |
| `$0`–`$9` | substitute a parameter from the invoking command line |
| `-V` | echo every line as it runs |
| `-N` | fetch but never execute — with `-V`, a listing |

Lines must fit the command line buffer. A comment counts against the same
budget as a command.

Any failing command ends the procedure. So does end of file, `^C` at a wait,
a read error, or running out of records.

### Example

`BDTEST.PRO` — a BDOS regression run:

```
: BDOS 5.9 regression via the shell
_BDOS 5.9 sequential tests
BDTEST S
BDTEST B
BDTEST P
BDTEST 2
_random and cursor tests
BDTEST R
BDTEST C
_done
DIR BDTEST.*
```

Invoked as `BDTEST.PRO`, this launches BDTEST six times with a different test
each time, returning to the shell and resuming the procedure between each:

```
%BDTEST.PRO
_BDOS 5.9 sequential tests
BDTEST60 - BDOS 5.9 + PSEL MEM/CUTOVER DIAG
DESTRUCTIVE ONLY TO BDTEST.DAT
S
BEGIN RECORDS=>0005 (HEX)
WRITE/CAPACITY CHECK OK
READ/PATTERN/EOF CHECK OK
DELETE/RECLAIM OK
*** TEST PASSED ***
BDTEST60 EXIT
...
_random and cursor tests
BDTEST60 - BDOS 5.9 + PSEL MEM/CUTOVER DIAG
R
BEGIN RANDOM ACCESS TEST
WRITE/CAPACITY CHECK OK
RANDOM READ/OVERWRITE/APPEND/REJECT CHECK OK
RANDOM CLOSE/REOPEN PERSISTENCE CHECK OK
DELETE/RECLAIM OK
*** TEST PASSED ***
BDTEST60 EXIT
_done
%
```

### How it runs

- **Entry** — `LOOK` opens the file, records its size, reads `FTY` from the
  FCB and branches to `DOPROC` when it is `PRO`.
- **`DOPROC`**, once per procedure — refuses to nest, sets `PROCSW` (which is
  what makes the shell fetch its next line from the file instead of
  prompting), resets the record cursor, saves the whole FCB to `MNAM`, takes
  its own record count in `PRECS`, and parses the invoking line into `PBUF`
  and `PPTRS` for `$n` substitution.
- **`PFETCH`**, once per record — restores the FCB from `MNAM`, reads into
  `PRBUF`, saves the advanced cursor back.
- **`BUFMOVE`**, once per character — assembles a line, terminating on `CR`;
  `LF` is discarded, and `>00`, `>FF` or `^Z` end the procedure.
- **`PEOL`**, once per line — applies the comment, echo, `-V`, `-N` and wait
  rules, then hands the line to `DOCMD`.
- **`UNPROC`** clears `PROCSW` and returns to the prompt.

Two details make it survive launching programs. Every command a procedure
runs reuses the FCB at `>0280`, so `PFETCH` restores it before each read.
And `INIT_LOADER` copies LOADERCODE over `>0300`, so the procedure buffers
into its own `PRBUF` rather than the common sector buffer.

---

## Building

```
A99 SHELLV63.A99
```

Produces an Intel HEX image at `>C000`. Transfer with the monitor's hex
loader, or convert with `hex2com` and send by XMODEM.

---

## Revision history

Kept in full at the head of `SHELLV63.A99`. Recent work:

- **6.3** — raw COM contract; C and D on physical page 1 during execution,
  with a common BDOS gateway that disables PSEL for the whole call.
- **6.3** — command lookup rewritten. A bare name previously got extension
  `???`, so the first directory entry of that name won — typing `HELLO` ran
  `HELLO.R99` and branched into an object file. Type now read from the FCB.
- **6.3** — procedure processing connected. `DOPROC` had never been called
  from anywhere, so `PROCSW` was never set and the fetch engine was
  unreachable. FCB save/restore, private record buffer, own record count.
