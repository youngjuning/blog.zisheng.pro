---
title: Omaha 到底是什么：一篇讲清 Chromium 自动更新的客户端、协议与服务端
date: 2026-08-25 10:56:36
description: Omaha 既是 Google Update 的开源项目名，也常被用来指 Chromium 更新协议与服务端。本文拆清 Chromium Updater、Omaha Protocol、CUP、CRX3 和 Authenticode 的责任边界，并串起一次完整的自动更新。
categories:
  - [软件工程]
tags:
  - Omaha
  - Chromium Updater
  - Auto Update
  - CUP
  - CRX3
  - Windows
  - Release Engineering
cover: /images/omaha-update-protocol.webp
---

![Omaha 自动更新的检查、下载与回报](/images/omaha-update-protocol.webp)

最近在梳理 Chromium 桌面应用的更新链路时，我频繁遇到一个词：`Omaha`。

它一会儿出现在 `Omaha endpoint` 里，一会儿又变成 `Omaha Protocol`、`Omaha Server` 或 `Omaha response`。继续查源码，还会遇到 `Chromium Updater`、`update_client`、`CUP` 和 `CRX3`。如果把这些名字都当成“自动更新”的同义词，很快就会把客户端、协议、服务端和安装制品混在一起。

<!-- more -->

## 一句话总结

**Omaha 最初是 Google Update 的开源客户端项目；在当今 Chromium 更新语境里，它更常指一套更新协议和云端决策接口。** 真正在本机定期检查、下载并安装新版本的，是 Chromium Updater 这类客户端；Omaha Server 回答“这台设备现在应该拿哪个版本、去哪里下载、如何处理”。

所以，一句“我们要接入 Omaha”，至少要继续追问三件事：

- 是否要编译和定制 Chromium Updater 客户端？
- 是否要实现 Omaha Protocol 的服务端响应？
- 更新制品、签名、CDN 和渠道激活由谁产出和管理？

## 同一个 Omaha，其实指三个不同对象

Google 公开的 [Omaha 仓库](https://github.com/google/omaha) 将它定义为 Google Update 的开源版，用于安装软件并保持更新。这份历史客户端代码以 Windows 为主，仓库已于 2025 年 8 月归档为只读。

现在的 Chromium 源码中，`chrome/updater` 下的 [Chromium Updater](https://chromium.googlesource.com/chromium/src/+/main/chrome/updater/) 是 Google Update/Omaha/Keystone 的开源可定制替代者，目标平台包括 Windows、macOS 和 Linux。但客户端更名与替换后，它与服务端交流时仍然使用 Omaha Protocol。

| 对象 | 是什么 | 负责什么 |
| --- | --- | --- |
| Omaha 历史项目 | Google Update 的开源客户端 | 安装软件、周期检查、下载和应用更新 |
| Omaha Protocol | 客户端与更新基础设施之间的 HTTP 应用层协议 | 描述已安装应用、版本、平台、渠道、更新指令和执行结果 |
| Omaha Server | 实现协议的更新决策服务 | 识别 App ID，按版本、平台、渠道或 cohort 返回 `noupdate` 或更新 pipeline |

这也解释了为什么 Chromium 文档里会出现“Chromium Updater 轮询 Omaha Server”这种句子：前者是本机执行者，后者是云端决策者。

## 一次完整的 Omaha 更新发生了什么

[Omaha Protocol 4.0](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/protocol_4.md) 把一次更新会话拆成三个阶段：Update Check、Download 和 Ping-Back。截至 2026 年 8 月 25 日，这份公开文档仍标注为 draft；真正实现时应固定到所使用的 Chromium commit，而不是假设 `main` 永远不变。

```text
已安装客户端
  │
  ├─ 1. POST 当前 App ID、版本、OS、CPU、渠道
  ▼
Omaha Server
  │
  ├─ 2. 返回 noupdate，或新版本 + 下载/处理 pipeline
  ▼
Download Server / CDN
  │
  ├─ 3. GET 更新 payload，校验大小、hash 与签名
  ▼
Chromium Updater
  │
  ├─ 4. 解包、执行 installer、切换版本
  └─ 5. POST 成功、失败或取消事件
```

### 1. Update Check：客户端不只上报一个版本号

客户端先用 HTTP POST 发送更新检查。请求中可以包含多个受管应用，每个应用都有独立的 `appid`、`version`、`cohort` 和 `updatecheck`。操作系统、CPU architecture、安装范围和前台/后台触发等信息，都可能改变服务端的决策。

下面是一份刻意缩减过的示意请求，用于展示结构，不是可直接投产的完整 payload：

```json
{
  "request": {
    "protocol": "4.0",
    "@updater": "chromium-updater",
    "updaterversion": "148.0.7778.97",
    "os": {
      "platform": "win",
      "arch": "x64"
    },
    "apps": [
      {
        "appid": "{YOUR-APP-ID}",
        "version": "1.2.3.4",
        "cohort": "stable",
        "updatecheck": {}
      }
    ]
  }
}
```

`appid` 是更新身份，不是展示名称。版本则是服务端和客户端共同消费的比较值。在 Chromium 发行工程里，我更倾向为它保留一条独立、单调递增的数值版本，而不是直接复用用户看到的 SemVer、npm package version 或 Git tag。它们的消费者和推进条件不同。

### 2. Server Decision：响应描述的是 pipeline

服务端不会把安装包塞进 Update Check 的 JSON response。它先回答 `noupdate` 或 `ok`；当有新版本时，再提供 `nextversion` 与一组按优先级排列的 pipeline。

在 Protocol 4 中，pipeline 是一系列 operation。它可以表达：

- 从哪些 URL 下载 payload；
- 预期的 size 和 SHA-256 hash；
- 是否经过 `xz`、Zucchini 或 Puffin 等压缩/差分操作；
- 如何处理 `crx3` 并运行安装程序。

服务端可以根据渠道、cohort、平台、当前版本和 rollout 策略对不同设备做不同决策。这才是 Omaha Server 的价值：**它不只存放“最新版本”，还决定谁现在应该收到什么。**

### 3. Download 与 Install：CDN 不替你执行更新

客户端收到 pipeline 后，才向 Download Server 或 CDN 发起 HTTP GET。下载成功只能证明一串 bytes 到了本地；Chromium Updater 还要校验大小与 hash，处理 CRX3，执行 installer，并根据 user/system scope 处理权限、进程和安装状态。

[Chromium Updater Functional Specification](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/functional_spec.md) 指出，Updater 接受 CRX3 更新包，包使用 publisher key 签名，相应公钥固化在客户端中；它还可以额外 pin 一枚 public key。在 Windows 上，最终执行的 Updater 与 installer 还应有正确的 Authenticode 签名。

### 4. Ping-Back：没有回报，发布者就看不到失败发生在哪里

安装尝试后，客户端会把结果回传给 Omaha Server。Protocol 4 为 download、CRX3 安装、installer 和其他 operation 定义了事件类型、结果、error category 与 error code。

这条回路不是装饰性 telemetry。如果只知道新版本文件已经上传，却看不到客户端是在检查、下载、验签、解包还是 installer 阶段失败，更新系统就没有可操作的反馈。

## CUP、CRX3 和 Authenticode 为什么一个都不能少

自动更新有一个很高的安全门槛：服务端的一次响应，有机会让大量客户端自动下载并执行代码。TLS 很重要，但 Chromium 的更新链路没有把全部信任都压在 TLS 上。

| 机制 | 主要保护的对象 | 能证明什么 | 不能代替什么 |
| --- | --- | --- | --- |
| TLS | HTTP 连接 | 传输加密和站点身份 | 不等于 payload 发布者签名 |
| CUP | Update Check 的 request/response 组合 | 响应完整性、新鲜度，以及对当次请求的绑定 | 不提供内容保密，不代替包签名 |
| CRX3 publisher signature | 更新 payload | 包来自客户端信任的 publisher key | 不代替 OS 对可执行文件的信任判断 |
| Authenticode | Windows 可执行文件 | 文件的签名者与签名后完整性 | 不代替 Omaha 渠道决策或 CUP |

[CUP 文档](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/cup.md) 对边界写得很明确：它通过 ECDSA key pair 保护请求体和响应体的完整性，并用 nonce 保护响应新鲜度；客户端内置公钥，服务端保存私钥。它不提供请求或响应的保密性，也不保护 HTTP header。

这意味着，自建 Omaha Server 时不能只写一个返回 JSON 的 API。CUP public key 必须在构建期进入客户端 trust root，服务端必须正确保管对应 private key，还要为旧客户端设计 key rotation 与历史 key 兼容。

## Omaha 不是 CDN、安装包或“最新版本 JSON”

在工程实践里，最容易跑偏的是用一个 `update.json` 包办整条更新链路。如果客户端只是拉取：

```json
{
  "version": "1.2.3",
  "url": "https://cdn.example.com/app.exe"
}
```

这可以是一个自定义 updater 的起点，但它还不是完整的 Omaha 链路。你仍然需要回答：

- App ID 和 user/system scope 如何注册、迁移和卸载？
- 不同 OS、CPU、channel 和 cohort 如何选择制品？
- 旧客户端如何确认 response 没有被替换？
- payload 如何验签，失败如何回退或切换备用 URL？
- 下载、解包、安装、重启各阶段的错误如何回报？
- 服务过载时，客户端如何 backoff，避免集体重试把服务压垮？

Omaha Protocol 对这些问题给出了一套经过大规模桌面软件验证的表达方式。但协议完整不等于你的发布系统已经完成：构建、签名、上传、渠道激活和旧客户端验收，仍然是五个独立状态。

## 自建 Chromium 桌面应用时，哪些东西必须属于你

Chromium Updater 是可定制的，但它不会自动让一个 Chromium fork 拥有自主更新能力。至少要管住下面几类产品身份和发布输入：

| 输入 | 为什么必须自有 |
| --- | --- |
| App ID、company/updater identity | 避免与其他 updater 冲突，并让客户端稳定识别已安装产品 |
| Omaha endpoint | 把检查请求发到自己控制的决策服务 |
| CUP public/private key | 建立客户端与服务端之间的更新指令信任根 |
| CRX3 publisher key | 让客户端只接受预期发布者的更新 payload |
| Code signing identity | 满足操作系统对 installer 和 executable 的信任要求 |
| 版本、channel 与 cohort 策略 | 决定谁可以在什么时候升级到哪个版本 |
| 不可变 artifact 与可变 descriptor | 先固化可回读的制品，再安全激活发布渠道 |

我的建议是，先画清这些责任边界，再决定服务端是自行实现、使用现成实现，还是采用另一套 updater。先选中一个开源项目，再反推产品 identity 和 key 如何放置，很容易留下无法平滑迁移的历史负担。

## 要不要用 Omaha：我的判断框架

### 值得上

你在做 Chromium fork、桌面客户端平台，或者需要一套兼容 Chromium Updater 的服务端；产品有明确的 App ID、多渠道或分批发布需求，也愿意持续管理 key、签名、制品和旧客户端兼容。

### 可以再等

你只有一个用户规模很小的内部工具，手动下载新版本的成本可接受；或者你还没有稳定的 code signing、artifact storage 和 release process。这时先把“可重复构建 + 正确签名 + 不可变上传”做稳，比匆忙增加 Omaha Server 更有价值。

### 重点关注

如果决定接入，优先验收一条真实旧客户端链路：发现新版本、验证 CUP response、下载 CRX3、校验 payload、执行已签名 installer、重启后读到新版本，同时保留用户数据。只调通一份 JSON response，证明不了这条链路。

这也是我现在对 Omaha 的最简单理解：**它是自动更新的决策与交互契约，不是一个替你完成整条发布的黑盒。**

## 参考资料

- [Omaha：Google Update for Windows](https://github.com/google/omaha)
- [Chromium Updater](https://chromium.googlesource.com/chromium/src/+/main/chrome/updater/)
- [Chromium Updater Design Document](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/design_doc.md)
- [Omaha Protocol 4.0](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/protocol_4.md)
- [Client Update Protocol（CUP）](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/cup.md)
- [Chromium Updater Functional Specification](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/functional_spec.md)
- [Chromium Updater Developer's Manual](https://chromium.googlesource.com/chromium/src/+/main/docs/updater/dev_manual.md)

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作。项目已开源，欢迎在 GitHub 点个 Star。
