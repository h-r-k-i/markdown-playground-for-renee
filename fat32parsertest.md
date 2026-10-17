# fat32 (Which Also Does FAT12 And FAT16, Sort Of, If You Uncomment Them)

A read-only FAT12/FAT16/FAT32 driver that runs in userspace against a disk image.
It reads the MBR, finds the FAT partitions, works out which FAT each one is,
resolves an absolute 8.3 path, and copies that file out to `output1.bin`.

That's it. That's the whole project. It finds exactly one file, `/INFO/HOMOS.TXT`,
because that path is hardcoded in `main`. Change it. Or just fucking copy-paste
the source code and hardcode your own paths.

## Why

So SBL can read files off a disk eventually. Doing this inside a bootloader with no
`printf`, no debugger, and no heap would be miserable, so it gets written here first
with `std::cout` and then ported once it works. Some of the design follows from that:
there are no heap allocations in the parsing path, partitions are parsed as they're
found instead of being collected into a vector, and `Disk` only hands out one sector
at a time like a real disk driver would.

## Dependencies

- `g++` or `clang++` that accepts GNU extensions. The code uses variable-length arrays
  (`uint8_t sector[BytesPerSector]`) and designated initializers, which C++17 doesn't
  technically allow. Both compilers accept them without `-pedantic`.
- A disk image with an **MBR** partition table and at least one FAT partition on it.
  GPT disks and superfloppies (a FAT volume with no partition table) won't work. See
  [Making a disk image](#making-a-disk-image).

There's no Makefile because there's only one file to compile. God has spared us this once.

## Build & run

```sh
g++ -std=c++17 -Wall -Wextra fat32driver.cpp -o fat32driver
./fat32driver
```

It opens `fatdisk.img` in the current directory. That's hardcoded too.

Example output (there's a lot of blank lines):

```
FAT32 partition detected. Parsing...

Path name: INFO
Path name: HOMOS.TXT
End of path reached. Directory state: Found
File HOMOS.TXT found. Cluster: 4, Size: 18, Is Directory: No
File read successfully. Size: 18 bytes
```

The file contents end up in `output1.bin`.

To read a different file, image, or FAT type, edit `main`:

- **Image:** `Disk disk("fatdisk.img");` (a `vdisk.img` line is commented out above it)
- **Path:** the string passed to `ResolvePath`. It must be absolute and use 8.3 names.
- **FAT12 / FAT16:** their blocks are commented out. Uncomment them to read from
  those partitions too. Each one writes to the same `output1.bin`, so whichever runs
  last wins.

## Making a disk image

With `sfdisk`, `dosfstools`, and `mtools` you can do this without root or loop devices:

```sh
truncate -s 1G fatdisk.img
echo 'start=2048, type=c' | sfdisk fatdisk.img        # one FAT32 LBA partition at 1 MiB
mkfs.fat -F 32 --offset 2048 fatdisk.img              # format it in place
mmd   -i fatdisk.img@@1M ::/INFO
mcopy -i fatdisk.img@@1M somefile.txt ::/INFO/HOMOS.TXT
```

1 GiB is enough clusters for `mkfs.fat` to make a real FAT32 volume. Much smaller and
the cluster count drops below 65525, and the driver (correctly) decides it's FAT16.

To test all three types, make several partitions and format each one with `-F 12`,
`-F 16`, and `-F 32`. The `fatdisk.img` this was developed against has three:

```
fatdisk.img1 : start=        2048, size=      296960, type=6
fatdisk.img2 : start=      299008, size=      661504, type=6
fatdisk.img3 : start=      960512, size=     1136640, type=c
```

## How it works

1. **MBR.** Sector 0 is cast to an `MBR`. `PartitionEntry::isValid` accepts FAT partition
   types `0x01 0x04 0x06 0x0B 0x0C 0x0E`, and returns `81` for extended partitions
   (`0x05 0x0F 0x85`). Extended partitions are then skipped (see below).
2. **Which FAT.** The partition's first sector is cast to a `VBR`, and
   `VBR::GetFATType` computes the data cluster count the way the spec says to:
   `< 4085` is FAT12, `< 65525` is FAT16, everything else is FAT32. The partition type
   byte and the `"FAT32   "` label aren't used, because they lie.
3. **Geometry.** The `FAT12` / `FAT16` / `FAT32` constructor reads the BPB again and
   computes `FATStartLBA`, `RootDirectoryLBA`, `DataStartLBA`, and `ClusterCount`. If
   the cluster count doesn't match the class, it sets `BytesPerSector = 0` and returns.
   FAT32 also respects `ExtendedFlags` bit 7 (mirroring off) and uses the active FAT
   that it names.
4. **Path.** `ResolvePath` splits the path on `/` and calls `FindInDirectory` once per
   component, starting from the root.
   - The FAT12/16 root directory is a fixed run of sectors after the FATs, so it is
     read directly.
   - Everything else, including the FAT32 root directory, is a cluster chain. It is
     walked with `GetNextCluster` until end-of-chain, with a visit counter capped at
     `ClusterCount` so that a looped FAT can't hang it.
   - `WalkDirectory` converts the name to padded 11-byte 8.3 form and compares it
     against each 32-byte entry. It skips deleted entries (`0xE5`), `.` and `..`,
     volume labels, and LFN entries, and stops at the first `0x00`.
5. **FAT entries.** Each subclass implements `GetFATEntryLocation`, `GetNextCluster`,
   `IsEndOfChain`, and `IsBadCluster`:

   | | Entry size | Bad | End of chain | Notes |
   | --- | --- | --- | --- | --- |
   | FAT12 | 12 bits | `0xFF7` | `> 0xFEF` | Entries are 1.5 bytes and can straddle two sectors, so it reads two |
   | FAT16 | 16 bits | `0xFFF7` | `> 0xFFEF` | |
   | FAT32 | 28 bits | `0x0FFFFFF7` | `> 0x0FFFFFEF` | Top 4 bits masked off |

6. **Read.** `ReadFile` follows the file's cluster chain and copies sector by sector into
   the caller's buffer until it has `FileSize` bytes.

## Known rough edges

- **Bad clusters count as end of chain.** All three `IsEndOfChain`s return true for the
  bad-cluster value (`0xFF7 > 0xFEF`, and so on). The `IsBadCluster` check inside
  `FindInDirectory`'s loop is never reached, because the loop condition exits first.
  So a bad cluster, or a failed FAT read (which returns the bad value), makes a
  directory look like it simply ended. You get `NotFound` instead of `Bad`. The
  thresholds also include the reserved range (`0x?FF0`–`0x?FF6`), which isn't strictly
  end-of-chain either.
- **`Disk` always reads 512 bytes.** `SetSectorSize` exists but nothing calls it. On a
  volume whose BPB says 4096-byte sectors, the `BytesPerSector`-sized buffers are only
  partly filled, and every LBA computed from the BPB points at the wrong place.
- **Extended partitions are skipped.** `main` has `// todo: extended partition parsing`.
  `ExtendedPartitionEntry::parse` and `::get` are written and never called.
- **MBR only.** A GPT protective MBR (type `0xEE`) fails `isValid`, so nothing is found.
  The `0x55AA` boot signature is never checked, and `main` ignores `getSector`'s return
  value.
- **Short names only.** There's no LFN support, so `I was in a dream.txt` has to be
  requested as `/IWASIN~1.TXT`. A first byte of `0x05` (an escaped `0xE5`, for Kanji
  names) isn't translated, and `.` / `..` can't appear in a path.
- **Everything in `main` is hardcoded:** the image, the path, the output file, and
  which FAT types are enabled.
- `if (buffer == nullptr)` after `new uint8_t[...]` never fires, because `new` throws.
- `GetNextCluster` prints `Warning: File is fragmented` to stderr on every non-contiguous
  hop (it's marked "remove in prod"). The FAT32 version compares the raw entry before
  masking, so the reserved top 4 bits can trigger the warning on a file that isn't
  fragmented.
- `ResolvePath` prints `Path name: ...` for every component. Also debug.
- `isValid()` returns an `int` so that it can return `81`, which is truthy, so
  `if (a.isValid() == false)` still does the right thing. By accident or not, nobody knows.
- `FSInfo` is fully defined and `static_assert`ed and never read. It's there "for
  completeness".
- `-Wall -Wextra` gives one `-Wsign-compare` warning in `FAT12::GetFATEntryLocation`.

## Files

| File | What |
| --- | --- |
| `fat32driver.cpp` | The whole driver: MBR/VBR structs, the `FAT` base class, `FAT12`/`FAT16`/`FAT32`, and `main`. |
| `disk.hpp` | `Disk`, a one-sector-at-a-time `std::ifstream` wrapper standing in for a real disk driver. |
| `fatdisk.img` | 1 GiB test image with FAT12/FAT16/FAT32 partitions. Gitignored. |
| `vdisk.img` | 10 GiB older test image. Gitignored. |
| `output1.bin`, `output.bin` | Files pulled out by earlier runs. Gitignored. |
| `fat32out` | Saved stdout from a run. Gitignored. |

## References

- [OSDev Wiki: FAT](https://wiki.osdev.org/FAT), which the struct comments are mostly paraphrasing
- [Wikipedia: FAT](https://en.wikipedia.org/wiki/File_Allocation_Table). Yes. Fucking Wikipedia. Fight me.
- [OSDev Wiki: MBR](https://wiki.osdev.org/MBR_(x86))

## License

No `LICENSE` file yet. The other repos are MIT, and this one will be too once I remember
to add one, but there are better repos to use.
