---
title: RSS 到底是什么：为什么 FiaOS 性能评测不能只看一个内存数字
date: 2026-08-24 20:47:17
description: 最近做 FiaOS 性能 baseline 时，同一轮空闲采样里，进程树 RSS 均值是 949.4 MiB，Physical Footprint 均值却只有 355.5 MiB。本文从这组真实数据出发，讲清 Resident Set Size 的统计边界、Chromium 多进程下的重复计数，以及判断内存回归和泄漏时应该怎样选指标。
categories:
  - [软件工程]
tags:
  - RSS
  - Resident Set Size
  - Performance
  - Chromium
  - macOS
  - FiaOS
  - Memory
cover: /images/rss-resident-set-size.webp
---

最近在整理 FiaOS 的性能 baseline 时，我碰到一组很容易被误读的数据：同一轮空闲采样中，完整进程树的 RSS 均值是 `949.4 MiB`，macOS Physical Footprint 均值却只有 `355.5 MiB`。

两个数字相差约 2.7 倍。是采集脚本错了，还是浏览器真的吃掉了近 1 GB 物理内存？都不是。真正的问题在于：我们把两个统计口径不同的指标，当成了同一个“内存占用”。这也是我想单独写清 RSS 的原因。

<!-- more -->

## 一句话总结

RSS（Resident Set Size）表示一个进程当前驻留在物理内存中的页面规模。它适合观察同一口径下的趋势和拆解 Browser、Renderer、GPU、Utility 等进程角色，但在 Chromium 多进程架构里，把各进程 RSS 直接相加可能重复计算共享页，所以它不等于应用独占的真实物理内存，更不能单凭一次上涨就判定内存泄漏。

在 macOS 上，我的判断很明确：

- 看整个应用给系统造成的内存压力，以进程树的 Physical Footprint 为主；
- 看内存主要分布在哪类进程、版本前后趋势是否异常，以同口径 RSS 为辅；
- 判断是不是泄漏，还要继续进入 heap、allocation、对象生命周期和 GC 证据，不能停在 RSS。

## RSS 到底统计了什么

操作系统按页管理内存。进程的虚拟地址空间可能很大，但其中只有一部分页面此刻真正驻留在 RAM；RSS 统计的就是这部分 resident pages 的规模。

在 macOS 上，`ps` 对 `rss` 的定义是进程的 real memory resident set，并以 `1024 bytes` 为单位输出。FiaOS 的采样器读取的是：

```bash
ps -axo pid=,ppid=,rss=,%cpu=,command=
```

这里的 `rss` 不是 JavaScript heap，也不是应用申请过的全部虚拟地址，更不是“这个应用独占了多少内存”。几个常见指标可以这样区分：

| 指标 | 回答的问题 | 主要局限 |
| --- | --- | --- |
| VSZ / Virtual Size | 进程映射了多大的虚拟地址空间 | 大量地址可能没有驻留，通常不能直接代表内存压力 |
| RSS | 这个进程当前有多少页面驻留在 RAM | 可能包含 clean、file-backed 和共享页；跨进程相加可能重复 |
| Heap | 某个运行时或分配器管理了多少对象内存 | 只是进程内存的一部分，不包含全部 native、映射文件和共享内存 |
| Physical Footprint | 从系统记账角度，这个进程或进程组带来多少物理内存负担 | 更适合 macOS 压力判断，但不直接告诉你具体是哪类对象在增长 |

Apple 在内存优化文档里也把 dirty、clean、compressed 等页面区别开来，并建议结合真实 memory pressure 理解应用影响，而不是只盯一个总数。参见 [Reducing your app’s memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)。

## 为什么 Chromium 多进程会把 RSS “加大”

Chromium 不是单进程程序。Browser、Renderer、GPU、Network Service、Storage Service 等角色彼此隔离，这也是它稳定性和安全性的重要基础。Chromium 官方的 [Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/) 文档对此有完整说明。

问题在于，不同进程可以映射同一份共享库、共享内存或 file-backed page。对单个进程看，这些页面确实属于它的 resident set；但如果把多个进程的 RSS 直接相加，同一批物理页就可能在多个进程名下重复出现。

可以把它想成一套公共技术手册：五个团队的桌上都放着它。按“每个团队正在使用多少资料”统计，这套手册会出现五次；按“公司仓库实际占了多少空间”统计，它只有一套。两个数字都没有错，只是在回答不同的问题。

这也是为什么 FiaOS 同时采集两条线：

```bash
# 拆解进程角色，并得到每个进程的 RSS
ps -axo pid=,ppid=,rss=,%cpu=,command=

# 对同一组 PID 做 macOS Physical Footprint 统计
footprint -j result.json -f bytes --noCategories <pid...>
```

`footprint` 在接收多个进程时，会把多重映射的对象去重，并把共享部分单独记账。因此，RSS 汇总和 Physical Footprint 出现明显差异，本身并不是异常。

## 回到 FiaOS：949.4 MiB 和 355.5 MiB 各代表什么

2026 年 8 月 19 日，我用 FiaOS CLI `1.2.0` 的本地 Release 构建，为 Fia `0.1.0` 建立了第一份 clean-profile 正式 baseline。测试环境是 macOS 26.5.2 arm64，临时干净 profile，只打开 `https://example.com/`，预热 5 秒后进入空闲采样。

| 采样项 | 结果 | 采样口径 |
| --- | ---: | --- |
| RSS 均值 | 949.4 MiB | 进程树 5 次采样，间隔 1 秒 |
| RSS 中位数 | 948.5 MiB | 同上 |
| Physical Footprint 均值 | 355.5 MiB | 同一进程树，前 3 次采样 |
| 空闲 CPU 均值 | 1.9% | 同一空闲窗口 |
| CDP reload CPU 峰值 | 66.2% | 40 次采样，间隔 200 ms |

RSS 的价值在角色拆解上尤其直观。下面是最后一次空闲采样的分布，总计 `948.5 MiB`：

| 进程角色 | 数量 | RSS |
| --- | ---: | ---: |
| Browser | 1 | 231.3 MiB |
| Renderer | 3 | 322.5 MiB |
| GPU | 1 | 109.3 MiB |
| Utility - Network | 1 | 78.8 MiB |
| Utility - Storage | 1 | 66.5 MiB |
| Runtime | 1 | 140.1 MiB |

这张表能快速回答“主要增长发生在哪个角色”。例如 Renderer 突然多出一个，或者 Runtime RSS 连续抬升，调查方向会完全不同。但它不能回答“这 948.5 MiB 是否全部由 Fia 独占”。对于后一个问题，`355.5 MiB` 的 Physical Footprint 更接近我关心的系统压力口径。

## RSS 上涨不等于发生了内存泄漏

内存泄漏会推高 RSS，但 RSS 上涨还有很多正常原因：

- Chromium 新建了 Renderer 或 Utility 进程；
- 页面缓存、图片解码缓存或磁盘缓存映射增加；
- V8 JIT 编译了更多代码，heap 也扩过容；
- allocator 暂时保留已释放页面，没有立即归还操作系统；
- file-backed 页面因为刚被访问而进入 resident set；
- 对象仍然可达，只是产品逻辑保留时间比预期更长。

因此，“跑一次后 RSS 比之前高”最多是一个调查信号，不是泄漏结论。更可靠的链路应该是：先固定 workload 和进程范围复现趋势，再用 heap snapshot、allocation profile、对象数量、GC 后基线和进程生命周期去定位原因。

如果多轮相同操作后，RSS 与 Physical Footprint 都阶梯式增长，强制 GC 或回到空闲状态后仍不下降，同时 heap 或 native allocation 里能看到稳定增长的保留路径，才有资格把结论推进到“疑似泄漏”。

## 性能评测里，怎样比较 RSS 才不误导

我现在会把 RSS 对比当成一个有前置条件的实验，而不是把两个数字直接相减。至少要锁定下面六件事：

| 必须一致的条件 | 为什么 |
| --- | --- |
| 指标定义 | RSS 只能和 RSS 比，不能拿 RSS 与 Physical Footprint 做版本涨跌 |
| 进程范围 | 只看 Browser 与统计完整进程树，结论会完全不同 |
| workload | 空白页、真实业务页、视频页的进程和缓存结构不同 |
| 生命周期 | 冷启动、预热后空闲、操作中、操作后稳定态不能混在一起 |
| 构建产物 | Debug、Release、符号、Feature Flag 都可能改变结果 |
| 采样窗口 | 单点很容易受进程启动、GC、缓存和后台任务干扰 |

这也解释了为什么这份 baseline 没有拿历史报告中的 `1.53–1.67 GB RSS` 直接宣称“大幅下降”：历史数据的进程树、页面负载和采样口径并不完全一致。当前这组数据首先是一条可复现的基线，只有后续版本在同机、同 workload、同范围、同窗口下复测，才形成有效比较。

Apple 对性能回归的建议也是建立可重复的测试和 baseline，再观察变化，而不是靠一次快照下结论。参见 [Preventing memory-use regressions](https://developer.apple.com/documentation/xcode/preventing-memory-use-regressions)。

## 要不要看 RSS / 我的判断框架

RSS 值得看，但要放在正确的位置上。

| 你要回答的问题 | 优先看什么 | 我的判断 |
| --- | --- | --- |
| 内存主要花在哪类进程 | 分角色 RSS | 值得看，定位速度很快 |
| 新版本是否出现同口径回归 | RSS 趋势 + Physical Footprint 趋势 | 两条线一起看，避免统计口径误导 |
| macOS 上整个应用造成多大内存压力 | 进程树 Physical Footprint + memory pressure | 以此为主，不用 RSS 汇总替代 |
| 是不是 JavaScript 对象泄漏 | Heap snapshot、allocation、GC 后保留对象 | RSS 只能报警，不能定罪 |
| 用户是否真的感知到退化 | 页面响应、切换延迟、崩溃、系统压力 | 最终仍要回到体验和稳定性 |

我的核心原则只有一句：先问指标在回答什么问题，再看数字大小。

RSS 不是“假数据”，Physical Footprint 也不是 RSS 的修正版。它们是从不同层面观察内存的两把尺子。FiaOS 性能治理真正需要的，不是寻找一个万能数字，而是把进程结构、系统压力、运行时分配和用户体验连成一条证据链。

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作，配图使用 [better-imagegen](https://github.com/zisheng-ai/better-imagegen) 生成。项目已开源，欢迎在 GitHub 点个 Star。
