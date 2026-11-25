# Changelog

All notable changes to ADOS for the SAM Coupé.

## 1.2 (25 November 2026)

### Fixed

- **Atom Lite cards formatted by AL-BDOS 2.x are now found as BDOS cards.** AL-BDOS 2.x puts a PC-style FAT16 boot sector on sector 0 of the card, so that a PC sees a normal card. ADOS 1.1 took these cards for a small FAT16 drive, and the records could not be reached. ADOS now recognises the AL-BDOS boot sector. It also checks that a FAT drive really has a FAT, so a card whose boot sector has been renamed is still found as BDOS.

## 1.1 (24 November 2026)

First public release.

- One DOS for floppy drives, the Atom Classic and Atom Lite IDE interfaces, and the Trinity SD card interface.
- BDOS records and PC FAT16 / FAT32 cards and drives, each with its own drive letter from C:.
- Reads and writes MasterDOS disks, including subdirectories and date stamps.
- Files dated from an Atom Lite, SAMBUS or Dallas clock.
- BDOS programs work as under BDOS: BDOS's DVARs, `D2:` for the BDOS card, and BDOS's record and sector hooks.
- Streams (hooks 158, 159, 161 and 192–194) and a direct entry (`CALL &8209`) for programs that have taken the machine over.
- Hook 195 gives a long name (up to 255 characters, with a drive and path) to the next hook that reads a header.

See `Docs/ADOS User Manual.html` for the full details.
