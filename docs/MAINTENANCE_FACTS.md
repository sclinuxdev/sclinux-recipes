# SCLinux Recipes 维护事实与架构设计规范 (Maintenance Facts & Architecture Decisions)

本文档记录了 **SCLinux 官方配方仓库 (`sclinux-recipes`)** 在依赖拓扑、子包切分粒度、系统冲突规避、双 Init 生态隔离及自举流水线中的核心维护事实与技术决策。

---

## 1. 单配方多子包拆分准则 (Granular Subpackage Splitting)

SCLinux 坚持“**单次密闭编译，声明式多包切分 (`[[subpackages]]`)**”原则。一个配方产物在 DESTDIR 中必须严格按照职责与使用场景细化，杜绝在生产 rootfs 中混入编译期依赖或非必需的辅助工具。

### 1.1 核心三段式切分规范
对于所有提供动态链接库与头文件的基础组件，必须执行三段式解耦：
1. **`*-libs`（运行时共享库）**：
   - 仅包含动态链接库（`usr/lib/*.so.*`）以及核心运行时数据（如 `ncurses` 的 terminfo 数据库）。
   - **依赖要求**：仅依赖 `virtual/libc` 或必要的同级 `*-libs`。严禁依赖编译器、Python 或任何命令行 CLI 工具。
   - **符号声明**：必须导出精确的 `so:lib<name>.so.<version>` 符号以及 `virtual/*-libs`。
2. **`*-dev`（开发与构建接口）**：
   - 包含头文件（`usr/include/**`）、动态库软链接（`usr/lib/*.so`）、静态库（`usr/lib/*.a`）与元数据（`usr/lib/pkgconfig/*.pc`）。
   - **依赖要求**：依赖对应的 `*-libs = <version>-<release>`。
   - **符号声明**：必须导出 `virtual/*-dev`。
3. **主包（命令行可执行工具集）**：
   - 通过 `default = "remaining"` 认领剩余的 CLI 工具、文档与服务单元。
   - 依赖对应的 `*-libs`，按需提供 `virtual/*` 抽象符号。

### 1.2 关键复合组件拆分事实

#### A. `system/systemd`（12 核心子包解耦）
为了实现极速启动与云原生/嵌入式场景的按需组装，`systemd`（261.2-1）被严格拆分为 12 个独立子包：
- **`systemd-libs`**：收纳 `libsystemd.so.*`、`libudev.so.*` 以及私有运行时库 `usr/lib/systemd/libsystemd-shared-*.so`，供所有子进程独立链接。
- **`systemd-udev`**：独立设备管理守护进程（`udevd`、`udevadm`、hwdb 与 rules），提供 `virtual/udev`。
- **`systemd-journald`**：独立日志收集与查询（`journalctl`、`systemd-journald`），提供 `virtual/journald`。
- **`systemd-logind`**：用户会话、多席位、合盖与电源事件管理（`loginctl`），提供 `virtual/logind`。
- **`systemd-sysvcompat`**：SysVinit 传统兼容层软链接（`/sbin/init`、`reboot`、`shutdown`、`poweroff`、`halt`），提供 `virtual/sysvcompat`。
- **`systemd-homed`**：便携家目录与现代 JSON 用户记录服务（`homectl`、`systemd-userdbd`）。
- **`systemd-networkd`**：独立 L2/L3 网络管理守护进程（`networkctl`）。
- **`systemd-resolved`**：独立 L7 DNS 缓存、本地存根与 DNSSEC 解析器（`resolvectl`）。
- **`systemd-timesyncd`**：轻量 SNTP 时钟同步客户端（`timedatectl`），提供 `virtual/ntp-client`。
- **`systemd-boot`**：UEFI 引导加载器与 `bootctl` 管理工具，提供 `virtual/bootloader`。
- **`systemd-container`**：轻量级容器运行时（`systemd-nspawn`、`machined`、`machinectl`）。
- **`systemd-dev`**：开发头文件与 pkg-config 接口，提供 `virtual/systemd-dev`。

#### B. `security/shadow`（三段式拆分）
- **`shadow-libs`**：收纳 `usr/lib/libsubid.so.*`，提供 `so:libsubid.so.5` 与 `virtual/shadow-libs`。满足容器运行时（Podman、rootless 容器等）仅需分配从属 UID/GID 的最小依赖。
- **`shadow-dev`**：收纳 `usr/include/subid.h`、`libsubid.so`，提供 `virtual/shadow-dev`。
- **`shadow`（主包）**：收纳 `useradd`、`userdel`、`passwd`、`su`、`login`、`chage` 等全套管理工具。

#### C. `net/iproute2`（开发接口分离）
- **`iproute2-dev`**：收纳 `usr/include/iproute2/**`，提供 `virtual/iproute2-dev`，专供需要编写 TC（Traffic Control）或 eBPF 网络过滤器的上层程序引用。
- **`iproute2`（主包）**：保留 `ip`、`ss`、`bridge`、`tc` 等核心网络诊断工具。

#### D. `devel/gcc`（原生 Slot 与通道隔离）
- **`channel = "gcc16"`, `slot = "16"`**：主编译器归入专有通道，原生支持未来多版本 GCC 并行安装。
- **`gcc-libs`**：生成的运行时核心库（`libstdc++.so.6`、`libgcc_s.so.1`、`libgomp.so.1`、`libatomic.so.1`）归入 **`system` 通道**（`slot = "16"`），使普通程序无需切换通道即可链接最新 C/C++ 运行时。

---

## 2. 核心网络解耦事实：为什么 `systemd-networkd` 绝不依赖 `systemd-resolved`

在 SCLinux 的配方设计中，`systemd-networkd` 与 `systemd-resolved` 保持完全解耦：

1. **二进制与符号链接无依赖（ELF Zero Linkage）**：
   `systemd-networkd` 二进制仅链接 `libc.so`、`libsystemd-shared-*.so`、`libcap.so`，与 `systemd-resolved` 没有任何动态符号链接。
2. **协议栈职责正交（L2/L3 vs L7）**：
   `systemd-networkd` 负责链路与网络层（IP 分配、DHCP 客户端、VLAN、路由、WireGuard）；`systemd-resolved` 负责应用层名称解析与 DNS 缓存。
3. **IPC 通信为弱耦合/可选通知（Optional Event Notification）**：
   `networkd` 在 DHCP 租约中获取 DNS 服务器后，仅在检测到总线存在 `org.freedesktop.resolve1` 时才异步发送 D-Bus/Varlink 通知。如果 `resolved` 未安装或未运行，`networkd` 直接静默忽略，网卡与路由完全正常工作。
4. **工业生产关键场景**：
   - **本地自建 DNS 服务（AdGuard Home / Pi-hole / CoreDNS / Unbound / BIND）**：这类服务必须监听 `0.0.0.0:53`，而 `systemd-resolved` 默认的本地存根监听在 `127.0.0.53:53` 极易引发端口冲突与递归死循环。业界标准实践是：**保留 `systemd-networkd` 配置网络，坚决不安装或彻底禁用 `systemd-resolved`**。
   - **极简容器与 MicroVM**：直接挂载宿主机 `/etc/resolv.conf`，完全不需要额外的 DNS 缓存守护进程。

---

## 3. 双 Init 与设备管理生态互斥事实 (Systemd vs Loom / Eudev)

### 3.1 为什么 `systemd` 绝不能使用 `eudev`？
1. **历史渊源**：Gentoo 社区正是为了让非 systemd 系统（OpenRC, runit, sysvinit）摆脱 systemd 捆绑，才 fork 出了 `eudev`。
2. **`.device` 动态单元合成机制**：`systemd` 作为 PID 1 在接收内核 uevent 时，强依赖 `systemd-udevd` 在规则中打上的私有标记（`TAG+="systemd"`、`ENV{SYSTEMD_WANTS}`、`SYSTEMD_ALIAS`）。PID 1 在内存中据此动态生成 `.device` 虚拟单元。`eudev` 根本没有这套私有机制。
3. **90 秒启动挂死灾难（The 90s Boot Hang）**：如果强行混用，挂载单元（`.mount`）在启动时将永远等不到对应的 `/dev/disk/by-uuid/...` 设备就绪，并在 90 秒倒计时超时后报错，**直接跌入 Emergency Mode 急救 Shell，导致系统启动失败**。
4. **Socket 激活协议绑定**：`systemd` 早期引导阶段通过 `LISTEN_FDS` 向 udevd 传递控制套接字，`eudev` 不支持此机制。

### 3.2 声明式生态隔离
SCLinux 明确隔离了两套开箱即用的系统运行时生态：

| 生态特性 | 生态 A：现代全功能系统 | 生态 B：极简传统 UNIX 系统 |
| :--- | :--- | :--- |
| **Init 系统 (`virtual/init`)** | `systemd` | `loom` |
| **设备管理 (`virtual/udev`)** | `systemd-udev` | `eudev` |
| **日志服务 (`virtual/journald`)** | `systemd-journald` | 标准终端 / kmsg |
| **会话管理 (`virtual/logind`)** | `systemd-logind` | 无 (标准 PAM/agetty) |
| **互斥声明 (`conflicts`)** | `conflicts = ["loom", "eudev"]` | `conflicts = ["systemd", "systemd-udev"]` |

**双层拦截保护**：
- **第一道防线（PubGrub 求解器）**：在用户于 `/etc/sage/system.toml` 声明配置时，求解器在内存推导阶段命中互斥规则，直接驳回并输出因果树，零磁盘开销。
- **第二道防线（LMDB 预检状态机）**：在事务落地前，LMDB 全局文件反向所有权索引捕获 `/sbin/init`、`/usr/bin/udevadm`、`/usr/lib/libudev.so.1` 的路径碰撞，触发 Fail-Closed 原子回滚。

---

## 4. 全局命令与文件所有权防冲突矩阵 (Global Conflict Matrix)

Linux 历史上存在多个软件包争抢同一标准命令所有权的情况。SCLinux 配方在编译参数（`configure_args`）中进行了严格的边界割接，杜绝任何文件碰撞：

| 争议命令 / 库 | 候选上游 | 权威归属包 | 编译期规避参数 |
| :--- | :--- | :--- | :--- |
| **`kill`** | coreutils / util-linux / procps-ng | **`util-linux`** | `coreutils`: `--enable-no-install-program=kill`<br>`procps-ng`: `--disable-kill` |
| **`uptime`** | coreutils / procps-ng | **`procps-ng`** | `coreutils`: `--enable-no-install-program=uptime` |
| **`login`, `su`, `nologin`** | util-linux / shadow | **`shadow`** | `util-linux`: `--disable-login --disable-nologin --disable-su --disable-runuser` |
| **`libuuid`, `libblkid`** | e2fsprogs / util-linux | **`util-linux-libs`** | `e2fsprogs`: `--disable-libuuid --disable-libblkid` |
| **`/sbin/init`** | systemd / loom | **互斥单选** | `systemd` 与 `loom` 双向声明 `conflicts`，由 `virtual/init` 约束 |
| **`udev`** | systemd-udev / eudev | **互斥单选** | `systemd-udev` 与 `eudev` 双向声明 `conflicts`，由 `virtual/udev` 约束 |

---

## 5. 自举计划与拓扑排序维护事实 (Bootstrap & DAG Topology)

### 5.1 自举环切断事实 (Bootstrap Cycle Breaking)
- **现象**：标准 Autotools 构建类（`rclass/autotools.toml`）通常会隐式注入 `implicit_build_dependencies = ["system/make"]`。
- **死锁环**：若 `devel/make` 或 `system/glibc` 自身继承 `autotools`，将在图计算中产生直接自锁（`make -> make`）或自举环（`glibc -> make -> glibc`）。
- **决策事实**：`make`、`glibc`、`pkgconf` 必须使用 `inherit = ["custom"]`，手动编写密闭的自举脚本（例如利用 make 自带的 `./build.sh` 自建二进制），从而切断依赖死锁。

### 5.2 全量拓扑并行层级 (Deterministic 8 Layers)
在运行 `sage mass-rebuild recipes --dry-run` 时，全量 40 个基础与工具链配方严格收敛于 8 个并行层级：
```text
Layer 1: filesystem
Layer 2: ca-certificates, glibc, linux-headers
Layer 3: libcap, make, pkgconf, zlib, zstd
Layer 4: binutils, coreutils, dhcpcd, e2fsprogs, gmp, grep, iproute2, m4, ncurses, patch, sed, shadow, tar, xz
Layer 5: bison, flex, isl, kmod, mpfr, procps-ng, readline, util-linux
Layer 6: bash, eudev, gawk, linux, mpc
Layer 7: gcc, loom, systemd
Layer 8: base
```
- **第 8 层收敛点 (`base`)**：`system/base` 纯声明式元包收敛所有核心依赖，提供一个经严格验证的、可开机启动的最小系统镜像基础。

---

## 6. 虚拟接口规范与去冗余事实 (Virtual Providers & KISS Pruning)

### 6.1 历史教训：过度抽象与“虚拟包膨胀”
在早期配方编写中，曾机械参考 Gentoo 的 `virtual/*` 模式，将许多唯一实现的库（如 `virtual/gmp`、`virtual/zlib`）以及常规工具（如 `virtual/tar`、`virtual/sed`、`virtual/coreutils`）全部抽象为虚拟提供者，甚至为子包追加了大量的 `virtual/*-dev`。

**弊端分析**：
1. **违背 KISS 与 YAGNI 原则**：世界上仅存在唯一的 GNU Coreutils、Tar、GMP、Zlib 等基础组件，没有任何生产级替代方案。凭空构造虚拟符号只会在依赖图与 LMDB 数据库中增加间接寻址层，产生毫无意义的抽象开销。
2. **符号污染与求解器迷航**：大量无用的虚拟符号（Unconsumed Virtuals）充斥图空间，使依赖诊断与 PubGrub 因果树变得繁冗晦涩。

### 6.2 收敛决策：确立 15 大核心多态接口
经过全面清理，全系统**严格确立 15 个真正具备操作系统级多态替换价值的核心接口**：

| 核心虚拟接口 | 典型提供者 | 架构替换与多态决策意义 |
| :--- | :--- | :--- |
| **`virtual/coreutils`** | `coreutils` (GNU) / `uutils-coreutils` (Rust) / `busybox` | 核心用户态基础命令集决策 |
| **`virtual/init`** | `systemd` / `loom` | 核心 PID 1 与服务管理器决策 |
| **`virtual/udev`** | `systemd-udev` / `eudev` | 硬件设备管理器与 uevent 响应机制决策 |
| **`virtual/libc`** | `glibc` / `musl` | 操作系统底层 C 运行库 ABI 决策 |
| **`virtual/libc-dev`**| `glibc-dev` / `musl-dev` | C 语言编译底层系统头文件决策 |
| **`virtual/kernel`** | `linux` / `linux-lts` | 内核镜像与槽位决策 |
| **`virtual/sh`** | `bash` / `dash` / `zsh` | POSIX 标准 Shell 运行时决策 |
| **`virtual/awk`** | `gawk` / `mawk` | POSIX 标准 awk 解释器决策 |
| **`virtual/compiler`**| `gcc` / `clang` | 基础 C/C++ 编译器总管 |
| **`virtual/c-compiler`**| `gcc` / `clang` | C 编译器 |
| **`virtual/cxx-compiler`**| `gcc` / `clang` | C++ 编译器 |
| **`virtual/bootloader`**| `systemd-boot` / `grub` | UEFI / BIOS 引导加载器决策 |
| **`virtual/pkg-config`**| `pkgconf` / `pkgconfig` | 编译期元数据探查工具 |
| **`virtual/filesystem`**| `filesystem` | FHS 基础目录骨架锚点 |
| **`virtual/base`** | `base` | 最小可启动系统元包锚点 |

所有其余依赖均遵循 **直球原则（Direct Dependency）**：使用具体包名（如 `util-linux`, `e2fsprogs`, `shadow`, `tar`, `xz`, `zstd`, `sed`, `grep`, `procps-ng`, `iproute2`, `dhcpcd`, `ca-certificates`）直接依赖，使配方库回归干净、透明、高效的极致形态。
