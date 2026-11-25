# ADOS commands

This guide covers every command ADOS adds to SAM BASIC, and how the ROM's own file commands (LOAD, SAVE, VERIFY) behave on each kind of drive. The same commands work on every drive: SAM floppies, BDOS records on a Trinity or Atom Lite, FAT16/FAT32 partitions, and SAM disk images kept on FAT drives.

For disk images, see [Disk Images](Disk%20Images.md). For machine code programs, including reading files a piece at a time, see [Hook Codes](Hooks.md). For the hidden `.info` files that hold SAM file details on FAT, see [SAM Info Format](SAM%20Info%20Format.md).

## Quick reference

| Command | What it does |
|---------|--------------|
| `DIR` | List the current directory |
| `DIR "X:name"` | List a drive, a folder (end the name with `\`) or matching files |
| `DIR ?` | List every file on a SAM disk, in any directory |
| `DIR 1` / `DIR 2` | List drive 1 (A:) or 2 (the BDOS card, or B:) |
| `DIR = "X:path"` | Change the current drive and directory |
| `PATH$` | The current drive and path, as a string |
| `LOAD`, `SAVE`, `VERIFY` | The ROM's commands, on any drive |
| `LOAD n` | Load file number n of the `DIR` listing (SAM disks and disk images) |
| `ERASE "name"` | Erase files (wildcards allowed) |
| `RENAME "old" TO "new"` | Rename a file or directory |
| `OPEN DIR "name"` | Make a directory |
| `ERASE DIR "name"` | Erase an empty directory |
| `RENAME DIR "old" TO "new"` | Rename a directory |
| `COPY "from" TO "to"` | Copy files between any drives |
| `COPY "A:" TO "D:X.MGT"` | Copy a whole SAM disk |
| `FORMAT "X:name"` | Make a blank disk image on a FAT drive |
| `OPEN "X:" TO "image"` | Mount a disk image on a free drive letter |
| `CLOSE "X:"` | Unmount it |
| `RECORD` | List the named BDOS records |
| `RECORD n` / `RECORD "name"` | Select a BDOS record |
| `BOOT`, F9 | Load and run the `AUTO*` file |

## Drives, names and paths

### Drive letters

ADOS finds the drives once, when the SAM starts. Cards and disks with partitions aren't scanned again until the next reset.

| Letter | Drive |
|--------|-------|
| A, B | Floppy drives 1 and 2 (`D1:` also works for A:) |
| C onwards | Every other drive and partition, in the order they are found |

A and B always belong to the floppies, even when there is no second drive (a disk image can still be mounted on B:). The other drives are found in this order: Atom Lite right bay (master, then slave), Atom Lite left bay, Atom Classic master, Atom Classic slave, Trinity. Each FAT partition, and the BDOS records, takes the next letter, so an Atom Classic with two partitions and an SD card as slave gives C: and D: on the hard disk and E: on the card. Up to 12 drive letters can be in use at once, counting the floppy drives and mounted disk images. If more drives and partitions are found than that, the extra ones get no letter, and a disk image can't be mounted until a letter is freed with CLOSE. A floppy drive that is fitted always has its letter, even with no disk in it.

`D2:` is the BDOS card in use, as in BDOS, or B: if there is no BDOS card. With more than one BDOS card it is the one used last (see [BDOS programs](#bdos-programs)).

A name with no drive letter uses the current drive. `DIR = "D:"` changes the current drive.

### Paths

The same path rules apply on every drive:

| Part | Meaning |
|------|---------|
| `\` or `/` at the start | The root of the drive |
| `\` or `/` between names | Separates folders |
| `..` | Up one level (`^` also works) |
| `GAMES.MGT` | A disk image on a FAT drive is a folder you can go into |

```
LOAD "D:\GAMES\CHAOS.MGT\chaos"
DIR = "A:\UTILS"
DIR = ".."
```

On SAM disks the folders are MasterDOS directories, so a disk made by MasterDOS works unchanged.

### File names

- **SAM disks** use names of up to 10 characters, as SAMDOS and MasterDOS do.
- **FAT drives** take long names of up to 64 characters (a longer name is cut to its first 64; a file made on a PC with a longer name can still be reached with a wildcard or its 8.3 name). Case is ignored when matching, and a file can be found by its long name or its short 8.3 name. A name can't start with `.` or contain `" * / : < > ? \ |`.
- **The whole name**, drive and path included, can be up to 255 characters, in ADOS's commands and in LOAD, SAVE, VERIFY and MERGE: `SAVE "D:\PICS\HOLIDAY SNAPS.SCR" SCREEN$`. The SAM ROM on its own refuses a LOAD or SAVE name over 14 characters. ADOS takes the longer ones on ROM 3.0; with an older ROM the limit stays at 14. Tape names stay at 10.

### Wildcards

`?` matches any one character and `*` matches the rest of the name. DIR, ERASE and COPY accept wildcards. SAVE, RENAME, OPEN DIR and FORMAT need a plain name and give error 89, "Bad file name", if the name has wildcards.

## Listing and moving around

### DIR

```
DIR                       the current directory
DIR "A:"                  another drive
DIR 1                     drive 1 (A:), as in SAMDOS and MasterDOS
DIR "D:\GAMES\"           what is in a folder
DIR "D:\GAMES\CHAOS.MGT\" what is in a disk image
DIR "*.BIN"               matching files only
DIR "A:CHAOS*"
```

To list what is in a folder or disk image, end the name with `\`. Without it the last part is a name to match, so `DIR "D:\GAMES"` lists just the `GAMES` entry.

`DIR 1` is drive A: and `DIR 2` is `D2:`: the BDOS card in use, or B: if there is none. `DIR 0` is the current drive, and any other number gives error 103, "No drive". The number must start with a digit: use `DIR "C:"` for the other drives.

On a SAM disk each line shows the entry number, name, size in sectors and file type, then a total such as `12 files, 640K free`. On a BDOS drive the heading also shows the record. As in MasterDOS, CODE files are shown as `C` with their start address and length, and a BASIC program that runs itself shows its auto-run line:

```
A:\
  1 INTRO       12 BASIC 10
  2 GAMES        1 <DIR>
  3 TITLE       49 SCREEN$
  4 MCODE       14 C 32768,6912
 4 files, 731K free
```

On a FAT drive the heading shows the full path, such as `D:\GAMES`. Each line shows the name and its size in bytes (or in K for files of 64K and over), or `<DIR>` for a folder, then the number of files. Hidden files and `.info` files aren't listed.

### DIR ?

```
DIR ?
```

Lists every file on the current SAM disk, whatever directory it is in. On a FAT drive it lists the current folder only.

### DIR =

```
DIR = "D:"                change drive
DIR = "D:\GAMES"          change drive and folder
DIR = "SAVES"             into a folder on the current drive
DIR = ".."                up one level
DIR = "\"                 to the root
DIR = "D:\GAMES\CHAOS.MGT"   into a disk image
```

Each drive remembers its own current directory. A missing folder gives error 116, "Dir not found". Programs can do the same through hook 145.

### PATH$

```
PRINT PATH$
```

Gives the current drive and path, for example `A:\UTILS` or `D:\GAMES\SAVES`. Inside a disk image it gives only the path inside the image (see [Not done yet](#not-done-yet)).

## Loading and saving

LOAD, SAVE, VERIFY and their forms (`CODE`, `SCREEN$`, `LINE`, and so on) are the ROM's own commands. ADOS supplies the file handling behind them, so they work the same on every drive:

```
SAVE "D:\WORK\GAME" LINE 10
LOAD "D:\WORK\GAME"
SAVE "A:PIC" SCREEN$
SAVE "D:DATA.BIN" CODE 32768, 1000
VERIFY "D:DATA.BIN" CODE
```

**Replacing a file.** SAVE replaces a file of the same name. A protected file (SAM) or a read-only file (FAT) isn't replaced, and gives error 109, "File name used". So does saving over a folder's name, or over a disk image that is mounted or that a drive is inside.

**Dates.** If a clock is fitted, SAVE stamps the file with the date and time. On a SAM disk the stamp goes where MasterDOS keeps it. On a FAT drive it is the file's date, the one a PC shows. With no clock, a SAM file has no date and a FAT file is dated 1 January 1980.

**Loading by number.** On a SAM disk (a floppy, a BDOS record, or a FAT drive inside a disk image) `LOAD n` loads the file that `DIR` lists as number n, as SAMDOS and BDOS do:

```
DIR
LOAD 3
```

The number is the file's entry in the directory, so a name that `DIR` shows in another folder, or a number with no file, gives error 107, "File not found". BASIC, CODE and SCREEN$ files load as `LOAD "name"` would. On a FAT drive outside an image there are no numbers, and you get "Bad device".

### Files on FAT drives

A file saved to a FAT drive holds only its data, so a PC can read it as it is. The SAM details (type, name, load and run addresses) go in a hidden file beside it with `.info` added to the name. For example, `GAME.BIN` gets `GAME.BIN.info`. ADOS keeps the two together: ERASE removes both and RENAME renames both.

A file with no `.info`, such as one copied from a PC, loads as CODE at 32768. Give an address to load it elsewhere:

```
LOAD "D:MUSIC.DAT" CODE 49152
```

The length always comes from the file's size on the card.

## Files and directories

### ERASE

```
ERASE "OLDFILE"
ERASE "D:\TEMP\*"
ERASE "A:TEST?"
```

Erases every matching file. Protected (SAM) and read-only (FAT) files are skipped. If nothing matches you get error 107, "File not found". ERASE skips a disk image that is mounted or that a drive is inside.

### RENAME

```
RENAME "OLD" TO "NEW"
RENAME "D:\WORK\DRAFT.TXT" TO "FINAL.TXT"
```

The new name stays on the same drive and in the same directory as the old one, so give only the name after TO. If the new name already exists you get error 109, "File name used". RENAME works on directories too. On a FAT drive the renamed file is dated with the time of the rename.

### OPEN DIR

```
OPEN DIR "GAMES"
OPEN DIR "D:\WORK\SAVES"
```

Makes a directory. On a SAM disk this is a MasterDOS directory. On a FAT drive it is an ordinary folder that a PC can use.

### ERASE DIR

```
ERASE DIR "GAMES"
```

Erases a directory. It must be empty, or you get error 117, "Dir not empty". On a FAT drive a hidden `.info` file still counts as something in the folder.

### RENAME DIR

```
RENAME DIR "GAMES" TO "OLDGAMES"
```

The same as RENAME. It's there for MasterDOS users.

## Copying

### COPY files

```
COPY "A:CHAOS" TO "D:\GAMES\CHAOS"      one file, new name
COPY "A:*" TO "D:\BACKUP\"              every file, same names
COPY "D:\PICS\*.SCR" TO "A:"            matching files
COPY "D:README.TXT" TO "D:\DOCS\"       same drive, another folder
```

COPY works between any two drives, SAM or FAT, in either direction. If the target's name is blank or has wildcards, each file keeps its own name. Directories are not copied.

While it copies, COPY shows each file's name and how far it has got, from 0% to 100%, on one line of the screen. The figure counts every sector read and written, so on a small file it can reach 99% early and then stay there until the file is finished.

As in MasterDOS, a file bigger than free memory is copied in several passes: COPY reads as much as fits, writes it, and goes back for the rest, so any file can be copied whatever its size. COPY needs at least 32K of free memory in one block (16K holds a copy of the DOS while it switches between the two files); with less it gives error 82, "No memory". The more memory is free, the fewer passes it takes.

The copy keeps the file's type and addresses. The copy is dated when it is made, as if it had just been saved. A PC file copied from one FAT drive to another gains a `.info`.

### COPY a whole disk

If one side is only a drive letter and that drive is a SAM disk, COPY copies the whole disk, every track and sector:

```
COPY "A:" TO "B:"                 floppy to floppy
COPY "A:" TO "D:\BACKUP.MGT"      floppy to a disk image (made if it isn't there)
COPY "C:" TO "D:\RECORD5.MGT"     the current BDOS record to a disk image
COPY "D:\BACKUP.MGT" TO "A:"      disk image to a floppy (it must be formatted)
```

The target is overwritten completely. The copy goes through 32K of free memory at a time, and shows how far it has got as a percentage. See [Disk Images](Disk%20Images.md#copying-whole-disks) for more.

## Disk images

### FORMAT

```
FORMAT "D:\WORK.MGT"
```

Makes a new, empty SAM disk image (819,200 bytes) on a FAT drive. If the name is already used you get error 109, "File name used". FORMAT doesn't format real floppies. Using it on a drive that isn't FAT gives error 91, "Bad device".

### OPEN "X:" TO

```
OPEN "B:" TO "D:\GAMES\CHAOS.MGT"
LOAD "B:chaos"
```

Gives a disk image its own drive letter. The letter must be free (not used by any drive), or you get error 91, "Bad device". You also get error 91 if all 8 drive letters are already in use. The mount lasts until CLOSE or reset.

### CLOSE "X:"

```
CLOSE "B:"
```

Unmounts a disk image. If the drive was the current drive, ADOS goes back to the default drive. CLOSE on a letter that isn't a mounted image gives error 91, "Bad device".

`OPEN #` and `CLOSE #` (streams) are still the ROM's.

## BDOS records

These work on a Trinity or Atom Lite holding BDOS records. On any other drive RECORD gives error 91, "Bad device", except as noted for FAT drives below.

### RECORD

```
RECORD
```

Lists the records that have names, with their numbers. A name is up to 16 characters, and ends early at a 0 byte: AL-BDOS ends names with zeros, BDOS pads them with spaces.

### RECORD n

```
RECORD 5
```

Selects record 5 on the current drive. You start in that record's root directory. A record that doesn't exist gives error 114, "No record".

### RECORD "name"

```
RECORD "GAMES DISK"
```

Selects a record by its name. Case is ignored.

### RECORD on a FAT drive

For users used to BDOS records, RECORD also works with disk images on a FAT drive:

```
RECORD "CHAOS.MGT"     the same as DIR = "CHAOS.MGT"
RECORD 0               leave the disk image
```

RECORD 0 can't leave a mounted image (error 91); use CLOSE instead.

### BDOS programs

Programs written for BDOS keep working: ADOS has BDOS's DOS variables and hook codes (see [Hook Codes](Hooks.md#bdos-compatibility)). BDOS had one card, its drive 2. In ADOS, drive 2 is the BDOS card used last, so with a card in the Atom and another in the Trinity, it follows whichever you used last. At boot it is the first one found. Each card keeps its own record.

`PEEK DVAR 25` is that card's current record, and `POKE DVAR 25,n` selects record n on it, as in BDOS. ADOS takes the new record the next time the card is used, from its root directory.

```
DIR "D:"                  D: is now the BDOS card that drive 2 and the DVARs mean
POKE DVAR 25,7: LOAD "D2:MENU"
```

## Booting

**F9** loads and runs the file whose name starts with `AUTO` on the current drive, in the current directory. If there isn't one, nothing happens.

**BOOT** does the same once ADOS is loaded, but reports error 101, "No AUTO file", if there isn't one.

The AUTO file must be BASIC, CODE or SCREEN$, or you get error 94, "Wrong type". BASIC programs run from their auto-start line, as with LOAD.

## Settings

ADOS keeps its settings in DOS variables, read with `PEEK DVAR n` and changed with `POKE DVAR n,value`. To keep a change for the next boot, save the DOS file again after changing it.

| DVAR | Setting | Default |
|------|---------|---------|
| 0 | Border flash while sectors are read or written. The colour is (DVAR 0 AND 7) AND the sector number. 0 turns the flash off | 7 |
| 1-45 | Laid out as in BDOS, for BDOS programs: DVAR 25 is the BDOS card's current record (see [BDOS programs](#bdos-programs)). The full list is in [Hook Codes](Hooks.md#bdos-compatibility) | |
| 7 | The version × 10 + 50 (63 is ADOS 1.3). Don't change it | 63 |
| 46 | Drive timeout, about 1.5 seconds per unit. A floppy drive with no disk gives "No disk" after it | 3 |
| 47 | The drive selected at boot. If there is no such drive, the first letter that has a drive is used | "A" |
| 48 | For machine code programs only: not 0 gives hook errors back to the program instead of reporting them. See [Hook Codes](Hooks.md#getting-errors-back-instead-of-a-basic-error) | 0 |

```
POKE DVAR 0,0             no border flash
POKE DVAR 47,CODE "C"     start on C: next time
```

Leave DVAR 48 at 0 in BASIC. While it is set, failed commands give no error message.

## Error messages

ADOS uses the same error numbers as SAMDOS, so programs that trap DOS errors keep working. Errors 114 onwards are new.

| No. | Message | Usual cause |
|-----|---------|-------------|
| 81 | Nonsense | A command's syntax is wrong, for example a missing TO |
| 82 | No memory | COPY has less than 32K of free memory in one block |
| 83 | Bad statement | Something extra after the command |
| 85 | Disk error | The drive reported a fault |
| 87 | No disk | No disk or card in the drive |
| 89 | Bad file name | A bad name, wildcards where they aren't allowed, or a name too long |
| 91 | Bad device | The wrong kind of drive for the command, or a letter already in use |
| 93 | Verify failed | VERIFY found a difference |
| 94 | Wrong type | The file isn't the type asked for |
| 101 | No AUTO file | BOOT found no `AUTO*` file |
| 103 | No drive | No drive has that letter |
| 104 | Protected | The disk, card or image can't be written to |
| 105 | Disk full | The drive is full |
| 106 | Dir full | No free directory entries |
| 107 | File not found | No file matches the name |
| 108 | End of file | A file ended sooner than expected |
| 109 | File name used | The name exists, or the file is protected |
| 113 | Bad hook | A program called a hook ADOS doesn't know |
| 114 | No record | The BDOS record doesn't exist |
| 115 | Not a SAM disk | The disk isn't in a format ADOS can read |
| 116 | Dir not found | No directory with that name |
| 117 | Dir not empty | ERASE DIR on a directory that still holds files |
| 118 | Fragmented | A disk image is in more than 12 pieces on the card |
| 119 | File not open | A stream hook with no file open (see [Hook Codes](Hooks.md)) |

## Not done yet

- **Other SAMDOS and MasterDOS commands.** Only the commands in this guide are in ADOS. Others, such as BASIC streams to a file (`OPEN #` to a file), aren't done yet. Machine code can read files as streams through the [hook codes](Hooks.md#streaming-a-file).
- **FAT12** drives can't be read.
- **Dates on FAT.** DIR doesn't show dates, and RENAME gives a FAT file the time of the rename. ADOS sets only the date a file was written, not the dates it was created and last opened.
- **Single-drive copies.** MasterDOS asks you to swap disks when you copy from one floppy to another in the same drive. ADOS doesn't, so copy through a second drive or a disk image.
- **Inside a disk image**, the DIR heading and PATH$ show only the path inside the image, not the FAT folder and image name.
