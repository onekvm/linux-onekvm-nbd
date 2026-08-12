# linux-onekvm-nbd

OneKVM 优化版网络块设备（NBD）客户端，面向浏览器虚拟介质场景。

本驱动并非 SG2002 硬件专用，而是 OneKVM 针对虚拟介质访问路径做的
Linux 块设备优化。在 SG2002 NanoKVM 上使用，也可以为其他兼容 Linux
5.10 的目标平台编译。

这是一个 Linux 5.10 树外内核模块。它保留 Linux NBD 标准 ioctl 和
netlink 协议，同时使用独立的 `onekvm_nbd` 设备名和 generic-netlink
family 名称，因此可以和原版 `nbd` 驱动并存。

## 与 Linux 原版 NBD 的区别

本实现基于 Linux 5.10 的 NBD 驱动，增加了以下 OneKVM 优化：

- 使用动态 block major，设备名为 `/dev/onekvm-nbd0`，不占用原版
  `nbd` major，也不使用 `nbd0` 名称。
- generic-netlink family 和 multicast group 分别使用 `onekvm_nbd`、
  `onekvm_nbd_mc`，避免和原版 `nbd` family 冲突。
- 默认只创建一个设备且不创建分区 minor，匹配 NanoKVM 单个浏览器虚拟
  介质会话的使用方式。
- 增加自适应读缓存上限 `max_cache_kb`，支持 128--4096 KiB 的 2 的幂值，
  并通过设备 sysfs 属性提供运行时调整。修改时会先冻结 request queue，
  同步更新队列限制和 read-ahead 参数。
- 接收 workqueue 使用 unbound 配置，并降低到普通任务的最低优先级，减少
  NBD 网络处理对视频和输入等低延迟任务的影响。
- debugfs 状态放在 `onekvm_nbd` 目录下，而不是原版的 `nbd` 目录。

NBD 的网络协议、request/reply 格式、ioctl ABI 和 netlink 属性保持与
Linux 5.10 原版 userspace 的兼容性。

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
