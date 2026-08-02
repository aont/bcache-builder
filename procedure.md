# Procedure for Building `bcache.ko` for the Rocky Linux 9 Standard Kernel

The following procedure assumes that you want to **build a `bcache.ko` module that can be loaded into the currently running Rocky Linux 9 standard kernel, without rebuilding the entire kernel**.

This procedure assumes that the standard kernel configuration is as follows:

```text
# CONFIG_BCACHE is not set
CONFIG_CRC64=y
```

When `CONFIG_CRC64=y`, CRC64 functionality is built into the kernel itself. Therefore, you do not need to build or install a custom `crc64.ko`.

The three key points are:

1. Use the `kernel-devel` package whose version exactly matches `uname -r`, along with its `Module.symvers`
2. Obtain the BCACHE source from the Source RPM that exactly matches the relevant Rocky Linux kernel
3. Use the kernel’s built-in `crc64_be()` implementation and build only `bcache.ko` as an external module

---

## Prerequisites

This procedure assumes the following:

* Rocky Linux 9
* Rocky Linux standard kernel
* The build target is the currently running kernel
* x86_64
* `CONFIG_BCACHE` is disabled
* `CONFIG_CRC64=y`
* The kernel itself will not be rebuilt
* The only module to be created is `bcache.ko`

First, set the variables.

```bash
KREL="$(uname -r)"
ARCH="$(uname -m)"
KDIR="/lib/modules/$KREL/build"

printf 'KREL=%s\nARCH=%s\nKDIR=%s\n' \
    "$KREL" \
    "$ARCH" \
    "$KDIR"
```

Example output:

```text
KREL=5.14.0-xxx.el9_x.x86_64
ARCH=x86_64
KDIR=/lib/modules/5.14.0-xxx.el9_x.x86_64/build
```

From this point onward, **do not mix in files built for any kernel whose `KREL` differs by even one character**.

---

# 1. Check Whether `bcache.ko` Is Already Provided

First, check whether the standard RPM packages already include `bcache.ko`.

```bash
modinfo -k "$KREL" bcache
```

If module information is displayed, the module is already provided and there is no need to build it yourself.

As an additional precaution, check the `kernel-modules-extra` package for the relevant kernel.

```bash
sudo dnf install -y "kernel-modules-extra-$KREL"
```

After installation, check again.

```bash
modinfo -k "$KREL" bcache
```

Next, check the standard kernel configuration.

```bash
CFG="/boot/config-$KREL"

grep -E \
'^(CONFIG_(BCACHE|CRC64|MODVERSIONS|MODULE_SIG_FORCE)=|# CONFIG_(BCACHE|CRC64|MODULE_SIG_FORCE) is not set)' \
"$CFG"
```

This procedure assumes output similar to the following:

```text
# CONFIG_BCACHE is not set
CONFIG_CRC64=y
CONFIG_MODVERSIONS=y
# CONFIG_MODULE_SIG_FORCE is not set
```

The results have the following meanings:

| Setting                      | Status                                                                  |
| ---------------------------- | ----------------------------------------------------------------------- |
| `CONFIG_BCACHE=m`            | The kernel is configured to provide `bcache.ko` for the standard kernel |
| `CONFIG_BCACHE=y`            | BCACHE is built into the kernel. `bcache.ko` is unnecessary             |
| `# CONFIG_BCACHE is not set` | Target configuration for this external module build                     |
| `CONFIG_CRC64=y`             | CRC64 is built into the kernel. `crc64.ko` is unnecessary               |
| `CONFIG_CRC64=m`             | CRC64 is provided as a separate module                                  |
| `# CONFIG_CRC64 is not set`  | Outside the scope of this procedure                                     |

This procedure applies to the following combination:

```text
# CONFIG_BCACHE is not set
CONFIG_CRC64=y
```

---

## Confirm That the CRC64 Symbol Is Provided by the Kernel

`bcache.ko` uses `crc64_be()`.

For the external module’s `modpost` stage to resolve this symbol, `crc64_be` must be registered in the `Module.symvers` file from the `kernel-devel` package that matches the running kernel.

After installing `kernel-devel`, confirm this with:

```bash
grep -w crc64_be "$KDIR/Module.symvers"
```

Expected example:

```text
0x........	crc64_be	vmlinux	EXPORT_SYMBOL_GPL
```

Depending on the kernel source or release, details such as the export type may differ slightly.

The important point is that the provider of `crc64_be` is registered as `vmlinux`.

---

# 2. Install the Exactly Matching `kernel-devel`

Install the required packages.

```bash
sudo dnf install -y \
    "kernel-devel-$KREL" \
    gcc \
    make \
    elfutils-libelf-devel \
    binutils \
    rpm-build \
    python3-devel \
    dnf-plugins-core
```

The RHEL 9 external module build procedure also uses `kernel-devel-$(uname -r)`, `gcc`, and `elfutils-libelf-devel` matching the running kernel. ([Red Hat Documentation][1])

Check the build directory.

```bash
readlink -f "$KDIR"
```

It will normally resemble the following:

```text
/usr/src/kernels/5.14.0-xxx.el9_x.x86_64
```

Check the files required to build an external module.

```bash
test -f "$KDIR/Makefile" &&
test -f "$KDIR/include/generated/autoconf.h" &&
test -s "$KDIR/Module.symvers" &&
echo "kernel-devel tree: OK"
```

You cannot proceed if `Module.symvers` does not exist or is empty.

```bash
ls -lh "$KDIR/Module.symvers"
```

Also check the CRC64 symbol.

```bash
grep -w crc64_be "$KDIR/Module.symvers"
```

If `crc64_be` cannot be found even though `CONFIG_CRC64=y`, do not continue building in that state. Confirm that the installed `kernel-devel` package exactly matches the running kernel.

```bash
rpm -q "kernel-devel-$KREL"
uname -r
```

---

## Do Not Run `modules_prepare`

With this method, you do not need to run the following commands against the downloaded source tree:

```bash
make prepare
make modules_prepare
```

The `kernel-devel` tree is already prepared for external module builds.

In addition, when `CONFIG_MODVERSIONS=y`, running `make modules_prepare` alone does not generate the `Module.symvers` file required for symbol versioning. The official Kbuild documentation also states that `modules_prepare` does not generate `Module.symvers`. ([Linux Kernel Documentation][2])

---

# 3. Obtain the Source RPM That Exactly Matches the Running Kernel

The Rocky Linux kernel contains distribution-specific modifications and backports.

Therefore, use the Rocky Linux Source RPM corresponding to the running kernel, rather than a generic `v5.14` source tree from kernel.org.

First, determine the Source RPM name referenced by the installed `kernel-core` package.

```bash
KCORE="kernel-core-$KREL"

rpm -q "$KCORE"
```

Then obtain the Source RPM name.

```bash
SRPM="$(rpm -q --qf '%{SOURCERPM}\n' "$KCORE")"

printf 'Source RPM: %s\n' "$SRPM"
```

Example:

```text
kernel-5.14.0-xxx.el9_x.src.rpm
```

Download the Source RPM.

```bash
DL="$HOME/rocky-kernel-srpm-$KREL"

mkdir -p "$DL"

dnf --enablerepo=baseos-source download \
    --source \
    --downloaddir "$DL" \
    "$KCORE"
```

Verify it.

```bash
ls -lh "$DL/$SRPM"
```

---

## If `baseos-source` Cannot Be Found

Check the available source repositories.

```bash
dnf repolist --all |
grep -i source
```

On Rocky Linux 9, the following repository ID is normally used:

```text
baseos-source
```

If the running kernel is old and its Source RPM is no longer available from the current BaseOS repository, obtain the **exactly matching `$SRPM`** shown earlier from the Rocky Linux Vault or an appropriate historical-release source repository.

Avoid substituting any of the following:

* The latest Rocky Linux kernel source
* Source with the same `5.14` version but a different release number
* kernel.org `v5.14`
* Source from a different CentOS Stream release
* A similar AlmaLinux or RHEL version

---

# 4. Extract the Source RPM and Apply the Rocky Linux Patches

Create a dedicated RPM build directory.

```bash
TOP="$HOME/rpmbuild-bcache-$KREL"

rm -rf "$TOP"

mkdir -p "$TOP"/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
```

Install the Source RPM contents.

```bash
rpm -ivh \
    --define "_topdir $TOP" \
    "$DL/$SRPM"
```

Locate the SPEC file.

```bash
SPEC="$(
    find "$TOP/SPECS" \
        -maxdepth 1 \
        -name '*.spec' \
        -print \
        -quit
)"

printf 'SPEC=%s\n' "$SPEC"

test -f "$SPEC"
```

Run only the `%prep` phase to extract the source and apply the Rocky Linux patches.

```bash
rpmbuild -bp \
    --nodeps \
    --define "_topdir $TOP" \
    --target "$ARCH" \
    "$SPEC"
```

This does not compile the kernel. It only extracts the source and applies the patches.

The kernel SPEC uses `pathfix.py` during `%prep`. This command is provided by the `python3-devel` package installed above. If `%prep` reports `pathfix.py: command not found`, install `python3-devel` before retrying.

If `rpmbuild -bp` fails because of other missing commands or BuildRequires dependencies, install the build dependencies.

```bash
sudo dnf builddep -y "$SPEC"
```

Then run it again.

```bash
rpmbuild -bp \
    --define "_topdir $TOP" \
    --target "$ARCH" \
    "$SPEC"
```

Locate the extracted kernel source.

```bash
BCACHE_MAKEFILE="$(
    find "$TOP/BUILD" \
        -type f \
        -path '*/drivers/md/bcache/Makefile' \
        -print \
        -quit
)"

test -n "$BCACHE_MAKEFILE"

KSRC="${BCACHE_MAKEFILE%/drivers/md/bcache/Makefile}"

printf 'KSRC=%s\n' "$KSRC"
```

Check the required files.

```bash
test -f "$KSRC/drivers/md/bcache/Kconfig"
test -f "$KSRC/drivers/md/bcache/Makefile"
test -f "$KSRC/include/linux/crc64.h"
test -f "$KSRC/include/uapi/linux/bcache.h"
test -f "$KSRC/include/trace/events/bcache.h"

echo "required sources: OK"
```

Because the CRC64 implementation is built into the kernel, you do not need to copy the following files into the working directory:

```text
lib/crc64.c
lib/gen_crc64table.c
```

You also do not need to generate `crc64table.h`.

---

# 5. Create a Working Directory for the External Module

Create a dedicated build directory.

```bash
WORK="$HOME/bcache-kmod-$KREL"

rm -rf "$WORK"

mkdir -p \
    "$WORK/include/linux" \
    "$WORK/include/uapi/linux" \
    "$WORK/include/trace/events"
```

Copy the complete BCACHE source directory.

```bash
cp -a \
    "$KSRC/drivers/md/bcache" \
    "$WORK/"
```

Copy the required headers.

```bash
cp \
    "$KSRC/include/linux/crc64.h" \
    "$WORK/include/linux/"

cp \
    "$KSRC/include/uapi/linux/bcache.h" \
    "$WORK/include/uapi/linux/"

cp \
    "$KSRC/include/trace/events/bcache.h" \
    "$WORK/include/trace/events/"
```

Check the directory structure.

```bash
find "$WORK" \
    -maxdepth 4 \
    -type f |
sort |
head -100
```

The working directory should contain approximately the following:

```text
bcache/
include/linux/crc64.h
include/uapi/linux/bcache.h
include/trace/events/bcache.h
```

Do not include the CRC64 C source files or generation tools.

---

# 6. Create the Kbuild File for `bcache.ko`

Create `$WORK/Kbuild`.

```bash
cat > "$WORK/Kbuild" <<'EOF'
# Build only bcache.ko as an external module
obj-m += bcache/

# Prefer headers copied from the exactly matching Rocky Linux kernel source
ccflags-y += -I$(src)/include
ccflags-y += -I$(src)/include/uapi

subdir-ccflags-y += -I$(src)/include
subdir-ccflags-y += -I$(src)/include/uapi
EOF
```

Verify it.

```bash
cat "$WORK/Kbuild"
```

The copied original Makefile is used in the BCACHE directory.

```bash
cat "$WORK/bcache/Makefile"
```

It should resemble the following:

```make
obj-$(CONFIG_BCACHE) += bcache.o

bcache-y := alloc.o bset.o btree.o closure.o debug.o extents.o \
            io.o journal.o movinggc.o request.o stats.o super.o sysfs.o \
            trace.o util.o writeback.o features.o
```

---

## Why CRC64 Is Not Built Alongside BCACHE

The standard kernel has the following configuration:

```text
CONFIG_CRC64=y
```

In this case, `crc64_be()` is provided by `vmlinux`.

During the external module build, `modpost` refers to the following file, which must exactly match the running kernel:

```text
$KDIR/Module.symvers
```

If this file registers `crc64_be` as a symbol exported by `vmlinux`, the symbol reference in `bcache.ko` is resolved against it.

Therefore, none of the following are required:

* Copying `crc64.c`
* Compiling `gen_crc64table`
* Generating `crc64table.h`
* Building `crc64.ko`
* Specifying `KBUILD_EXTRA_SYMBOLS`
* Running a combined `modpost` for `crc64.ko` and `bcache.ko`
* Installing or loading `crc64.ko`

---

# 7. Build Only `bcache.ko`

Run the following:

```bash
make -C "$KDIR" \
    M="$WORK" \
    CONFIG_BCACHE=m \
    -j"$(nproc)" \
    modules
```

You may explicitly pass the existing CRC64 configuration as follows:

```bash
make -C "$KDIR" \
    M="$WORK" \
    CONFIG_BCACHE=m \
    CONFIG_CRC64=y \
    -j"$(nproc)" \
    modules
```

However, the generated kernel configuration in `kernel-devel` normally already contains `CONFIG_CRC64=y`, so explicitly specifying `CONFIG_CRC64=y` is unnecessary.

This process compiles approximately only the following:

* The required `.c` files under BCACHE
* The module-specific `.mod.c` file
* `bcache.ko`

It does not build the kernel image, the CRC64 implementation, or any other kernel modules.

The standard external-module build form is as follows. ([Linux Kernel Documentation][2])

```text
make -C /lib/modules/$(uname -r)/build M=<module-directory> modules
```

---

## Verify the Generated Files

```bash
ls -lh \
    "$WORK/bcache/bcache.ko" \
    "$WORK/Module.symvers"
```

The primary generated files are:

```text
$WORK/bcache/bcache.ko
$WORK/Module.symvers
```

No `crc64.ko` is generated.

Verify this.

```bash
test -f "$WORK/bcache/bcache.ko"
test ! -f "$WORK/crc64.ko"

echo "bcache-only build: OK"
```

---

# 8. Verify CRC64 Symbol Resolution

Confirm again that the standard kernel’s `Module.symvers` contains `crc64_be`.

```bash
grep -w crc64_be "$KDIR/Module.symvers"
```

Expected example:

```text
0x........	crc64_be	vmlinux	EXPORT_SYMBOL_GPL
```

You can also confirm that `bcache.ko` references `crc64_be`.

```bash
nm -u "$WORK/bcache/bcache.ko" |
grep -w crc64_be
```

Expected example:

```text
U crc64_be
```

This is not an error.

Within the external module, `crc64_be` remains an undefined symbol and is resolved against the kernel’s exported symbol when the module is loaded.

---

## Check Module Dependencies

```bash
modinfo -F depends "$WORK/bcache/bcache.ko"
```

When `CONFIG_CRC64=y`, CRC64 is built into the kernel rather than provided as a module.

Therefore, `crc64` will normally not appear in the dependency list.

The expected result is either an empty field or a list containing only dependencies unrelated to CRC64.

You should not expect output such as:

```text
crc64
```

A `crc64` dependency normally appears only when CRC64 itself is built as a separate module.

---

# 9. Check `vermagic`

Check the generated `bcache.ko` module’s `vermagic`.

```bash
modinfo -F vermagic "$WORK/bcache/bcache.ko"
```

Its beginning must exactly match:

```bash
uname -r
```

For easier comparison, run:

```bash
printf 'running kernel : %s\n' "$KREL"

printf 'bcache vermagic: %s\n' \
    "$(modinfo -F vermagic "$WORK/bcache/bcache.ko")"
```

Example:

```text
running kernel : 5.14.0-xxx.el9_x.x86_64
bcache vermagic: 5.14.0-xxx.el9_x.x86_64 SMP preempt mod_unload modversions ...
```

Do not install the module if the kernel release at the beginning does not match.

---

# 10. About BTF Warnings

If the only message is similar to the following, it is usually not a fatal error:

```text
Skipping BTF generation for ... due to unavailability of vmlinux
```

The RHEL 9 external module build example also successfully generates a `.ko` file while displaying this message. ([Red Hat Documentation][1])

You may proceed if the following file was generated:

```bash
test -f "$WORK/bcache/bcache.ko"
```

However, if there are compilation errors or `modpost` errors in addition to the BTF warning, do not ignore those errors.

---

# 11. Check Secure Boot

Check the Secure Boot state.

```bash
mokutil --sb-state
```

You can also inspect the kernel command line.

```bash
cat /proc/cmdline
```

Module signing is required if any of the following apply:

* Secure Boot is enabled
* The kernel command line contains `module.sig_enforce=1`
* Signature enforcement is enabled in the kernel configuration

When Secure Boot is enabled, the kernel loads only modules signed with a trusted key.

On RHEL 9, you can enroll a public key in MOK and sign the module using the `scripts/sign-file` utility included in `kernel-devel`. ([Red Hat Documentation][3])

---

## If Secure Boot Is Disabled

Proceed to the installation step without signing the module.

Loading an unsigned external module may taint the kernel.

---

## If Secure Boot Is Enabled

The following assumes that you already have a key enrolled in MOK.

Example:

```bash
SIGN_KEY="/root/module-signing/sb_cert.priv"
SIGN_CERT="/root/module-signing/sb_cert.cer"
```

Sign only `bcache.ko`.

```bash
sudo "$KDIR/scripts/sign-file" \
    sha256 \
    "$SIGN_KEY" \
    "$SIGN_CERT" \
    "$WORK/bcache/bcache.ko"
```

Check the signer information.

```bash
modinfo "$WORK/bcache/bcache.ko" |
grep -E '^(signer|sig_key|sig_hashalgo):'
```

Important points:

* Sign only `bcache.ko`
* Do not run `strip` after signing
* Do not relink the module after signing
* If you rebuild the module, sign it again
* The module cannot be loaded unless the public key is enrolled in MOK
* CRC64 is built into the kernel, so no additional CRC64 signature is required

---

# 12. Install `bcache.ko`

Create the installation directory.

```bash
DEST="/lib/modules/$KREL/extra/bcache"

sudo install -d -m 0755 "$DEST"
```

Copy only `bcache.ko`.

```bash
sudo install -m 0644 \
    "$WORK/bcache/bcache.ko" \
    "$DEST/bcache.ko"
```

Restore the SELinux context.

```bash
sudo restorecon -RF "$DEST"
```

Verify the contents.

```bash
ls -lh "$DEST"
```

Expected contents:

```text
bcache.ko
```

Do not place a custom `crc64.ko` in this directory.

CRC64 is built into the standard kernel.

---

# 13. Run `depmod`

```bash
sudo depmod -a "$KREL"
```

This scans the files under `/lib/modules/$KREL/` and updates indexes such as:

```text
modules.dep
modules.alias
modules.symbols
modules.dep.bin
modules.alias.bin
```

The RHEL external module procedure also runs `depmod -a` after installing the module and then loads it with `modprobe`. ([Red Hat Documentation][1])

Check the recognized module path.

```bash
modinfo -k "$KREL" -F filename bcache
```

Expected output:

```text
/lib/modules/5.14.0-.../extra/bcache/bcache.ko
```

Because CRC64 is not a module, it is not a problem if the following command fails:

```bash
modinfo -k "$KREL" crc64
```

Functionality configured with `CONFIG_CRC64=y` is built into the kernel and does not necessarily exist as a `crc64.ko` file or as a target for `modinfo crc64`.

---

# 14. Inspect What Will Be Loaded

Check what `modprobe` will do when loading `bcache`.

```bash
modprobe --show-depends bcache
```

Because `CONFIG_CRC64=y`, the output should generally show only `bcache.ko`.

```text
insmod /lib/modules/.../extra/bcache/bcache.ko
```

No `insmod` command for `crc64.ko` will be displayed.

The following output is not expected with this procedure:

```text
insmod /lib/modules/.../crc64.ko
insmod /lib/modules/.../bcache.ko
```

CRC64 functionality is already available from the kernel itself.

---

# 15. Load `bcache.ko`

Load the BCACHE module.

```bash
sudo modprobe -v bcache
```

There is no need to load CRC64 separately.

Do not run:

```bash
sudo modprobe crc64
```

Check the loaded module.

```bash
lsmod |
grep -E '^bcache\b'
```

Expected example:

```text
bcache    ...
```

Because CRC64 is built into the kernel, it will normally not appear in `lsmod`.

Check the BCACHE sysfs directory.

```bash
ls -ld /sys/fs/bcache
```

Check the kernel log.

```bash
sudo journalctl -k -b --no-pager |
tail -n 100
```

Alternatively:

```bash
sudo dmesg |
tail -n 100
```

You do not need to register a block device with BCACHE merely to complete these checks.

---

# 16. Unload the Module

If no BCACHE device has been registered yet, you can unload the module with:

```bash
sudo modprobe -r bcache
```

Verify it.

```bash
lsmod |
grep -E '^bcache\b'
```

CRC64 is part of the kernel itself and cannot be unloaded.

Do not run:

```bash
sudo modprobe -r crc64
```

If a BCACHE device is in use, unloading `bcache` will fail. Do not force the module to unload in that situation.

---

# 17. Common Errors

| Error                                     | Primary cause                                                                                     | Resolution                                                                                                                    |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `modpost: "crc64_be" undefined`           | The `Module.symvers` being used does not contain the kernel’s `crc64_be` symbol                   | Check `CONFIG_CRC64=y` and `$KDIR/Module.symvers`, and use the `kernel-devel` package that exactly matches the running kernel |
| `Unknown symbol crc64_be`                 | The build-time and load-time kernels differ, or the symbol versions do not match                  | Recheck `uname -r`, `vermagic`, `kernel-devel`, and `Module.symvers`                                                          |
| `Invalid module format`                   | Mismatch in `vermagic`, configuration, compiler conditions, kernel release, or similar properties | Compare `uname -r` with `modinfo -F vermagic`                                                                                 |
| `disagrees about version of symbol`       | `Module.symvers` from a different kernel was used                                                 | Use the `kernel-devel` package that exactly matches the running kernel                                                        |
| `Key was rejected by service`             | The module is unsigned for Secure Boot, or the signing key is not trusted                         | Sign `bcache.ko` with a key enrolled in MOK                                                                                   |
| `Module bcache not found`                 | There is a problem with the installation path or `depmod`                                         | Check `/lib/modules/$KREL/extra/bcache/` and run `depmod -a "$KREL"`                                                          |
| `Skipping BTF generation...`              | `vmlinux` is unavailable                                                                          | Normally only a warning if `bcache.ko` was generated                                                                          |
| `modprobe: FATAL: Module crc64 not found` | An attempt was made to load CRC64 as a module even though `CONFIG_CRC64=y`                        | CRC64 is built into the kernel, so do not run `modprobe crc64`                                                                |
| `crc64` does not appear in `lsmod`        | CRC64 is built into the kernel                                                                    | Normal. Functionality configured with `CONFIG_CRC64=y` does not appear in `lsmod`                                             |

Check detailed kernel errors with:

```bash
sudo journalctl -k -b --no-pager |
grep -Ei 'bcache|crc64|module|symbol|vermagic|signature|key'
```

Do not use the following workarounds:

```text
insmod -f
modprobe --force
Ignoring unresolved symbols with KBUILD_MODPOST_WARN=1
Editing vermagic with a binary editor
Building a custom crc64.ko and adding it to the standard kernel
```

Even if the build appears to succeed, these methods eliminate any assurance that the module can operate safely as a storage module.

---

# 18. Handling Kernel Updates

If Rocky Linux updates the kernel and the following value changes:

```bash
uname -r
```

Repeat all of the following for the new kernel:

* Install the exactly matching `kernel-devel`
* Confirm that the new kernel still has `CONFIG_CRC64=y`
* Confirm that the new `$KDIR/Module.symvers` contains `crc64_be`
* Obtain the exactly matching Source RPM
* Rebuild `bcache.ko`
* Sign `bcache.ko` again in a Secure Boot environment
* Install it under the new `/lib/modules/<new-KREL>/extra/bcache/`
* Run `depmod -a <new-KREL>`

Do not copy a `bcache.ko` built for an older kernel into the directory for a newer kernel.

Also, if the state of `CONFIG_CRC64` changes after a kernel update, this procedure may no longer apply unchanged.

Always check:

```bash
grep -E \
'^(CONFIG_(BCACHE|CRC64)=|# CONFIG_(BCACHE|CRC64) is not set)' \
"/boot/config-$(uname -r)"
```

---

# Overall Process

```text
Exactly matching Rocky Linux Source RPM
        │
        └── drivers/md/bcache/*
                  │
                  ▼
/lib/modules/$(uname -r)/build
        │
        ├── Standard kernel configuration
        │       └── CONFIG_CRC64=y
        ├── Generated headers
        └── Standard kernel Module.symvers
                └── crc64_be is provided by vmlinux
                  │
                  ▼
Build only BCACHE as an external module
        │
        └── bcache/bcache.ko
                  │
                  ▼
Sign bcache.ko if Secure Boot is enabled
                  │
                  ▼
/lib/modules/$(uname -r)/extra/bcache/bcache.ko
                  │
                  ▼
depmod -a
                  │
                  ▼
modprobe bcache
        │
        ├── Load bcache.ko
        └── Use the CRC64 functionality built into the kernel
```

This method creates only the required `bcache.ko` without modifying the standard kernel’s `.config` or rebuilding the kernel itself.

Because CRC64 is built into the kernel through `CONFIG_CRC64=y`, none of the following are required:

* Building `crc64.ko`
* Signing `crc64.ko`
* Installing `crc64.ko`
* Creating a dependency between `bcache.ko` and `crc64.ko` with `depmod`
* Running `modprobe crc64`
* Running `modprobe -r crc64`

Note that verifying that the module can be loaded is separate from writing BCACHE metadata to an actual device.

Device initialization with tools such as `make-bcache` can destroy existing data, so it is not included in this module-loading verification procedure.

[1]: https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/proc_compiling-custom-kernel-modules_managing-kernel-modules
[2]: https://docs.kernel.org/kbuild/modules.html
[3]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/signing-a-kernel-and-modules-for-secure-boot_assembly_managing-kernel-command-line-parameters-with-uki
