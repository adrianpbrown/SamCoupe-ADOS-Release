# Changelog

All notable changes to ADOS for the SAM Coupé.

## 1.3 (25 November 2026)

### Added

- **ADOS can boot from an Atom Lite card**, with no floppy disk, using the Atom Lite boot ROM (AL BOOT ROM). On a BDOS card, hold B while resetting to boot BDOS instead. A card boot shows no start-up screen. Tested in SimCoupe, not yet on real hardware.
- **MAKEBOOT** sets a card up for this, on the SAM itself: FAT cards with free sectors before the partition or in their reserved area, and AL-BDOS cards with a record that has never been formatted. It is on the ADOS disk; see Docs/Utilities.md.
- **BDOS2FAT** turns a BDOS drive into a FAT32 drive, on the SAM itself, with every record that holds files as an 800K disk image. It is on the ADOS disk; see Docs/Utilities.md.
- **`LOAD n`** loads a file by its number in the `DIR` listing, on SAM disks, BDOS records and disk images.

### Fixed

- **Loads that carry on after one header now work**, such as a SCREEN$ and its palette.
- **Hook 129 works when IX points at the caller's own memory** (&8000 and above).
- **`RECORD` no longer shows `?` after record names.** AL-BDOS ends a name with 0 bytes rather than spaces, and ADOS printed them. A name now ends at its first 0 byte.

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
