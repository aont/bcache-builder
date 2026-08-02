# bcache-builder

> [!WARNING]
> It is currently not possible to build `bcache.ko` independently for the
> standard Rocky Linux 9 kernel with the approach in this repository. The
> kernel's `CONFIG_BCACHE=n` configuration creates an incompatible
> `task_struct` layout, and the remaining bcache source is incompatible with
> backported EL9 block-layer APIs. See [error_analysis.md](error_analysis.md)
> for the detailed investigation and possible alternatives.

The intended goal of this project is to build `bcache.ko` as an external
module for the standard Rocky Linux 9 x86_64 kernel without rebuilding the
kernel itself.

Rocky Linux 9's standard kernel may be configured with BCACHE disabled while
CRC64 is built in:

```text
# CONFIG_BCACHE is not set
CONFIG_CRC64=y
```

This project uses the matching Rocky Linux kernel source and `kernel-devel`
package to build only `bcache.ko`. The module relies on the kernel's exported
`crc64_be()` symbol, so it does not build or install a separate `crc64.ko`.

## Automated build

The workflow described below documents the attempted build process, but it
cannot currently produce a usable module because of the incompatibilities
described above.

The **Build and release bcache.ko** GitHub Actions workflow is designed to
build a module for a specified Rocky Linux 9 kernel release and publish it in
a new GitHub release.

1. Open the repository's **Actions** tab.
2. Select **Build and release bcache.ko**.
3. Choose **Run workflow**.
4. Enter the complete x86_64 kernel release reported by `uname -r`, for
   example `5.14.0-503.40.1.el9_5.x86_64`.
5. Download `bcache-<kernel-release>.ko` and its `.sha256` file from the
   release created by the workflow.

The workflow obtains the exactly matching Rocky Linux source RPM, applies its
distribution patches, builds against the matching `kernel-devel` tree, checks
the resulting module's CRC64 reference and `vermagic`, and generates a SHA-256
checksum.

## Manual build and installation

See [procedure.md](procedure.md) for the complete manual process, including:

- checking the target kernel configuration and exported symbols;
- obtaining and preparing the matching Rocky Linux source RPM;
- building and validating `bcache.ko`;
- signing the module when Secure Boot is enabled; and
- installing, loading, and troubleshooting the module.

## Important safety notes

- A kernel module must be built for the **exact** target kernel release. Do not
  use a module whose `vermagic` does not begin with the output of `uname -r`.
- Rebuild the module after every kernel update.
- Secure Boot systems require a module signed by a key trusted by that system.
- This repository covers building and loading the kernel module. It does not
  initialize block devices or write BCACHE metadata; those operations can
  destroy existing data.
- Never force-load a mismatched or unresolved module with `insmod -f` or
  `modprobe --force`.
