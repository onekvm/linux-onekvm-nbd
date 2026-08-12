# linux-sg2002-nbd

OneKVM's dedicated network block device client for browser-backed virtual
media on NanoKVM.

This is an out-of-tree Linux 5.10 kernel module. It intentionally uses the
Linux NBD userspace ABI and netlink protocol while exposing the device as
`/dev/onekvm-nbd0` and providing OneKVM-specific adaptive read-cache controls.

## Build

```sh
make -C /path/to/linux \
    M="$PWD" \
    ARCH=riscv \
    CROSS_COMPILE=riscv64-linux-gnu- \
    modules
```

The kernel must export the workqueue attribute helpers used by this module:
`alloc_workqueue_attrs`, `apply_workqueue_attrs`, and `free_workqueue_attrs`.
OneKVM's NanoKVM kernel carries those exports in its local kernel patch set.

## License

GPL-2.0-or-later. See the SPDX identifier in `onekvm_nbd.c`.
