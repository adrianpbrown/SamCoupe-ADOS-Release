# ADOS for the SAM Coupé

ADOS is a single DOS for the SAM Coupé. It takes the place of SAMDOS, BDOS (Trinity) and AL-BDOS (Atom Lite), reads and writes MasterDOS disks, and adds PC FAT16 and FAT32 cards and drives. Load it once and every drive you have gets its own letter.

ADOS fits in one 16K DOS page. Its start-up code is thrown away after boot and the memory is reused for buffers, so it needs no more memory than SAMDOS.

Current version: **1.2**.

## Features

- **One DOS for all storage**: floppy drives, the Atom Classic and Atom Lite IDE interfaces, and the Trinity SD card interface.
- **Drive letters**: A: and B: are the floppies; every BDOS card and FAT partition gets the next letter from C:.
- **Compatible**: disks made by SAMDOS, MasterDOS, BDOS and AL-BDOS work as they are. Error numbers match SAMDOS, so programs that trap DOS errors keep working.
- **MasterDOS directories**: subdirectories, paths and date stamps on SAM disks.
- **BDOS records**: 800K records on a BDOS card or drive, each used like a floppy. BDOS programs see the same DVARs and hooks they expect.
- **PC cards and drives**: FAT16 and FAT32 with long file names (up to 255 characters with the path) and folders. A PC can read what ADOS saves. SAM file details are kept in a small `.info` file next to each file.
- **COPY between any drives**: single files, wildcards and whole disks, with a progress percentage.
- **Disk images on FAT**: an 800K `.MGT` file can be opened as a folder or mounted on a drive letter.
- **Real-time clocks**: files are dated from an Atom Lite, SAMBUS or Dallas clock when one is fitted.
- **For machine code**: file streams with seeking, long-name hooks, and a direct entry point for programs that have taken over the machine.

## Supported hardware

| Device | Media | Letter |
|--------|-------|--------|
| Floppy drives 1 and 2 | 800K SAM disks | A: and B: |
| Atom Lite (left and right bays, master and slave) | CompactFlash | C: onwards, in the order found |
| Atom Classic (master and slave) | IDE hard drive or CompactFlash | |
| Trinity | SD and SDHC cards | |
| Atom Lite, SAMBUS or Dallas clock | Date and time stamps | |

Cards made on a PC must use an MBR partition table (or no partition table at all) and FAT16 or FAT32. GPT, exFAT and NTFS are not supported. SDHC on the Trinity has not been tested on real hardware yet.

## Getting started

1. Write [Release/ADOS-1.2.mgt](Release/ADOS-1.2.mgt) to a floppy disk, or attach it as drive 1 in SimCoupe.
2. Press **F9** or type `BOOT`. ADOS lists the interfaces, drives and clock it found and selects a drive.
3. Use SAM BASIC as normal:

```text
DIR                       list the current drive
DIR "C:"                  list drive C:
DIR = "C:\GAMES"          change the current drive and folder
LOAD "A:GAME"             load from drive A:
COPY "A:*" TO "C:\BACKUP\"
```

To load ADOS from another DOS or loader instead, [Release/ADOS-1.2.bin](Release/ADOS-1.2.bin) is the DOS code on its own (load address 49161), with its SAM header in `ADOS-1.2.bin.info`.

ADOS finds drives only at start-up. If you fit a card after booting, reset and boot again.

## Documentation

| Document | Contents |
|----------|----------|
| [ADOS User Manual](Docs/ADOS%20User%20Manual.pdf) ([HTML](Docs/ADOS%20User%20Manual.html)) | The full manual: setup, drives, every command, interfaces, clocks, errors and settings |
| [Commands](Docs/Commands.md) | Every BASIC command, with a quick reference |
| [Disk Images](Docs/Disk%20Images.md) | Using `.MGT` images on FAT drives |
| [Hook Codes](Docs/Hooks.md) | For machine code: hooks, streams, long names, BDOS compatibility and the direct entry |
| [SAM Info Format](Docs/SAM%20Info%20Format.md) | The `.info` files that hold SAM file details on FAT |

## Not done yet

- FAT12 drives.
- Other SAMDOS and MasterDOS commands, such as `OPEN #` streams to files from BASIC.
- Disk-swap prompts for single-drive floppy copies.

See Appendix C of the manual for the full list.

## Author

ADOS is © 2026 Adrian Brown.
