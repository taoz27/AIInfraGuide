---
title: "第5章：Speculative Decoding"
description: "掌握投机解码的 Draft + Verify 框架与无精度损失的验证机制，沿'猜得准、猜得快、系统权衡'三条主线深入 EAGLE-3、DeepSeek MTP、DFlash 与 DSpark 的原理与取舍"
pubDate: 2026-04-16
updatedDate: 2026-09-18
category: "inference-optimization"
order: 34
tags: ["Speculative Decoding", "投机解码", "EAGLE-3", "MTP", "DFlash", "DSpark", "接受长度"]
---

## 本章简介

LLM 是自回归模型，Decode 阶段一个 Token 一个 Token 串行输出，且是 Memory Bound——这是推理慢的总根源。Speculative Decoding（投机解码/投机采样）利用"验证 k 个 Token 与验证 1 个几乎一样便宜"这一红利：小草稿模型先猜、Base 模型一次前向统一验证，在**无精度损失**的前提下打破串行瓶颈。

**核心原理**讲清 Draft + Verify 框架：以贪婪采样为例逐步拆解验证流程（Logits 逐位比对、最长接受前缀、奖励 Token），论证为什么无损，并推导加速比与接受长度的关系——草稿质量太差时接受长度过低，反而会更慢。这是全章的公理层。

**猜得更准（提升接受长度）**：EAGLE-3 指出"到 logits/最后一层 Hidden States 时很多语言信息已丢失"，将 Base 低、中、高多层特征喂给草稿，并用 top-k 展开加概率剪枝保留等宽草稿树；MTP 则是训练优化的附加收益——主模型自带的 MTP 头在推理时顺手当草稿，形成线性无分支的草稿序列。

**猜得更快（压缩草稿耗时）**：DFlash 注意到草稿模型自己也是自回归的，用块扩散（双向注意力）一次前向生成整块草稿，并将 Base 的 Hidden States 映射后直接注入草稿 KV 计算以保住接受率，代价是块内独立性带来的后缀衰减。

**系统级权衡**：DSpark 兼取 EAGLE-3 与 DFlash 之长——半自回归结构（并行主干 + 串行采样头）与预测接受概率的 Confidence Head；系统层面围绕"全局吞吐量 = 接受长度 × 每秒 Decode 次数"，按 Token 接受概率与系统资源动态决定验证预算，是投机解码从算法走向系统联合优化的代表。

## 本章小节

- **5.1 投机解码核心原理**：Draft + Verify、无精度损失的验证机制、加速比与接受长度
- **5.2 EAGLE-3**：多层 Hidden States 特征级草稿、top-k 等宽树状草稿、Training-Time Test
- **5.3 MTP**：训练优化的附加收益、线性无分支草稿、一个头的工程现实
- **5.4 DFlash**：块扩散一次前向出整块草稿、Base 特征注入 KV、后缀衰减
- **5.5 DSpark**：半自回归结构、Confidence Head、接受长度 × Decode 次数的系统权衡
