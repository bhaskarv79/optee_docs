# Power loss and maintenance

Why every operation is atomic across a power cut, what is left behind and how the file system maintains itself.

Part of the [SPI-NOR secure storage low-level design](../README.md).

## Power loss and atomicity

A change is all-or-nothing because nothing committed is overwritten: new blocks, nodes and heads go to the inactive version, and only the dirf.db head (then the anchor) switches to them. A cut before the dirf.db head write leaves the previous state; a cut after it leaves the new one.

| Cut during | State after reboot | Why |
| --- | --- | --- |
| object block or node writes | old object | new items are in the inactive versions; the dirent still holds the old root hash |
| object head write | old object | the object is opened by the dirent's root hash, which matches the old root node version |
| after the object sync, before the dirf.db commit | old object | dirf.db was not changed; the new items are unreferenced |
| dirf.db node writes | old state | same rule, one level up |
| dirf.db head write | old state, on SPI-NOR FS | a torn record fails its CRC, so the slot keeps its previous head (counter − 2) and the other slot's head (counter − 1) wins; on REE FS this relies on the REE file system writing the 68-byte head atomically |
| after the dirf.db commit, before the anchor raise | new state, anchor one behind | the image from just before this commit is accepted until the next commit raises the anchor |
| create, before `set_name()` commits | object absent | file N exists but no dirent names it; its file number is reused by a later create, which drops it (`be->create()`) |
| remove, after the dirf.db commit | object absent | the file is unreferenced; REE FS leaves it on disk, SPI-NOR FS keeps its records until the file number is reused |
| truncate | old or new length | on topic/spinor the blocks past the new end are dropped only after the commit |

### SPI-NOR store specifics

| Cut during | Effect at mount |
| --- | --- |
| a record program | bad CRC: the record is ignored and its sector is closed for appending; the item keeps its previous record |
| a data sector program, before its record | no record references the sector: it is free |
| compaction, while copying records | both copies are valid; the copy with the higher `seq` wins |
| compaction, after the header is cleared | the sector reads as free and is erased before reuse |
| an erase | header already 0: free |
| a `DEL` or `TRUNC` marker | marker ignored: the file or blocks are still there, as before the call |

The store self test (xtest 1001 with `CFG_SPI_NOR_SIM=y`) covers torn records, log wrap-around, compaction past items written once, two blocks on one sector, and fill then dirf.db update and removal. Power cut at an arbitrary point is not yet tested on QEMU.

## Maintaining the file system

Neither hash-tree file system has a format step, a background task or a checker. Everything happens on the first storage call after boot and inside the operations themselves; leftovers from power cuts are reclaimed when their space or file number is reused.

### First use after boot

1. Boot: tee_fs_init_key_manager() derives the SSK. If that fails, the log says "Secure storage will not be available" and every FEK wrap fails.
2. First storage call on the instance: get_dirh() calls fs->open_dirh().
3. SPI-NOR FS only: fs_store_ready() → find_region() takes the optee-fs partition (from the DT or a GPT), else CFG_SPI_NOR_FS_OFFSET / CFG_SPI_NOR_FS_SIZE, which is then registered as partition optee-fs. If the DT declares optee-fs but no probed flash has it, storage is unavailable: the store never moves to another region.
4. sfs_init() checks the region: the erase size divides 4 KiB, the region is 4 KiB aligned, and it has 8 to 65,535 sectors (records hold a 16-bit sector number).
5. sfs_mount() → scan() reads the 16-byte header of every sector, replays the log sectors in log sequence order and rebuilds the index in RAM.
6. spinor_select_anchor(), once per boot, then the anchor is read.
7. tee_fs_dirfile_open() checks dirf.db (see [Check order](integrity.md#check-order-at-open-and-read)) and builds the bitmap of used file numbers.

dirf.db then stays open while any object is open, through the cached handle.

### Format and reset

- No mkfs: a region with no valid log header mounts as empty, every sector free. Whatever it held is erased sector by sector as space is allocated, so foreign data in the region is taken over silently. Only the partition table keeps the store apart from other data.
- The first dirf.db is created with its head counter starting at the anchor's value (create_and_sync() with min_counter); the other head slot gets a dummy head.
- dirf.db missing while the anchor is above 0 fails the mount, unless CFG_SPI_NOR_FS_ALLOW_RESET (REE FS: CFG_REE_FS_ALLOW_RESET) is set; then a new, empty dirf.db starts at the anchor's value.
- Factory reset is erasing the partition. Objects cannot be recovered from it afterwards: their FEKs are wrapped by keys that exist only on the device, and the dirents that name them are gone.

### File numbers and names

- tee_fs_dirfile_get_tmp() takes the lowest clear bit of the bitmap; tee_fs_dirfile_remove() clears it.
- REE FS: the file name is the file number in hex under CFG_TEE_FS_PARENT_PATH (for example /var/lib/tee/1f); dirf.db is named dirf.db.
- SPI-NOR FS: the file number is the file field of each record; dirf.db is 0x7fffffff (SFS_DIRF_FILE), and object numbers must stay below it.
- TEE_RenamePersistentObject() changes only the oid of the dirent; the file and its hash tree stay as they are.

### Leftovers and how they are reclaimed

| Leftover | Cause | Reclaimed by |
| --- | --- | --- |
| Unreferenced object file | power cut after the object was written but before set_name() committed, or after a remove committed but before be->remove() | reuse of its file number: REE FS creates with O_CREAT \| O_TRUNC in tee-supplicant; SPI-NOR FS be_create() appends a DEL marker for that number first |
| Inactive versions | each item keeps its version from before the last commit | never: they are the other half of the two-version scheme. A REE file is up to twice its data; on SPI-NOR an object block holds up to two data sectors |
| Superseded records | every write replaces the record with the same key | SPI-NOR FS: the old data sector is free at once (set_item()); log space comes back through compact() |
| Blocks past a truncate | TEE_TruncateObjectData() | REE FS: ftruncate() after the commit; SPI-NOR FS: a TRUNC marker |
| Duplicate file number in two dirents | corruption | cleared in the later dirent at dirf.db open |

### Space and RAM on SPI-NOR

| Item | Flash used |
| --- | --- |
| Data block | one 4 KiB sector + a 28-byte record |
| Node | a 96-byte record (28-byte header + 66, padded to 4) |
| Head | a 96-byte record (28 + 68) |
| DEL or TRUNC marker | a 28-byte record |
| Log sector | a 16-byte header, then 4,080 bytes of records |
| Reserve | 2 sectors for compaction, plus 2 more that object data cannot take |

In RAM the store keeps one 28-byte struct sfs_item per live item (grown 32 at a time, searched linearly), 1 state byte and a 4-byte log sequence number per sector, a 4 KiB I/O buffer and a 96-byte record buffer. A 1 MiB partition is 256 sectors; an object that fits in one block uses at most two data sectors, so such a partition holds roughly 120 small objects.

### Wear and limits

- Sectors are taken round-robin from s->cursor to spread erases, but sfs_mount() resets the cursor to 0, so on a device that reboots often the first sectors of the region get more erases. Erase counts are not stored.
- A small TA write costs two sector programs and up to two erases: the object's block and dirf.db's block (see the [worked example](spinor-store.md#worked-example-one-write-on-spi-nor)). A NOR sector is typically rated for about 100,000 erase cycles.
- Record and log sequence numbers are 32-bit and never wrap: at UINT32_MAX the store returns TEE_ERROR_STORAGE_NO_SPACE for good. At six records per small write, that is roughly 700 million writes.
- The dirf.db head counter grows by 1 per commit (uint32_t), and the anchor follows it; RPMC counters are 32-bit too.

