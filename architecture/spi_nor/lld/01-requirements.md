# 1. Requirements

Each requirement has an ID, the statement, where it comes from, how it is
verified and its status on `topic/integration`. IDs are stable: a requirement
that is dropped keeps its ID with status "Dropped".

**Sources**
- **#8240**: the design discussion in
  [OP-TEE/optee_os#8240](https://github.com/OP-TEE/optee_os/issues/8240),
  including the maintainers' requests (basic SPI-NOR storage first, replay
  protection as a follow-up, SPI-NOR FS in QEMU CI).
- **GP**: GlobalPlatform TEE Internal Core API, Trusted Storage.
- **Design**: decisions recorded in this document set.
- **Review**: findings of the code reviews of the series.

**Verification methods**
- **Test**: an automated test (`xtest 1001` core self tests, `xtest _60` and
  the 6000 storage tests, the QEMU boot runs).
- **Inspection**: code or configuration review.
- **Analysis**: an argument in [05-design](05-design/), for example a power-cut
  table.

**Status**: Met, Partly met (what is missing is named), Open, Dropped.

## 1.1 Functional

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-FUN-01 | OP-TEE provides a secure storage whose data stays on a SPI-NOR flash only reachable from the secure world, under the storage ID `TEE_STORAGE_PRIVATE_SPI_NOR` (`0x80000300`). | #8240 | Test: storage tests 6001 to 6022 on that ID | Met |
| REQ-FUN-02 | TAs use it through the unchanged GP Trusted Storage API: create, open, read, write, seek, truncate, rename, delete, enumerate. No new TA API. | GP | Test: 6000 series | Met |
| REQ-FUN-03 | With `CFG_SPI_NOR_FS_DEFAULT=y`, `TEE_STORAGE_PRIVATE` selects SPI-NOR FS, so TAs and core users (TA version database, StandaloneMM variables with `CFG_STMM_STORAGE_SPI_NOR`) store there without change. | #8240 | Test: xtest detects the storage behind `TEE_STORAGE_PRIVATE` | Met |
| REQ-FUN-04 | SPI-NOR FS works without tee-supplicant: TAs that only use it are enumerated at TEE bus stage 1. | Design | Inspection: `core/pta/device.c`, `user_ta_header.h` | Met |
| REQ-FUN-05 | The store region is the flash partition labelled `optee-fs`, taken from the secure DT or a GPT, else `CFG_SPI_NOR_FS_OFFSET` / `CFG_SPI_NOR_FS_SIZE` of device `CFG_SPI_NOR_FS_DEVICE`. If the DT declares `optee-fs` but no flash has it, the storage is unavailable; it never falls back to another region. | Design | Test: store self test; Inspection: `find_region()` | Met |
| REQ-FUN-06 | Flashes are discovered with JEDEC SFDP (JESD216), with a JEDEC ID table as fallback, and accessed with polled read, page program and 4 KiB sector erase. | #8240 | Test: `spi_nor_self_test()` on the simulator; QEMU boot | Met |
| REQ-FUN-07 | SPI controllers are reached through a spi-mem operation interface; existing `struct spi_chip` controllers (PL022) work through an adapter. | #8240 | Test: spi-mem self test (`core/pta/tests/spi_mem.c`) | Met |
| REQ-FUN-08 | Flashes and their partitions can be described in the secure DT (`jedec,spi-nor` nodes with a `partitions` subnode). | Design | Test: DT tests on `topic/dt` | Met |
| REQ-FUN-09 | Partitions can be read from a GPT on the flash; the table is parsed as untrusted input, with fallback to the backup GPT. | Design | Test: GPT self tests (valid, bad primary, both bad, out of range, overlap); host fuzzing, about 633,000 cases | Partly met: an entry that is not erase aligned rejects the whole table (Review) |
| REQ-FUN-10 | The normal world can read, and where allowed program and erase, partitions marked `optee,ns-access = "ro"` or `"rw"`, through the SPI-NOR PTA, so that for example firmware update can write a boot partition. | #8240 | Test: NS access policy self tests | Met (to be discussed in #8240 before it is proposed) |
| REQ-FUN-11 | A flash simulator in secure RAM (`CFG_SPI_NOR_SIM`) with NOR program/erase semantics and an SFDP table allows tests without hardware. | Design | Test: all self tests run on it | Met |
| REQ-FUN-12 | On QEMU, OP-TEE drives the secure PL022 and SPI-NOR flash of `-machine virt,spi=secure`, reports the flash and its partitions at boot, and keeps flash and RPMC state across boots. | #8240 (QEMU CI) | Test: QEMU boot runs with persistence | Met, waits for the QEMU patches upstream |

## 1.2 Security

Threat model: the normal world is untrusted; the flash content is untrusted
when it is read back (a physical attacker can reprogram it, replay an old
image or move it to another device); the SPI bus is secure-only (see
[04-assumptions.md](04-assumptions.md)).

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-SEC-01 | Confidentiality: every stored byte of object data and metadata is encrypted with a per-file key that only this device and the owning TA can unwrap. | #8240 | Analysis: [integrity.md](05-design/integrity.md) | Met (hash-tree file system unchanged) |
| REQ-SEC-02 | Integrity: any change to stored data, metadata or the directory is detected before data is returned. | #8240 | Analysis: tamper matrix; Test: torn-record and corruption tests | Met |
| REQ-SEC-03 | Identity binding: an object cannot be read as another object, by another TA or on another device. | Design | Analysis: [integrity.md](05-design/integrity.md) | Met |
| REQ-SEC-04 | Rollback protection: replaying an older flash image is detected, using an RPMC counter of the flash, else the platform counter `nv_counter_*_spinor_fs()`. | #8240 (follow-up PR) | Test: `spinor_fs_rollback_self_test()`; QEMU 4-boot replay run | Met on `topic/rpmc`. Gap: after a power cut between the dirf.db commit and the anchor raise, the anchor lags one commit (Review) |
| REQ-SEC-05 | Without an anchor, the storage mounts only in an insecure build (`CFG_INSECURE=y`) and prints a warning. | Design | Inspection: `spinor_select_anchor()` | Met |
| REQ-SEC-06 | A counter that answers but misbehaves never makes OP-TEE fall back to a weaker anchor; products can pin the anchor with `optee,rollback-anchor`. | Design | Inspection; Test: anchor selection runs | Met |
| REQ-SEC-07 | The normal world cannot read or write the `optee-fs` partition, nor any partition without `optee,ns-access`. Partitions cannot overlap, so an NS partition cannot cover the store. | #8240 | Test: NS access policy tests; Inspection: `spi_nor_add_partition()` | Met |
| REQ-SEC-08 | RPMC root keys are derived from the HUK per flash and counter, never stored by OP-TEE; writing a root key is possible only in a provisioning build (`CFG_RPMC_WRITE_ROOT_KEY`, warned at build time). | Design | Inspection: `derive_root_key()`, `mk/config.mk` | Met |
| REQ-SEC-09 | Data read from the flash (records, GPT, SFDP) is bounds-checked; malformed content fails cleanly, without memory errors or a panic. | Review | Test: GPT fuzzing; Inspection | Partly met: a GPT error on a DT flash without partitions fails the driver probe, which panics the boot (Review) |
| REQ-SEC-10 | Test-only code (simulator, test NV counter) cannot be built into a secure image: it requires `CFG_INSECURE=y`. | Design | Inspection: `mk/config.mk` errors | Met |

## 1.3 Reliability

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-REL-01 | Every create, write, truncate, rename and delete is atomic across power loss: after a reboot the object is entirely in its old or its new state. | GP | Analysis: [power-loss.md](05-design/power-loss.md); Test: torn-record tests | Met in analysis and self tests; power cut at arbitrary points not yet tested on QEMU |
| REQ-REL-02 | NOR semantics are respected: no program over unerased bytes, every sector erased before reuse. | Design | Test: the simulator programs like NOR (bits only cleared, wrap inside the page), so a program over unerased bytes corrupts data and fails the store self tests | Met |
| REQ-REL-03 | A full store reports `TEE_ERROR_STORAGE_NO_SPACE` and still allows removing objects and updating dirf.db. | Review | Test: fill, then dirf.db update and removal | Met |
| REQ-REL-04 | Compaction always makes progress: 100,000 overwrites of one object on a minimal store never report the store full. | Review (earlier critical finding) | Test: host harness | Met |
| REQ-REL-05 | Erases are spread over the region. | Design | Inspection: round-robin `alloc_sector()` | Partly met: the allocation cursor restarts at sector 0 after each boot |

## 1.4 Resources and limits

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-RES-01 | RAM use is bounded and proportional to the number of live items: one 28-byte index entry per item, 5 bytes per sector, one 4 KiB buffer. | Design | Analysis: [spinor-store.md](05-design/spinor-store.md) | Met |
| REQ-RES-02 | The store works on regions from 8 to 65,535 sectors of 4 KiB, on flashes whose erase size divides 4 KiB. | Design | Inspection: `sfs_init()` | Met |
| REQ-RES-03 | Flashes above 16 MiB are addressed with 4-byte addresses as their SFDP tables require. | Review | Inspection | Partly met: 4-byte mode is entered with B7h; SFDP dword 16 (entry method) is not read |

## 1.5 Compatibility

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-CMP-01 | REE FS keeps its on-disk format and behaviour; the backend split is a refactoring. | #8240 | Test: full xtest with REE FS | Met |
| REQ-CMP-02 | Existing platforms see no change unless they enable the new options; every new option defaults to `n`. | Design | Inspection: `mk/config.mk` | Met |
| REQ-CMP-03 | REE FS can use an RPMC counter instead of RPMB as its anchor (`CFG_REE_FS_INTEGRITY_RPMC`), exclusive with `CFG_REE_FS_INTEGRITY_RPMB`. | Design | Inspection; build error when both are set | Met |
| REQ-CMP-04 | Objects written by older OP-TEE versions stay readable (`CFG_CORE_HUK_SUBKEY_COMPAT`, `CFG_REE_FS_HTREE_HASH_SIZE_COMPAT` unchanged). | Design | Inspection | Met |

## 1.6 Configuration

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-CFG-01 | Every feature is a separate `CFG_*` option, with documented dependencies checked by `cfg-depends-all`. The list is in [architecture.md](05-design/architecture.md#build-options). | OP-TEE practice | Inspection; per-commit builds | Met |
| REQ-CFG-02 | Platform-specific values (store region, anchor policy, NS access) can be set per board in the secure DT without rebuilding. | Design | Test: DT tests | Met on `topic/dt` |

## 1.7 Test and upstream readiness

| ID | Requirement | Source | Verification | Status |
| --- | --- | --- | --- | --- |
| REQ-TST-01 | Each commit builds on its own, with the new options on and off. | OP-TEE practice | Per-commit builds of every branch | Met (arm32 sim and QEMU configurations; arm64 per-commit not run) |
| REQ-TST-02 | Self tests for spi-mem, SPI-NOR core, store, rollback, GPT, RPMC and NS access run in `xtest 1001`. | #8240 | Test | Met |
| REQ-TST-03 | SPI-NOR FS runs in the OP-TEE CI on QEMUv8 with the simulated flash. | #8240 (maintainer request) | `.github/workflows/ci.yml` | Met |
| REQ-TST-04 | optee_test covers `TEE_STORAGE_PRIVATE_SPI_NOR` in the 6000 storage tests and skips it on builds without it. | #8240 | Test | Met (optee_test `spinor` branch) |
| REQ-TST-05 | Commits pass checkpatch, carry SPDX headers and a `Signed-off-by`. | OP-TEE practice | Gates of the code review | Met |
| REQ-TST-06 | optee_docs describes the feature, the DT properties and the GPT behaviour. | Review | Inspection | Open |

## 1.8 Dropped

| ID | Requirement | Why dropped |
| --- | --- | --- |
| REQ-DRP-01 | Emulate RPMB on top of SPI-NOR and RPMC, so RPMB FS could run on it. | Superseded by SPI-NOR FS, which needs no RPMB layer (design decision D-8). |
| REQ-DRP-02 | A Linux MTD driver for the NS partitions. | Replaced by the SPI-NOR PTA; a normal-world user-space library is planned later. |
