# Conclusion

The failure is **not** occurring during the download or extraction of the Rocky Linux kernel SRPM.

The source extraction performed by `rpmbuild -bp` completes successfully. The workflow then fails during the subsequent file existence check:

```bash
test -f "$KSRC/include/uapi/linux/bcache.h"
```

In Rocky Linux 9 / EL9 kernel sources, this file has already been moved:

```text
Old: include/uapi/linux/bcache.h
New: drivers/md/bcache/bcache_ondisk.h
```

Therefore, the root cause is that **the workflow assumes the directory layout of an older Linux kernel source tree**. Because `set -e` is enabled, the external GitHub Actions shell exits as soon as `test -f` returns false (exit status 1). The workflow explicitly checks for the old path and later copies that old header. ([GitHub][1])

---

# What Actually Completed Successfully

Tracing the log shows the following sequence.

## 1. The target kernel was correctly identified

The target release is:

```text
5.14.0-687.30.1.el9_8.x86_64
```

The `crc64_be` lookup in `Module.symvers` also succeeds, producing:

```text
0xeaf3cb23 crc64_be vmlinux EXPORT_SYMBOL_GPL
```

This means the following commands completed successfully:

```bash
test -f "$KDIR/Makefile"
test -s "$KDIR/Module.symvers"
grep -w crc64_be "$KDIR/Module.symvers"
```

The logs also show that execution proceeded to the SRPM download stage afterward. ([GitHub][2])

## 2. The matching SRPM exists

The official Rocky Linux source repository contains the corresponding SRPM:

```text
kernel-5.14.0-687.30.1.el9_8.src.rpm
```

Therefore, this is **not** a case of "matching source RPM not found." ([Rocky Linux][3])

## 3. `rpmbuild -bp` completed successfully

The end of the log contains:

```text
+ RPM_EC=0
++ jobs -p
+ exit 0
##[error]Process completed with exit code 1.
```

At first glance this appears contradictory, but these exit codes come from **different shells**. ([GitHub][2])

The execution flow is approximately:

```text
GitHub Actions bash
  ├─ rpmbuild -bp
  │    └─ RPM %prep shell
  │         └─ exit 0
  │
  └─ Source file validation
       └─ test -f ... → exit 1
            └─ Outer bash exits with status 1
```

The log entry:

```text
+ exit 0
```

belongs to the RPM `%prep` script, **not** to the overall GitHub Actions step.

After `%prep` completes, the outer GitHub Actions shell continues with:

```bash
BCACHE_MAKEFILE="$(find ...)"
test -n "$BCACHE_MAKEFILE"

KSRC="${BCACHE_MAKEFILE%/drivers/md/bcache/Makefile}"

for file in ...; do
    test -f "$KSRC/$file"
done
```

The outer shell is configured as:

```text
shell: bash --noprofile --norc -e -o pipefail
```

and the script itself also enables:

```bash
set -euo pipefail
```

Consequently, if **any** required file is missing, the entire workflow step exits immediately with status 1. ([GitHub][2])

---

# The Missing File

The workflow checks these five files:

```bash
drivers/md/bcache/Kconfig
drivers/md/bcache/Makefile
include/linux/crc64.h
include/uapi/linux/bcache.h
include/trace/events/bcache.h
```

Comparing against the CentOS Stream 9 `kernel-5.14.0-687.el9` source tree:

| File checked by the workflow        | Status in EL9 |
| ----------------------------------- | ------------- |
| `drivers/md/bcache/Kconfig`         | Present       |
| `drivers/md/bcache/Makefile`        | Present       |
| `include/linux/crc64.h`             | Present       |
| `include/uapi/linux/bcache.h`       | **Missing**   |
| `include/trace/events/bcache.h`     | Present       |
| `drivers/md/bcache/bcache_ondisk.h` | Present       |

EL9 contains `drivers/md/bcache/bcache_ondisk.h`, while the old path `include/uapi/linux/bcache.h` no longer exists. The remaining files are present.

Therefore, the command most likely failing is:

```bash
test -f "$KSRC/include/uapi/linux/bcache.h"
```

This command produces no output whether it succeeds or fails. Since the outer shell does not have `set -x` enabled, the executed `test` command never appears in the log. As a result, the log effectively looks like:

```text
rpmbuild internal: exit 0
Outer test: no output, exits with status 1
GitHub Actions: Process completed with exit code 1
```

---

# Why the Header Was Moved

EL9 includes a change that relocates the bcache header.

According to the commit message, `include/uapi/linux/bcache.h` was not actually a userspace API header but instead defined the on-disk metadata format used internally by bcache. It was therefore moved to:

```text
include/uapi/linux/bcache.h
    ↓
drivers/md/bcache/bcache_ondisk.h
```

The rationale was that `bcache-tools` already maintains its own copy, so exposing it as kernel UAPI was unnecessary. This change has been present in the CentOS Stream 9 kernel series since 2021. ([GitLab][4])

Correspondingly, EL9's `drivers/md/bcache/bcache.h` no longer includes:

```c
#include <linux/bcache.h>
```

Instead, it includes the local header:

```c
#include "bcache_ondisk.h"
```

([GitLab][5])

An important point is that although the Rocky Linux 9 kernel reports version `5.14.0`, its source tree is **not identical** to upstream Linux 5.14. RHEL/EL9 kernels include extensive backports from newer kernel releases. Consequently, the assumption that "version 5.14 implies the upstream 5.14 directory layout" does not hold.

---

# There Is Another Workflow Issue

Even if the failing existence check is removed, the next workflow step ("Build bcache.ko") will fail for the same reason.

The workflow currently contains:

```bash
cp -a "$KSRC/drivers/md/bcache" "$WORK/"
cp "$KSRC/include/linux/crc64.h" "$WORK/include/linux/"
cp "$KSRC/include/uapi/linux/bcache.h" "$WORK/include/uapi/linux/"
cp "$KSRC/include/trace/events/bcache.h" "$WORK/include/trace/events/"
```

The following line will fail because the file no longer exists:

```bash
cp "$KSRC/include/uapi/linux/bcache.h" "$WORK/include/uapi/linux/"
```

([GitHub][1])

However, the workflow already copies the entire bcache directory:

```bash
cp -a "$KSRC/drivers/md/bcache" "$WORK/"
```

In EL9, that directory already contains:

```text
drivers/md/bcache/bcache_ondisk.h
```

Therefore, when building against EL9 kernel sources, there is no need to copy the legacy UAPI header separately.

---

# Recommended Fix

No repository modifications have been made, but the workflow should be updated as follows.

## If targeting only EL9

Replace the old header check with the new header:

```bash
for file in \
  drivers/md/bcache/Kconfig \
  drivers/md/bcache/Makefile \
  drivers/md/bcache/bcache_ondisk.h \
  include/linux/crc64.h \
  include/trace/events/bcache.h; do
  test -f "$KSRC/$file"
done
```

Remove this obsolete copy operation:

```bash
cp "$KSRC/include/uapi/linux/bcache.h" "$WORK/include/uapi/linux/"
```

Since `bcache_ondisk.h` is already copied by:

```bash
cp -a "$KSRC/drivers/md/bcache" "$WORK/"
```

no additional action is necessary.

## If supporting both old and new kernel layouts

A more robust approach is to accept either location:

```bash
if [[ ! -f "$KSRC/drivers/md/bcache/bcache_ondisk.h" &&
      ! -f "$KSRC/include/uapi/linux/bcache.h" ]]; then
  echo "Missing bcache on-disk format header" >&2
  exit 1
fi
```

Similarly, copy the legacy header only when it exists:

```bash
cp -a "$KSRC/drivers/md/bcache" "$WORK/"

if [[ -f "$KSRC/include/uapi/linux/bcache.h" ]]; then
  cp "$KSRC/include/uapi/linux/bcache.h" "$WORK/include/uapi/linux/"
fi
```

---

# Improving Error Reporting

The current workflow simply uses `test -f`, making it impossible to determine which file caused the failure.

Replacing it with explicit diagnostics would make future failures immediately obvious:

```bash
for file in \
  drivers/md/bcache/Kconfig \
  drivers/md/bcache/Makefile \
  include/linux/crc64.h \
  include/trace/events/bcache.h; do
  if [[ ! -f "$KSRC/$file" ]]; then
    echo "Required kernel source file is missing: $KSRC/$file" >&2
    exit 1
  fi
done
```

For the current source tree, the expected diagnostic would be:

```text
OK      drivers/md/bcache/Kconfig
OK      drivers/md/bcache/Makefile
OK      drivers/md/bcache/bcache_ondisk.h
OK      include/linux/crc64.h
MISSING include/uapi/linux/bcache.h
OK      include/trace/events/bcache.h
```

---

# Summary

The immediate cause of the failure is:

```bash
test -f "$KSRC/include/uapi/linux/bcache.h"
```

The underlying cause is that `bcache-builder` assumes the legacy bcache source tree layout and does not account for the header relocation that was backported into the EL9 kernel.

The failure is **not** caused by:

* A missing `crc64_be` symbol
* A missing Rocky Linux kernel SRPM
* Failure of `rpmbuild -bp`
* A missing bcache source directory
* The `mockbuild` / `mock` user and group warnings in the log

To resolve the observed source preparation failure, the workflow must either switch to `drivers/md/bcache/bcache_ondisk.h` or support both header layouts conditionally. This also eliminates the subsequent failure caused by copying the obsolete header. Whether the module itself builds successfully afterward still requires verification with an actual build.

No commits, pushes, or repository modifications have been made.

[1]: https://github.com/aont/bcache-builder/blob/main/.github/workflows/build-bcache.yml "https://github.com/aont/bcache-builder/blob/main/.github/workflows/build-bcache.yml"
[2]: https://github.com/aont/bcache-builder/raw/refs/heads/main/6_Download%20and%20prepare%20matching%20Rocky%20kernel%20source.txt "https://github.com/aont/bcache-builder/raw/refs/heads/main/6_Download%20and%20prepare%20matching%20Rocky%20kernel%20source.txt"
[3]: https://download.rockylinux.org/pub/rocky/9/BaseOS/source/tree/Packages/k/ "https://download.rockylinux.org/pub/rocky/9/BaseOS/source/tree/Packages/k/"
[4]: https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/commit/1e4006f2decf7931bea9b74856acdbe84389fca8 "https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/commit/1e4006f2decf7931bea9b74856acdbe84389fca8"
[5]: https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/drivers/md/bcache/bcache.h "https://gitlab.com/redhat/centos-stream/src/kernel/centos-stream-9/-/raw/kernel-5.14.0-687.el9/drivers/md/bcache/bcache.h"
