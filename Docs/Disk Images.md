# SAM disk images on FAT

A FAT16 or FAT32 drive can hold SAM floppy disk images. There are two ways to use one:

- **As a folder.** Go into the image with a path, as with any folder. While inside it, the drive works as a SAM floppy, with the same directory, file layout and limits as a real disk.
- **Mounted on its own drive letter.** `OPEN "B:" TO "D:GAMES.MGT"` makes B: a SAM floppy until you unmount it. This suits software that only uses certain drives, and games on more than one disk.

Software that needs a SAMDOS or MasterDOS disk runs from an image unchanged either way.

## Which files are images

An image is any file of exactly 819,200 bytes, whatever its extension (`.MGT`, `.DSK`, or none). It holds the raw 800K disk: 80 tracks, 2 sides, 10 sectors of 512 bytes, in MGT order (track 0 side 1, track 0 side 2, track 1 side 1, and so on). ADOS does not read EDSK or SAD files.

An image doesn't need to sit in one piece on the card, but it must be in no more than 12 pieces. A file copied to a freshly formatted card is almost always in one piece. A badly fragmented image gives error 118, "Fragmented". To fix it, copy the image to another drive and back, or defragment the card on a PC.

An image that is read-only on the FAT drive can't be written to. Trying gives error 104, "Protected".

## Making a blank image

```
FORMAT "D:\WORK.MGT"
```

FORMAT makes a new, empty SAM disk image (800K of zeros, which is an empty SAMDOS disk). If a file with that name already exists, you get "File name used". FORMAT only makes images on FAT drives; it doesn't format real floppies.

## Paths

The same path rules apply on every drive:

| Path | Meaning |
|------|---------|
| `\` or `/` at the start | The root of the drive. On a FAT drive this is outside any image. |
| `..` | Up one level. `^` also works. |
| `GAMES.MGT` | Into the image, as if it were a folder. |

An image can appear anywhere in a path:

```
DIR = "D:\GAMES\CHAOS.MGT"        the drive is now the SAM disk in CHAOS.MGT
DIR                               lists the SAM disk
DIR = ".."                        back to D:\GAMES
LOAD "D:\GAMES\CHAOS.MGT\chaos"   load from the image without going into it
COPY "D:\README.TXT" TO "D:\WORK.MGT\README"     into an image on the same drive
COPY "D:\WORK.MGT\SCORES" TO "D:\SCORES.BIN"     and back out
```

MasterDOS folders inside an image work as on a floppy (`OPEN DIR`, `DIR = "SUB"`, `..`). `..` from the image's root goes back to the FAT folder that holds the image.

`RECORD "CHAOS.MGT"` on a FAT drive does the same as `DIR = "CHAOS.MGT"`, and `RECORD 0` leaves the image. This is for users used to BDOS records.

A drive stays inside its image until you leave it or reset.

Inside an image, or on a mounted drive, `DIR` numbers the files and `LOAD n` loads one by its number, as on a SAMDOS or BDOS disk: `DIR = "D:\GAMES\CHAOS.MGT"`, then `DIR`, then `LOAD 2`.

## Mounting an image on a drive letter

```
OPEN "B:" TO "D:\GAMES\CHAOS.MGT"   B: is now the SAM disk in CHAOS.MGT
LOAD "B:chaos"
CLOSE "B:"                          unmount it
```

- **The letter must be free.** A letter that belongs to a drive or partition gives error 91, "Bad device". A letter is free when no drive has it, for example B: when there is no second floppy drive.
- **Up to 8 drive letters** can be in use at once, counting mounts.
- **A mounted drive stays inside its image.** `..` and `\` take you to the image's root, never out to the FAT drive. RECORD 0 can't unmount it; use CLOSE.
- **A mount lasts until CLOSE or reset.** It isn't saved.
- **The image can't be erased or replaced while mounted.** ERASE skips it, the way it skips read-only files, and SAVE over it gives "File name used". The same applies to an image a drive has been left inside with `DIR =`. RENAME is still allowed.
- **Changing disks.** For a game on two disks, use `CLOSE "B:"`, then `OPEN "B:" TO "D:GAME2.MGT"`.

## Copying whole disks

COPY copies a whole disk when one side is only a drive letter and that drive is a SAM disk: a floppy, the current BDOS record, or a FAT drive that is inside an image.

```
COPY "A:" TO "D:\BACKUP.MGT"      floppy to a new image (made if it isn't there)
COPY "C:" TO "D:\RECORD5.MGT"     the current BDOS record to an image
COPY "D:\BACKUP.MGT" TO "A:"      image back to a floppy (it must be formatted)
COPY "A:" TO "B:"                 floppy to floppy
```

Every track and sector is copied, so the target is overwritten completely. If the target image already exists it is reused, but a file of another size with that name gives "File name used". A copy goes through 32K of free memory at a time, so it isn't limited by how much memory is free.

To copy one image file to another, name both images: `COPY "D:OLD.MGT" TO "E:NEW.MGT"`. Neither side is a bare drive letter, so this is an ordinary file copy. Like any big file it is copied in several passes, and the new file gets a `.info`.

## Not done yet

- The DIR heading and PATH$ inside an image show only the SAM path (`D:\SUB`), not the FAT folder and image name.
