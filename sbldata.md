# SBL Bootloader Map

## Structure

The SBL bootloader struct appears somewhat like this:

```c
struct SBL_Data {
    uint64_t magic;
    uint64_t checksum;
    uint64_t flags;
    uint64_t sgalf;
    uint64_t first;
}
```

Each node appears as follows:
```c
struct node_t {
    uint16_t data;
    uint16_t next;
}
```

## `SBL_Data`
#### `magic`
`magic` is a value that should always be set to `0x4E454B4F00000000` (OKEN). This is one of several checks to ensure the bootloader properly produced the map.

#### `checksum`
`checksum` is a value that that is set such that the sum of `magic`, `flags`, `sgalf`, and `checksum` is equal to `0xFFFFFFFFFFFFFFFF`.

#### `flags`/`sgalf`
The flags set by the bootloader are as follows:
```
0: Memory map
1: Reserved
2: Text framebuffer
3: Video framebuffer
4: Protected Mode
5: Random
6: Floating-Point
7: ACPI
8: SMBIOS
9: Reserved
10: Bootloader
11: Topology
12: GDT
13: VM
14: Reserved
15: Reserved
16: RTC
17: Display
18: Serial
19-62: Reserved
63: Identity
```

#### `first`
Due to the nature of pointers in long mode, this value is assumed to be a 64-bit pointer. This value corresponds effectively to `node_t *` and points to the first node of the bootloader map.

## `node_t`
#### `data`
Depending on the flag that this corresponds to and the antiflag value, this value may either correspond to a 64-bit pointer or a 64-bit data value.

#### `next`
This is a 64-bit pointer pointing towards the location of the next node. If this is set to 0 then this is the last node.

## `flags`/`sgalf`
The flag and antiflag values in the header may be more specifically listed as follows:

#### `flags`
```
0: Memory map present
2: Text framebuffer info available
3: Video framebuffer info available
4: Protected Mode data available
5: Entropy data available
6: Floating-Point data available
7: ACPI data available
8: SMBIOS data available
10: Bootloader memory map data present
11: CPU topology data present
12: GDT data present
13: Virtual machine data available
16: Real time clock data available
17: Monitor data available
18: Serial bus data available
63: Bootloader identity data available
```

#### `sgalf`
```
0: Error obtaining a memory map
2: Error getting text framebuffer data
3: Error getting video framebuffer data
4: Error producing protected/long mode data
5: Error getting entropy data
6: Error getting floating-point data
7: Error getting ACPI data
8: Error getting SMBIOS data
10: Error loading bootloader memory map
11: Error getting CPU topology data
12: Protected mode not available and therefore no GDT is set up
13: Error getting virtual machine data
16: Error getting RTC data
17: Error getting monitor data
18: Error getting serial bus data
63: Error loading bootloader identity data
```
The bootloader will produce nodes from the least significant bit towards the most siginificant bit.

### More in-depth information on each node:
#### Memory map

##### flags/sgalf

| flag | galf | Meaning                               |
|------|------|---------------------------------------|
| 0    | 0    | Failure state                         |
| 0    | 1    | Memory map obtained via 88h           |
| 1    | 0    | Memory map obtained via E820h         |
| 1    | 1    | Memory map obtained via E801h         |

##### Information
This will always be the first node of the header list. The node's `data` field points to the location in memory where the memory map was written, if one was written. The flag/antiflag pair, uniquely, states which memory mode was employed rather than encoding a standard success/error state. In a failure state, any other values the bootloader has produced should be ignored; the bootloader is designed to fail outright if it cannot produce a memory map, and should not be able to pass control on to a third-stage bootloader or kernel.

If the memory map was produced using the modern E820h method, the memory map shall appear as follows:
```
[data]: 0xE820
[data + 2]: The count of the entries within the memory map.
[data + 4]: System map entries, 24-byte aligned
```
where each system map entry appears as follows:
```
[entry]: Base address
[entry + 8]: Length of map in bytes
[entry + 16]: Type of address
[entry + 20]: Reserved
```

If the memory map was produced using the legacy E801h method, the memory map shall appear as follows:
```
[data]: 0xE801
[data + 2]: Extended memory between 0x100000 and 0x1000000, in 1024-byte blocks
[data + 4]: Extended memory above 0x1000000, in 65536-byte blocks
[data + 6]: Configured memory between 0x100000 and 0x1000000, in 1024-byte blocks
[data + 8]: Configured memory above 0x1000000, in 65536-byte blocks
```

If the memory map was produced using the legacy 88h method, the memory map shall appear as follows:
```
[data]: 0x0088
[data + 2]: Size of memory above 0x100000, in 1 KB pages
```

The caller is obligated to manually seek through the memory map in order to find the values needed.

#### Text framebuffer
##### flags/sgalf
not configured
##### Information
not configured

#### Video framebuffer
##### flags/sgalf
not configured
##### Information
not configured

#### Protected Mode
##### flags/sgalf
not configured
##### Information
not configured

#### Random
##### flags/sgalf
not configured
##### Information
not configured

#### Floating-Point
##### flags/sgalf
not configured
##### Information
not configured

#### ACPI
##### flags/sgalf
not configured
##### Information
not configured

#### SMBIOS
##### flags/sgalf
not configured
##### Information
not configured

#### Bootloader
##### flags/sgalf
not configured
##### Information
not configured

#### Topology
##### flags/sgalf
not configured
##### Information
not configured

#### GDT
##### flags/sgalf
not configured
##### Information
not configured

#### VM
##### flags/sgalf
not configured
##### Information
not configured

#### RTC
##### flags/sgalf
not configured
##### Information
not configured

#### Display
##### flags/sgalf
not configured
##### Information
not configured

#### Serial
##### flags/sgalf
not configured
##### Information
not configured

#### Identity
##### flags/sgalf
| flag | galf | Meaning                               |
|------|------|---------------------------------------|
| 0    | X    | Failure state                         |
| 1    | 0    | Identity node successfully produced   |
| 1    | 1    | Failure state                         |
##### Information
This will always be the last string of the 
