---
title: darwin-arm64、win32-x64、linux-x64 到底是什么：一篇讲清安装包命名
date: 2026-08-24 22:27:09
description: 下载 CLI、桌面应用或原生依赖时，我们经常遇到 darwin-arm64、win32-x64、linux-x64、AMD64、AArch64、gnu、musl 等名字。本文从 OS、CPU、ABI 与 release channel 四个维度拆解这些标签，解释它们的历史来源、兼容关系与选包方法。
categories:
  - [软件工程]
tags:
  - CPU Architecture
  - macOS
  - Windows
  - Linux
  - x64
  - arm64
  - ABI
  - Cross Compilation
cover: /images/platform-architecture-package-naming.webp
---

![安装包命名里的 OS、CPU 与 ABI 坐标](/images/platform-architecture-package-naming.webp)

最近整理多平台发行物时，我又遇到了一排很像“暗号”的文件名：

```text
tool-v1.8.0-darwin-arm64.tar.gz
tool-v1.8.0-darwin-x64.tar.gz
tool-v1.8.0-win32-x64.zip
tool-v1.8.0-linux-x64-gnu.tar.gz
tool-v1.8.0-linux-arm64-musl.tar.gz
```

它们其实没有看上去那么乱。每一段都在回答一个兼容性问题：给哪个操作系统、哪种 CPU、哪套运行环境，以及用什么方式分发。

真正容易误解的是，大家常把这些维度混成一句“Windows 有 x64 渠道，Linux 也有”。`x64` 不是渠道，它是 CPU architecture；`stable`、`beta`、`nightly` 才是 release channel。把两者分开后，大部分安装包命名都能直接读懂。

<!-- more -->

## 一句话总结

`darwin-arm64` 可以读成“面向 Darwin/macOS 平台、使用 64-bit Arm 指令集的构建”；`win32-x64` 是“面向 Windows 平台、使用 x86-64 指令集的构建”；`linux-x64-gnu` 则继续补充了“面向 Linux、使用 x86-64，并依赖 GNU/glibc 运行环境”。

我习惯把完整文件名看成一组坐标：

```text
<product>-<version>-<os>-<cpu>-<abi>-<channel>.<package-format>
```

现实中的项目不一定把字段写全，顺序也可能不同，但背后通常逃不开下面几层：

| 维度 | 它回答的问题 | 常见值 |
| --- | --- | --- |
| OS / platform | 这份程序调用哪套操作系统接口 | `darwin`、`win32`、`windows`、`linux` |
| CPU architecture | CPU 能不能执行这套机器指令 | `x64`、`x86_64`、`amd64`、`arm64`、`aarch64`、`ia32` |
| ABI / runtime | 二进制如何调用系统库、传参和链接 | `gnu`、`musl`、`msvc`、`gnueabihf` |
| package format | 用什么容器或安装机制交付 | `.zip`、`.tar.gz`、`.dmg`、`.msi`、`.deb`、`.rpm`、`AppImage` |
| release channel | 这是哪条发布节奏 | `stable`、`beta`、`canary`、`nightly`、`LTS` |

**OS、CPU 和 ABI 决定“能不能跑”，package format 决定“怎么装”，channel 决定“拿哪条版本线”。**

## `darwin` 为什么代表 macOS

`darwin` 不是 Apple 给 macOS 起的下载渠道名，而是 macOS 底层系统家族的名字。

Apple 在 2000 年发布 Darwin 1.0 时，把它描述为 Mac OS X 的 operating system core，其中包括 Mach kernel 和 BSD layers。今天我们日常使用的 macOS 还包含图形界面、Cocoa framework 和大量闭源组件，但在编译器、Unix 工具链和 Runtime 眼里，`darwin` 仍是一个稳定的平台标识。

这也是为什么 Node.js 的 `process.platform` 在 Mac 上返回 `darwin`，Rust 会使用 `x86_64-apple-darwin` 与 `aarch64-apple-darwin`，Clang/LLVM 也保留 Darwin 作为 Apple OS family 的 target 名称。

所以：

```text
darwin-arm64 = macOS 平台 + 64-bit Arm
darwin-x64   = macOS 平台 + 64-bit x86
```

这里的 `darwin` 决定程序会按照 Apple 平台的 Mach-O、system framework 与 ABI 规则构建；它不表示这份程序能在所有基于 Darwin 的系统上任意互换。macOS、iOS、Simulator 仍可能拥有不同的 platform 与 SDK 边界。

## `win32` 为什么也会出现在 64-bit Windows 上

`win32-x64` 是最容易让人误判的一种组合：前面写着 `32`，后面又写 `64`，到底谁算数？

答案是两个字段描述的对象不同：

- `win32` 是 Windows platform/API 家族的历史名称；
- `x64` 才是当前二进制的 CPU architecture。

Microsoft 现在更常使用 “Windows API” 这个名字，但官方文档也说明它过去称为 Win32 API，而且这套 API 同样支持 64-bit Windows。Node.js 沿用了 `win32` 作为 Windows 的 `process.platform` 固定值，所以 Electron、Node CLI 和不少 npm native package 会自然产生 `win32-x64`、`win32-arm64` 这样的标签。

这几个名字可以这样读：

| 名称 | 实际含义 |
| --- | --- |
| `win32-ia32` / `win32-x86` | Windows + 32-bit x86 |
| `win32-x64` / `windows-x64` / `win-amd64` | Windows + 64-bit x86 |
| `win32-arm64` / `windows-arm64` | Windows + 64-bit Arm |

因此，看到 `win32` 时不要据此判断 CPU 位数。继续找同一文件名里的 `ia32`、`x86`、`x64` 或 `arm64`，那一段才负责回答架构问题。

## `linux` 只是第一层，后面往往还要看 libc

`linux-x64` 表示 Linux + 64-bit x86，但对包含 native code 的程序来说，这个坐标有时还不够精确。

Linux 生态不像 macOS、Windows 那样由单一厂商提供完整的用户态运行环境。相同的 Linux kernel 之上，可以有不同 distribution、libc、dynamic linker 和 system library 版本。于是发行物里经常继续出现：

```text
x86_64-unknown-linux-gnu
x86_64-unknown-linux-musl
aarch64-unknown-linux-gnu
aarch64-unknown-linux-musl
```

`gnu` 通常表示 GNU toolchain / glibc environment，`musl` 表示 musl libc environment。CPU 和 kernel 都对，libc 不匹配，程序仍可能在启动时报告 dynamic linker 不存在、symbol version 找不到，或者某个 native dependency 无法加载。

`.deb`、`.rpm`、`AppImage` 又是另一层：它们主要描述交付与安装方式。

| 名称片段 | 描述对象 | 不能单独证明什么 |
| --- | --- | --- |
| `linux-arm64` | OS + CPU | 不保证 libc 与最低系统版本兼容 |
| `gnu` / `musl` | libc / ABI environment | 不表示 package manager |
| `.deb` / `.rpm` | package format | 不表示 CPU architecture |
| `ubuntu` / `debian` / `rhel` | distribution compatibility target | 仍要继续看版本、CPU 与 library baseline |

这也是 Linux 安装包经常比 macOS 多出一截后缀的原因：发布者需要把更多用户态兼容边界写进名字。

## `x86`、`x64`、`amd64`、`x86_64` 为什么像一组近义词

这组名字背后是一段很长的兼容历史。

### `x86` 来自一串以 86 结尾的处理器

Intel 在 1978 年推出 8086，后续又有 80186、80286、80386、80486。因为这些型号都以 `86` 结尾，业界逐渐用 `x86` 指代这一指令集家族。

到了 32-bit 时代，`i386`、`i486`、`i586`、`i686` 常被用来表达不同代际或最低指令集 baseline。`IA-32` 则是 Intel 对 32-bit architecture 的正式叫法之一。Node.js 使用 `ia32` 表示 32-bit x86 构建。

这导致 `x86` 在不同页面里可能有两种语义：

1. 广义上指整个 x86 family，包含 32-bit 与 64-bit；
2. 下载按钮里常特指 32-bit x86，用来和 `x64` 区分。

选包时要服从当前下载页的对照关系，不能只凭词源判断。

### x86 的 64-bit 扩展最早叫 AMD64

AMD 把传统 x86 扩展到 64-bit，并保持对既有 16-bit、32-bit 软件模型的兼容，这套 architecture 被称为 `AMD64`。Intel 后来实现兼容的 64-bit architecture，并使用 `Intel 64` 这个名称。

软件生态于是留下了几种常见写法：

| 名称 | 常见生态 | 一般指向 |
| --- | --- | --- |
| `x86_64` | Unix、Linux、Clang、Rust、Python wheel | 64-bit x86 |
| `amd64` | Debian/Ubuntu、Go、Windows 内部与云镜像 | 64-bit x86 |
| `x64` | Windows、Node.js、Electron、普通下载页 | 64-bit x86 |
| `Intel 64` | Intel 官方文档 | Intel 对兼容 64-bit x86 的命名 |

Microsoft 的官方文档直接把 `x64` 定义为同时包含 AMD64 与 Intel 64。对普通安装包选择来说，`x64`、`amd64`、`x86_64` 通常可以视为同一 instruction-set family 的不同生态写法；到了 compiler flags、calling convention、object format 或 container manifest 层面，仍应使用对应工具链规定的精确字符串。

## `arm64` 与 `aarch64` 又是什么关系

Armv8-A 引入了新的 64-bit execution state，Arm 官方称它为 `AArch64`；对应的 instruction set 是 A64。与它并列的 32-bit execution state 叫 `AArch32`。

不同生态对同一个 64-bit Arm 家族选了不同标签：

| 名称 | 常见位置 |
| --- | --- |
| `AArch64` / `aarch64` | Arm 文档、Linux `uname`、GNU toolchain、Rust target |
| `arm64` | Apple、Windows、Node.js、Docker platform、普通下载页 |

因此，Apple Silicon Mac 下载 `darwin-arm64`；Rust 对同一类目标使用 `aarch64-apple-darwin`。它们不是两种互不兼容的 CPU，而是不同工具链对同一 64-bit Arm architecture family 的常用命名。

不过，“同属 AArch64”仍不等于二进制可以跨 OS 直接运行。一个 macOS arm64 Mach-O executable 不能因为 CPU 相同，就直接当作 Linux arm64 ELF executable 使用。OS API、object format、ABI 与 system library 仍然不同。

## 把短名字展开，就是 target triple

编译器需要比下载页更精确的坐标。Clang/LLVM、Rust 与 GNU toolchain 常用 target triple 描述目标：

```text
ARCHITECTURE-VENDOR-OPERATING_SYSTEM[-ENVIRONMENT]
```

LLVM 官方文档说明，经典形式包含 architecture、vendor、operating system 三段，也可以再加 environment。即使今天经常出现四段，工具链仍沿用 “target triple” 这个历史名称，因为它最初确实只有三个字段。

拆几个真实例子：

| Target | Architecture | Vendor | OS | Environment / ABI |
| --- | --- | --- | --- | --- |
| `aarch64-apple-darwin` | `aarch64` | `apple` | `darwin` | Apple 平台默认环境 |
| `x86_64-pc-windows-msvc` | `x86_64` | `pc` | `windows` | MSVC ABI/toolchain |
| `x86_64-pc-windows-gnu` | `x86_64` | `pc` | `windows` | GNU/MinGW environment |
| `x86_64-unknown-linux-gnu` | `x86_64` | `unknown` | `linux` | GNU/glibc environment |
| `aarch64-unknown-linux-musl` | `aarch64` | `unknown` | `linux` | musl environment |

`unknown` 不表示作者不知道 CPU 或系统，只表示这个目标没有需要单独编码的 vendor。`pc` 也不等于“只能在某品牌 PC 上运行”，它是工具链 target 命名的一部分。

短文件名 `linux-x64` 可以看成产品面向普通用户的压缩写法；`x86_64-unknown-linux-gnu` 则是编译器和 linker 需要的精确坐标。

## macOS 为什么还有 Universal Binary

macOS 经历过多次 CPU architecture 迁移。Apple 的技术文档记录了从 PowerPC 到 Intel，再从 Intel 到 Apple Silicon 的兼容需求。为了让同一份 App 同时包含多个 architecture 的机器码，macOS 使用 Universal Binary，也常被称为 fat binary。

今天常见的 universal macOS executable 同时包含：

```text
x86_64 arm64
```

系统启动时会选择适合当前机器的 slice。Apple Silicon 上还可以通过 Rosetta 运行只有 `x86_64` 指令的 Mac app；但这属于 binary translation，不等于它已经变成原生 `arm64` 构建。

三种发行方式各有取舍：

| 发行物 | 优点 | 代价 |
| --- | --- | --- |
| `darwin-arm64` | Apple Silicon 原生，体积只含一个 architecture | Intel Mac 不能运行 |
| `darwin-x64` | 支持 Intel Mac，Apple Silicon 在 Rosetta 条件下可运行 | 在 Apple Silicon 上不是原生执行；插件和 native module 还要一起兼容 |
| `universal` / `universal2` | 一个 artifact 覆盖两种 Mac architecture | 体积更大，所有 embedded binaries 都要包含正确 slice |

可以用 `lipo` 检查一个 macOS binary 里到底有哪些 slice：

```bash
lipo -archs /path/to/MyApp.app/Contents/MacOS/MyApp
```

看到 `x86_64 arm64` 才能证明主 executable 同时包含两种 architecture。对复杂 App，还要继续检查 framework、plugin、helper 与 native library，不能只验主程序。

## Windows ARM64 和模拟执行怎么理解

Windows 11 on Arm 支持运行 Arm64 native app，也能通过 emulation 运行 x86 与 x64 app。于是 Arm Windows 用户经常同时看到两个都“能装”的选项：`windows-x64` 和 `windows-arm64`。

我的选择顺序仍然是：**有可信的 Arm64 native build 就优先 Arm64；只有 x64 build 时，再把 emulation 当兼容路径。**

原因并不神秘。模拟层能翻译 CPU instructions，但应用还可能加载 driver、shell extension、DLL、JIT、plugin 或 native addon。这些组件存在各自的 architecture 与 ABI 边界。Microsoft 的 WOW64 文档也明确指出，32-bit 与 64-bit DLL 不能随意注入不同位数的进程；“主程序能启动”不能替代整条依赖链验收。

Windows 还有 `ARM64EC` 这类面向混合迁移的 ABI，让同一进程在特定规则下组合 Arm64EC 与兼容 x64 code。它主要是开发和迁移概念，不是普通用户下载页面上的首要选择。看到它时应按产品官方说明处理，不要把 `EC` 当作另一个 CPU family。

## 为什么 OS 和 CPU 都对，程序还是跑不起来

一个 native executable 能否运行，至少要连续通过四层匹配：

```text
OS / object format
    ↓
CPU instruction set
    ↓
ABI / libc / calling convention
    ↓
system library、driver、plugin 与最低版本
```

几个典型反例：

- `linux-x64-musl` 与机器的 x64 CPU 匹配，但程序期望的 dynamic loader 可能和当前 glibc environment 不同；
- `darwin-x64` 能通过 Rosetta 启动，App 内部某个只含 arm64 或只含 x86_64 的 plugin 仍可能加载失败；
- `windows-x64` 可以在 Windows on Arm 的 emulation 下启动，但 kernel driver 不能按普通 user-space app 的方式处理；
- 同样是 `linux-x64-gnu`，新 glibc 编译出的 binary 也可能因为 symbol version baseline 过高而无法运行在旧 distribution 上。

所以“架构兼容”只是必要条件，不是充分条件。发行工程真正要验证的是整条 artifact：可执行文件、动态库、native addon、插件、安装器和更新器都必须对齐。

## 怎么确认自己该下载哪一个

### 跨平台最省事：看当前 Runtime 的编译目标

如果机器上已经有 Node.js：

```bash
node -p "process.platform + '-' + process.arch"
```

常见输出：

```text
darwin-arm64
win32-x64
linux-x64
```

Node.js 官方定义得很清楚：`process.platform` 表示这份 Node binary 面向哪个 OS platform 编译，`process.arch` 表示它面向哪个 CPU architecture 编译。这里检测的是当前 Runtime，不一定是物理硬件。

例如 Apple Silicon 上如果运行的是 x64 Node，`process.arch` 可能仍是 `x64`。对 native addon 来说这恰好有用，因为 addon 必须匹配当前 process；对“我应该安装哪一版 Node”来说，你还要继续确认硬件与翻译状态。

### macOS

```bash
uname -m
sysctl -in sysctl.proc_translated 2>/dev/null || true
```

常见的 `uname -m` 输出是 `arm64` 或 `x86_64`。在 Apple Silicon 上，`sysctl.proc_translated` 返回 `1` 表示当前 process 正在 Rosetta translation environment 中运行，返回 `0` 表示 native process。

如果只想通过图形界面判断：Apple 菜单 →“关于本机”，看到 Apple M 系列芯片就选 `arm64`；看到 Intel processor 就选 `x64` / `x86_64`。

### Windows PowerShell

如果安装了现代 .NET runtime，可以同时查看 OS 与当前 process：

```powershell
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture
[System.Runtime.InteropServices.RuntimeInformation]::ProcessArchitecture
```

两者可能不同。例如 Arm64 Windows 正在运行 x64 process 时，OS architecture 与 process architecture 就不能混为一个值。没有把握时，优先遵循产品下载页对 Windows on Arm 的明确说明。

### Linux

```bash
uname -m
ldd --version
```

常见 architecture 映射：

| `uname -m` | 下载页常见标签 |
| --- | --- |
| `x86_64` | `x64` / `amd64` / `x86_64` |
| `aarch64` | `arm64` / `aarch64` |
| `armv7l` | `armv7` / `armhf`，还要确认 hard-float ABI |
| `ppc64le` | `ppc64le` |
| `s390x` | `s390x` |
| `riscv64` | `riscv64` |

`ldd --version` 的输出通常能帮助判断当前环境属于 glibc 还是 musl，但不同 distribution 的实现与输出不完全一致。发布者若提供了专门的 `gnu` / `musl` artifact，应继续对照其支持矩阵，而不是只看 `uname -m`。

## 看到一个文件名，我会按什么顺序判断

假设下载页给出：

```text
app-3.2.0-linux-arm64-musl-beta.tar.gz
```

我会从兼容性到发布策略依次拆：

1. `linux`：目标 OS；
2. `arm64`：目标 CPU architecture；
3. `musl`：目标 libc / ABI environment；
4. `beta`：release channel；
5. `.tar.gz`：archive format，不是 CPU 或 ABI。

这套顺序有一个好处：即使项目把字段换成 `aarch64-unknown-linux-musl`、`linux-aarch64` 或 `linux-arm64-alpine`，你仍然是在找相同的几个坐标。

还有三个实用判断：

- **原生优先。** Apple Silicon 选 `arm64`，Windows on Arm 有成熟 native build 时也优先 `arm64`；
- **模拟是兼容手段，不是构建证明。** x64 app 能通过 Rosetta 或 Windows emulation 启动，不代表 native module、plugin 与 driver 全部通过；
- **Linux 多看一层 libc 与最低版本。** CPU 匹配后，再核对 `gnu/musl`、distribution baseline 和 package format。

## 要不要记住所有名字：我的判断框架

不需要背完整词典。记住三条主线就够了：

**第一，先分维度。** `darwin`、`win32`、`linux` 是 OS/platform；`x64`、`arm64` 是 CPU architecture；`gnu`、`musl`、`msvc` 是 ABI/toolchain environment；`stable`、`beta` 才是 channel。

**第二，识别同义标签。** `x64 ≈ amd64 ≈ x86_64`，`arm64 ≈ aarch64`。这里的“≈”表示 ordinary package selection 中通常指向同一个 instruction-set family，不代表不同 OS、ABI 下的 binary 可以互换。

**第三，把检测对象说清楚。** 你检测的是 physical machine、operating system，还是当前 process？在 Rosetta、WOW64、container 和 cross-compilation 环境里，这三个答案完全可能不同。

下一次再看到 `darwin-arm64`，不用把它当成一串项目私有缩写。把连字符理解成坐标分隔符：平台、架构、环境、渠道，一层层读下去即可。

## 参考资料

- [Apple：Apple Releases Darwin 1.0 Open Source](https://www.apple.com/newsroom/2000/04/05Apple-Releases-Darwin-1-0-Open-Source/)
- [Apple Developer：Building a universal macOS binary](https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary)
- [Apple Developer：About the Rosetta translation environment](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment)
- [Node.js：`process.platform` 与 `process.arch`](https://nodejs.org/api/process.html)
- [Intel：What is x86 Architecture?](https://newsroom.intel.com/tech101/x86-architecture-foundation-of-modern-computing)
- [AMD：AMD64 Architecture Programmer’s Manual](https://docs.amd.com/v/u/en-US/40332_4.09_APM_PUB)
- [Arm：Armv8-A Instruction Set Architecture](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Armv8-A%20Instruction%20Set%20Architecture.pdf)
- [Microsoft Learn：x64 architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/x64-architecture)
- [Microsoft Learn：Windows on Arm FAQ](https://learn.microsoft.com/en-us/windows/arm/faq)
- [LLVM：Target Triple](https://llvm.org/docs/LangRef.html#target-triple)
- [Rust：Platform Support](https://doc.rust-lang.org/rustc/platform-support.html)

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作。项目已开源，欢迎在 GitHub 点个 Star。
