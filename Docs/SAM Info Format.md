# SAM `.info` files on FAT

A file that ADOS saves to a FAT16 or FAT32 volume holds only its data, byte for byte. ADOS does not add a header, and it drops the 9-byte SAMDOS header. What the SAM needs to load the file again goes in a second file beside it, with `.info` added to the full name:

| Data file    | Info file         |
|--------------|-------------------|
| `GAME`       | `GAME.info`       |
| `GAME.BIN`   | `GAME.BIN.info`   |
| `Saved Game` | `Saved Game.info` |

ADOS sets the hidden attribute on the `.info` file. Readers must not depend on that attribute, because zip, git and many copy tools drop it. ADOS recognises a `.info` file by its name. DIR does not list `.info` files, and LOAD never matches one with a wildcard.

When a file has no `.info` file, ADOS loads it as CODE at 32768. The length always comes from the data file's size, never from the `.info` file.

## Layout

All numbers are little-endian.

| Offset | Size | Contents |
|--------|------|----------|
| 0 | 4 | `SAMI` |
| 4 | 1 | Version, 1 |
| 5 | ... | Entries |

Each entry is:

| Size | Contents |
|------|----------|
| 2 | Type |
| 2 | Length of the data |
| n | Data |

A type of 0 ends the list and has no length field. Each entry stores a single item.

A reader skips any type it does not know by moving past *length* bytes. A reader also skips a known type whose length is not the expected one. Types 0x8000–0xFFFF are free for other tools; ADOS never writes them.

## Types

Addresses are plain numbers as BASIC sees them: `SAVE "x" CODE 32768,100` stores a start address of 32768. Lengths are byte counts.

| Type | Item | Length | ADOS writes it | ADOS reads it |
|------|------|--------|----------------|---------------|
| 1 | SAM file type: 16 BASIC, 17 numeric array, 18 string array, 19 CODE, 20 SCREEN$ | 1 | Always | Yes |
| 2 | SAM file name, space padded, without drive or path | 10 | Always | Yes |
| 3 | Start address | 3 | Always | Yes |
| 4 | Execute address | 3 | CODE files that have one | Yes |
| 5 | BASIC program length, without variables | 3 | BASIC | No |
| 6 | BASIC program + numeric variables length | 3 | BASIC | No |
| 7 | BASIC program + numeric variables + gap length | 3 | BASIC | No |
| 8 | BASIC autostart line | 2 | Reserved | |
| 9 | Screen mode | 1 | Reserved | |
| 10 | Array name | 1–10 | Reserved | |
| 11 | Flags (UIFA byte 15: bit 0 hidden, bit 1 protected) | 1 | Always | No |
| 12 | The ROM header (UIFA) bytes 15–47, exactly as saved | 33 | Always | Yes |

Types 5–7 hold the ROM header's own fields (UIFA bytes 16–24), converted to plain numbers.

If type 12 is present, ADOS loads from it, so a file saved from the SAM comes back exactly as it was, BASIC included. Types 3 and 4 are there so other tools can read the addresses without knowing the ROM's header layout. A tool that creates a `.info` file only needs types 1, 2 and 3, plus 4 for a CODE file that should run.

## Example

The `.info` for `SAVE "KEEP.BIN" CODE 40960,5000,40976`:

```
53 41 4D 49 01                     "SAMI", version 1
01 00 01 00  13                    type 1:  19 (CODE)
02 00 0A 00  4B 45 45 50 ...       type 2:  "KEEP.BIN  "
03 00 03 00  00 A0 00              type 3:  40960
04 00 03 00  10 A0 00              type 4:  40976
0B 00 01 00  00                    type 11: flags
0C 00 21 00  00 ...               type 12: UIFA bytes 15-47 (33 bytes)
00 00                              end
```
