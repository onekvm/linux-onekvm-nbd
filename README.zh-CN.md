# linux-onekvm-nbd

OneKVM 面向浏览器虚拟介质的优化版网络块设备（NBD）客户端。

本驱动不是 SG2002 硬件专用，而是 OneKVM 针对虚拟介质访问路径做的 Linux
块设备优化。在 SG2002 NanoKVM 上使用，也可以为其他兼容 Linux 5.10 的目标
平台编译。

这是一个 Linux 5.10 树外内核模块。它保留 Linux NBD 标准 ioctl 和 netlink
协议，同时使用独立的 `onekvm_nbd` 设备名和 generic-netlink family 名称，
因此可以和原版 `nbd` 驱动并存。

## 与 Linux 原版 NBD 的区别

本实现基于 Linux 5.10 的 NBD 驱动，增加了以下 OneKVM 优化：

- 使用动态 block major，设备名为 `/dev/onekvm-nbd0`，不占用原版 `nbd`
  major，也不使用 `nbd0` 名称。
- generic-netlink family 和 multicast group 分别使用 `onekvm_nbd`、
  `onekvm_nbd_mc`，避免和原版 `nbd` family 冲突。
- 默认只创建一个设备且不创建分区 minor，匹配 NanoKVM 单个浏览器虚拟介质
  会话的使用方式。
- 增加自适应读缓存上限 `max_cache_kb`，支持 128--4096 KiB 的 2 的幂值，
  并通过设备 sysfs 属性提供运行时调整。修改时会先冻结 request queue，
  同步更新队列限制和 read-ahead 参数。
- 增加 `io_depth` 模块参数（范围 1--1024，默认 128），用于控制每个设备
  允许同时在途的请求数量，从而支持按需调整请求级异步 I/O 深度。
- 接收 workqueue 使用 unbound 配置，并降低到普通任务的最低优先级，减少
  NBD 网络处理对视频和输入等低延迟任务的影响。
- debugfs 状态放在 `onekvm_nbd` 目录下，而不是原版的 `nbd` 目录。

NBD 的网络协议、request/reply 格式、ioctl ABI 和 netlink 属性保持与 Linux
5.10 原版 userspace 的兼容性。

## 编译

针对已经配置好的 Linux 内核树编译模块：

```sh
make -C /path/to/linux \
    M="$PWD" \
    ARCH=riscv \
    CROSS_COMPILE=riscv64-linux-gnu- \
    modules
```

内核必须导出本模块使用的 workqueue 属性辅助函数：
`alloc_workqueue_attrs`、`apply_workqueue_attrs` 和 `free_workqueue_attrs`。
OneKVM NanoKVM 内核的本地补丁集中包含这些导出符号。

## 许可证

GPL-2.0-or-later，详见 `onekvm_nbd.c` 中的 SPDX 标识。
