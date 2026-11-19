# Concurrency and locking

Which locks serialize storage operations and in what order they are taken.

Part of the [SPI-NOR secure storage low-level design](../README.md).

## Concurrency and locking

Each file system instance has one mutex that serializes every operation on it, from any TA session or core thread; the locks below it only protect shared driver state. All are sleeping mutexes, so an REE FS operation can hold its mutex across an RPC to the normal world.

| Lock | File | Protects | Taken by |
| --- | --- | --- | --- |
| `pobjs_mutex` | `tee_pobj.c` | the list of objects in use, `refcnt`, `creating` | `tee_pobj_get()`, `tee_pobj_release()`, `tee_pobj_create_final()` |
| `pobjs_usage_mutex` | `tee_pobj.c` | `objectUsage` of a shared object (only when it was opened with a share flag) | open, enumeration, `TEE_RestrictObjectUsage1()` |
| `fs_instance.mutex` (`ree_fs_inst`, `spinor_fs_inst`) | `tee_ree_fs.c` | the cached dirf.db handle, every hash tree of the instance, the backend, and for SPI-NOR FS the whole store (`struct sfs` has no lock of its own) | every `tee_file_operations` entry of that instance |
| `nor->lock` | `core/drivers/spi_nor/core.c`, `rpmc.c` | one flash device: commands, busy state | `spi_nor_read/write/erase()`, each RPMC command |
| `sessions_mu` | `core/drivers/spi_nor/rpmc_counter.c` | RPMC sessions (HMAC keys) per (flash, counter) | `spi_nor_rpmc_counter_get/incr_to()` |

Lock order is top to bottom: `fs_instance.mutex`, then `sessions_mu` (through the anchor), then `nor->lock`. REE FS and SPI-NOR FS have separate instance mutexes, so they run in parallel, except that both may reach the same flash through `nor->lock` when REE FS uses an RPMC counter (`CFG_REE_FS_INTEGRITY_RPMC`).

The `busy` flag of `struct tee_obj` marks a key object in use by a crypto operation (`tee_svc_cryp.c`); it plays no part in storage. Two handles on one object, or two objects of one instance, are serialized by the instance mutex.

