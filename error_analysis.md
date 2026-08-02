## Conclusion  

The build failure for target `5.14.0-687.30.1.el9_8.x86_64` is not caused by simple missing headers or dependency packages. The root cause consists of two independent issues.  

1. **The target kernel is built with `CONFIG_BCACHE=n`, making its `task_struct` layout incompatible with what bcache expects.**  
2. **The bcache source remaining in the Rocky/RHEL 9 kernel source has not kept up with backported block layer APIs.**  

Therefore, the method assumed by the current repository—"build only `bcache.ko` as an external module from the same kernel source without modifying the standard Rocky Linux 9 kernel"—**is not feasible with the unmodified in-tree bcache source**. The logs show an exact copy of the SRPM source is used and `make -C ... M=... CONFIG_BCACHE=m modules` is executed, but it stops at the compilation stage. ([GitHub][1])  

---  

## Cause 1: Mismatch between `task_struct` and `CONFIG_BCACHE`  

The first fundamental error is:  

```text  
struct task_struct has no member named 'sequential_io_avg'  
struct task_struct has no member named 'sequential_io'  
```  

`add_sequential()` and `check_should_bypass()` in `request.c` directly reference the following members:  

```c  
task->sequential_io  
task->sequential_io_avg  
```  

The numerous diagnostics for `minmax.h`, `__UNIQUE_ID()`, `BUILD_BUG_ON_MSG()` appearing later in the log are **secondary errors** resulting from passing these non-existent members to the `max()` macro. `minmax.h` itself is not broken. ([GitHub][1])  

In the analogous CentOS Stream 9 source, these members are conditionally defined in `include/linux/sched.h`:  

```c  
#if defined(CONFIG_BCACHE) || defined(CONFIG_BCACHE_MODULE)  
        unsigned int sequential_io;  
        unsigned int sequential_io_avg;  
#endif  
```  

Immediately following these are other `task_struct` members like `kmap_ctrl`. ([GitLab][2])  

On the other hand, in standard Rocky/RHEL configurations, bcache is disabled. The repository's own README describes this state as the build target. ([GitHub][3])  

### Why passing `CONFIG_BCACHE=m` does not resolve the issue  

The workflow executes:  

```bash  
make -C "$KDIR" M="$WORK" CONFIG_BCACHE=m modules  
```  

This functions as a Make variable to cause Kbuild to evaluate `obj-$(CONFIG_BCACHE)` etc., and include bcache objects in the compilation target. Indeed, the official Kbuild documentation explains that when `obj-$(CONFIG_FOO)` evaluates to `y` or `m`, the target is built. ([GitHub][4])  

However, `CONFIG_BCACHE_MODULE` as referenced by the kernel C source is defined in `include/generated/autoconf.h`, generated when the target kernel is configured via Kconfig. Passing a Make variable on the command line does **not** change the target kernel's `.config`, the generated `autoconf.h`, or the layout of the running kernel's `task_struct`. ([Linux Kernel Documentation][5])  

Therefore, the current state is:  

```text  
From Kbuild's perspective:  
  CONFIG_BCACHE=m, so bcache/*.c is compiled  

From the C preprocessor's / running kernel's perspective:  
  CONFIG_BCACHE_MODULE is undefined  
  task_struct does not have sequential_io*  
```  

### `-DCONFIG_BCACHE_MODULE` is dangerous  

Simply adding the following compilation option does not resolve the issue:  

```bash  
-DCONFIG_BCACHE_MODULE  
```  

This would make the module *believe* that `task_struct` contains two integer members. However, the running kernel's `task_struct` lacks them.  

Consequently, when bcache attempts to write to `sequential_io`, it would actually corrupt different fields like the subsequent `kmap_ctrl`. This is not merely a load failure but **an ABI mismatch that leads to kernel memory corruption**.  

---  

## Cause 2: Mismatch between bcache source and EL9 block layer API  

Alongside `request.c`, `super.c` also encountered independent errors.  

```text  
'q' undeclared  
implicit declaration of blk_queue_max_discard_sectors  
implicit declaration of blk_queue_logical_block_size  
QUEUE_FLAG_NONROT undeclared  
too many arguments to blk_queue_write_cache  
implicit declaration of blkdev_put  
implicit declaration of blk_queue_io_opt  
implicit declaration of blkdev_get_by_path  
```  

This is not the kind of issue solvable by adding a single header. ([GitHub][1])  

### `q` undeclared is a source-side inconsistency from the distribution  

In the analogous public CentOS Stream 9 `kernel-5.14.0-687.el9` source, `bcache_device_init()` lacks a declaration for `struct request_queue *q;` but later executes:  

```c  
q = d->disk->queue;  
```  

Therefore, `q undeclared` is not due to the external module extraction script removing variables; it is **a problem present in the kernel source itself at the point of extraction**. ([GitLab][6])  

Further historical investigation shows that in the commit immediately before the block layer cache settings were moved from `queue->flags`, the declaration `struct request_queue *q;` existed. In the immediate subsequent commit, only this declaration was removed while the old code `q = d->disk->queue` and its surrounding logic remained. ([GitLab][7])  

The commit message indicates that changes to bcache were dropped as a conflict resolution. Consequently, only part of the upstream change was incorporated, leaving bcache internally inconsistent. ([GitLab][8])  

### Old block API remains  

The target `super.c` uses the following old format:  

```c  
blk_queue_max_discard_sectors(q, UINT_MAX);  
blk_queue_logical_block_size(q, ...);  
blk_queue_flag_set(QUEUE_FLAG_NONROT, ...);  
blk_queue_write_cache(q, true, true);  
```  

However, in the target `kernel-devel`, for example, `blk_queue_write_cache()` is now a 1-argument state query function, not a 3-argument setter. The log also displays the actual declaration from the header side. ([GitHub][1])  

In the current upstream Linux bcache implementation, this area has been migrated to building a `struct queue_limits` before disk creation and passing it to `blk_alloc_disk(&lim, ...)`. Therefore, a similar API port is needed for this EL9 tree. However, since RHEL/EL9 is a mixed state with both old and new APIs backported, simply copying the current upstream `super.c` wholesale is not an appropriate method. ([GitHub][9])  

---  

## Error-by-Error Assessment  

| Error                                  | Assessment                                    |  
| ------------------------------------ | ------------------------------------- |  
| `task_struct` lacks `sequential_io*` | Most critical structural issue. Incompatible with target kernel's build config/ABI.        |  
| Numerous `minmax.h`, `__UNIQUE_ID` related errors    | Secondary macro expansion errors due to missing `sequential_io*`. |  
| `q undeclared`                       | Backport inconsistency in the distributed bcache source.              |  
| `blk_queue_*`, `QUEUE_FLAG_NONROT`    | Failure to adapt to block layer API updates.                |  
| `blkdev_get_by_path`, `blkdev_put`    | Failure to adapt to block device open/close API updates.    |  
| CRC64, `crc64_be`                     | Not the cause of this failure. Compilation stops before linking.               |  
| Secure Boot, signing                       | Not the cause of this failure. Compilation stops before module generation.           |  

---  

## Feasible Fix Strategies  

### 1. If the standard Rocky kernel is not to be modified  

bcache needs to be ported as a true external module. At minimum, this requires:  

* Removing dependency on `task_struct.sequential_io` and `sequential_io_avg`.  
* Migrating sequential I/O statistics to bcache's own hash / I/O stream state, etc.  
* Converting `bcache_device_init()` to the `struct queue_limits` API of the target EL9.  
* Converting the configuration of `QUEUE_FLAG_NONROT`, write cache, FUA, discard, block size, and `io_opt` to the new API.  
* Converting `blkdev_get_by_path()` / `blkdev_put()` to the file-based bdev API of the target kernel.  
* After compilation, verifying with `MODPOST` and `Module.symvers` that all required symbols are exported for external modules.  
* Validating load/unload, flush/FUA, discard, writeback, and failure recovery on a VM.  

This is not a 1-2 line fix to the build script but **a porting effort of bcache itself for EL9**.  

### 2. If the kernel itself can be rebuilt  

Configure and build the entire kernel with `CONFIG_BCACHE=m` via Kconfig. This ensures the kernel and module share the same `task_struct` layout.  

However, the block API inconsistency in `super.c` identified here is a separate issue, so **simply changing the config to `m` is highly unlikely to make this EL9 source compilable**. The bcache-side API porting is a prerequisite.  

### 3. If using a kernel with bcache already enabled  

The safest method is to use a kernel package that has been built and verified with `CONFIG_BCACHE=m` enabled. The kernel binary and `bcache.ko`'s struct layout, configuration, and symbol exports are managed integrally.  

---  

## Reasonable Immediate Actions for the Repository  

If the current source is not yet to be ported, the workflow should check the following before building and explicitly stop if conditions are unmet:  

```bash  
KREL=5.14.0-687.30.1.el9_8.x86_64  
KDIR="/usr/src/kernels/$KREL"  

grep -E '^CONFIG_BCACHE=|^# CONFIG_BCACHE' "$KDIR/.config" || true  

grep -E '^#define CONFIG_BCACHE(_MODULE)? ' \  
  "$KDIR/include/generated/autoconf.h" || true  

grep -n -A4 -B2 \  
  'defined(CONFIG_BCACHE).*defined(CONFIG_BCACHE_MODULE)' \  
  "$KDIR/include/linux/sched.h"  
```  

If `include/generated/autoconf.h` lacks `CONFIG_BCACHE_MODULE` and `sched.h` contains the conditional `sequential_io*` members, building unmodified bcache as an external module is unsafe.  

Furthermore, the following superficial fixes are insufficient or dangerous:  

* Simply adding `struct request_queue *q;`  
  → Will stop at the next block API error.  
* Adding `-DCONFIG_BCACHE_MODULE`  
  → Causes `task_struct` ABI mismatch.  
* Adding `-Wno-error`  
  → Does not resolve function deletions or signature changes.  
* Copying the entire current upstream bcache  
  → Likely to cause new incompatibilities with EL9-specific backported APIs.  
* Fixing CRC64 or module signing first  
  → The build does not even reach that stage.  

**The investigation concludes that the primary cause is "attempting to externally mount in-tree bcache, which assumes structural changes in the kernel itself, onto a kernel built with `CONFIG_BCACHE=n`." The failure to adapt to the block layer API is a separate, secondary obstacle.**  

No commits or pushes have been made.  

[1]: https://github.com/aont/bcache-builder/raw/refs/heads/main/logs/7_Build%20bcache.ko.txt "https://github.com/aont/bcache-builder/raw/refs/heads/main/logs/7_Build%20bcache.ko.txt"  
[2]: https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/include/linux/sched.h "https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/include/linux/sched.h"  
[3]: https://github.com/aont/bcache-builder "https://github.com/aont/bcache-builder"  
[4]: https://github.com/aont/bcache-builder/raw/refs/heads/main/.github/workflows/build-bcache.yml "https://github.com/aont/bcache-builder/raw/refs/heads/main/.github/workflows/build-bcache.yml"  
[5]: https://docs.kernel.org/kbuild/kconfig.html "https://docs.kernel.org/kbuild/kconfig.html"  
[6]: https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/drivers/md/bcache/super.c "https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/drivers/md/bcache/super.c"  
[7]: https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/6b121db89ebf97182e0f485aaddb2bc5987fc31f/drivers/md/bcache/super.c "https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/6b121db89ebf97182e0f485aaddb2bc5987fc31f/drivers/md/bcache/super.c"  
[8]: https://gitlab.com/api/v4/projects/redhat%2Fcentos-stream%2Fsrc%2Fkernel%2Fcentos-stream-9/repository/commits?page=1&path=drivers%2Fmd%2Fbcache%2Fsuper.c&per_page=20&ref_name=kernel-5.14.0-687.el9 "https://gitlab.com/api/v4/projects/redhat%2Fcentos-stream%2Fsrc%2Fkernel%2Fcentos-stream-9/repository/commits?page=1&path=drivers%2Fmd%2Fbcache%2Fsuper.c&per_page=20&ref_name=kernel-5.14.0-687.el9"  
[9]: https://github.com/torvalds/linux/blob/master/drivers/md/bcache/super.c "https://github.com/torvalds/linux/blob/master/drivers/md/bcache/super.c"
