# 配方编写与打包指南 (Packaging Guidelines)

## 1. 命名与子包切分惯例

为保持整个 SCLinux 发行版生态的一致性与可预测性，包名及子包名遵循以下约定：

| 包类型 | 命名后缀 | 典型包含路径 (`files`) | 依赖关系建议 |
| :--- | :--- | :--- | :--- |
| **主包** | `<pkgname>` | `usr/bin/*`, `usr/sbin/*`, `etc/*` | 依赖自身的 `-libs` |
| **动态运行库** | `<pkgname>-libs` | `usr/lib/*.so.*` | 依赖 `virtual/libc`，导出 `so:...` |
| **开发头文件** | `<pkgname>-dev` | `usr/include/**`, `usr/lib/*.so`, `usr/lib/pkgconfig/*` | 精确依赖对应版本的 `-libs` |
| **文档/手册** | `<pkgname>-doc` | `usr/share/doc/**`, `usr/share/man/**` | 通常无运行时依赖 (arch="any") |

## 2. 常用虚拟符号 (Virtual Symbols)

当软件包存在多种替代实现时，使用统一的 `virtual/` 符号解耦：
- `virtual/libc`: 由 `glibc` 或 `musl` 提供
- `virtual/init`: 由 `systemd` 或 `openrc` 提供
- `virtual/awk`: 由 `gawk` 或 `mawk` 提供
- `virtual/sh`: 由 `bash`、`dash` 或 `busybox` 提供

## 3. SPDX 常见合法许可证速查

在 `recipe.toml` 中的 `license` 字段受严格的 SPDX 表达式校验器审计，常见有效写法：
- `MIT`
- `Apache-2.0`
- `BSD-2-Clause` / `BSD-3-Clause`
- `GPL-2.0-only` / `GPL-2.0-or-later`
- `GPL-3.0-only` / `GPL-3.0-or-later`
- `LGPL-2.1-or-later` / `LGPL-3.0-or-later`
- 多许可证组合使用布尔大写：`"BSD-2-Clause AND Apache-2.0"` 或 `"MIT OR Apache-2.0"`

## 4. 补丁放置与管理

补丁文件存放在专属版本目录下的 `patches/` 文件夹中：
```text
recipes/<category>/<pkgname>/<arch>/<pkgname>-<version>-<release>/
├── recipe.toml
└── patches/
    ├── 0001-fix-cross-compilation.patch
    └── 0002-adjust-install-prefix.patch
```
补丁需保持单一职责与标准 Unified Diff 格式 (`diff -u` / `git format-patch`)。
