---
tags:
  - 科技/AI/微调
publish date: 2025-07-13T07:33:00
reviewed date: 
title: 【2】模型的解码策略 Decoding Strategy
source: https://www.bilibili.com/video/BV15mPke1EiT/?spm_id_from=333.1387.collection.video_card.click&vd_source=cff3eef3abcdb3fcf7537244dd23cb21
description: 
obsidian-note-status:
  - completed
---
--- 

## 目录
- [0. 问题](#0-问题)
- [1. 解码策略 (Decoding Strategy) 概述](#1-解码策略-decoding-strategy-概述)
- [2. Greedy Search (贪心搜索)](#2-greedy-search-贪心搜索)
- [3. Beam Search (束搜索)](#3-beam-search-束搜索)
- [4. 代码实践演示：Greedy Search vs. Beam Search](#4-代码实践演示greedy-search-vs-beam-search)
- [5. Top-K Sampling](#5-top-k-sampling)
- [6. Top-P (Nucleus) Sampling](#6-top-p-nucleus-sampling)
- [7. Temperature (温度)](#7-temperature-温度)
- [8. 总结与实际应用](#8-总结与实际应用)
- [AI 总结](#ai-总结)

---
## 0. 问题
在使用大语言模型中，我们可以看到在对话界面的侧面有一些参数可调，在此之前只是简单实用用，没有细究过参数是什么，有什么作用。
下面就来研究下
![image.png](https://build-web.oss-cn-qingdao.aliyuncs.com/my_pic_file/20250712192101.png)
## 1. 解码策略 (Decoding Strategy) 概述

解码策略是指在模型进行推理（Inference）时，如何从预测的概率分布中选择下一个词元（Token）的方法。当模型根据输入文本预测下一个词元时，它会为词汇表中的所有词元生成一个概率分布。不同的选择策略会产生完全不同的输出文本，影响生成内容的相关性、多样性和创造性。
![image.png](https://build-web.oss-cn-qingdao.aliyuncs.com/my_pic_file/20250713075639.png)



主要讨论的解码策略包括：
- `Greedy Search`
- `Beam Search`
- `Top-K Sampling`
- `Top-P (Nucleus) Sampling`
- `Temperature`

## 2. Greedy Search (贪心搜索)

`Greedy Search` 是最直接、最常用（约99%的情况下）的解码策略，也是许多大语言模型的默认设置。

- **核心机制**：在每一步推理时，永远选择当前概率最高的那个词元作为输出。
- **优点**：
    - **速度最快**：计算开销最小，因为它只考虑一个选择。
    - **实现简单**：只需在模型的 `logits` 输出上应用 `softmax` 函数，然后取概率最大值的索引（`argmax`）。
- **缺点**：
    - **局部最优陷阱**：每一步都选择局部最优解（当前概率最高的词元），但这不保证最终生成的整个序列是全局最优的。
    - **缺乏多样性**：对于相同的输入，输出永远是固定的，内容可能显得重复和呆板。
- **推理过程**：
    1.  输入文本被分词器（Tokenizer）转换为词元序列。
    2.  模型根据当前词元序列预测下一个词元的概率分布。
    3.  选择概率最高的词元，并将其附加到现有序列的末尾。
    4.  将更新后的序列作为新的输入，重复第2步，直到达到最大长度或生成结束符。
- **示例**：
    - 输入：“一颗苹果树”
    - 模型预测下一个词元的概率：`下` (0.2), `，` (0.19), ...
    - `Greedy Search` 会选择 `下`，序列变为“一颗苹果树下”，然后基于这个新序列继续预测。如果当初选择了 `，`，后续生成的文本可能会完全不同（例如，引出牛顿的故事）。

## 3. Beam Search (束搜索)

`Beam Search` 是对 `Greedy Search` 的一种改进，它通过同时探索多条可能的路径来寻找全局更优的序列。

- **核心机制**：在每一步，保留 `k` 个（`k` 称为 `beam_width` 或束宽）概率最高的候选序列。在下一步，从这 `k` 个序列出发，生成所有可能的扩展序列，并再次选出总概率最高的 `k` 个。
- **推理过程**：
    1.  定义束宽 `k`（例如，`k=3`）。
    2.  在第一步，选择概率最高的 `k` 个词元，形成 `k` 个独立的初始序列（“光束”）。
    3.  在后续每一步，对这 `k` 个序列中的每一个进行扩展，生成新的候选序列。
    4.  计算所有候选序列的累积概率（通常是累加对数概率以避免浮点数下溢）。
    5.  从所有候选中选出累积概率最高的 `k` 个，作为新的“光束”，并丢弃其余的。
    6.  重复此过程，直到所有“光束”都达到结束条件。
    7.  最终，选择累积概率最高的那个完整序列作为最终输出。
- **优点**：
    - **生成质量高**：通过探索多条路径，能有效避免 `Greedy Search` 的局部最优问题，生成的句子通常更连贯、更合理。
- **缺点**：
    - **计算成本高**：需要维护 `k` 个序列，计算量和内存消耗是 `Greedy Search` 的 `k` 倍。
    - **不适用于流式输出**：必须等到所有“光束”都生成完毕后，才能比较并选出最优序列。这导致用户需要等待较长时间才能看到完整结果，不适合聊天机器人等需要逐字输出的应用场景。
- **适用场景**：研究、离线文本生成、翻译等对生成质量要求极高且不要求实时响应的任务。

## 4. 代码实践演示：Greedy Search vs. Beam Search

本节使用 `Hugging Face Transformers` 库和 `GPT-2` 模型进行演示。

- **输入提示 (Prompt)**：`"Beijing is the capital of"`
- **Greedy Search (默认) 结果**：
  - 输出：`"Beijing is the capital of China is China's second largest economy"`
  - **分析**：这是一个未经指令微调的基础语言模型。在训练数据中，"China" 后面很可能经常出现 "is"，导致模型在生成 "China" 后，以最高概率选择了 "is"，产生了一个不合逻辑的重复句子。
- **Beam Search 结果分析**：
  - 假设 `beam_width=2`，模型在第一步预测：
    - `China` (概率: 37.26%)
    - `the` (概率: 26.16%)
  - **路径1 (从 `China` 开始)**：`China` -> `is` -> `second` -> `largest` -> `economy`。这条路径的初始概率高，但后续词元的概率可能并不突出。
  - **路径2 (从 `the` 开始)**：`the` -> `People's` (概率极高) -> `Republic` (99%) -> `of` (95%) -> `China`。虽然 `the` 的初始概率较低，但后续词元的概率非常高。
  - **结论**：`Beam Search` 会计算两条路径的整体累积概率。路径2（生成 "the People's Republic of China"）的累积概率总和会超过路径1，因此 `Beam Search` 会选择这个更合理、更完整的答案作为最终输出。

## 5. Top-K Sampling

`Top-K Sampling` 是一种引入随机性的采样策略，旨在增加生成文本的多样性。

- **核心机制**：在每一步，不再只选择概率最高的一个，而是从概率最高的 `K` 个词元中进行随机抽样。
- **推理过程**：
    1.  模型预测出所有词元的概率分布。
    2.  筛选出概率最高的 `K` 个词元。
    3.  在这 `K` 个词元中，根据它们的相对概率（重新归一化）进行随机抽样，选择一个作为下一个词元。
- **优点**：
    - **增加多样性**：通过随机性，即使输入相同，每次生成的输出也可能不同。
    - **计算相对简单**：比 `Beam Search` 快，因为它只维护一条生成路径。
- **缺点**：
    - **固定的 `K` 值**：`K` 是一个固定值。当模型对下一个词非常确定时（例如，某个词概率为95%），`Top-K` 仍然会考虑其他 `K-1` 个选项，可能引入不相关的词。

## 6. Top-P (Nucleus) Sampling

`Top-P Sampling`（也称 `Nucleus Sampling`）是 `Top-K` 的一种更智能的改进，它使用动态的候选集大小。

- **核心机制**：选择一个累积概率阈值 `P`（例如，0.9）。然后，从按概率降序排列的词元中，选择一个最小的集合，使其累积概率刚好超过 `P`。最后，从这个集合（称为“核”，Nucleus）中进行随机抽样。
- **推理过程**：
    1.  将所有词元按概率从高到低排序。
    2.  从高到低累加它们的概率，直到总和大于等于 `P`。
    3.  将这些词元构成候选集（Nucleus）。
    4.  从这个候选集中随机抽样。
- **优点**：
    - **自适应候选集**：
        - 当模型非常**确定**时（某个词概率很高），候选集会很小（可能只有1个词），生成结果更稳定。
        - 当模型非常**不确定**时（多个词概率相近），候选集会变大，允许更多的多样性和创造性。
    - 通常被认为比 `Top-K` 更优越。

## 7. Temperature (温度)

`Temperature` 是一个非常常用且简单的参数，用于控制生成文本的随机性，它通过调整 `logits` 来改变 `softmax` 函数的输出分布。

- **核心机制**：在应用 `softmax` 之前，将模型输出的 `logits` 除以一个称为 `temperature` 的值。
  - **公式**: `new_logits = logits / temperature`
- **参数效果**：
    - **`Temperature < 1.0` (例如 0.5, 0.1)**：
        - `logits` 的值被放大，使得高概率词元和低概率词元之间的差距变大。
        - `softmax` 后的概率分布会变得更“尖锐”（sharper），模型会更倾向于选择概率最高的词元。
        - **效果**：降低随机性，使输出更具确定性、更保守。`T` 趋近于0时，效果类似于 `Greedy Search`。
    - **`Temperature = 1.0`**：
        - `logits` 不变，保持模型原始的概率分布。
    - **`Temperature > 1.0` (例如 1.5, 2.0)**：
        - `logits` 的值被缩小，使得高概率和低概率词元之间的差距变小。
        - `softmax` 后的概率分布会变得更“平坦”（flatter），增加了低概率词元被选中的机会。
        - **效果**：增加随机性，使输出更具多样性、更富创造性，但可能会牺牲一定的连贯性。
- **示例**：
	![image.png](https://build-web.oss-cn-qingdao.aliyuncs.com/my_pic_file/20250713081136.png)

| 原始概率 (T=1.0)                 | T=0.5 后的概率                   | T=0.1 后的概率                   |
| ---------------------------- | ---------------------------- | ---------------------------- |
| A: 63%, B: 2%, C: 34%, D: 1% | A: 77%, B: 0%, C: 23%, D: 0% | A: 100%, B: 0%, C: 0%, D: 0% |


*(注：表格为视频中展示的示意数据，说明降低温度会放大最高概率项的优势)*

## 8. 总结与实际应用

- **常用组合**：在实际应用中，最常见的组合是 **`Greedy Search` + `Temperature`**。这种方式既能保证推理速度（`Greedy Search` 的优势），又能通过调整 `Temperature` 来控制输出的多样性。
- **高级组合**：为了获得更高质量的生成结果，可以组合使用多种策略，例如 **`Temperature` + `Top-P` + `Beam Search`**。
- **实际场景中的选择**：
    - **聊天机器人 (Chatbot)**：通常使用流式输出（逐字显示），因此不适合 `Beam Search`。大多采用 `Greedy Search` 或 `Top-P`/`Top-K` 采样，并配合 `Temperature` 来平衡回答的准确性和趣味性。
    - **需要确定性答案的任务**（如数学计算、代码生成、事实问答）：倾向于使用低 `Temperature`（如 0.1）和 `Greedy Search`，以确保输出最可靠的结果。
    - **需要创造性的任务**（如写诗、故事创作）：倾向于使用较高的 `Temperature` 和 `Top-P` 采样，以激发模型的创造力。
- **开发者界面**：`ChatGPT` 等面向用户的产品通常不会暴露这些参数，但其背后的 API 和开发者平台（如 Playground）会提供这些选项供开发者微调。

---

## AI 总结
本视频详细介绍了大型语言模型在生成文本时所采用的多种**解码策略**。这些策略决定了模型如何从一系列可能的下一个词元中进行选择，直接影响了输出内容的质量、多样性和创造性。

核心策略包括：
1.  **`Greedy Search`**：最简单快速的方法，每一步都选择概率最高的词元。虽然高效，但可能导致结果单调且陷入局部最优。
2.  **`Beam Search`**：通过同时探索多个（`k`个）高概率路径，来寻找全局最优的文本序列。它能生成更高质量的文本，但计算成本高，且不适用于需要实时逐字返回的流式应用。
3.  **采样方法 (`Top-K` 和 `Top-P`)**：这些策略通过从一个经过筛选的候选词元集中进行随机抽样，为生成过程引入了多样性。`Top-P` (Nucleus Sampling) 因其能根据模型置信度动态调整候选集大小，通常被认为比固定数量的 `Top-K` 更为智能。
4.  **`Temperature`**：这是一个关键参数，通过调整模型输出的概率分布来控制随机性。低温（<1.0）使输出更具确定性和保守性，而高温（>1.0）则增加创造性和多样性。

在实际应用中，`Greedy Search` 结合 `Temperature` 是一个兼顾速度和可控多样性的流行方案，尤其适用于聊天机器人等流式场景。而对于追求极致生成质量的离线任务，则可以组合使用 `Beam Search`、`Top-P` 和 `Temperature` 等多种策略。理解这些策略的原理和权衡对于有效利用和开发大语言模型至关重要。