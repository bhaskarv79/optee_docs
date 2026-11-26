# QEMU reference platform

How the SPI-NOR stack runs on QEMU `virt` (Armv7, `vexpress-qemu_virt`) with a
secure SPI-NOR flash: the QEMU side, the OP-TEE platform code, the flash image
made by the build repository, and what the boot log shows. This is the
reference for porting to a real board.

Part of the [SPI-NOR secure storage low-level design](../README.md). Code:
`core/arch/arm/plat-vexpress/spi_nor_qemu.c`, `conf.mk` (`CFG_QEMU_SPI_NOR`);
build repository `qemu.mk`, `common.mk`, `qemu-spinor-image.sh`.

## Hardware model

| Item | QEMU `virt,spi=secure` |
| --- | --- |
| SPI controller | ARM PL022, secure only |
| Chip select | PL061 GPIO, pin 0, secure only |
| Flash | `w25r256jv` (Winbond, ID ef 40 19), 32 MiB, SFDP, RPMC (JESD260) |
| Flash content | `-drive if=mtd,index=0,file=spinor.img,format=raw` |
| RPMC state (root keys, counters) | `-drive if=mtd,index=1,file=spinor-rpmc.img,format=raw` |

| Block | Base address in OP-TEE (`spi_nor_qemu.c`) | Size |
| --- | --- | --- |
| PL022 | `0x090c0000` (`QEMU_SPI_BASE`) | `PL022_REG_SIZE` |
| PL061 | `0x090d0000` (`QEMU_SPI_GPIO_BASE`) | `PL061_REG_SIZE` |

These match the QEMU `spinor-v10.0` branch. The version prepared for
qemu-devel (`rpmc-m25p80`, on QEMU master) moved them to `0x090d0000` and
`0x090e0000` because newer QEMU uses `0x090c0000` for ACPI PCI hotplug; the
OP-TEE side must follow whichever is merged (RSK-04).

Both blocks are mapped `MEM_AREA_IO_SEC` with `register_phys_mem_pgdir()`.
Without `spi=secure` QEMU has no such devices, and OP-TEE prints "QEMU
SPI-NOR: no flash, is QEMU run with spi=secure?".

## Initialization (`qemu_spi_nor_init()`, `driver_init` level)

| Step | Detail |
| --- | --- |
| 1. GPIO | `pl061_init()`, `pl061_register()` |
| 2. SPI controller | PL022 with 24 MHz input clock, 12 MHz bus, SPI mode 0, 8-bit frames, chip select by GPIO pin 0 driven automatically per transfer (`PL022_CS_CTRL_AUTO_GPIO`); `pl022_init()`, `configure()` |
| 3. spi-mem | `spi_mem_legacy_get_dev()` wraps the PL022 `spi_chip` |
| 4. Flash | `spi_nor_probe()`: RDID, SFDP, EN4B (32 MiB needs 4-byte addresses) |
| 5. Partitions | `spi_nor_gpt_add_partitions()`; then `set_ns_access()` by label. If the flash has no GPT, or `CFG_SPI_NOR_GPT=n`, `add_partitions()` registers the built-in table instead |
| 6. Report | `report()` lists the flash and every partition with its content |

SPI-NOR FS mounts later, at the first storage call, on the `optee-fs`
partition.

## Flash layout

| Offset | Size | Partition | Normal world | Content |
| --- | --- | --- | --- | --- |
| `0x0000000` | 17 KiB | (GPT) | none | protective MBR, primary GPT header, 128 entries |
| `0x0100000` | 1 MiB | `boot` | read-only | U-Boot as a uImage "U-Boot (bl33)" |
| `0x0200000` | 8 MiB | `kernel` | read-only | Linux zImage as a uImage "Linux zImage" |
| `0x0a00000` | 6 MiB | `ns-data` | read-write | erased |
| `0x1000000` | 1 MiB | `optee-fs` | none | SPI-NOR FS (256 sectors) |
| `0x1ffbe00` | 16.5 KiB | (GPT) | none | backup entries and header |

The built-in table (`CFG_QEMU_SPI_NOR_{BOOT,KERNEL,NS,FS}_{OFFSET,SIZE}`) has
the same four partitions; a size of 0 leaves one out. The first MiB is kept
free for the GPT. On QEMU the normal-world access of GPT partitions is set by
name from the built-in table, since GPT attributes are not used.

## Flash image (build repository)

`SPINOR=flash` (the default on the build `spinor` branch) makes `make run`
call `qemu-spinor-image.sh` once:

| Output in `out/bin` | Content |
| --- | --- |
| `spinor.img` | 32 MiB, `0xff` filled, GPT with 512-byte blocks, the two uImages |
| `spinor-rpmc.img` | 4 KiB of zeros: RPMC state, no root keys yet |
| `spinor-gpt-main.bin` | bytes 0 to 17,407: MBR, primary header, entries |
| `spinor-boot.uimg`, `spinor-kernel.uimg` | partition contents |
| `spinor-gpt-backup.bin` | last 16,896 bytes: backup entries and header |
| `spinor-layout.txt` | offset, size, partition and file of each piece, for a programmer that writes regions |

GPT details: type GUID `0fc63daf-8483-4772-8e79-3d69d8477de4` (Linux
filesystem data), partition and disk GUIDs from `uuid5` of the partition name
so the image is reproducible, revision 1.0, 92-byte headers, 128 entries of
128 bytes, CRC32 over header and entries. `sgdisk -v` reports no problems.

Existing images are kept, so objects and counters survive reboots;
`make spinor-clean` deletes them. An old image without "EFI PART" at offset 512
is replaced together with its RPMC state.

| `SPINOR=` | OP-TEE options | QEMU |
| --- | --- | --- |
| `n` | none | plain `virt,secure=on` |
| `sim` | `CFG_DRIVERS_SPI_MEM CFG_DRIVERS_SPI_NOR CFG_SPI_NOR_SIM`, 1 MiB simulated flash, `CFG_SPI_NOR_FS` on 512 KiB | plain `virt,secure=on`; storage lost at every boot |
| `flash` | `CFG_QEMU_SPI_NOR CFG_SPI_NOR_FS CFG_SPI_NOR_RPMC CFG_RPMC_WRITE_ROOT_KEY CFG_SPI_NOR_NS_ACCESS CFG_SPI_NOR_PTA` | `virt,secure=on,spi=secure` with the two `-drive if=mtd` images |

`CFG_RPMC_WRITE_ROOT_KEY=y` lets the first boot provision the RPMC counters
of the fresh image, as a factory would.

## Boot log

From a boot after the storage has been used (on the very first boot `optee-fs` shows "erased"):

```
I/TC: SPI-NOR: ef 40 19, 32768 KiB, SFDP
I/TC: SPI-NOR: primary GPT, 512-byte blocks, 4 of 128 entries
I/TC: QEMU SPI-NOR: Winbond ef 40 19, 32 MiB, GPT partitions:
I/TC:   boot      0x00100000   1024 KiB  ns:ro   uImage "U-Boot (bl33)"
I/TC:   kernel    0x00200000   8192 KiB  ns:ro   uImage "Linux zImage"
I/TC:   ns-data   0x00a00000   6144 KiB  ns:rw   erased
I/TC:   optee-fs  0x01000000   1024 KiB  secure  SPI-NOR FS
```

`describe()` names a partition's content from its first 64 bytes:

| Content | Recognized by |
| --- | --- |
| erased | all `0xff` |
| uImage "name" | big-endian `0x27051956`, name at offset 32 |
| FDT/FIT, size | big-endian `0xd00dfeed` |
| Linux zImage | little-endian `0x016f2818` at offset `0x24` |
| SPI-NOR FS | little-endian `0x4c464e53` (a log sector) |
| data | anything else |

`optee-fs` shows "erased" until the first object is stored. The uImage name
comes from the flash; on a product where the normal world can write that
partition it is untrusted text.

## Porting to a board

| QEMU piece | Board equivalent |
| --- | --- |
| `spi_nor_qemu.c` probe code | A DT node `jedec,spi-nor` under a DT SPI controller (PR 2), or platform code like this file |
| PL022 + PL061 | The board's controller: a `spi_chip` driver through the adapter, or a native spi-mem driver |
| Built-in table / GPT | Secure DT `partitions` or a GPT written at the factory |
| `ns_access` by label | `optee,ns-access` in the secure DT |
| `CFG_RPMC_WRITE_ROOT_KEY=y` | Only in the provisioning build |
| `-drive if=mtd` images | The real flash |
