---
title: 从 rcedit 到 resedit：给 Bun 单文件 EXE 修改 VERSIONINFO 的兼容层设计
date: 2026-09-01 18:44:35
description: 最近一次 Windows 构建工具迁移，让我重新理解了 PE、VERSIONINFO 与 Bun 单文件可执行程序之间的关系。本文从零解释 Windows PE，并复盘如何用 resedit 替代归档的 rcedit，同时通过不变量校验保护 Bun 的自定义 section。
cover: /images/from-rcedit-to-resedit-bun-pe.webp
categories:
  - [软件工程]
tags:
  - Windows
  - PE
  - Bun
  - resedit
  - Release Engineering
  - Build Tooling
---

最近在精简一个 Windows 发行版时，我看到源码目录里放着一个 `rcedit-x64.exe`。它不属于产品业务，也不会在应用运行时被调用，却跟着发行版源码保存了很多年。

最初的问题只是“这个文件能不能删”。继续追下去，问题变成了三个：谁负责修改 Windows 可执行文件的图标和版本信息？这类工具应该留在发行版，还是归构建框架管理？如果迁移到官方建议的 `resedit`，它能不能安全处理 Bun 生成的单文件 EXE？

这次迁移最后没有停在替换依赖。`resedit` 确实可以取代 `rcedit.exe` 的资源编辑职责，但 Bun 在 PE 末尾增加了自己的 `.bun` section，触发了底层解析库的保护性报错。要兼容它，必须先理解 PE 的结构，再把“兼容”写成可验证的不变量。

![一个 Windows PE 容器中，仅资源模块被替换，Bun payload 保持封闭完整](/images/from-rcedit-to-resedit-bun-pe.webp)

<!-- more -->

## 一句话总结

Windows PE 是 `.exe`、`.dll` 等文件使用的结构化容器。版本信息和图标通常位于 `.rsrc`，Bun 的单文件程序还会带有保存 Runtime 与应用 payload 的 `.bun` section。

迁移到 `resedit` 时，可靠的做法是只允许已知的 Bun 布局，并在写回前证明：除资源以外的 section，其虚拟地址、大小、属性和内容都没有变化。

## Windows PE 是什么

PE 的全称是 Portable Executable。按照 [Microsoft 的 PE/COFF 规范](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)，它是 Windows 家族操作系统使用的 executable image 和 object file 格式。常见的 `.exe`、`.dll`、`.sys` 都建立在这套结构上。

把 PE 理解成“Windows Loader 能读懂的带目录容器”会比较直观。文件内部不只是一段连续的机器码，而是由 Header、Section Table 和多块 section 共同组成：

| 组成部分 | 作用 |
| --- | --- |
| DOS Header / PE Header | 声明文件身份、目标架构、section 数量等基础信息 |
| Optional Header | 记录入口地址、Image Base、对齐方式和 Data Directory；名字叫 Optional，但 executable image 通常离不开它 |
| Section Table | 描述每个 section 的名称、磁盘位置、内存地址、大小和访问属性 |
| Sections | 保存代码、只读数据、可写数据、资源、重定位信息以及工具自定义 payload |

常见 section 如下：

| Section | 通常保存什么 |
| --- | --- |
| `.text` | 可执行机器码 |
| `.rdata` | 只读数据 |
| `.data` | 可写数据 |
| `.rsrc` | 图标、字符串、Dialog、Manifest、`VERSIONINFO` 等资源 |
| `.reloc` | Image 无法加载到首选地址时需要使用的重定位信息 |
| `.bun` | Bun 单文件 executable 使用的自定义 payload section |

![Windows PE 的简化结构，以及修改 VERSIONINFO 时允许变化和必须保护的边界](/images/windows-pe-section-layout.webp)

`.rsrc` 内部也有严格结构。Windows 按 Type、Name、Language 组织资源树，最终叶子节点指向资源数据。资源编辑工具必须解析这棵树、修改指定节点，再重新生成合法的 section。

### RVA 和磁盘 offset 不是一回事

理解 PE 修改风险，还需要分清两个位置概念。

`PointerToRawData` 表示一块 section 在磁盘文件里的位置。前面的 `.rsrc` 变大后，后续 section 在文件中的 raw offset 可以顺延，只要 Section Table 同步更新即可。

`VirtualAddress` 通常用 RVA 表示，是 image 映射进进程地址空间后，相对 Image Base 的地址。代码、数据和 Loader 读取的目录会依赖这些地址。修改资源时如果无意中移动后续 section 的 RVA，就会改变 executable 的内存布局。

因此，磁盘 offset 变化不一定是错误；非资源 section 的 RVA、大小或内容变化，则必须高度警惕。

## VERSIONINFO 保存了什么

Windows 文件资源管理器中“详细信息”页显示的 Product name、File description、Company、File version、Product version，主要来自 PE 资源里的 `VERSIONINFO`。

这也是 `rcedit` 长期存在的原因。构建 Electron 或其他 Windows 应用时，可以在命令行里修改图标、Manifest 和版本字段，不必手写 `.rc` 文件并重新链接整个程序。

2026 年 4 月，Electron 团队将 `rcedit` 仓库归档，并在 [No Longer Maintained](https://github.com/electron/rcedit/issues/184) 中说明：Electron 的内部程序化使用已经迁移到 `resedit`，仍依赖 `node-rcedit` 的项目也可以将它作为替代方案。

`resedit` 的边界更适合现代 JavaScript 构建链。它以 JavaScript/TypeScript 实现，可以在 Node.js 和 Web bundler 环境中运行，不需要在仓库里额外保存一个 `rcedit-x64.exe`。调用方可以直接读取 PE、修改 `VersionInfo`，再生成新的 binary。

```js
import * as ResEdit from 'resedit'

const executable = ResEdit.NtExecutable.from(binary)
const resources = ResEdit.NtExecutableResource.from(executable)
const [versionInfo] = ResEdit.Resource.VersionInfo.fromEntries(
  resources.entries,
)

versionInfo.setFileVersion('1.2.3.0', 1033)
versionInfo.setProductVersion('1.2.3.0', 1033)
versionInfo.setStringValues(
  { lang: 1033, codepage: 1200 },
  {
    ProductName: 'Example Runway',
    FileDescription: 'Example Runway',
    CompanyName: 'Example Inc.',
  },
)
```

不过，换成 JavaScript library 不等于自动获得安全性。`resedit` 自己的 README 也提醒，修改或签名 executable 的覆盖测试仍有限，调用方应谨慎检查输出文件。构建系统不能把“API 没报错”当成交付证据。

## 为什么 Bun 让迁移复杂了一步

Bun 支持用 `bun build --compile` 把 JavaScript 或 TypeScript 与 Runtime 一起编译成单文件 executable，也支持生成 Windows x64 目标。Bun 生成的 PE 除了常规 section，还包含保存自身数据的 `.bun` section。

在实际生成的 Windows x64 文件中，资源附近出现了 `.rsrc → .reloc → .bun` 的顺序。

原始 `resedit` 解析这份文件时会抛出错误：

```text
After Resource section, sections except for relocation are not supported
```

这条异常来自 `resedit` 底层 PE library 的保守假设：资源 section 之后只接受 relocation，并不表示 Bun executable 已损坏。普通链接器常把 `.rsrc` 和 `.reloc` 放在靠后位置，但 PE 规范允许工具定义自己的 section；Bun 恰好在后面保留了 `.bun`。

Bun 本身也提供 `--windows-title`、`--windows-version`、`--windows-publisher` 等 metadata 参数，但 [Bun executable 文档](https://bun.sh/docs/bundler/executables) 明确指出，这些参数依赖 Windows API，当前不能用于 cross-compiling。对于需要在 macOS 上交叉生成 Windows artifact 的 host-neutral 构建链，它不是完整替代。

于是出现了一个容易做错的选择：为了让 `resedit` 继续运行，直接删掉检查、忽略所有后续 section。这样能消除异常，却无法证明 `.bun` payload 没被移动或重写。

## 兼容层的设计：先定义不能变的东西

这次实现采用 fail-closed 策略。只有同时满足以下条件，兼容路径才会启用：

1. PE 中存在资源 section。
2. `.rsrc` 之后除了 `.reloc`，只多出一个 `.bun`。
3. 写入新资源后，整个 image 的虚拟大小不变。
4. 所有非资源 section 的名称、RVA、virtual size、raw size、attributes 和二进制内容不变。
5. 重新解析输出文件后，所有目标 `VERSIONINFO` 字段与预期值一致。

遇到 `.custom` 等未知 section，构建立即失败；资源内容过长，导致后续虚拟布局需要移动时，构建同样失败。这两种情况都不会覆盖原始 executable。

### 第一步：记录布局快照

资源写入前，构建工具遍历所有非资源 section，保存它们的结构和内容：

```js
function snapshotExecutableLayout(executable, resourceSection) {
  return {
    imageSize: executable.newHeader.optionalHeader.sizeOfImage,
    sections: executable
      .getAllSections()
      .filter((section) => section !== resourceSection)
      .map((section) => ({
        name: section.info.name,
        virtualAddress: section.info.virtualAddress,
        virtualSize: section.info.virtualSize,
        sizeOfRawData: section.info.sizeOfRawData,
        characteristics: section.info.characteristics,
        data: Buffer.from(section.data),
      })),
  }
}
```

这里刻意没有把 `PointerToRawData` 设为不变量。`.rsrc` 增长时，后续 section 在磁盘文件中的位置可以移动；只要 PE Header 正确反映新位置，Loader 仍能读取它。兼容层需要锁住的是虚拟布局与 payload 内容。

### 第二步：只绕过一个已知假设

兼容层不会修改或删除 `.bun`。它只在 `NtExecutableResource.from()` 执行保守顺序检查时，暂时从 `getAllSections()` 的返回值中隐藏唯一的 `.bun` section。资源解析结束后立即恢复原方法。

这个处理范围很窄：放行的是已经识别并验证过的 Bun 布局，不是把任意 PE 都交给宽松解析器。

### 第三步：写入后逐项比对

`resedit` 生成新资源后，构建工具再次读取所有 section。只要 section 数量、对象、名称、RVA、大小、属性或内容有一项变化，就拒绝生成 artifact。

随后把完整输出重新解析一次，检查 `FileVersion`、`ProductVersion`、`ProductName`、`FileDescription`、`CompanyName`、`InternalName` 和 `OriginalFilename`。只有结构保护和业务字段验证同时通过，才覆盖构建目录里的 executable。

这类兼容层的价值不在于“支持 Bun”这个结果，而在于把成功条件写成了机器可以拒绝的契约。

## 测试不能只看文件属性页

Windows 文件属性页能证明目标字段可见，却不能证明其他 section 没有受损。测试按四层组织更可靠。

| 层级 | 验证内容 | 能证明什么 |
| --- | --- | --- |
| 合成 PE 回归测试 | 构造 `.rsrc → .reloc → .bun`，确认原始 `resedit` 失败、兼容层成功 | 精确复现并覆盖已知冲突 |
| 负向结构测试 | 在资源后放入未知 `.custom` section | 未知布局不会被静默放行 |
| 资源增长测试 | 写入超长 metadata | 虚拟 section 需要移动时会失败且不覆盖原文件 |
| 真实 Bun artifact | 交叉编译 Windows x64 executable，写入 metadata，再解析并打包 | 兼容层能处理 Bun 的真实产物 |

真实 artifact 验证完成后，仍不能把 macOS 上的结构检查描述成 Windows 原生验收。正式发布还需要在 Windows 上启动 executable、回读 File Properties 或 PowerShell `VersionInfo`，并在 metadata 修改之后完成 Authenticode 签名。

如果目标 executable 需要 Authenticode，顺序也必须固定：

![resedit 修改 Windows executable 的安全构建顺序](/images/resedit-safe-build-pipeline.webp)

签名后的 PE 再被修改，原签名就不再对应当前文件内容。`resedit` 文档同样说明，解析签名文件需要显式选择，而重新生成 binary 会得到未签名输出。因此，资源编辑应处于签名前的构建阶段。

## 构建工具和发行版应该怎么分工

这次清理还解决了一个 ownership 问题。`rcedit-x64.exe` 原来位于产品发行版源码目录，看起来像产品运行依赖，实际只服务于 Windows 构建。

更清晰的边界是：

| 层级 | 应该拥有的内容 |
| --- | --- |
| Distribution | 品牌名、版本号、publisher、图标等产品输入 |
| Build Kit | PE 解析、资源写入、结构保护、hash 与打包顺序 |
| Target artifact | 修改完成的 executable，不包含 `resedit` 或 `rcedit.exe` |
| Windows release gate | 原生启动、File Properties、签名和安装验收 |

这次落地将 `resedit` 精确锁定在 `3.1.0`，作为 Build Kit 的 npm dependency。它参与构建，但不会被复制到 Runway payload，也不会作为一个孤立的 `.exe` 混入 Browser App。

这种边界比“把工具换成 npm 包”更重要。它决定了依赖由谁升级、兼容问题在哪里修复、最终安装包应该包含什么，以及故障由哪一层负责阻断。

## 要不要用：我的判断框架

### 值得迁移到 resedit

项目使用 Node.js/JavaScript 构建链，需要程序化修改未签名 PE 的图标或 `VERSIONINFO`，并希望移除仓库中的预编译 `rcedit.exe`，可以优先评估 `resedit`。Electron 已经给出了明确迁移方向，library API 也更容易接入结构检查和测试。

### 可以直接使用 Bun 参数

构建始终发生在原生 Windows，Bun 提供的 title、publisher、version、description 和 copyright 已经覆盖需求时，直接使用 Bun metadata 参数更短。此时仍要验证 Bun 版本、目标 executable 和签名顺序。

### 需要停下来审计

只要 PE 带有未知自定义 section、已经签名、资源增长可能改变虚拟布局，或者发行链没有原生 Windows 验收，就不应该把“生成成功”当成可发布。先建立结构快照、负向测试和重签名路径，再决定是否修改。

这次迁移给我的判断很直接：兼容第三方 binary format，最危险的实现是让异常消失，最可靠的实现是明确哪些变化被允许，并对其余变化全部失败。

## 参考资料

1. [Microsoft Learn：PE Format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)
2. [Electron rcedit：No Longer Maintained](https://github.com/electron/rcedit/issues/184)
3. [resedit-js](https://github.com/jet2jet/resedit-js)
4. [Bun：Single-file executable](https://bun.sh/docs/bundler/executables)

> 出品人：俊宁（Aaron），阿里高级工程师、AI 开发者，拥有 10 年研发经验，现专注 AI 应用、Agent、AI Infra 与软件工程，FlyOS 作者。

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作，配图使用 [better-imagegen](https://github.com/zisheng-ai/better-imagegen) 生成。项目已开源，欢迎在 GitHub 点个 Star。
