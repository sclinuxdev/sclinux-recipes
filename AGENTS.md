# SCLinux Package Recipes Developer & Agent Guidelines

欢迎来到 **SCLinux Package Recipes**（SCLinux 官方配方与系统自举仓库）开发规范与 Agent 指南。
本仓库基于 **Sage 声明式包管理器体系**，所有配方与元数据 Schema 版本统一起步为 **`1`**。

---

## 1. 核心设计原则 (Core Principles)

1. **各架构严格独立维护 (Strict Per-Architecture Independence)**：
   - 路径规则：`recipes/<category>/<pkgname>/<arch>/<pkgname>-<version>-<release>/recipe.toml`。
   - 硬件架构目录：`amd64`、`aarch64`、`riscv64`，以及纯数据无二进制包专属的 `any`。
   - 严禁跨架构隐式继承与回退。各架构拥有专属的 `recipe.toml` 与 `patches/` 目录，编译参数、补丁集与修订号（`release`）完全独立演进。
2. **单配方多子包声明式拆分 (Single-Recipe Multi-Subpackages)**：
   - 坚持“单次编译、零重复构建”。
   - 通过 `[[subpackages]]` 的 Glob 规则（如 `usr/lib/*.so.*`）将产物严格互斥切分为 `-libs`（动态库）、`-dev`（头文件与静态库）、`-doc`（文档），主包通过 `default = "remaining"` 认领剩余可执行程序。
3. **严格 SPDX 许可校验 (Valid SPDX License Expressions)**：
   - `license` 字段受 `spdx` 表达式校验器审计。必须填写合法的 SPDX 标识符（如 `GPL-3.0-or-later`, `LGPL-2.1-or-later`, `MIT`, `Apache-2.0`, `BSD-2-Clause`）。
4. **零命令生命周期脚本 (Zero Interactive Lifecycle Hooks)**：
   - 严禁传统的 `preinst`/`postinst`/`prerm`/`postrm` Shell 脚本。
   - 系统用户全量由 `[[sysusers]]` 声明；命令替代项由 `[[alternatives]]` 声明；系统缓存与模块更新全量由 `triggers/*.toml` 声明。
5. **绝对零硬编码与完全可扩展 (`rclass`)**：
   - 编译器/构建工具链逻辑由 `rclass/*.toml` 驱动（如 `cmake`, `meson`, `autotools`, `cargo`），Init 服务生成由 `rclass/init-*.toml` 驱动。

---

## 2. 标准分类体系 (Category Taxonomy)

所有配方必须归入以下 8 大标准分类之一：

| 分类 (Category) | 范围与职责 | 典型软件包示例 |
| :--- | :--- | :--- |
| **`devel`** | 编译器、链接器、解释器、构建系统与调试器 | `gcc`, `clang`, `llvm`, `cmake`, `ninja`, `binutils` |
| **`lib`** | 通用基础运行时动态库、压缩库、FFI 库 | `zlib`, `zstd`, `libarchive`, `libffi`, `ncurses` |
| **`net`** | 网络协议栈、客户端、守护进程与网络管理工具 | `curl`, `openssh`, `dhcpcd`, `iproute2`, `wget` |
| **`security`** | 加密库、鉴权认证模块、权限管理、根证书 | `openssl`, `ca-certificates`, `shadow`, `sudo`, `pam` |
| **`system`** | 操作系统核心基石、Init 系统、C 运行库、FHS 骨架 | `filesystem`, `glibc`, `musl`, `systemd`, `kmod` |
| **`text`** | 文本流处理、行编辑器、语法分析与全文检索 | `gawk`, `sed`, `grep`, `diffutils`, `less`, `vim` |
| **`tools`** | 磁盘与文件系统工具、硬件检测、打包压缩工具 | `e2fsprogs`, `btrfs-progs`, `tar`, `xz`, `gzip` |
| **`utils`** | 进程管理、系统监控、终端工具集 | `coreutils`, `procps-ng`, `util-linux`, `findutils` |

---

## 3. 配方目录树规范 (Recipe Layout)

```text
recipes/<category>/<pkgname>/
├── amd64/
│   └── <pkgname>-<version>-<release>/
│       ├── recipe.toml               # 架构专属配方 (Schema v1)
│       └── patches/                  # 专用硬件/架构补丁 (可选)
│           └── 0001-fix-build.patch
├── aarch64/
│   └── <pkgname>-<version>-<release>/
│       └── recipe.toml
└── any/                              # 仅限纯数据/跨架构通用包
    └── <pkgname>-<version>-<release>/
        └── recipe.toml
```

---

## 4. Git 提交与代码风格规范 (Commit & Documentation Rules)

### 4.1 语言强制要求 (English Only)
- **提交信息 (Commit Messages)**：必须 **全英文书写**。
- **配方注释与文档 (Recipe Comments & Docstrings)**：配方元数据与注释优先英文，文档可中英双语。

### 4.2 提交格式 (Conventional Commits)
```text
<type>(<scope>): <short description in English>

[optional body in English]
```

#### 允许的 Type
- `feat`: 新增软件包配方、新子包或新增功能
- `fix`: 修复配方依赖、编译参数、补丁缺陷或元数据错误
- `refactor`: 配方重构、子包切分优化或依赖精简
- `docs`: 文档或打包指南更新
- `chore`: 工具链、rclass、triggers 或自举计划调整

#### 合法 Scope 范围
- 分类与包名：如 `devel/binutils`, `system/glibc`, `security/ca-certificates` 或分类名 `system`, `devel`
- 基础设施：`bootstrap`, `rclass`, `triggers`, `templates`, `repo`

### 4.3 质量门禁要求 (Pre-Submission Verification Gates)
在推送或发起 PR 之前，必须在本地通过 Sage 的干跑验证：
1. **全依赖图无环与并行层级验证**：
   ```bash
   sage mass-rebuild recipes --dry-run
   ```
   必须成功推导出各并行构建层（Layers），无死锁环依赖或 Schema 报错。
2. **自举计划完整性验证**：
   ```bash
   sage bootstrap bootstrap.toml --dry-run
   ```
   必须成功推进各个 Stage，所有在计划中引用的配方必须存在且合法。
3. **单包语法与沙箱检查**（若构建单包）：
   ```bash
   sage build <recipe-directory> --dry-run
   ```
