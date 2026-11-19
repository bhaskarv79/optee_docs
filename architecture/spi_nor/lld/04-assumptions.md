# 4. Assumptions

The design is correct only if these hold. Each assumption says why it
matters, what happens if it fails, and how a platform can check it.

## 4.1 Platform and threat model

| ID | Assumption | Why it matters | If it does not hold | How to check |
| --- | --- | --- | --- | --- |
| AS-PLT-01 | The SPI controller, its MMIO, its chip select and the flash are reachable only from the secure world. | SPI-NOR FS has no transport encryption; the normal world must not issue flash commands. | The normal world can read ciphertext, erase or rewrite the store at will (data loss, denial of service; rollback still detected by the anchor). | Platform memory map and TZPC/TZASC settings; the DT node is in the secure DT only |
| AS-PLT-02 | The HUK is unique per device, secret, and stable for the device's life. | Every key (SSK, TSK, RPMC root keys) derives from it. | A shared HUK lets one device read another's store; a changed HUK makes all data unreadable and RPMC counters unusable. | Platform `tee_otp_get_hw_unique_key()`; never the weak default |
| AS-PLT-03 | OP-TEE is the only user of the flash at run time. Boot ROM and boot loaders may read it but do not write `optee-fs`. | The store assumes nobody else programs or erases its sectors. | Silent corruption, detected as `TEE_ERROR_CORRUPT_OBJECT` or a failed mount. | Boot flow review |
| AS-PLT-04 | Only one OP-TEE instance, and one core at a time under the instance mutex, accesses a given flash. | The store has no lock of its own (see [concurrency.md](05-design/concurrency.md)). | Interleaved records; data loss. | Platform design |
| AS-PLT-05 | Attackers: the normal world is fully hostile; a physical attacker can read, reprogram, replay or move the flash, but cannot read secure RAM or the HUK. | Defines what the integrity and rollback mechanisms protect against. | Attacks on secure RAM, side channels and fault injection are out of scope. | Product threat model |
| AS-PLT-06 | The target is little-endian (all Arm OP-TEE platforms). | Integers are stored in CPU byte order. | A big-endian build cannot read a store written by a little-endian one. | n/a |

## 4.2 Flash media

| ID | Assumption | Why it matters | If it does not hold | How to check |
| --- | --- | --- | --- | --- |
| AS-MED-01 | Programming only clears bits; erase sets a whole sector to 0xff; the erase unit divides 4 KiB. | The log store appends without erasing and erases whole 4 KiB sectors. | The store refuses the region (`sfs_init()` checks the erase size). | SFDP erase types |
| AS-MED-02 | A power cut during a program or an erase leaves the affected bytes in an unknown state, but does not disturb other sectors. | The power-loss analysis treats a torn record or sector as garbage limited to that place. | A cut that damages neighbouring committed sectors could lose committed data. | Flash datasheet |
| AS-MED-03 | Programming 4 bytes of 0 over a sector header does not need an erase and completes or leaves the header invalid. | `compact()` invalidates a sector before erasing it. | A half-cleared header could still validate; its CRC makes this negligible. | Datasheet |
| AS-MED-04 | Sectors endure enough erase cycles for the product's write rate (typically 100,000 per sector). | Each small TA write costs about two sector programs and up to two erases. | Wear-out of the store region. | Write-rate budget against region size |
| AS-MED-05 | Reads return what was last programmed, or a detectable error. | Data CRCs and GCM tags catch silent bit flips, but the design does not correct them. | A flipped bit makes the object unreadable (it is removed as corrupt). | Datasheet, data retention |
| AS-MED-06 | The flash supports single-wire read (03h), page program (02h), the 4 KiB erase opcode its SFDP names, status polling (05h) and, above 16 MiB, 4-byte addressing entered with B7h. | Only these commands are used. | Parts that need another 4-byte entry method fail above 16 MiB (open item, REQ-RES-03). | SFDP; datasheet |

## 4.3 Partitions and layout

| ID | Assumption | Why it matters | If it does not hold | How to check |
| --- | --- | --- | --- | --- |
| AS-LAY-01 | The `optee-fs` partition is fixed for the device's life. | The store never moves; a missing declared partition makes storage unavailable rather than using another region. | Moving the partition loses all objects. | Flash layout control |
| AS-LAY-02 | When partitions come from a GPT, the normal world cannot write the sectors holding the protective MBR, the primary and the backup GPT. | Partition bounds, and through them NS access, are read from that table. | The normal world could redraw partitions, for example to place an NS partition over the store. Partitions are checked for overlap, but the table itself is trusted once its CRCs pass. | No NS partition covers the GPT areas |
| AS-LAY-03 | Partitions are erase aligned and do not overlap. | `spi_nor_add_partition()` enforces it. | The partition is rejected (from a GPT, today the whole table: open item 2). | Layout tooling |
| AS-LAY-04 | The QEMU layout is an example: GPT in the first MiB, boot 1 MiB at 1 MiB, kernel 8 MiB at 2 MiB, ns-data 6 MiB at 10 MiB, optee-fs 1 MiB at 16 MiB. | Products choose their own. | n/a | n/a |

## 4.4 Rollback anchor

| ID | Assumption | Why it matters | If it does not hold | How to check |
| --- | --- | --- | --- | --- |
| AS-ANC-01 | RPMC root keys are written once, at provisioning, by a build with `CFG_RPMC_WRITE_ROOT_KEY=y`, in a trusted environment. | The root key travels in clear on the SPI bus when it is written. | An attacker who sees the key can forge counter updates. | Factory process |
| AS-ANC-02 | The RPMC counter used by SPI-NOR FS (`CFG_SPI_NOR_FS_RPMC_COUNTER`) is used by nothing else. | The anchor must only move with dirf.db commits. | Another user raising it makes dirf.db look old: storage refused. | Configuration review |
| AS-ANC-03 | A platform counter behind `nv_counter_*_spinor_fs()` is monotonic, survives power loss and is out of the attacker's reach. | It replaces RPMC as the anchor. | Rollback goes undetected. | Platform design |
| AS-ANC-04 | Products exposed to physical attack pin the anchor (`optee,rollback-anchor = "rpmc"`) rather than `auto`. | With `auto`, a flash without RPMC selects the weaker anchor. | Swapping the flash downgrades the protection. | Secure DT review |
| AS-ANC-05 | A 32-bit counter is enough: one increment per commit. | Counters never wrap. | After about 4 billion commits the store stops accepting writes. | n/a |

## 4.5 Software environment

| ID | Assumption | Why it matters | If it does not hold | How to check |
| --- | --- | --- | --- | --- |
| AS-SW-01 | TAs keep objects small (keys, certificates, state), typically a few KiB. | The store and RAM index are sized for small objects; a write copies whole 4 KiB blocks. | Large objects fill the region quickly and slow every open (it verifies every node). | TA design |
| AS-SW-02 | Storage is used after boot, from TA sessions or core services; nothing calls it from interrupt context. | All locks are sleeping mutexes. | Deadlock or panic. | Code review |
| AS-SW-03 | Product builds use `CFG_INSECURE=n`. | The simulator, test NV counter and the "no anchor" mode require `CFG_INSECURE=y`. | Test code in a product. | Build configuration |

## 4.6 Risks and open questions

| ID | Risk or question | Impact | Mitigation |
| --- | --- | --- | --- |
| RSK-01 | Anchor lags one commit after a power cut between the dirf.db commit and the anchor raise | The image from before that commit is accepted until the next commit | Raise the anchor first, or record the pending raise; under discussion |
| RSK-02 | Allocation restarts at sector 0 after each mount | Uneven wear on devices that reboot often | Start the cursor after the newest log sector |
| RSK-03 | With anchor 0, a corrupt dirf.db is removed and storage restarts empty | Loss of all objects in insecure or unanchored builds | Existing REE FS behaviour; document it |
| RSK-04 | QEMU memory map may change while the QEMU patches are reviewed | The QEMU platform code must follow | Keep `spi_nor_qemu.c` addresses in one place; update after QEMU merge |
| RSK-05 | Maintainers may prefer a different NS access interface | Rework of the PTA | Discuss in #8240 before posting |
| Q-01 | Should GPT partition names grant NS access on products, or only the DT? | Security of the NS access policy | Open |
