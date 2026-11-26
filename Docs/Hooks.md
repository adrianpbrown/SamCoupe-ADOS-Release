# ADOS hook codes

Machine code programs call the DOS with `RST 8` followed by a hook code. This guide lists the hooks ADOS supports, how they behave on FAT drives, and the stream hooks that read any file a piece at a time.

For the BASIC commands, see [Commands](Commands.md).

## Calling a hook

Put the arguments in the registers, then:

```
        RST  8
        DB   161          ; the hook code
        EXX               ; results come back in the alternate registers
```

- **Arguments** go in the main registers: A, HL, DE, BC, and IX for a 48-byte header (normally the ROM's at &4B00).
- **Results** come back in the alternate registers: A', HL', DE', BC'. Use `EXX` (and `EX AF,AF'` for A') to reach them. A hook that gives no result returns its arguments there unchanged.
- **Errors** are reported by the ROM in the usual way, for example "107 File not found". The numbers are listed in [Commands](Commands.md#error-messages). Set DVAR 48 to get them back instead (see below).

## Getting errors back instead of a BASIC error

Normally a failed hook ends in a BASIC error, and the program only gets control back by patching the ROM's error handling. If DVAR 48 is not 0, ADOS gives the error back to the caller instead, the way a [direct call](#direct-entry) does:

- **Carry set in F'** and **A' = the error number**, for example 107. HL', DE' and BC' are undefined.
- **On success, carry is clear in F'.** The results are in A', HL', DE' and BC', as usual.
- BASIC sees no error.

```
        RST  8
        DB   158          ; open the file in the header at IX
        EX   AF,AF'
        JR   C,FAILED     ; A = the error number
        EX   AF,AF'
```

DVAR 48 is at offset &0250 in the DOS page. Find its address with hook 139, as for DVAR 7 (see [Is ADOS loaded?](#is-ados-loaded)), or from BASIC with `POKE DVAR 48,1`. Check that the DOS is ADOS first, because DVAR 48 means something else under MasterDOS and BDOS.

> **Set it only around your own calls, and put it back to 0 afterwards.** While it is set, every ADOS error behaves like this, including commands typed in BASIC and the ROM's own LOAD and SAVE. Those then fail without any message, and the ROM may carry on as if they had worked.

## Is ADOS loaded?

Codes 128 to 174 are shared with SAMDOS and MasterDOS. Codes 192 and up are ADOS's own. Under MasterDOS an unknown code above 174 crashes the machine, so check for ADOS before using them.

The system variable DOSFLG (&5BC2) is 0 if no DOS is loaded. If one is, ask it for DVAR 7, the DOS version, with hook 139. Every DOS has hook 139: it is what BASIC's `DVAR` uses, and SAMDOS, MasterDOS, BDOS and ADOS all give the address of DVAR n. Don't read DVAR 7 from a fixed place in the DOS page, because not every DOS keeps its DVARs in the same place.

ADOS writes DVAR 7 as version × 10 + 50, so 60 is ADOS 1.0, 61 is ADOS 1.1, 62 is ADOS 1.2 and 63 is ADOS 1.3. That keeps it clear of SAMDOS (20), MasterDOS (42 = 2.2, 43 = 2.3), BDOS (5) and Betasoft's MasterBASIC, which changes DVAR 7 to 50 when combined with MasterDOS and 21 when combined with SAMDOS. From BASIC:

```
IF PEEK DVAR 7 >= 61 THEN ...
```

From machine code, put n on the calculator stack, call hook 139, and turn the number it gives back into a page and an offset:

```
        LD   A,(&5BC2)    ; DOSFLG
        AND  A
        JR   Z,NoADOS     ; no DOS loaded
        XOR  A            ; 7 on the calculator stack (A, E, D, C, B =
        LD   E,A          ; 0, 0, 7, 0, 0 is the small integer 7)
        LD   D,7
        LD   C,A
        LD   B,A
        CALL &0127        ; JSTKSTORE
        RST  8
        DB   139          ; DVAR: 7 becomes the address of DVAR 7
        CALL &0124        ; JSTKFETCH: A = exponent, E, D, C, B = mantissa
        SET  7,E          ; the mantissa's top bit (stored as the sign)
        SUB  &96
        JR   Z,Shifted
        NEG
        LD   B,A          ; shift E:D:C right &96 - exponent times
Shift:  SRL  E
        RR   D
        RR   C
        DJNZ Shift
Shifted:                  ; E:D:C = the address * 4
        DEC  E            ; E = the page
        SRL  D
        RR   C
        SRL  D
        RR   C            ; D:C = the offset
        SET  7,D          ; seen in section C
        IN   A,(251)      ; HMPR
        LD   B,A          ; keep it
        AND  &E0
        OR   E
        OUT  (251),A      ; the page in section C
        LD   L,C
        LD   H,D
        LD   C,(HL)       ; DVAR 7
        LD   A,B
        OUT  (251),A      ; section C back
        LD   A,C
        CP   60
        JR   C,NoADOS     ; below 60: not ADOS
        ; ADOS if A is 60 or more (61 or more for the direct entry and hook 195)
```

Run this from section A or B, not from section C, because it pages section C over. It needs the ROM and the system variables, so a program that takes the machine over should do it first.

The DOS variables DVAR 0 to 45 are laid out as in BDOS (see [BDOS compatibility](#bdos-compatibility)), and DVAR 0 to 7 mean the same as in SAMDOS and MasterDOS too, so programs that read or POKE them are safe. ADOS uses DVAR 0 as MasterDOS does: the border flashes while sectors are read or written, in colour (DVAR 0 AND 7) AND the sector number, and `POKE DVAR 0,0` turns it off. ADOS's own variables are DVAR 46 to 48: the drive timeout (46), the drive selected at boot (47) and the error setting above (48). In ADOS the DVARs are at offset &0220 of the DOS page, and, as in BDOS, the word at offset 0 holds their address as seen with the DOS page in section C (&8220). The direct entry relies on the fixed offset (see below), but programs that can use hook 139 should.

### Making sure it is ADOS

Checking DVAR 7 works with the DOSes there are, but it is safest to check for ADOS's name as well. Version numbers aren't reserved: each DOS chose its own, and nobody keeps a list. Another DOS, or a later version of one, could use 61 and be taken for ADOS. MasterBASIC is an example: it changes the DOS's DVAR 7 to its own number. A program would then call codes 192 and up, and under MasterDOS that crashes the machine.

ADOS puts `ADOS` at offset &0212 in its page, followed by its version as two bytes (major, then minor). In another DOS those four bytes could be anything, but they won't spell `ADOS` by chance. After the DVAR 7 check:

```
        LD   A,(&5BC2)    ; DOSFLG
        LD   B,A
        IN   A,(251)      ; HMPR
        PUSH AF
        AND  &E0
        OR   B
        OUT  (251),A      ; the DOS page in section C
        LD   HL,(&8212)   ; "AD"
        LD   DE,(&8214)   ; "OS"
        LD   BC,(&8216)   ; C = major, B = minor
        POP  AF
        OUT  (251),A      ; section C back
        ; ADOS if HL = "AD" (&4441) and DE = "OS" (&534F)
        ; then C.B is the version, for example 1.1
```

## Hooks ADOS supports

| Code | What it does | Arguments | Results |
|------|--------------|-----------|---------|
| 128 | Load and run the `AUTO*` file if there is one (F9) | | |
| 129 | Find a file and give its header | IX = header with type and name | Header at &4B50 |
| 130 | Load the file found by 129 | HL = address (section C), C = 16K pages, DE = bytes mod 16K | |
| 131 | Verify it | As 130 | |
| 132 | Save | IX = full 48-byte header | |
| 134 | Verify sectors (BDOS) | As 162 | |
| 135 | Select the default drive | | |
| 136 | As 128, but error 101 if there is no `AUTO*` file | | |
| 139 | DVAR: the address of DOS variable n (BASIC's `DVAR n`) | n on the calculator stack | The address on the calculator stack |
| 142 | PATH$ | | String on the calculator stack |
| 145 | Set the current drive and directory (`DIR =`) | A = page, DE = address, BC = length of the name | |
| 149 | Write a sector | As 160 | |
| 156 | Select a record on the BDOS drive (BDOS) | A = 0: HL = the record number. A = 1: HL = the address of a 16-character record name | |
| 157 | Verify a block from the stream (BDOS) | As 161 | As 161 |
| 158 | Open a file as a stream | IX = header with type and name | Header at &4B50, DE':HL' = length |
| 159 | Read one byte from the stream | | A' = the byte |
| 160 | Read a sector | A = drive, D = track, E = sector, HL = address (16384-65535) | |
| 161 | Read a block from the stream | A = 16K pages, DE = bytes mod 16K, HL = address (section C) | C':DE' = bytes read, B':HL' = next free byte |
| 162 | Read sectors (BDOS) | A = drive, D = track, E = sector, C = page, HL = offset in it, IX = how many | |
| 163 | Write sectors (BDOS) | As 162 | |
| 164 | Restore the drive head (BDOS). Does nothing | | |
| 165 | Print a directory listing | IX = header with the name or pattern | |
| 166 | Erase a file | IX = header with the name | |
| 192 | Move the stream to a position | DE:HL = position | |
| 193 | The stream's position | | DE':HL' = position |
| 194 | Close the stream | | |
| 195 | Give a long name for the next hook that reads a header (from 1.1, see [Long names](#long-names)) | A = page, DE = address, BC = length of the name (0 cancels) | |

158, 159 and 161 have the same numbers and arguments as in MasterDOS and BDOS, and 149 and 160 as in SAMDOS. The codes marked BDOS are explained under [BDOS compatibility](#bdos-compatibility). Other SAMDOS, MasterDOS and BDOS codes from 128 to 194 aren't supported yet and do nothing, as BDOS's own unused codes do. That includes SAMDOS's hooks for writing a file a piece at a time (147, 148, 150 and 152). A code below 128 or above 195 gives error 113 "Bad hook".

**Long names from BASIC.** ROM 3.0 refuses a LOAD, SAVE, VERIFY or MERGE name over 14 characters with error 18 before the DOS sees it. So ADOS puts a 32-byte routine at &4BA0 in the system page and points RST8V (&5AEE) at it. The ROM's error handler calls RST8V first. For that one error the routine hands the whole name (up to 255 characters) to ADOS through hook 195 (see [Long names](#long-names)), then sends the ROM on as if the name were 14 characters long. The next header hook (129 or 132) then uses the whole name. Every other RST 8 goes straight through. A program that uses RST8V itself replaces this, and long names from BASIC then give error 18 again. The vector and routine are put back each time hook 135 runs.

## Long names

The name in a header has room for 14 characters. Hook 195 (from ADOS 1.1) gives ADOS a longer one, up to 255 characters, for the next hook that reads a header: 129 (and so 130 and 131), 132, 158, 165 or 166. It is the name BASIC would take in `LOAD`: a drive, a path and a FAT long name are all allowed, for example `D:\GAMES\Monkey Island Part 1.dat`.

- **The arguments are those of hook 145:** A = the page the name is in, DE = its address as if that page were in section C (&8000-&BFFF), BC = its length. See the table below.
- **The name in the header is then ignored**, but the rest of the header (type, addresses, length) is used as usual.
- **It is used once.** The hook after it that reads a header takes it, and the next one uses its own header again.
- **Call 195 just before that hook.** Anything else that reads a string from you in between, such as hook 145 or a BASIC command with a name, cancels it, and so does any BASIC error. BC = 0 cancels it too.
- **More than 255 characters** gives error 89 "Bad file name" from hook 195 itself.
- **It works through the direct entry too** (see [Direct entry](#direct-entry)).

The name can be anywhere from 16384 up. With BASIC's paging:

| Name's address | Section | A | DE |
|----------------|---------|---|----|
| 16384-32767 | B (page 0, the system page) | 0 | address + &4000 |
| 32768-49151 | C (page 1) | 1 | address |
| 49152-65535 | D (page 2) | 2 | address - &4000 |
| 0-16383 | A (ROM 0 is over it) | can't be used | |

That is, A = (address / 16384) - 1 and DE = (address AND &3FFF) + &8000. A program that changes the paging takes the page from the ports instead: section B holds page (LMPR + 1) AND 31, section C HMPR AND 31 and section D (HMPR + 1) AND 31. DE is worked out the same way. With the direct entry the name can be in any page, paged in or not.

Save 10 bytes from 32768 as `D:Hook long name test.bin`, with the name assembled at 32768-49151:

```
        LD   A,1          ; NAME is at 32768-49151: page 1
        LD   DE,NAME
        LD   BC,NAMEEND-NAME
        RST  8
        DB   195          ; the long name
        LD   IX,&4B00     ; header: type 19, start page 1, &8000, 0 pages, 10 bytes
        RST  8
        DB   132          ; save
        ...
NAME:   DM   "D:Hook long name test.bin"
NAMEEND:
```

## BDOS compatibility

A program written for BDOS 1.5 (Trinity) or AL-BDOS (Atom Lite) finds what it expects when ADOS runs with a BDOS card: the DVARs laid out as BDOS has them, drive 2 as the card, and BDOS's record and sector hooks.

### The BDOS drive

BDOS has one hard drive, its drive 2. ADOS can have several BDOS cards, for example one in the Atom and one in the Trinity. So drive 2 and the BDOS DVARs mean the BDOS drive used last:

- At boot it is the first BDOS card found (Atom Lite right bay, Atom Lite left bay, Atom Classic, then Trinity).
- Using another BDOS drive makes that one the BDOS drive, and its details replace the DVARs. That includes LOAD, SAVE, DIR, RECORD and a sector hook given its letter.
- Each BDOS drive keeps its own record. Changing to another one doesn't change the record selected on either.
- With no BDOS card, drive 2 is floppy drive 2 (B:), as in SAMDOS, and DVAR 2 is 0.

`D1:` in a file name is A:, and `D2:` is the BDOS drive (B: if there is none).

### DVARs

As in BDOS, the word at offset 0 of the DOS page holds the address of DVAR 0 as seen with the DOS page in section C (&8220). Hook 139 gives the same address in BASIC's form.

| DVAR | BDOS meaning | In ADOS |
|------|--------------|---------|
| 0 | Border flash | As SAMDOS and MasterDOS (see above) |
| 1 | Drive 1 tracks | &D0 |
| 2 | Drive 2 tracks | The BDOS drive's current record: &D0 (80 tracks, 2 sides), or DVAR 6 for its last record. 0 with no BDOS drive |
| 3, 4 | Default device | "D", 1 (not used) |
| 5 | Space character | " " (not used) |
| 6 | Tracks in the last record | The BDOS drive's |
| 7 | Version | ADOS's: version × 10 + 50 (BDOS has 5) |
| 8, 12-13, 10 | First sector of the current record | Bits 0-7, 8-23 and 24-31 of its sector number on the card, as AL-BDOS has them in LBA mode |
| 9, 11, 14-17 | Sectors per track, heads, cylinders | 0. AL-BDOS doesn't use them in LBA mode, and DVAR 34 says LBA |
| 18-20 | Total sectors | 0 (AL-BDOS doesn't set them either) |
| 21-22 | First sector of record 1 | The BDOS drive's |
| 23-24 | The last record (the number of records) | The BDOS drive's |
| 25-26 | The current record | The BDOS drive's. POKE it to select a record: the next use of the drive takes it, from the record's root directory |
| 27 | Write protected | 0. ADOS checks a record's protection when it writes |
| 28-29 | Temporary record | As DVAR 25 |
| 30-33 | Alternative record, command record | 0 (not used) |
| 34 | Drive select | &E0 for a master and &F0 for a slave (LBA mode). Bit 0 is set for the Atom Lite's left bay |
| 35-37 | `JP` to start the DOS again | Runs hook 135 again. Call it with the DOS page in section C, as in BDOS |
| 38 | Date flags | 1 (not used) |
| 39-44 | Date and time: day, month, year, hour, minute, second | Read from the clock when hook 139 asks for DVAR 39, as in BDOS. The year counts from 1900 (126 is 2026), as ADOS stamps files. The day is 0 with no clock |
| 45 | Spare | 0 |
| 46 | ADOS: drive timeout | 3 |
| 47 | ADOS: drive selected at boot | "A" |
| 48 | ADOS: errors back to the program | 0 |

### Records

Hook 156 selects a record on the BDOS drive, by number (A = 0, HL = the number) or by its 16-character name in the record list (A = 1, HL = its address, 16384-65535, read as for hook 160). As in BDOS, the BDOS drive becomes the current drive, and record 0 makes floppy drive 1 (A:) current instead. It gives "No drive" (103) if there is no BDOS card, and "No record" (114) if the record isn't there.

### Sectors

| Code | What it does | Arguments |
|------|--------------|-----------|
| 160 | Read a sector | A = drive, D = track, E = sector, HL = address 16384-65535 |
| 149 | Write a sector | As 160 |
| 162 | Read sectors | A = drive, D = track, E = sector, C = page, HL = offset in it (0-&3FFF), IX = how many |
| 163 | Write sectors | As 162 |
| 134 | Verify sectors: compare them with memory | As 162. "Verify failed" (93) if they differ |

- **The drive** is 1 (A:), 2 (the BDOS drive, or B:) or, in ADOS, 0 for the current drive or a drive letter such as "C". It must be a SAM disk: a floppy, a BDOS record (the current one), or a disk image that a FAT drive is in. Otherwise the hook gives "Bad device" (91).
- **Use 0 in a loader** that reads its own disk a sector at a time. It then works wherever it was loaded from: a floppy, a BDOS record, or a disk image after `DIR = "D:\GAMES\GAME.MGT"`. A loader that asks for drive 1 always reads floppy A:, so it can't find its files when it is run from an image. 0 is an ADOS addition, so a loader that must also run under another DOS should use 1 there.
- **The address** for 160 and 149 is read as BASIC's memory is paged, as in BDOS and MasterDOS: 16384-32767 is page 0, 32768-49151 page 1 and 49152-65535 page 2. Below 16384 gives error 30, "Integer out of range". For 162, 163 and 134 the page is given in C.
- **Each sector** goes 512 bytes on from the one before, on into the following pages. The track and sector go on through sector 10 and then the next track. Track 79 is followed by track 0 of side 2 (128).
- **The track and sector** must be on the disk: tracks 0-79 on either side (128-207 for side 2) and sectors 1-10. Otherwise, or if the sectors run off the end of the disk, the hook gives "Disk error" (85). Asking for no sectors (IX = 0) gives error 30.
- **The directory isn't read first,** so these hooks work on any disk with SAM tracks and sectors, and a floppy's head moves only to the sectors asked for.
- **Not for direct calls.**

### Not the same as BDOS

- BDOS numbers some errors differently (its 81-83 are about records and sectors). ADOS keeps SAMDOS's numbers (see [Commands](Commands.md#error-messages)), and gives "No record" (114) and "Disk error" (85) instead.
- SAMDOS's hooks for writing a file a piece at a time (147, 148, 150 and 152) aren't supported yet.
- BASIC's `DEVICE D2` doesn't change the current drive. Use `D2:` in the file name, or `DIR =` with the BDOS drive's letter.
- BDOS's `READ AT`, `WRITE AT`, `HIDE`, `PROTECT` and `.` commands aren't in ADOS.

## Headers on FAT drives

On a SAM disk the header is the one the file was saved with. On a FAT drive (see [SAM Info Format](SAM%20Info%20Format.md)):

- **With a `.info` file**, the type, name and addresses come from the `.info`.
- **Without one**, for example a file copied from a PC, the file is CODE, loads at 32768, and has no run address.
- **The length is always the file's size** on the card.

Hooks 129 and 158 both work this way, so a program can load or stream any file on any drive.

The length field in the header (16K pages and the rest) holds up to 4 MB. Hook 158 also returns the full length as a 32-bit number in DE':HL', which is correct for any file size.

## Streaming a file

A stream reads a file a piece at a time, from any position, on any drive: a SAM floppy, a BDOS record, a FAT drive, or a SAM disk image on a FAT drive.

- **One file at a time.** Opening a file closes the one that was open.
- **Positions count data bytes only.** A SAM file's 9-byte header isn't counted, and FAT files have no header, so position 0 is the first byte of the data on every drive.
- **Reads cross page boundaries.** A read carries on through the following pages and stops at the end of the file. It returns how many bytes it read, which is 0 at the end. Reading past the end isn't an error, except for hook 159, which gives error 108 "End of file".
- **Seeking past the end** gives error 108. Seeking to the end exactly is allowed.
- **Other DOS work can come between stream calls.** LOAD, SAVE, DIR, changing drive and so on all work, and the next stream call carries on where it left off.
- **Stream calls with no file open** give error 119 "File not open", except hook 194 (close), which does nothing.

The name in the header can be up to 14 characters, drive letter included. For a longer name, give it with hook 195 first (see [Long names](#long-names)). To open a file deeper in a folder or inside a disk image, either put the whole path in the long name or set the directory first with hook 145 (as `DIR = "D:\GAMES\CHAOS.MGT"`) and then open the file by its own name.

### Example

Read the first 1000 bytes of `D:MUSIC.DAT` to &9000, then the 50 bytes from position 20000 to &A000:

```
        LD   IX,&4B00     ; header: type 0 (any), then the name
        RST  8
        DB   158          ; open (the length is in DE':HL')

        XOR  A            ; 0 whole pages
        LD   DE,1000
        LD   HL,&9000
        RST  8
        DB   161          ; read

        LD   DE,0         ; position 20000
        LD   HL,20000
        RST  8
        DB   192          ; seek

        XOR  A
        LD   DE,50
        LD   HL,&A000
        RST  8
        DB   161

        RST  8
        DB   194          ; close
```

The header at &4B00 for that open is type 0 then `D:MUSIC.DAT` padded with spaces to 14 characters.

### Seeking

A seek reads only the sector the new position is in:

- **On FAT** it follows the file's cluster chain, which reads only the FAT, from where it is (forwards) or from the start (backwards). The sectors in between aren't read.
- **On a SAM disk, BDOS record or disk image**, a file whose sectors are all in one run (as the SAM DOSes save a file when the disk has the room) goes straight to the right sector. ADOS checks the file's sector map when it is opened.
- **Other SAM files** (saved in pieces on a full or fragmented disk) are read through to the position, from where it is or from the start, because each sector holds the link to the next.

### Limits

- **Read only.** Writing through a stream isn't supported yet.
- **Disk changes.** If the floppy is changed while a file is open, ADOS doesn't notice. Close the file before the disk is changed and open it again afterwards: an open always reads the directory from the disk.
- **ERASE and SAVE** don't check for an open stream, so don't erase or replace a file while it is open.

## Direct entry

A program that has taken the machine over can't use `RST 8`: that needs ROM 0, the system variables and the header area at &4B00. The direct entry needs only the DOS page. A game that pages ROM 0 out, reuses page 0 and runs its own interrupts can still load and save through ADOS, from any drive.

### Finding it

Before taking the machine over, read the DOS page number from DOSFLG (&5BC2) and check for `ADOS` at offset &0212 of that page, as above. The version is the two bytes at offset &0216 (major, then minor). The direct entry and hook 195 are there from 1.1.

ADOS keeps no note of which page it is in, so the program can copy the 16K page somewhere else and use the copy. Its drive details, current directories and the open file are all inside the page. After booting, ADOS uses no other memory.

### Calling it

Page the DOS page into section C and call offset &0209 in it (&8209), with the hook code in the byte after the `CALL`. It is in ADOS's jump block, after the hook, syntax and NMI entries the ROM uses (&8200, &8203 and &8206).

```
        LD   A,DOSPAGE    ; HMPR: the DOS in section C (section D is the page after it)
        OUT  (251),A
        LD   IX,HEADER
        CALL &8209
        DB   158          ; open
        JR   C,ERROR      ; A = the error number
```

- **Arguments** are the hook's, in the main registers (A, HL, DE, BC, IX).
- **Results** come back in the main registers, not the alternate ones. On success carry is clear and A = 0. On an error carry is set and A is the error number, for example 107 "File not found" or 87 "No disk". The program carries on either way.
- **Registers:** IX, IY and the alternate registers are corrupted.
- **Your code, your stack and the header at IX** must be in section A or B, or in section D. Section D holds the page after the DOS page (or external memory, if HMPR bit 7 is set). It can't be in section C, which holds the DOS. Hooks 145 and 161 take a page number and an offset, so their data can be in any page.
- **Paging (LMPR and HMPR) and the interrupt state** are as they were when it was called.

These hooks can be called this way:

| Code | What it does | Arguments | Results |
|------|--------------|-----------|---------|
| 132 | Save (replaces a file of the same name) | IX = 48-byte header: type, name (14 characters, drive allowed), start page (byte 31), offset as &8000-&BFFF (32-33), 16K pages (34), length mod 16K (35-36) | |
| 145 | Set the current drive and directory | A = page of the path, DE = its address as if that page were in section C (&8000-&BFFF), BC = length | |
| 158 | Open a file | IX = 48-byte header: type (0 = any), then the name (14 characters, drive allowed) | DE:HL = length |
| 161 | Read | **C = destination page**, HL = offset in it (0 to &3FFF), A = 16K pages, DE = bytes mod 16K. It carries on into the following pages. | C:DE = bytes read (C 16K pages), B:HL = where the next byte would go (B page, HL as &8000-&BFFF) |
| 192 | Seek | DE:HL = position | |
| 193 | Position | | DE:HL = position |
| 194 | Close | | |
| 195 | Long name for the next 132 or 158 (see [Long names](#long-names)) | As 145 | |

Other hooks use the system variables or the ROM, so don't call them this way.

### Paging and interrupts during a call

While a direct call runs:

| Section | Holds |
|---------|-------|
| A | ROM 0. The page below the DOS is underneath it, but ADOS doesn't write there. |
| B | The DOS. It runs on its own stack at the top of its page. |
| C | The DOS. The page being read into or saved from is only paged in while a piece of it is copied (up to 512 bytes), and interrupts are off while it is. |
| D | The page after the DOS. Section D only changes while that copy is made. |

ADOS never enables interrupts that were off. It disables them only while it pages data in, and for each floppy sector (see below). So a program can keep interrupts on during calls, provided that:

- **The IM2 vector table and handler are in the page after the DOS page,** where they are seen in section D. For example, with I = &E9 the table and handler at &E900-&EAFF are at offset &2900-&2AFF of that page. Otherwise, call with interrupts off.
- **The handler uses only section D.** It runs on ADOS's stack, which has about 50 bytes spare, and must not change the paging.

A floppy sector is read or written with interrupts off, from the command until the sector is done. That is up to one revolution of the disk (200 ms), so a frame or more can be missed. With no disk in the drive it is until the drive timeout. Hard drives and cards keep interrupts on except for the copies.

### Other things to set

- **DVAR 0** (offset &0220): the border flashes while sectors are read, and a direct call doesn't put the border back afterwards. POKE it with 0 to turn the flash off.
- **DVAR 46** (offset &024E): the drive timeout. A floppy drive with no disk gives "No disk" (87) after it. The unit is a fixed count of status reads: about 1.5 seconds with interrupts off, longer with them on. The default is 3. The read can wait twice (for the drive and for the data), so 1 suits a program that retries.
- **Disk changes:** close the file, ask for the disk, then open again. An open reads the directory from the disk every time, so the new disk is seen. With every file in one folder on a hard drive or card, the opens all succeed and no disk ever needs changing.

### Direct entry example

Open `MI1.DAT` in `D:\GAMES\MONKEY`, then read 2000 bytes from position 300000 into page 12 at offset &1000:

```
        LD   A,(DOSPAGE)
        OUT  (251),A          ; DOS in section C
        LD   A,CODEPAGE       ; the page PATH is in
        LD   DE,PATH+&4000    ; PATH is in section B, so &4000 more in section C
        LD   BC,PATHEND-PATH
        CALL &8209
        DB   145              ; DIR = "D:\GAMES\MONKEY"
        JR   C,ERROR
        LD   IX,HEADER
        CALL &8209
        DB   158              ; open
        JR   C,ERROR
        LD   DE,&0004         ; position 300000 = &0493E0
        LD   HL,&93E0
        CALL &8209
        DB   192              ; seek
        JR   C,ERROR
        XOR  A                ; 0 whole pages
        LD   DE,2000
        LD   HL,&1000
        LD   C,12             ; into page 12
        CALL &8209
        DB   161              ; read
        JR   C,ERROR
        ...
PATH:   DM   "D:\GAMES\MONKEY"
PATHEND:
HEADER: DB   0
        DM   "MI1.DAT       "
        DS   33
```

`Test/direct.asm` in the ADOS sources is a full example. It takes the machine over the way a game does: it pages ROM 0 out, overwrites page 0, copies the DOS page and runs its own IM2 interrupts. Then it opens, seeks, reads, saves and checks every byte.
