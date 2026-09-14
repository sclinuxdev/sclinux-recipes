# SCLinux Package Recipes Repository (`sclinux-recipes`)

本仓库为 **SCLinux** 官方软件包配方（Recipes）仓库，基于 **Sage 声明式包管理器体系** 构建与维护。
所有配方统一遵循 **Schema v1** 规范，采用严格的**单次编译多子包拆分**与**各硬件架构独立维护**原则。

---

## 1. 目录结构与分类体系 (Taxonomy & Layout)

所有包配方按 **8 大标准分类** 进行组织，并在包级目录下严格按**硬件架构**做物理隔离：

```text
sclinux-recipes/
├── README.md                     # 仓库总览与维护指南
├── .gitignore                    # 忽略构建缓存与二进制产物
├── bootstrap.toml                # 全系统自举与阶段编排计划 (v1)
├── docs/                         # 架构决策与配方维护事实文档
│   └── MAINTENANCE_FACTS.md      # 依赖拓扑、冲突矩阵与解耦维护事实
├── templates/                    # 配方模板与规范速查
│   ├── recipe.template.toml      # 完整配方声明模板
│   ├── service.template.toml     # 跨 Init 服务与激活声明模板
│   └── README.md                 # 打包与 SPDX 规范速查
└── recipes/
    ├── devel/                    # 编译器、构建系统、调试器 (gcc, clang, cmake, binutils...)
    ├── lib/                      # 基础动态运行时库、压缩库 (zlib, zstd, libffi, openssl...)
    ├── net/                      # 网络协议栈、工具与守护进程 (curl, openssh, iproute2...)
    ├── security/                 # 鉴权安全、证书、加密 (ca-certificates, pam, shadow...)
    ├── system/                   # 核心基石、C 运行库、Init (filesystem, glibc, systemd...)
    ├── text/                     # 文本处理工具、流编辑器 (sed, gawk, grep, vim...)
    ├── tools/                    # 存储与归档工具、内核模块 (tar, xz, e2fsprogs, kmod...)
    └── utils/                    # 基础进程控制、系统工具 (coreutils, procps-ng, which...)
```

### 架构独立维护规范 (Strict Per-Architecture Tree)

```text
recipes/<category>/<pkgname>/
├── amd64/                            # x86-64 专属目录
│   └── <pkgname>-<version>-<release>/# 专属版本与构建号 (如 gcc-16.2.0-1)
│       ├── recipe.toml               # amd64 专属编译配置与依赖
│       └── patches/                  # 专用硬件或编译补丁 (可选)
├── aarch64/                          # ARM64 专属目录
│   └── <pkgname>-<version>-<release>/# 专属版本与构建号
│       └── recipe.toml
├── riscv64/                          # 64位 RISC-V 专属目录
│   └── <pkgname>-<version>-<release>/
│       └── recipe.toml
└── any/                              # 架构无关包 (无二进制编译: 纯数据、文档、纯 Python 库)
    └── <pkgname>-<version>-<release>/
        └── recipe.toml
```

> [!NOTE]
> - **禁止跨架构隐式继承**：不同硬件架构的指令集调优、补丁集、ABI 与 release 修订号严格解耦，单架构升级互不干扰。
> - **纯数据包归宿 (`any`)**：仅当产物无任何架构相关 ELF 二进制时（如 `filesystem`、`ca-certificates`、纯文本字体），才放置在 `any/`。

---

## 2. 核心打包设计原则

1. **单次编译，声明式多包拆分 (`[[subpackages]]`)**：
   - 源码仅在 Bubblewrap 沙箱中编译一次至统一 `DESTDIR`。
   - 通过 Glob 规则（如 `usr/lib/*.so.*`）将动态库切分为 `-libs`，头文件切分为 `-dev`，主包认领剩余工具，文件集合严格互斥，杜绝重复编译。
2. **绝对零硬编码与完全可扩展 (`rclass`)**：
   - 通过 `[build] inherit = ["cmake"]` / `["meson"]` / `["autotools"]` / `["cargo"]` 驱动构建，阶段逻辑完全由类模板编排。
3. **严格 SPDX 许可证检查**：
   - `license` 字段必须是合法的 SPDX 表达式（如 `GPL-3.0-or-later`, `BSD-2-Clause`, `MIT`, `Apache-2.0`）。
4. **确定性无环依赖图**：
   - `sage mass-rebuild` 与 `sage bootstrap` 依靠 Kahn 拓扑排序计算最大并行构建层，自举环通过 `bootstrap.toml` 阶段分步切断。

---

## 3. 常用构建与维护命令

```bash
# 1. 语法与依赖拓扑干跑验证 (Dry-run 检查图连通性与并行层级)
sage mass-rebuild /home/ir/sclinux-recipes/recipes --dry-run

# 2. 单配方独立密闭构建 (产出 *.pkg.tar.zst)
sage build recipes/system/glibc/amd64/glibc-2.41-1

# 3. 按阶段执行自举全量构建 (Bootstrap Pipeline)
sage bootstrap bootstrap.toml --output /var/cache/sage/bootstrap-pool
```

---

## 4. 维护事实与架构决策文档

详细的组件切分粒度、运行时解耦论证、全局命令防碰撞矩阵及双 Init 生态隔离推演，请参阅：
- [MAINTENANCE_FACTS.md](docs/MAINTENANCE_FACTS.md)：记录包括 `systemd-networkd` 与 `resolved` 解耦事实、`systemd` 与 `eudev` 互斥原因、`shadow` 三段式拆分、命令权威归属矩阵及自举环解法等核心决策。
