# OP-TEE SPI-NOR Secure Storage: Low-Level Design

This document set describes the SPI-NOR driver stack and the SPI-NOR FS secure
storage proposed for OP-TEE in
[OP-TEE/optee_os#8240](https://github.com/OP-TEE/optee_os/issues/8240). It is
written against the `topic/*` branches of
[bhaskarv79/optee_os](https://github.com/bhaskarv79/optee_os), based on
upstream optee_os `f77757010`.

> Status: working draft in Markdown. optee_docs is built from reStructuredText
> by Sphinx, so these files are not part of the published documentation yet;
> they are converted once the design is agreed.

## Contents

| File | What it answers |
| --- | --- |
| [01-requirements.md](01-requirements.md) | What the feature must do, with an ID, source, verification and status per requirement |
| [02-scope-of-work.md](02-scope-of-work.md) | What is delivered, split by pull request; what is left out; status and open items |
| [03-dependencies.md](03-dependencies.md) | Code bases, specifications, platform hooks, hardware and tools the work relies on; order of the pull requests |
| [04-assumptions.md](04-assumptions.md) | What the design takes as given about the platform, the threat model and the media; risks if an assumption fails |
| [05-design/](05-design/) | Low-level design, layer by layer |

### 05-design

| File | Content |
| --- | --- |
| [architecture.md](05-design/architecture.md) | Layers, source map, storage IDs, build options, call paths |
| [driver-stack.md](05-design/driver-stack.md) | The SPI-NOR driver in seven layers: partitions, core API, command layer, discovery, spi-mem, controllers; a write traced through them |
| [file-system.md](05-design/file-system.md) | What a TA gets (GP objects, not POSIX), the hash-tree file system, dirf.db, RPMB FS for comparison |
| [spinor-store.md](05-design/spinor-store.md) | The SPI-NOR log store: sectors, records, mount, allocation, compaction, a worked example |
| [integrity.md](05-design/integrity.md) | Keys, exact inputs of every tag and hash, identity binding, check order, tamper matrix |
| [rollback.md](05-design/rollback.md) | Rollback anchors, policy, open and commit sequences, the one-commit gap, platform counter contract |
| [power-loss.md](05-design/power-loss.md) | Atomicity, the state after a power cut at each step, maintenance and limits |
| [partitions-gpt.md](05-design/partitions-gpt.md) | Flash discovery, partitions from configuration, DT or GPT, normal-world access |
| [rpmc.md](05-design/rpmc.md) | RPMC driver: SFDP table, command packets and signatures, root and session keys, sessions, provisioning, errors |
| [qemu-platform.md](05-design/qemu-platform.md) | QEMU reference platform: hardware model, init, flash layout and image, boot log, porting to a board |
| [concurrency.md](05-design/concurrency.md) | Execution model, locks, lock order per path, what runs in parallel, unlocked data |
| [testing.md](05-design/testing.md) | Self tests, GP storage tests, CI, QEMU runs, host fuzzing and harness, mapped to requirements |

## Reading order

1. [01-requirements.md](01-requirements.md) and
   [02-scope-of-work.md](02-scope-of-work.md) for what is built and why.
2. [05-design/architecture.md](05-design/architecture.md) for the big picture. [driver-stack.md](05-design/driver-stack.md)
   zooms into the SPI-NOR driver.
3. [05-design/file-system.md](05-design/file-system.md), then
   [spinor-store.md](05-design/spinor-store.md), for the on-media format.
4. [integrity.md](05-design/integrity.md), [rollback.md](05-design/rollback.md)
   and [power-loss.md](05-design/power-loss.md) for the security and
   reliability arguments.
5. [03-dependencies.md](03-dependencies.md) and
   [04-assumptions.md](04-assumptions.md) before porting to a new platform.

## Glossary

| Term | Meaning |
| --- | --- |
| Anchor | A monotonic counter outside the storage that dirf.db's counter must reach; detects replay of an older image |
| dirf.db | The directory file of the hash-tree file system: one entry per object, with its root hash |
| FEK | File Encryption Key: random, 16 bytes, one per file, stored wrapped by the TSK |
| GP | GlobalPlatform; the TEE Internal Core API defines the Trusted Storage API that TAs use |
| GPT | GUID Partition Table (UEFI 2.10 section 5.3) |
| HUK | Hardware Unique Key of the device, from `tee_otp_get_hw_unique_key()` |
| NS, normal world | The non-secure world: Linux, U-Boot, UEFI |
| PTA | Pseudo Trusted Application: a service in OP-TEE core reachable through the TEE client API |
| REE FS | Secure storage in files of the normal world's file system, through tee-supplicant |
| RPMB | Replay Protected Memory Block of eMMC and UFS |
| RPMC | Replay Protected Monotonic Counter of SPI-NOR flashes (JEDEC JESD260) |
| SFDP | Serial Flash Discoverable Parameters (JEDEC JESD216) |
| SPI-NOR FS | The secure storage of this proposal: the hash-tree file system on a secure SPI-NOR partition |
| SSK | Secure Storage Key, derived from the HUK |
| TA | Trusted Application |
| TSK | TA Storage Key, derived from the SSK and the TA UUID |
