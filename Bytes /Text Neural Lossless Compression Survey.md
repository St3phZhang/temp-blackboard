# Bytes / Text Neural Lossless Compression 综述：统一框架、模型层、编码层与工程技巧

## 摘要

2023 年以后，neural lossless compression 在文本和 byte stream 上形成了几条清晰路线：

1. **大语言模型概率编码路线**：用 LLM 预测下一个 token/byte 的概率，再接 arithmetic/range coding。代表是 `LLMZip`、`Language Modeling Is Compression`、`FineZip`、`Nacrith`。
2. **rank / error-event 近似路线**：不用完整概率分布，改为记录真实 token 的 rank，或只记录预测失败事件，再用传统 compressor 二次压缩。代表是 `AlphaZip`、`FineZip` 主实用路径、`Llamazip`。
3. **局部适配与混合路线**：解决静态 LLM 对 OOD 文本适配差的问题，用 wPoE、频率表、n-gram、LoRA、adaptive bias 等修正概率。代表是 `Test-Time Steering / wPoE`、`Context-Aware Frequency Adaptation`、`Nacrith`、`Haiku to Opus` 的 LoRA 部分。
4. **轻量序列模型路线**：用 RWKV、Mamba/SSM、xLSTM、小型 MLP/CNN predictor 替代大 Transformer，以牺牲部分极限压缩率换取速度、端侧部署或自包含。代表是 `L3TC`、`StateSMix`、`BOA Constrictor`、`Chained Lightweight Neural Predictors`。
5. **结构化 byte / 多源数据路线**：把文本、音频、图像、float、genome、heterogeneous byte stream 统一看作 byte 序列或结构化 record，用轻量 learned predictor、autoencoder、并行 pipeline 压缩。代表是 `ByteZip`、`MSDZip`、`EDPC`、`FADE`。
6. **非文本 bytes 序列化路线**：核心问题不是“能否用 LM”，而是如何把 float/audio 等非文本 bytes 变成 LM 可建模的符号流。代表是 `LM-GC` 和 `Trilobyte`。

总体来看，几乎所有严格无损方案都可归纳为：

```text
输入数据与符号化
  -> 上下文构造
  -> 概率模型 / 预测器
  -> 概率校准、混合、适配
  -> 概率到整数 CDF / rank / residual / error-event 的表示
  -> entropy coder 或 secondary compressor
  -> 文件格式、模型共享与解码同步
  -> 工程加速与部署约束
```

这篇综述按这个框架重新组织当前本地已整理的 bytes/text 相关论文。

---

## 1. 统一框架：从数据到 bitstream

### 1.1 数据与符号化层

符号化层决定模型看见什么，是 bytes/text neural compression 的第一分叉。

| 符号化方式 | 代表论文 | 优点 | 问题 |
|---|---|---|---|
| 原始 byte | `Compression via Pre-trained Transformers`、`BOA`、`EDPC`、`FADE` | 模态无关、严格可逆、适合任意 byte stream | token 信息密度低，模型需学习底层局部结构 |
| 字符 / ASCII | `LLMZip` 的 text8 设置、传统 benchmark | 与 bpc 指标直接对应 | 不适合 Unicode、多语种、二进制 |
| BPE / subword token | `LLMZip`、`FineZip`、`Nacrith`、`L3TC`、`StateSMix` | 每 token 覆盖更多 bytes，文本预测更强 | 需要共享 tokenizer；大词表带来 CDF 量化开销 |
| rank 序列 | `AlphaZip`、`FineZip`、`Context-Aware Frequency Adaptation` | 可把 LLM 输出转成偏斜整数序列，再用传统压缩器 | 丢失概率幅度信息，不如直接 AC 精确 |
| error event / failed prediction | `Llamazip` | 预测命中时几乎零成本；适合模型记忆文本 | 对未见文本不稳定；强依赖训练集污染 |
| fixed record / chunk tensor | `ByteZip` | 利用结构化 byte stream 的字段布局 | 需要预处理 schema / record 对齐 |
| non-text serialization | `LM-GC`、`Trilobyte` | 把 float/audio 等边界数据转成 LM 可处理符号 | serialization 设计本身成为核心难题 |

**关键结论**：符号化不是无关紧要的预处理。对于 LLM/text 压缩，它决定语言模型的预测质量；对于 non-text bytes，它决定模型能否看见原始数据的结构边界。

### 1.2 上下文构造层

压缩等价于预测，预测依赖上下文。各论文的上下文策略可分为：

1. **固定滑窗**：`LLMZip`、`Llamazip`。简单，但长上下文慢，短上下文损失压缩率。
2. **chunk 内 dynamic context**：`FineZip`。chunk 内第 `i` 个 token 只看前 `i-1` 个 token，便于 batch 推理。
3. **固定 byte chunk**：`Language Modeling Is Compression`、`wPoE`、`Compression via Pre-trained Transformers`。方便实验和模型批处理，但会伤害传统 compressor 的长上下文优势。
4. **Equal-Info Windows**：`Training LLMs over Neurally Compressed Text`。按输出 bit 数切窗，让压缩文本更可学习。
5. **在线 predict-encode-update**：`StateSMix`、`MSDZip`、`EDPC`、`FADE`。模型状态随输入在线变化，解码端必须同步更新。
6. **多流/多 chunk 并行上下文**：`BOA`、`Nacrith`、`FADE`。牺牲跨流上下文，换取并行速度。
7. **结构化扫描顺序**：`ByteZip` 的 record/chunk、`Trilobyte` 的 sample-to-byte、`LM-GC` 的 float-to-hex grouping。

### 1.3 概率模型层

模型层负责估计：

```text
p(symbol_t | context_t)
```

现有论文使用的模型选项如下：

| 模型层选项 | 代表论文 | 定位 |
|---|---|---|
| 大 LLM / foundation model | `LLMZip`、`Language Modeling Is Compression`、`FineZip`、`Hybrid-LLM` | 极强文本概率模型，速度和模型体积是瓶颈 |
| 小型 LLM / 135M 级模型 | `Nacrith` | 在压缩率和工程成本之间折中 |
| GPT-2 / LLaMA rank predictor | `AlphaZip`、`FineZip`、`Context-Aware Frequency Adaptation` | 生成 rank 序列供传统 compressor 压缩 |
| byte-level Transformer | `Compression via Pre-trained Transformers`、`Training LLMs over...` | 统一 raw byte 建模，可跨文本/图像/音频 |
| RWKV | `L3TC` | 低复杂度文本 compressor，强调端侧解码速度 |
| Mamba / SSM | `StateSMix`、`BOA` | 线性时间序列建模，适合在线文本或结构化 byte stream |
| xLSTM | `MSDLC`（源码索引）、相关多源压缩 | 多源 byte 数据的轻量序列模型选项 |
| MLP/CNN lightweight predictor | `EDPC`、`FADE`、`Chained Predictors` | 提升吞吐、降低显存，更适合通用 byte stream |
| Autoencoder + mixture density | `ByteZip` | 非自回归块级结构化 byte 压缩 |
| n-gram / Naive Bayes / frequency model | `wPoE`、`Nacrith`、`StateSMix`、`Context-Aware Frequency Adaptation` | 作为局部统计专家修正神经模型 |

### 1.4 概率校准、混合和适配层

这层是近两年论文最活跃的地方。单一预训练模型通常不够，需要让分布更贴近当前文件/当前领域。

| 技巧 | 代表论文 | 作用 |
|---|---|---|
| LoRA / QLoRA online memorization | `FineZip` | 在压缩前微调，使待压文本概率更高 |
| domain LoRA + router | `Haiku to Opus` | 不同领域用不同 adapter，提高 LLM-generated text 压缩率 |
| Weighted Product of Experts | `Test-Time Steering / wPoE` | 把 LM 与 Naive Bayes 等专家用乘积方式融合，提升 OOD 表现 |
| Context-aware frequency bias | `Enhancing LLM-Based Text Compression...` | 局部频率表加到 logits 上，低成本自适应 |
| token n-gram + context mixer | `Nacrith` | 结合 LLM 全局语义和文件内局部重复 |
| adaptive log-space bias head | `Nacrith` | 在线修正当前文档中的 LLM 偏差 |
| entropy-adaptive n-gram logit bias | `StateSMix` | SSM 不确定时提高 n-gram 权重，SSM 自信时退场 |
| information inheritance | `Chained Lightweight Predictors` | 高阶 predictor 继承低阶 logits，类似神经版 PPM backoff |
| local/global feature decoupling | `FADE` | 将局部微语法与长程宏语义分流建模 |
| CDF precision / logit quantization | `Nacrith`、`Hybrid-LLM` | 前者降低大词表量化浪费，后者保证跨硬件确定性 |

### 1.5 编码层

编码层把概率或中间表示变成 bitstream。

| 编码层 | 代表论文 | 说明 |
|---|---|---|
| arithmetic coding | `LLMZip`、`Language Modeling Is Compression`、`Nacrith`、`wPoE`、`StateSMix`、`LM-GC`、`Trilobyte` | 主流严格无损后端，能直接利用任意条件概率 |
| range coding | `BOA` | arithmetic coding 的整数流式变体，适合 streaming |
| rank + zlib / bzip2 / Brotli | `AlphaZip`、`FineZip`、`Context-Aware Frequency Adaptation` | 工程简单，但丢掉概率幅度信息 |
| error run-length + gzip | `Llamazip` | 只记录预测失败位置和 token，适合模型记忆文本 |
| rANS / ANS | `RAS` 以及概率电路相关 prior | 高吞吐 entropy coding 方向，硬件友好 |
| residual / mixture density + arithmetic coder | `ByteZip` | 结构化 byte tensor 的非自回归多尺度编码 |
| learned discrete latent | `Seq2Seq2Seq` | 让模型直接学压缩 token 序列，边界上不如 AC 路线严格 |

### 1.6 同步、文件格式和可部署性

无损压缩的关键不是“编码端能压”，而是“解码端能产生完全相同的概率”。这里有四类同步模式：

1. **共享静态大模型**：`LLMZip`、`Language Modeling Is Compression`、`AlphaZip`。简单但模型体积巨大。
2. **共享静态模型 + 小 adapter / 参数**：`FineZip`、`Haiku to Opus`。需把 LoRA 或 adapter 作为 side information。
3. **在线自包含同步训练**：`StateSMix`。双方从同一随机初始化开始，根据已解码 token 同步更新模型。
4. **显式容器 / 确定性协议**：`Nacrith`、`Hybrid-LLM`、`BOA`。开始重视 header、chunk table、模型 fingerprint、logit quantization、CRC 等工程问题。

---

## 2. LLM 文本压缩路线

### 2.1 直接概率编码：LLM 作为 entropy model

`LLMZip` 是这一路线的起点：

```text
文本 -> tokenizer -> LLM next-token PMF -> arithmetic coder -> bitstream
```

它在 text8 1MB 子集上达到约 **0.7101 bpc**，在一个 2023 后发布的新书片段上约 **0.8426 bpc**。但速度极慢，后续论文引用 LLMZip 压 10MB 可到 **9.5 天**量级。

`Language Modeling Is Compression` 则把这件事扩展为一般原理：任何预测模型都能通过 arithmetic coding 变成 compressor；反过来，任何 compressor 也可看作 predictor。它还强调 adjusted compression rate：如果把模型大小算进压缩文件，70B LLM 往往不划算。

### 2.2 rank 路线：用 LLM 排序，传统 compressor 收尾

`AlphaZip` 和 `FineZip` 的实用路径都大量使用 rank：

```text
文本 -> LLM logits -> true token rank -> rank sequence -> zlib / bzip2 / Brotli
```

rank 的优点是实现简单、可绕过高精度 CDF 问题；缺点是丢失概率幅度信息。`FineZip` 发现 bzip2 压 rank 序列较好，并通过 dynamic context 让 chunk 内 token 并行预测。它把 LLMZip 的 10MB 任务从约 9.5 天压到约 **4 小时**，但绝对速度仍很慢。

### 2.3 失败事件路线：只存模型猜错的 token

`Llamazip` 更激进：如果 LLaMA3 预测正确，就什么也不存；只记录连续正确次数和失败 token。它在疑似训练集文本如 `alice29` 上可达到 **0.200 bpc**，但这强烈依赖模型记忆。对未见文本表现明显下降，因此更适合作为训练集检测/模型记忆分析工具，而不是通用压缩器。

### 2.4 局部适配：从静态 LLM 到当前文件分布

静态 LLM 的核心问题是 distribution mismatch。几篇论文分别从不同层面修正它：

- `FineZip`：压缩前用 LoRA/QLoRA 对当前文本做 online memorization。
- `Test-Time Steering / wPoE`：用 weighted product of experts 把 LM 与 Naive Bayes/Laplace smoothing 合成一个新分布，理论上最优权重不差于最佳单 expert。
- `Context-Aware Frequency Adaptation`：在每个 window 内构造局部频率表，把 `α log P_local` 加到 LLM logits。
- `Nacrith`：把 SmolLM2-135M 与 token n-gram、adaptive bias head、confidence skip 组合成一个 ensemble。
- `Haiku to Opus`：用领域 LoRA adapter + RAG router 提升 LLM-generated text 的 lossless arithmetic coding。

这些方法的共同点是：

```text
预训练模型给全局语义先验；局部统计或小参数适配负责当前文件分布。
```

### 2.5 系统化：Nacrith 与 Hybrid-LLM

`Nacrith` 是目前这批文本论文里最接近“完整压缩系统”的工作之一。它在 `enwik8` 上达到 **0.9389 bpb**，在 `alice29` 上 **0.918 bpb**，OOD 政府报告上 **0.723 bpb**。系统贡献包括：

- CDF-24，缓解大词表 arithmetic coding 的量化浪费；
- llama.cpp 后端；
- KV cache 原地 shift；
- n-gram + adaptive bias + confidence skip；
- NC05/NC06 容器；
- binary chunk fallback。

`Hybrid-LLM` 则从存储系统角度处理 LLM 压缩的可部署性：用 Zstd scout 做 block routing，只有适合语义压缩的 block 才走 LLM；同时用 logit quantization 解决跨硬件浮点非确定性。它在 memorized literature 上报告 **0.39 BPC**，在 unseen news 上 **0.75 BPC**，但约比 Zstd 慢 **2600×**，定位 ultra-cold archival。

### 2.6 反例与边界

`Comparing Text Compression Capabilities...` 是重要降温论文：LLaMA-3.2-1B 在非英语文本和打乱/替换文本上明显输给传统 compressor。它说明：LLM text compression 的强结果往往依赖英文、自然语言结构、训练分布与模型先验。

---

## 3. Raw byte 与 compressed-token 路线

### 3.1 Raw byte Transformer

`Compression via Pre-trained Transformers` 直接训练 small decoder-only Transformer 处理 raw byte sequence，避免 tokenizer 预压缩。它在 1GB OOD 数据上将模型大小计入压缩率，报告如 audio compression ratio **0.49**，优于 FLAC **0.54**。

但速度极慢。1GB text 压缩用时约 **452,355 秒**，远慢于 gzip 的 **102 秒**。它的意义更多是证明“合适规模的小 byte Transformer + 大训练集”可在 adjusted compression ratio 上进入传统 codec 竞争区间，而不是实用速度突破。

### 3.2 Equal-Info Windows：压缩文本作为 LLM tokenization

`Training LLMs over Neurally Compressed Text` 关注的不是文件压缩，而是能否训练 LLM 读取被神经 compressor 压过的文本。它发现：朴素 arithmetic-coded bitstream 过于随机，标准 Transformer 学不会。解决方案是 Equal-Info Windows：每个窗口压成相同 bit 数，并重置 AC 与 M1 上下文。

最强 setting `EqualInfoAC[b=16, v=65k]` 在 2B 模型上约 **0.94 bits/byte**，比 byte baseline 好，但仍弱于 SentencePiece。它说明一个重要原则：

```text
最优熵编码输出 ≠ 最适合下游 Transformer 学习的 tokenization。
```

### 3.3 Discrete latent token 压缩

`Seq2Seq2Seq` 用 T5-small + RL 学习一个可变长离散中间表示。它不是显式概率模型 + coder，而是 compressor 生成 token IR，decompressor 重构原文。enwik8 上 compression ratio 约 **4.12**，略好于 XZ **4.0**，弱于 NNCP **6.7**。

这个方向更像 learned discrete representation，而不是成熟 entropy-coded compressor。综述中应将其作为边界路线：它把“编码层”也学习化了，但严格无损性和可复现性需要更谨慎看。

---

## 4. 高效序列模型：RWKV、Mamba、轻量 predictor

### 4.1 RWKV / L3TC

`L3TC` 的目标是解决 learned text compression 太慢。它比较 Transformer、Transformer-XL、RWKV，最后选择 RWKV。最终系统包括：

- outlier-aware tokenizer；
- RWKV backbone；
- high-rank reparameterization；
- arithmetic coding；
- outlier byte bypass。

指标上，L3TC-3.2M 在 enwik9 上 CR 约 **16.23%**，计入模型的 ACR 约 **16.87%**，相比 gzip 约省 **48% bits**。速度上，L3TC-200K 在 A100 上解码 **4.35 MB/s**，移动端 iPhone12 ANE 上 **1.30 MB/s**。

### 4.2 在线 Mamba / StateSMix

`StateSMix` 不使用预训练模型，而是从随机初始化开始，边压缩边训练一个小型 Mamba-style SSM。它还叠加 sparse n-gram logit bias、LZ hash、recency bias、global frequency prior。

它的独特同步模式是：编码端和解码端都根据已处理 token 执行相同在线训练，因此模型知识隐含在码流诱导的训练轨迹里。无需外部权重，文件自包含。

指标：

- enwik8 10MB：**2.161 bpb**，比 `xz -9e` 好 0.7%；
- enwik8 100MB：**2.130 bpb**；
- 4 CPU cores + AVX2：约 **2,000 tok/s**。

### 4.3 Mamba byte predictor / BOA

`BOA Constrictor` 直接做 byte-level Mamba compression，面向高能物理结构化数据。输入是 256 byte alphabet，模型输出下一 byte 分布，range coder 编码。

指标：

- CMS：BOA **44.14×**，LZMA-9 **27.14×**；
- ATLAS：BOA **2.21×**，LZMA-9 **1.69×**；
- HEPMC：BOA **8.99×**，LZMA-9 **5.39×**。

这说明 Mamba 不只适合文本 token，也适合结构化 byte stream。但论文承认 PyTorch 浮点实现跨硬件不够 deterministic，吞吐也仍需优化。

### 4.4 Chained Lightweight Neural Predictors

`Chained Lightweight Neural Predictors` 把输入先 BPE tokenization，再用按 Markov order 组织的一串轻量 predictor：`s={1,2,3,4,8,16}`。高阶 predictor 通过 information inheritance 继承低阶 logits，类似神经版 PPM backoff。

它是半自适应方案：为当前文件训练部分模型，再把量化后的模型权重和词表一起写入 header。优点是解码端不需要在线 backprop。

指标：

- 压缩率接近 PAC；
- 编码吞吐约 **43–244 KB/s**；
- 解码吞吐约 **98–429 KB/s**；
- 相比 PAC，解码加速 **2.8–12.3×**。

---

## 5. 结构化 bytes 与多源 learned compression

### 5.1 ByteZip：结构化 byte stream 的非自回归方案

`ByteZip` 面向已知或可定义结构的 byte stream。它把重复 packet 合并为 record，对变长 record padding，再 reshape 成固定 `N x H x W` chunk。模型使用多尺度 autoencoder + discrete logistic mixture，绕开逐 byte 自回归。

指标：

- 比 TRACE 压缩速度快 **64×**；
- 平均 size reduction 只损失约 **5%**；
- 比 Gzip 多 **23% size reduction**；
- 比 7z 多 **16% size reduction**。

它的关键启发是：对结构化 byte stream，不应直接逐 byte 建模，而应先恢复 record / field layout，再做块级条件概率建模。

### 5.2 MSDZip：多源动态 byte compressor

`MSDZip` 仍以 byte 为符号，用固定历史窗口预测下一 byte，但通过 Local-Global-Deep Mixing 解决特征建模问题，通过 Stepwise-parallel 解决多 GPU 并行时 cold-start 恶化问题。

指标：

- 压缩率相对 advanced LLC 改善 **3.418%–69.874%**；
- 吞吐提升 **31.171%–495.649%**；
- 总体吞吐约 **15,469 bytes/s**。

### 5.3 EDPC：概率模型与编码流水线解耦

`EDPC` 面向 online autoregressive byte compression。其模型层使用 Multi-path Byte Refinement Block 和 Latent Transformation Engine，系统层使用 Decoupled Pipeline Compression Architecture。

指标：

- 压缩吞吐 **170.23 KB/s**；
- 比 PAC 快 **2.7×**；
- 比 TRACE 快 **>7×**；
- 压缩率比先进 baseline 提升 **3.2%**。

### 5.4 FADE：local/global 双流 + 并行解压

`FADE` 进一步把上下文拆成 micro-syntactic 和 macro-semantic 两类。模型层使用：

- Dual-Stream Multi-Scale Decoupler；
- Hierarchical Gated Refiner；
- Content-Adaptive Router；
- Concurrent Stream-Parallel Pipeline。

指标：

- Silesia 上压缩吞吐 **4571 KB/min**；
- 解压吞吐 **4144 KB/min**；
- 总吞吐 **4347 KB/min**；
- 相比 EDPC，解压吞吐高 **45.1%**。

FADE 的关键贡献是把并行化同时推到压缩和解压两端，不只是让编码端 GPU/CPU pipeline 更快。

---

## 6. 非文本 bytes：序列化是核心

### 6.1 LM-GC：float gradient -> hex text

`LM-GC` 面向 32-bit 浮点梯度压缩。它不直接把浮点数十进制文本喂给 LLM，而是：

```text
float32 bits -> 4-bit nibble -> hex symbol -> grouped text with separators -> LLM -> arithmetic coding
```

关键经验：separator 和 grouping 很重要。若直接 ISO byte 映射或不加分隔，LLM 概率质量会显著下降。

指标：相比传统/已有无损方法压缩率改善约 **10%–17.2%**。但吞吐很低，论文提到压缩 28MB 约需 4 小时量级。

### 6.2 Trilobyte：full-fidelity audio -> byte tokens

`Trilobyte` 解决高 bit-depth 音频 sample-level tokenization 词表爆炸问题。16-bit sample 词表是 65,536，24-bit 是 16,777,216，不可 tractable；Trilobyte 把每个 sample 拆成 byte token，使词表固定为 256。

指标：

- 8-bit：比 FLAC 平均好 **217%**；
- 16-bit：比 FLAC 平均好 **18%**；
- 24-bit：约 **1.48×**，低于 FLAC **1.63×**。

核心结论：非文本 byte 数据进入 LM compression 前，必须设计能暴露结构的 serialization，而不是粗暴把 bytes 当字符。

---

## 7. 横向指标对比

### 7.1 压缩率代表结果

| 论文 / 方法 | 数据 | 指标 |
|---|---|---:|
| Chinchilla 70B + AC (`Language Modeling Is Compression`) | enwik9 | **0.664 bpb** |
| `LLMZip` | text8 1MB | **0.7101 bpc** |
| `Nacrith` | enwik8 | **0.9389 bpb** |
| `Nacrith` | OOD gov text | **0.723 bpb** |
| `FineZip` | enwik8 | 约 **1.024 bpb**（Nacrith 引用） |
| `StateSMix` | enwik8 100MB | **2.130 bpb** |
| `L3TC` | enwik9 | 约 **16.23% CR** |
| `BOA` | CMS HEP data | **44.14×** |
| `Pre-trained byte Transformer` | OOD audio | compression ratio **0.49** |
| `LLM-generated text compression` | synthetic text | 超过 **20×** |

### 7.2 速度代表结果

| 论文 / 方法 | 速度 |
|---|---:|
| `BOA` | 摘要版报告压缩 **3.5–45 MB/s**，解压 **1.5–25 MB/s** |
| `L3TC-200K` | A100 解码 **4.35 MB/s**，iPhone12 ANE **1.30 MB/s** |
| `StateSMix` | 约 **2,000 tok/s**，4 CPU cores AVX2 |
| `EDPC` | **170.23 KB/s** |
| `FADE` | 总吞吐 **4347 KB/min ≈ 72.5 KB/s** |
| `Chained Predictors` | 编码 **43–244 KB/s**，解码 **98–429 KB/s** |
| `FineZip` | 10MB 约 **4 小时** |
| `LLMZip` | 10MB 可到 **9.5 天**量级 |
| `Pre-trained byte Transformer` | 1GB text 约 **452,355 秒** |

---

## 8. 对金融行情 bytes / Parquet / DataFrame / PCAP 的启示

虽然本综述的主体是 bytes/text，但它对金融行情很有启发。

### 8.1 不要直接压 raw PCAP，先做结构恢复

`ByteZip` 和 `BOA` 都说明，结构化 byte stream 的压缩潜力来自 schema/field/record 规律。行情数据中应优先解析：

```text
packet header -> protocol header -> message type -> sequence -> symbol -> price/size/order fields
```

再按字段或 message type 重排，而不是把整段 PCAP 当无结构 bytes。

### 8.2 对 Parquet/DataFrame，列间相关比大模型更重要

对行情表：bid/ask/mid/spread/level/timestamp/sequence/cum_volume 之间有强相关性。`Virtual`、`L3 GPU-native`、`DeepMapping` 这类表格/列式思想比 LLMZip/Nacrith 更适合。

### 8.3 对高频流，Mamba/RWKV 比大 LLM 更现实

行情压缩更看重吞吐和确定性。`BOA`、`L3TC`、`StateSMix`、`SerLiC` 的启发是：

- 用 SSM/RWKV 这类线性时间模型；
- 把价格/时间/序号变成 delta 或 tick；
- 按 symbol/channel/message type 建独立流；
- 用 range/rANS 做最终 entropy coding。

---

## 9. 开放问题

### 9.1 模型大小是否计入压缩文件

大 LLM raw bpb 很强，但若模型不能预共享，压缩率不成立。`Language Modeling Is Compression` 的 adjusted compression rate 是必须保留的评价维度。

### 9.2 浮点确定性

无损压缩要求解码端概率完全一致。`Hybrid-LLM` 和 `BOA` 都指出：GPU 浮点非确定性会导致 arithmetic/range decoder 崩溃。未来需要 fixed-point / quantized logits / deterministic kernels。

### 9.3 熵编码吞吐

很多 learned compressor 的瓶颈已不是模型，而是 CPU/GPU 间切换、arithmetic coding 串行性、CDF 构造。`EDPC`、`FADE`、`RAS` 这类系统工作会越来越重要。

### 9.4 tokenization 与 CDF 精度

BPE 减少 token 数，但大词表带来 CDF floor overhead。`Nacrith` 的 CDF-24 说明：编码层精度不是细节，而可能决定 0.5 bpb 级别的差距。

### 9.5 压缩率和速度不可兼得

当前最强文本压缩往往是 B/s 到 KB/s；最快 structured byte neural compressor 则依赖领域结构。未来真正可用的系统必须分数据类型选择模型，而不是追求一个 universal neural compressor。

---

## 10. 逐篇定位速查

| 论文 | 主要贡献层 |
|---|---|
| `LLMZip` | LLM PMF + arithmetic coding 基线 |
| `Language Modeling Is Compression` | 理论框架、模型大小计入、跨模态 raw compression |
| `AlphaZip` | rank 序列 + traditional compressor |
| `FineZip` | dynamic context + LoRA memorization + batching |
| `wPoE` | 测试时概率混合 / OOD 适配 |
| `LLM-generated text compression` | LLM 生成文本作为特殊高可压分布 |
| `Llamazip` | 预测失败事件编码 / 训练集检测 |
| `Nacrith` | 小 LLM ensemble + CDF-24 + 文件格式 + binary fallback |
| `Hybrid-LLM` | 路由 + logit quantization + archival system |
| `Haiku to Opus` | domain LoRA lossless + interactive lossy extension |
| `Comparing LLM text compression` | 多语言负结果 / 实用性边界 |
| `Context-Aware Frequency Adaptation` | 局部频率表 bias logits |
| `Compression via Pre-trained Transformers` | raw byte small Transformer + adjusted compression ratio |
| `Training LLMs over compressed text` | Equal-Info Windows / 可学习 compressed tokenization |
| `Seq2Seq2Seq` | RL 学习离散 latent token 压缩表示 |
| `L3TC` | RWKV 文本压缩 / 端侧 MB/s 解码 |
| `StateSMix` | 在线 Mamba + sparse n-gram / 自包含文件 |
| `BOA` | Mamba next-byte predictor / 结构化科学 bytes |
| `Chained Predictors` | Markov-order predictor chain + information inheritance |
| `ByteZip` | structured byte stream 的非自回归 AE 压缩 |
| `MSDZip` | 多源 byte compressor + stepwise parallel |
| `EDPC` | 轻量概率模型 + GPU/CPU 解耦 pipeline |
| `FADE` | local/global 双流 + 压缩/解压全流程并行 |
| `LM-GC` | float gradient -> hex grouped text -> LLM prior |
| `Trilobyte` | full-fidelity audio sample -> fixed 256 byte vocabulary |

## 结论

如果把所有 bytes/text neural lossless compression 放进同一框架，最重要的分化不是“用了什么模型”，而是：

1. **输入如何符号化**：byte、BPE、rank、error event、record、hex、sample bytes。
2. **上下文如何构造**：滑窗、chunk、dynamic context、equal-info window、online update、多流并行。
3. **概率模型如何产生分布**：LLM、small Transformer、RWKV、Mamba、MLP/CNN predictor、autoencoder。
4. **概率如何适配当前文件**：LoRA、wPoE、frequency table、n-gram、adaptive bias、logit quantization。
5. **编码层如何消费概率**：arithmetic/range/rANS、rank + secondary compressor、error-event、learned latent。
6. **解码端如何同步**：共享模型、保存 adapter、在线同步训练、自包含 header、确定性量化。
7. **系统如何跑得动**：batching、KV cache、llama.cpp、OpenMP/AVX2、multi-GPU、CPU/GPU pipeline。

目前的研究状态可以概括为：

> 大 LLM 给出了极强的文本压缩上界，小模型/SSM/RWKV 试图把它变得可部署，structured byte stream 方向则表明只要数据结构足够强，领域模型可以同时获得高压缩率和可接受速度。真正的下一步不是单纯换更大模型，而是把 **数据结构化、概率适配、熵编码精度、解码确定性和系统吞吐** 放在同一个设计闭环里。 


# Neural compression candidate bibliography (2023-present)

## File note

Saved in `neural_compression_2023_present/` because that directory already contains the prior archive note (`research_report.md`), the URL ledger, and downloaded first-party PDFs. This file is a candidate bibliography, not a final inclusion list.

Companion archive/download ledger: `neural_compression_2023_present/download_urls.csv`.

## Scope

Primary-source only archive candidate list for 2023-present neural compression papers that are explicitly lossless or rely on entropy/probabilistic coding. Bytes/text and Transformer/SSM/advanced linear models are prioritized; other data types are included only when lossless is explicit.

## Conventions

- `arXiv URL` points to the abstract page when available.
- `PDF URL` points to a downloadable first-party PDF or publisher PDF page when available.
- `Lossless explicit?` means the paper title/venue/page explicitly frames the method as lossless, or the first-party source explicitly describes lossless coding / entropy coding / arithmetic coding / range coding / rANS.
- The `Evidence` cell intentionally contains both the 1-line inclusion rationale and the primary-source URL (`Source: ...`), so the bibliography stays self-contained while the CSV keeps archive-oriented fields.

| Title | Year | Authors | Venue/source | ArXiv URL | PDF URL | Data type | Model family | Lossless explicit? | Evidence |
|---|---:|---|---|---|---|---|---|---|---|
| LLMZip: Lossless Text Compression using Large Language Models | 2023 | Chandra Shekhara Kaushik Valmeekam; Krishna Narayanan; Dileep Kalathil; Jean-Francois Chamberland; Srinivas Shakkottai | arXiv / OpenReview | https://arxiv.org/abs/2306.04050 | https://arxiv.org/pdf/2306.04050 | text | LLaMA / Transformer LLM | Yes | Core 2023 LLM text-compression baseline. Source: https://arxiv.org/abs/2306.04050. |
| Language Modeling Is Compression | 2024 | Gregoire Deletang; Anian Ruoss; Paul-Ambroise Duquenne; Elliot Catt; Tim Genewein; Christopher Mattern; Jordi Grau-Moya; Li Kevin Wenliang; Matthew Aitchison; Laurent Orseau; Marcus Hutter; Joel Veness | ICLR 2024 / arXiv | https://arxiv.org/abs/2309.10668 | https://arxiv.org/pdf/2309.10668 | text; image; audio | Transformer foundation models; small byte-level Transformers | Yes | Foundational 2023/2024 paper connecting LM log-loss and lossless compression. Source: https://arxiv.org/abs/2309.10668. |
| DeepMapping: Learned Data Mapping for Lossless Compression and Efficient Lookup | 2024 | Lixi Zhou; K. Selcuk Candan; Jia Zou | ICDE 2024 / arXiv | https://arxiv.org/abs/2307.05861 | https://arxiv.org/pdf/2307.05861 | tabular / key-value maps | multi-task neural networks | Yes | Included as database/tabular lossless neural compression, not bytes/text. Source: https://arxiv.org/abs/2307.05861. |
| Training LLMs over Neurally Compressed Text | 2024 | unknown from search snippet | arXiv | https://arxiv.org/abs/2404.03626 | https://arxiv.org/pdf/2404.03626 | text | LLMs over neural compressed representations | Yes | Related to text neural compression representations; not primarily a storage compressor. Source: https://arxiv.org/abs/2404.03626. |
| Lossless data compression by large models | 2024 | Ziguang Li; Chao Huang; Xuliang Wang; Haibo Hu; Cole Wyeth; Dongbo Bu; others | arXiv | https://arxiv.org/abs/2407.07723 | https://arxiv.org/pdf/2407.07723 | text; image; audio; video | large generative models / LLMs | Yes | LMCompress; broad multimodal lossless compression claims. Source: https://arxiv.org/abs/2407.07723. |
| Language Models as Zero-shot Lossless Gradient Compressors: Towards General Neural Parameter Prior Models | 2024 | Hui-Po Wang; Mario Fritz | arXiv | https://arxiv.org/abs/2409.17836 | https://arxiv.org/pdf/2409.17836 | neural gradients / floating point tensors | LLM prior | Yes | Important example of byte/float serialization with LLM prior. Source: https://arxiv.org/abs/2409.17836. |
| AlphaZip: Neural Network-Enhanced Lossless Text Compression | 2024 | Swathi Shree Narashiman; Nitin Chandrachoodan | arXiv | https://arxiv.org/abs/2409.15046 | https://arxiv.org/pdf/2409.15046 | text | GPT-style Transformer / LLM | Yes | Rank-based LLM text compression. Source: https://arxiv.org/abs/2409.15046. |
| FineZip: Pushing the Limits of Large Language Models for Practical Lossless Text Compression | 2024 | Fazal Mittu; Yihuan Bu; Akshat Gupta; Ashok Devireddy; Alp Eren Ozdarendeli; Anant Singh; others | arXiv | https://arxiv.org/abs/2409.17141 | https://arxiv.org/pdf/2409.17141 | text | LLaMA / Transformer LLM with online memorization | Yes | Practicality-oriented successor to LLMZip. Source: https://arxiv.org/abs/2409.17141. |
| Compression via Pre-trained Transformers: A Study on Byte-Level Multimodal Data | 2025 | unknown from search snippet | arXiv | https://arxiv.org/abs/2410.05078 | https://arxiv.org/pdf/2410.05078 | byte-level text; image; audio | small decoder-only Transformers | Likely/implicit | Directly relevant to bytes and multimodal small Transformer compressors. Source: https://arxiv.org/abs/2410.05078. |
| Lightweight Correlation-Aware Table Compression | 2024 | Mihail Stoian; Alexander van Renen; Jan Kobiolka; Ping-Lin Kuo; Josif Grabocka; Andreas Kipf | arXiv / TRL 2024 | https://arxiv.org/abs/2410.14066 | https://arxiv.org/pdf/2410.14066 | tabular | sparse learned/correlation functions; optional tiny neural functions | Yes | Included as lossless learned/correlation-aware table compression; not neural-only. Source: https://arxiv.org/abs/2410.14066. |
| Large Language Models for Lossless Image Compression: Next-Pixel Prediction in Language Space is All You Need | 2025 | Kecheng Chen; Pingping Zhang; Hui Liu; Jie Liu; Yibing Liu; Jiaxin Huang; Shiqi Wang; Hong Yan; Haoliang Li | NeurIPS 2025 / arXiv / OpenReview | https://arxiv.org/abs/2411.12448 | https://arxiv.org/pdf/2411.12448 | image | LLM / Transformer | Yes | Lossless image compression using LLM tokenization of pixels. Source: https://arxiv.org/abs/2411.12448. |
| L3TC: Leveraging RWKV for Learned Lossless Low-Complexity Text Compression | 2025 | Junxuan Zhang; Zhi-Lin Cheng; Yan Zhao; Shihao Wang; Dajiang Zhou; Guoxing Lu; others | AAAI 2025 / arXiv | https://arxiv.org/abs/2412.16642 | https://arxiv.org/pdf/2412.16642 | text | RWKV; Transformer and Transformer-XL comparisons | Yes | Key advanced linear/recurrent model paper; RWKV backbone. Source: https://arxiv.org/abs/2412.16642. |
| Efficient and Generic Point Model for Lossless Point Cloud Attribute Compression | 2024 | Kang You; Pan Gao; Zhan Ma | arXiv | https://arxiv.org/abs/2404.06936 | https://arxiv.org/pdf/2404.06936 | point cloud attributes | locality-aware attention / point model | Yes | PoLoPCAC; lossless PC attribute compression. Source: https://arxiv.org/abs/2404.06936. |
| Test-Time Steering for Lossless Text Compression via Weighted Product of Experts | 2025 | Qihang Zhang; Muchen Li; Ziao Wang; Renjie Liao; Lele Wang | Findings of EMNLP 2025 / arXiv | https://aclanthology.org/2025.findings-emnlp.110/ | https://aclanthology.org/2025.findings-emnlp.110.pdf | text | autoregressive LMs + Naive Bayes expert; byte-level Transformers; GPT-2/LLaMA | Yes | Important 2025 text compression robustness/adaptation paper. Source: https://aclanthology.org/2025.findings-emnlp.110/. |
| Lossless Compression of Large Language Model-Generated Text via Next-Token Prediction | 2025 | Yu Mao; Holger Pirk; Chun Jason Xue | arXiv | https://arxiv.org/abs/2505.06297 | https://arxiv.org/pdf/2505.06297 | LLM-generated text | LLMs / Transformer next-token predictors | Yes | Targets synthetic/LLM-generated text workloads. Source: https://arxiv.org/abs/2505.06297. |
| Llamazip: Leveraging LLaMA for Lossless Text Compression and Training Dataset Detection | 2025 | Soren Dreano; Derek Molloy; Noel Murphy | arXiv | https://arxiv.org/abs/2511.17589 | https://arxiv.org/pdf/2511.17589 | text | LLaMA3 / Transformer | Yes | Also uses compression as training-data detection signal. Source: https://arxiv.org/abs/2511.17589. |
| MSDZip: Universal Lossless Compression for Multi-source Data via Stepwise-parallel and Learning-based Prediction | 2025 | Huidong Ma; Hui Sun; Liping Yi; Yanfeng Ding; Xiaoguang Liu; Gang Wang | WWW 2025 / OpenReview / ACM | https://openreview.net/forum?id=BgSrNpCMqO | https://openreview.net/pdf?id=BgSrNpCMqO | multi-source data: text; image; audio; DNA; others | learning-based predictor; local-global-deep mixing | Yes | OpenReview PDF returned 403/HTML in this environment; metadata and DOI/code pages available. Source: https://openreview.net/forum?id=BgSrNpCMqO. |
| Genomics Data Lossless Compression with (S,K)-Mer Encoding and Deep Neural Networks | 2025 | Hui Sun; Liping Yi; Huidong Ma; Yongxia Sun; Yingfeng Zheng; Wenwen Cui; Meng Yan; Gang Wang; Xiaoguang Liu | AAAI 2025 | https://ojs.aaai.org/index.php/AAAI/article/view/33371 | https://ojs.aaai.org/index.php/AAAI/article/download/33371/35526 | genomics | Transformer + BiGRU | Yes | DeepGeCo; genomics lossless compression. Source: https://ojs.aaai.org/index.php/AAAI/article/view/33371. |
| PMKLC: Parallel Multi-Knowledge Learning-based Lossless Compression for Large-Scale Genomics Database | 2025 | Hui Sun; Yanfeng Ding; Liping Yi; Huidong Ma; Gang Wang; Xiaoguang Liu; Cheng Zhong; Wentong Cai | KDD 2025 / arXiv | https://arxiv.org/abs/2507.12805 | https://arxiv.org/pdf/2507.12805 | genomics | BiLSTM static models + Transformer-like dynamic model; multi-knowledge learning | Yes | Large-scale genomics compressor. Source: https://arxiv.org/abs/2507.12805. |
| EDPC: Accelerating Lossless Compression via Lightweight Probability Models and Decoupled Parallel Dataflow | 2025 | Zhonghui Lu; Xi Ma; Yujun Huang; Minxiao Chen; Bin Chen; Baoyi An; others | arXiv | https://arxiv.org/abs/2507.18969 | https://arxiv.org/pdf/2507.18969 | multi-source multimedia data | lightweight probability models; dual-path byte refinement | Yes | System/model optimization for neural lossless compression. Source: https://arxiv.org/abs/2507.18969. |
| Enhancing and Evaluating Probabilistic Circuits for High-Resolution Lossless Image Compression | 2025 | Daniel Severo; Jingtong Su; Anji Liu; Jeff Johnson; Brian Karrer; Guy Van den Broeck; Matthew Muckley; Karen Ullrich | DCC 2025 | https://starai.cs.ucla.edu/publications/browse.php?key=SeveroDCC25 | https://starai.cs.ucla.edu/papers/SeveroDCC25.pdf | image | probabilistic circuits | Yes | High-resolution lossless image study; notes diminishing gains. Source: https://starai.cs.ucla.edu/publications/browse.php?key=SeveroDCC25. |
| AstroCompress: A benchmark dataset for multi-purpose compression of astrophysics data | 2025 | unknown from search snippet | arXiv | https://arxiv.org/abs/2506.08306 | https://arxiv.org/pdf/2506.08306 | astronomical / scientific imaging | IDF; L3C; PixelCNN++; VDM benchmarked | Yes | Benchmark/challenge rather than a single new architecture. Source: https://arxiv.org/abs/2506.08306. |
| Efficient LiDAR Reflectance Compression via Scanning Serialization | 2025 | Jiahao Zhu; Kang You; Dandan Ding; Zhan Ma | arXiv | https://arxiv.org/abs/2505.09433 | https://arxiv.org/pdf/2505.09433 | LiDAR point cloud reflectance | Mamba / SSM | Yes | SerLiC; key SSM/Mamba lossless point-cloud attribute paper. Source: https://arxiv.org/abs/2505.09433. |
| Hierarchical Attention Networks for Lossless Point Cloud Attribute Compression | 2025 | Yueru Chen; Wei Zhang; Dingquan Li; Jing Wang; Ge Li | arXiv / DCC 2025 poster listing | https://arxiv.org/abs/2504.00481 | https://arxiv.org/pdf/2504.00481 | point cloud attributes | hierarchical attention / Transformer-like context model | Yes | Lossless point-cloud attribute compression. Source: https://arxiv.org/abs/2504.00481. |
| LINR-PCGC: Lossless Implicit Neural Representations for Point Cloud Geometry Compression | 2025 | Wenjie Huang; Qi Yang; Shuting Xia; He Huang; Zhu Li; Yiling Xu | arXiv | https://arxiv.org/abs/2507.15686 | https://arxiv.org/pdf/2507.15686 | point cloud geometry | implicit neural representation; SparseConv coding network | Yes | First INR-based lossless PC geometry claim from search result. Source: https://arxiv.org/abs/2507.15686. |
| UniPCGC: Towards Practical Point Cloud Geometry Compression via an Efficient Unified Approach | 2025 | Kangli Wang; Wei Gao | arXiv | https://arxiv.org/abs/2503.18541 | https://arxiv.org/pdf/2503.18541 | point cloud geometry | sparse convolution / learned geometry coding | Yes | Supports lossless and lossy modes; included for lossless mode. Source: https://arxiv.org/abs/2503.18541. |
| BOA Constrictor: A Mamba-based lossless compressor for High Energy Physics data / scientific data | 2026 | Akshat Gupta; Caterina Doglioni; Thomas Joseph Elliott | arXiv 2025; MLST 2026; IOP | https://arxiv.org/abs/2511.11337 | https://arxiv.org/pdf/2511.11337 | scientific byte streams / HEP data | Mamba SSM | Yes | Key Mamba byte-level lossless scientific-data compressor. Source: https://arxiv.org/abs/2511.11337. |
| AgentGC: Evolutionary Learning-based Lossless Compression for Genomics Data with LLM-driven Multiple Agent | 2026 | Hui Sun; Yanfeng Ding; Huidong Ma; Chang Xu; Keyan Jin; Lizheng Zu; others | arXiv / ACL Findings 2026 listing | https://arxiv.org/abs/2601.13559 | https://arxiv.org/pdf/2601.13559 | genomics | LLM-driven multi-agent controller + learning-based compressor | Yes | Agentic orchestration around genomics compressor. Source: https://arxiv.org/abs/2601.13559. |
| Seq2Seq2Seq: Lossless Data Compression via Discrete Latent Transformers and Reinforcement Learning | 2026 | Mahdi Khodabandeh; Ghazal Shabani; Arash Yousefi Jordehi; Seyed Abolghasem Mirroshandel | arXiv | https://arxiv.org/abs/2602.12146 | https://arxiv.org/pdf/2602.12146 | general/text benchmark | T5 seq2seq Transformer + reinforcement learning | Yes | Claims lossless; weaker than NNCP in reported enwik8 but relevant Transformer/RL direction. Source: https://arxiv.org/abs/2602.12146. |
| Nacrith: Neural Lossless Compression via Ensemble Context Modeling and High-Precision CDF Coding | 2026 | Roberto Tacconelli | arXiv | https://arxiv.org/abs/2602.19626 | https://arxiv.org/pdf/2602.19626 | text; arbitrary binary hybrid mode | SmolLM2 Transformer + n-gram ensemble | Yes | Highly relevant 2026 practical text/binary neural compressor. Source: https://arxiv.org/abs/2602.19626. |
| Benchmarking Language Modeling for Lossless Compression of Full-Fidelity Audio | 2026 | P. E. Long; Zachary Novack; Chris Donahue | arXiv | https://arxiv.org/abs/2603.08683 | https://arxiv.org/pdf/2603.08683 | audio bytes / waveforms | decoder-only Transformers | Yes | Byte-level tokenization for 16/24-bit audio; relevant beyond text. Source: https://arxiv.org/abs/2603.08683. |
| Investigating the Fundamental Limit: A Feasibility Study of Hybrid-Neural Archival | 2026 | Marcus Armstrong; ZiWei Qiu; Huy Q. Vo; Arjun Mukherjee | arXiv | https://arxiv.org/abs/2603.25526 | https://arxiv.org/pdf/2603.25526 | text / archival data | LLaMA3 / hybrid LLM + classical router | Likely/implicit | Focuses determinism and cold archival feasibility. Source: https://arxiv.org/abs/2603.25526. |
| Haiku to Opus in Just 10 bits: LLMs Unlock Large Compression Gains | 2026 | Roy Rinberg; Annabelle Michael Carrell; Simon Henniger; Nicholas Carlini; Keri Warr | arXiv | https://arxiv.org/abs/2604.02343 | https://arxiv.org/pdf/2604.02343 | LLM-generated text | LLM + LoRA adapters; interactive protocols | Yes | Contains both lossless and lossy; CSV inclusion is for lossless LoRA arithmetic-coding part only. Source: https://arxiv.org/abs/2604.02343. |
| Lossless Compression via Chained Lightweight Neural Predictors with Information Inheritance | 2026 | Yuriy Kim; Evgeny Belyaev | arXiv | https://arxiv.org/abs/2604.15472 | https://arxiv.org/pdf/2604.15472 | general multi-domain data | chained lightweight neural predictors / MLPs | Yes | Lightweight neural probability-estimation architecture. Source: https://arxiv.org/abs/2604.15472. |
| StateSMix: Online Lossless Compression via Mamba State Space Models and Sparse N-gram Context Mixing | 2026 | Roberto Tacconelli | arXiv | https://arxiv.org/abs/2605.02904 | https://arxiv.org/pdf/2605.02904 | text / bytes | online Mamba-style SSM + sparse n-grams | Yes | Key SSM/Mamba online compressor. Source: https://arxiv.org/abs/2605.02904. |
| PACE: Post-Causal Entropy Modeling for Learned LiDAR Point Cloud Compression | 2026 | unknown from search snippet | arXiv | https://arxiv.org/abs/2605.01320 | https://arxiv.org/pdf/2605.01320 | LiDAR point cloud geometry | learned entropy model; non-causal backbone + causal predictor | Yes | Point-cloud lossless geometry / occupancy coding. Source: https://arxiv.org/abs/2605.01320. |
| Efficient Learned Data Compression via Dual-Stream Feature Decoupling | 2026 | Huidong Ma; Xinyan Shi; Hui Sun; Xiaofei Yue; Xiaoguang Liu; Gang Wang; Wentong Cai | ACL 2026 | https://aclanthology.org/2026.acl-long.324/ | https://aclanthology.org/2026.acl-long.324.pdf | multi-source data including text/scientific sequences | dual-stream CNN/MLP learned compression model | Likely/implicit | FADE; efficient learned data compression. Source: https://aclanthology.org/2026.acl-long.324/. |
| RAS: A Bit-Exact rANS Accelerator For High-Performance Neural Lossless Compression | 2025 | Yuchao Qin; Anjunyi Fan; Bonan Yan | arXiv | https://arxiv.org/abs/2511.04684 | https://arxiv.org/pdf/2511.04684 | neural lossless coding hardware / image workloads | rANS accelerator paired with neural probability models | Yes | System accelerator for neural lossless compression rather than a probability model paper. Source: https://arxiv.org/abs/2511.04684. |
| Learning Lossless Compression for High Bit-Depth Medical Imaging | 2023 | Kai Wang; Yuanchao Bai; Deming Zhai; Daxin Li; Junjun Jiang; Xianming Liu | ICME 2023 / IEEE | https://doi.org/10.1109/ICME55011.2023.00434 | https://doi.org/10.1109/ICME55011.2023.00434 | medical images / high bit-depth | autoregressive entropy model conditioned on MSB image | Yes | Only DOI/IEEE page found; public PDF not located during this run. Source: https://doi.org/10.1109/ICME55011.2023.00434. |
| Rethinking Learning-Based Method for Lossless Genome Compression | 2023 | Han Yang; Fei Gu; Jieping Ye | ICASSP 2023 / IEEE | https://doi.org/10.1109/ICASSP49357.2023.10096124 | https://doi.org/10.1109/ICASSP49357.2023.10096124 | genomics | CompressBERT / position-driven Transformer | Yes | Only DOI/IEEE page found; no public PDF located. Source: https://doi.org/10.1109/ICASSP49357.2023.10096124. |
| GenCoder: A Novel Convolutional Neural Network Based Autoencoder for Genomic Sequence Data Compression | 2024 | K. S. Sheena; Madhu S. Nair | IEEE/ACM TCBB 2024 | https://doi.org/10.1109/TCBB.2024.3366240 | https://doi.org/10.1109/TCBB.2024.3366240 | genomics | convolutional autoencoder | Yes | Only DOI/IEEE page found; no public PDF located. Source: https://doi.org/10.1109/TCBB.2024.3366240. |
| ByteZip: Efficient Lossless Compression for Structured Byte Streams Using DNNs | 2024 | Parvathy Ramakrishnan P; Satyajit Das | IJCNN 2024 / IEEE | https://doi.org/10.1109/IJCNN60899.2024.10650523 | https://doi.org/10.1109/IJCNN60899.2024.10650523 | structured byte streams | autoencoders + density mixture models | Yes | Only DOI/IEEE page found; no public PDF located. Source: https://doi.org/10.1109/IJCNN60899.2024.10650523. |
| Adaptive Lossless Compression for Genomics Data by Multiple (s,k)-mer Encoding and XLSTM | 2025 | Hui Sun; Yanfeng Ding; Liping Yi; Huidong Ma; Haonan Xie; Gang Wang; others | ICASSP 2025 / IEEE | https://doi.org/10.1109/ICASSP49660.2025.10887721 | https://doi.org/10.1109/ICASSP49660.2025.10887721 | genomics | xLSTM | Yes | AGDLC; only DOI/IEEE page found. Source: https://doi.org/10.1109/ICASSP49660.2025.10887721. |
| Multi-source Data Lossless Compression via Parallel Expansion Mapping and xLSTM | 2025 | Huidong Ma; Hui Sun; Liping Yi; Xiaoguang Liu; Gang Wang | ICASSP 2025 / IEEE | https://doi.org/10.1109/ICASSP49660.2025.10889184 | https://doi.org/10.1109/ICASSP49660.2025.10889184 | multi-source data | xLSTM + Deep Spatial Gating Module | Yes | MSDLC; DOI/IEEE page found; code repo available. Source: https://doi.org/10.1109/ICASSP49660.2025.10889184. |
| Enhancing LLM-Based Text Compression with Context-Aware Frequency Adaptation | 2025 | Rishika Kinger; R. Murali; P. Ramya Krishna; Meghana Goru; Sujatha R. Upadhyaya | ECAI 2025 / IEEE | https://doi.org/10.1109/ECAI65401.2025.11095536 | https://doi.org/10.1109/ECAI65401.2025.11095536 | text | LLaMA-3.2-1B / Transformer LLM | Yes | Only DOI/IEEE page found; include because directly text/LLM lossless. Source: https://doi.org/10.1109/ECAI65401.2025.11095536. |
| DICOMP: Deep Reinforcement Learning for Integer Compression | 2025 | unknown from search snippet | ScienceDirect 2025 | https://www.sciencedirect.com/science/article/pii/S2666827025001392 | https://www.sciencedirect.com/science/article/pii/S2666827025001392 | integers | deep reinforcement learning | Yes | Publisher page only; included as neural/DRL lossless integer compression. Source: https://www.sciencedirect.com/science/article/pii/S2666827025001392. |
| L3: A GPU-Native Co-Designed Data Format for Learned Lossless Lightweight Compression | 2026 | Youyang Xia; Feng Zhang; Junda Pan; Yihao Liu; Jiawei Guan; Hailong Zhang; others | ACM 2026 | https://doi.org/10.1145/3802078 | https://doi.org/10.1145/3802078 | GPU analytics / correlated numeric data | learned lightweight predictor / GPU-native format | Yes | Publisher DOI only; included for learned lossless lightweight compression systems. Source: https://doi.org/10.1145/3802078. |

## Coverage note

This bibliography intentionally keeps borderline items where the primary source and metadata point to lossless or entropy-coded neural compression, even if the model family is not Transformer/SSM-centric. It is meant as a local archive candidate list rather than a final ranked set.
