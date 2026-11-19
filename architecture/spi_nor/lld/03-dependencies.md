# 3. Dependencies

## 3.1 Code bases

| Component | Version or branch | What the work needs from it |
| --- | --- | --- |
| optee_os | upstream `master` `f77757010` | Base of every topic branch. Existing parts reused unchanged: `fs_htree.c` (one change in PR 4), `fs_dirfile.c`, `tee_fs_key_manager.c`, `huk_subkey.c`, `nv_counter.c` weak hooks, the DT driver framework, `pl022_spi.c`, `pl061_gpio.c` |
| optee_test | `bhaskarv79/optee_test` `spinor` (2 commits on the manifest pin) | 6000 storage tests on `TEE_STORAGE_PRIVATE_SPI_NOR`; core self tests through `xtest 1001` |
| optee_client | upstream | Nothing new: SPI-NOR FS needs no tee-supplicant. REE FS still uses it. |
| build | `bhaskarv79/build` `spinor` | `SPINOR=n\|sim\|flash`, the flash image script, QEMU arguments |
| manifest | `bhaskarv79/manifest` `spinor` | `spinor.xml`, `spinor-topic-*.xml` |
| QEMU | `bhaskarv79/qemu` `spinor-v10.0` (demo), `rpmc-m25p80` (for qemu-devel) | `-machine virt,spi=secure`: a secure PL022 with a PL061 chip select and an RPMC-capable m25p80 flash (w25r256jv, 32 MiB); `-drive if=mtd` index 0 for the flash, index 1 for the RPMC state |
| TF-A, U-Boot, Linux, Buildroot | as in the upstream QEMU manifest | Nothing new |

## 3.2 Order of the pull requests

| PR | Depends on | Can merge in parallel with |
| --- | --- | --- |
| PR 0 pl022-trace | master | everything |
| PR 1 spinor | master | PR 0 |
| PR 2 dt | PR 1 | PR 3, PR 4 |
| PR 3 gpt | PR 1 | PR 2, PR 4 |
| DT/GPT hook commit | PR 2 and PR 3 | goes with the second of them |
| PR 4 rpmc | PR 1 | PR 2, PR 3 |
| QEMU platform | PR 1, PR 4 (and PR 2/3 for its partitions); QEMU patches merged | |
| NS access + PTA | PR 1, PR 2 (`optee,ns-access`) | |

Each child PR is opened against master as a draft that says "Depends on #N",
or after its parent is merged.

## 3.3 Specifications

| Specification | Used for | Where |
| --- | --- | --- |
| GlobalPlatform TEE Internal Core API, Trusted Storage | TA-facing API and storage IDs | unchanged libutee |
| JEDEC JESD216 (SFDP) | Flash size, erase types, page size, address modes | `sfdp.c` |
| JEDEC JESD260 (RPMC) | Root key, HMAC key, counter read and increment; SFDP RPMC parameter table | `rpmc.c`, `rpmc_counter.c` |
| UEFI 2.10 section 5.3 (GPT) | Partition table format, CRC32, backup table | `gpt.c`, build repo image script |
| Devicetree: `jedec,spi-nor` nodes with a `partitions` subnode (`label`, `reg`) | Flash and partition description | `core.c`, `partition.c` |
| OP-TEE DT properties (new): `optee,ns-access`, `optee,rollback-anchor` | Normal-world access, anchor policy | `partition.c`, `spinor_fs_store.c`; to be documented in optee_docs |
| ARM PrimeCell PL022, PL061 TRMs | SPI controller and GPIO chip select | existing drivers |

## 3.4 Platform hooks

A platform that enables SPI-NOR FS must provide or check these:

| Hook | Needed for | Default | Notes |
| --- | --- | --- | --- |
| `tee_otp_get_hw_unique_key()` | SSK, TSK, RPMC root keys | weak, all zeros, only with `CFG_INSECURE=y` | Must be unique per device and secret on a product |
| `nv_counter_get_spinor_fs()`, `nv_counter_incr_spinor_fs_to()` | Rollback anchor without RPMC | weak, `TEE_ERROR_NOT_IMPLEMENTED` | Optional when the flash has RPMC |
| A SPI controller driver | Flash access | PL022 on QEMU | Either spi-mem `ops` or a `struct spi_chip` through the adapter; chip select through the controller or a GPIO |
| Secure DT or build options | Flash, partitions, store region, anchor policy | `CFG_SPI_NOR_FS_*` | DT needs PR 2 |
| Bus and memory map security | Secure-only SPI controller and its MMIO | platform `main.c` / TZASC / TZPC | See [04-assumptions.md](04-assumptions.md) |

## 3.5 Hardware

| Item | Requirement |
| --- | --- |
| SPI-NOR flash | SFDP table (JESD216) or an entry in `ids.c`; 4 KiB sector erase; for rollback protection, RPMC (JESD260) with at least one counter |
| SPI controller | Single-wire transfers; secure-only, or reachable only from the secure world |
| Flash region | `optee-fs` of at least 32 KiB (8 sectors), 4 KiB aligned |
| Provisioning | An RPMC root key per counter, written once in a trusted environment |

## 3.6 Tools

| Tool | Used for |
| --- | --- |
| Arm GCC toolchains from `make toolchains` | All builds |
| Python 3 | Build scripts, the GPT image writer |
| `mkimage` (built from U-Boot by the build) | uImages in the flash image |
| `checkpatch.pl` from the Linux tree | Style gate |
| `sgdisk` / `sfdisk` (optional) | Inspecting the GPT of a flash image |
