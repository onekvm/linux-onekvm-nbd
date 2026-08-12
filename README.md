# linux-onekvm-nbd

OneKVM's optimized network block device client for browser-backed virtual
media. This driver is not SG2002 hardware-specific; it is a Linux block
driver optimization used by OneKVM on SG2002 NanoKVM systems and can be
built for any compatible Linux 5.10 target.

This is an out-of-tree Linux 5.10 kernel module. It keeps the standard Linux
NBD ioctl and netlink wire/userspace protocol, but uses the separate
`onekvm_nbd` device and generic-netlink family names so it can coexist with
the upstream `nbd` driver.

## Differences from the upstream NBD driver

The implementation is based on the Linux 5.10 NBD driver, with these
OneKVM-specific changes:

- The module registers a dynamic block major and exposes `/dev/onekvm-nbd0`
  instead of claiming the upstream `nbd` major and `nbd0` name.
- The generic-netlink family and multicast group are named `onekvm_nbd` and
  `onekvm_nbd_mc`, avoiding collisions with the upstream `nbd` family.
- It defaults to one device with no partition minors, matching the single
  browser-backed virtual-media session used by NanoKVM.
- It adds an adaptive read-cache ceiling (`max_cache_kb`, 128--4096 KiB,
  power-of-two values) and exposes it through the device's sysfs attribute.
  Changing it updates the request queue and read-ahead settings safely while
  the queue is frozen.
- It exposes `io_depth` (1--1024, default 32) to control the number of
  requests that may be in flight per device, allowing userspace to select a
  suitable amount of request-level asynchronous I/O.
- Adaptive read-ahead starts conservatively and is enabled after
  `adaptive_read_threshold_kb` contiguous read data (default 16384 KiB).
- Receive workqueues are unbound and lowered to the lowest normal priority,
  preventing NBD network processing from competing with latency-sensitive
  video/input work.
- Debugfs state is placed below `onekvm_nbd` rather than `nbd`.

The NBD protocol, request/reply format, ioctl ABI, and netlink attributes are
otherwise intentionally kept compatible with Linux 5.10 NBD userspace.

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
