# 2022 年以来神经网络与大模型驱动数据压缩研究综述

## Executive Summary

- 从 **2022 年到 2026 年上半年**，真正把 **神经网络 / Transformer / LLM / SSM** 当作**压缩器本体**或**概率模型**来做数据压缩的论文，已经形成了一条比较清晰的技术链：早期是面向一般 byte stream 的轻量 Transformer 在线压缩器（如 TRACE），随后出现 DNA / 文本上的 Transformer 压缩器（GeneFormer、LLMZip），再到把 foundation model 直接当作通用压缩器来评估与优化（Language Modeling Is Compression、Compression via Pre-trained Transformers、LMCompress），最近则出现了更工程化的 LLM/SSM 压缩系统（FineZip、Nacrith、StateSMix、BOA）。citeturn20search0turn29search2turn10view0turn16view0turn18view0turn32view0turn21view0turn23view0turn24view0

- 在**无损文本/序列压缩**里，最稳健也最主流的路线不是“端到端 latent codec”，而是 **“神经概率模型 + arithmetic/range coding”**：TRACE 用 Transformer 预测字节分布后交给 arithmetic coder；GeneFormer 用 Transformer + dynamic arithmetic coding；LLMZip、Nacrith、StateSMix、BOA 也都遵循“预测分布 → 熵编码”的范式。citeturn20search0turn29search6turn10view0turn21view0turn23view0turn24view0

- **LLM 的压缩率优势是真实存在的，但速度与系统开销仍是最大瓶颈。** LLMZip 报告了非常激进的 bits-per-character 结果，但实验规模有限，且作者自己明确指出 text8 可能与预训练语料重叠、不能在全 100MB 上完整运行。FineZip 通过 LoRA 式“在线记忆”与 dynamic context 大幅提速，但仍远未达到传统压缩器的实用吞吐。Nacrith 和 BOA / StateSMix 则把工程优化（llama.cpp、KV cache、跳过高置信 token、并行 range coder、AVX2/OpenMP）推到了前台。citeturn10view0turn12view0turn21view0turn23view0turn24view0

- **文本压缩的“结果可信度”差异很大。** ICLR/ICML/Nature/ICDE/WWW 等正式 venue 的代表作通常更适合当作高可信证据；而部分 2024–2026 arXiv 工作虽然结果亮眼，但实验集偏小、基准不标准、或仍缺少大规模独立复现，因此更适合作为“前沿信号”而不是稳定结论。citeturn17search3turn19search1turn32view0turn30search3turn20search4turn21view0turn23view0

- 在**科学数据 / 浮点数组 / HPC 数据**方向，2022 年以来的核心成果大多仍然是 **有损或近无损**，而且非常强调 **error-bound、PSNR、科学分析可用性、I/O 和吞吐**。代表性工作包括层次 autoencoder（HAE）、super-resolution 加强型 error-bounded 压缩（SRN-SZ）、在线学习增强传统科学压缩器（NeurLZ），以及用于时间变化体数据的 INR/KD-INR 路线。真正“神经网络做 lossless floating-point compression”的公开工作，在本次检索中依然很少见。citeturn36view1turn37view0turn39view0turn26search9turn4search7

- **结构化 / 表格数据压缩**不是这条文献线里最热的方向，但并非空白。DeepMapping 把表格视为多任务 key-value 映射，使用深度网络“记忆”映射并用轻量辅助结构纠错，目标同时兼顾**无损压缩**与**高效 lookup**；它说明数据库社区的“learned compression”与传统信息论压缩并不完全同构，但确实属于“直接压缩数据”的神经方法。citeturn37view1turn30search3turn30search9

- **SSM / Mamba 用于压缩**是一个很新的分支。到 2025–2026 才开始出现较明确的直接压缩器：BOA Constrictor（HEP 结构化科学数据）和 StateSMix（在线文本压缩）都把 Mamba/SSM 当作 next-symbol predictor，再与 range/arithmetic coding 结合。就本次纳入语料看，这条线还没有达到 Transformer/LLM 路线的成熟度，但它在**长序列、线性复杂度、在线训练**上的潜力非常明显。citeturn24view0turn23view0

- 很多被社群称为“compression”的 ACL / LLM 工作——例如 **LLMLingua、LongLLMLingua、context compression**——本质上是**推理成本压缩**或**任务语义压缩**，而不是本文关心的“对原始数据做 source coding / rate-distortion compression”。这些工作很重要，但如果没有**可逆恢复、bits-per-*、compression ratio、error bound**这类真正压缩指标，我不把它们纳入主语料。citeturn8search0turn8search4turn8search1

## 检索范围与筛选口径

本综述以 **2022-01-01 到 2026-06-03** 为时间窗口，优先使用**原始论文页、官方会议页、期刊页、作者公开代码页**作为证据，而不是只依赖二手摘要。最终纳入的主语料主要来自 arXiv、ICLR 2024 proceedings、ICML 2025 PMLR、WWW 2022、ICDE 2024、Nature Machine Intelligence，以及部分 IEEE/ACM/领域官方页面。citeturn17search3turn19search1turn20search4turn30search3turn32view0turn25search7

我采用了四条硬筛选规则。第一，论文必须把**数据本身**作为压缩对象，而不是只压缩模型权重。第二，论文需要提供明确的压缩指标，例如 **bits-per-character、bits-per-byte、compression ratio、PSNR、error bound、rate-distortion** 等。第三，神经网络 / Transformer / LLM / SSM 必须是**压缩模型本体**、**概率模型**或**核心增强模块**，而不是无关配角。第四，若论文只做 prompt/context shortening，但不涉及源编码意义上的压缩建模，我将其视为**相邻问题**，不纳入主表。citeturn10view0turn18view0turn36view1turn39view0turn8search0turn8search4

据此，我**排除**了大量“模型压缩”“LLM 权重量化/剪枝”“KV cache compression”“prompt compression survey”之类论文；这些确实与“压缩”有关，但压缩对象是模型或推理状态，而不是文本、字节流、浮点数组、表格或一般数据。LLMLingua、LongLLMLingua、工具使用场景中的 context compression 很适合放在“任务导向语义压缩”或“推理成本优化”综述里，但不属于本文的主任务定义。citeturn8search0turn8search4turn8search1turn8search2

本综述中的“可信度”是我对**可复现性与结果稳健性**的综合判断，而不是对论文正确性的裁决。大致标准是：**高**表示正式同行评审且基准清晰、最好有代码；**中**表示正式发表或较成熟预印本，但代码/实验规模/可复现性仍有限；**低**表示预印本、非标准基准较多，或关键结果主要停留在摘要级别。

## 论文总览表

下表列出我认为 **最相关、最值得纳入主综述** 的论文。为控制篇幅，表中“关键结果”只保留最具代表性的数字或结论；更细的实验设计与局限在后面的详细综述与深读部分展开。

| 年份 | 论文                                                         | 压缩对象                               | 类型                   | 核心模型                                                     | 关键结果                                                     | 代码                             | 重要性 | 可信度 |
| ---- | ------------------------------------------------------------ | -------------------------------------- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------- | ------ | ------ |
| 2022 | **TRACE: A Fast Transformer-based General-Purpose Lossless Compressor**（WWW 2022）citeturn20search0turn20search4 | 一般 byte stream                       | 无损                   | 单层 Transformer + arithmetic coding                         | 约 **3× 提速**，压缩率与 SOTA **可比**citeturn20search0   | GitHub citeturn20search1      | A      | 高     |
| 2022 | **GeneFormer: Learned Gene Compression using Transformer-based Context Modeling** citeturn29search2turn29search6 | 基因序列 / 符号序列                    | 无损                   | Transformer-XL 变体 + latent array + dynamic arithmetic coding | 相对前作**节省 29.7% bitrate**，并提升解码速度citeturn29search2turn29search6 | 未确认                           | B+     | 中     |
| 2023 | **LLMZip: Lossless Text Compression using Large Language Models** citeturn9view0turn10view0 | 文本 / token sequence                  | 无损                   | LLaMA-7B + SentencePiece + arithmetic coding / rank coding   | 在 1M text8 片段上报告 **0.7101 bpc**；但样本有限且作者提示可能有训练集重合偏乐观citeturn10view0 | 未确认                           | A      | 中     |
| 2023 | **Hierarchical Autoencoder-based Lossy Compression for Large-scale High-resolution Scientific Data** citeturn36view1 | 科学网格 / 气候数据                    | 有损                   | Hierarchical autoencoder                                     | 基准集 CR **140**；CESM 2D 气候模拟数据 CR **200**，重建误差“可忽略”citeturn36view1 | GitHub 实现 citeturn38search2 | A-     | 中高   |
| 2023 | **SRN-SZ: Scientific Error-bounded Lossy Compression with Super-resolution Neural Networks** citeturn37view0 | 科学浮点数据                           | 有损 / error-bounded   | Super-resolution network（HAT）+ 科学压缩框架                | 同误差界下 CR 最多提升 **75%**；同 PSNR 下最多提升 **80%**citeturn37view0 | 未确认                           | A      | 中高   |
| 2023 | **DeepMapping: Learned Data Mapping for Lossless Compression and Efficient Lookup**（ICDE 2024）citeturn37view1turn30search3 | 表格 / key-value 结构化数据            | 无损                   | 多任务 DNN + 辅助纠错结构                                    | 在 TPC-H / TPC-DS 等上更好平衡**压缩率、读取速度与内存占用**citeturn37view1 | GitHub citeturn30search9      | A-     | 高     |
| 2024 | **Language Modeling Is Compression**（ICLR 2024）citeturn16view0turn17search3 | 文本，也扩展到图像 patch / 音频 sample | 无损                   | Foundation LM / Transformer + arithmetic coding 视角         | Chinchilla 70B 将 ImageNet patch 压到 **43.4%**，LibriSpeech 压到 **16.4%**，优于 PNG/FLAC 基线citeturn16view0 | GitHub citeturn17search0      | S      | 高     |
| 2024 | **Deep Dict: Deep Learning-based Lossy Time Series Compressor for IoT Data**（ICC 2024）citeturn36view0 | IoT 时间序列                           | 有损                   | Bernoulli Transformer Autoencoder + distortion constraint    | 在 10 个数据集上，相对 SOTA **最高提升 53.66% CR**citeturn36view0 | 未确认                           | B+     | 中     |
| 2024 | **FineZip: Pushing the Limits of Large Language Models for Practical Lossless Text Compression** citeturn11view0turn12view0 | 文本                                   | 无损                   | Llama-3 8B + LoRA 在线记忆 + dynamic context                 | 压 10MB 文本约 **4 小时**，较 LLMZip **54× 提速**；rank-based 方案显著优于 gzip/bzip2/zlibciteturn12view0 | GitHub citeturn12view0        | A      | 中高   |
| 2024 | **AlphaZip: Neural Network-Enhanced Lossless Text Compression** citeturn13view0turn14view0turn15view1 | 文本                                   | 无损                   | GPT-2 / Transformer rank predictor + GZIP/Brotli/Huffman     | 在文学文本小实验里，raw GPT-2 压缩比约 **3.5**，对比 GZIP 约 **2**；蒸馏 GPT-2 + Brotli 在 Alice 上到 **4.65**citeturn15view1 | GitHub citeturn14view0        | B      | 低到中 |
| 2024 | **Training LLMs over Neurally Compressed Text**（TMLR）citeturn35view0 | 压缩文本 tokenization / 学习表示       | 邻接问题               | Equal-Info Windows + arithmetic-coded text                   | 不是“更强压缩器”，而是研究**如何直接在神经压缩文本上训练 LLM**；说明强压缩输出未必可学习citeturn35view0 | 未确认                           | B      | 中     |
| 2025 | **Compression via Pre-trained Transformers: A Study on Byte-Level Multimodal Data**（ICML 2025）citeturn18view0turn19search1turn19search5 | 原始 byte（文本/图像/音频）            | 无损                   | 小型 vanilla Transformer                                     | 百万级参数模型在计入模型体积后仍可超过 gzip/LZMA2，音频压缩率 **0.49**，优于 FLAC **0.54**citeturn18view0 | 未确认                           | S      | 高     |
| 2025 | **NeurLZ: An Online Neural Learning-Based Method to Enhance Scientific Lossy Compression**（ICS 2025）citeturn39view0 | HPC 科学数据（Nyx/Miranda/Hurricane）  | 有损 / error-regulated | 轻量 skipping DNN + online learning + cross-field learning   | 前 5 个学习 epoch 就有 **89% bit-rate reduction**；继续优化可达约 **94%**citeturn39view0 | 未确认                           | A      | 中高   |
| 2025 | **Lossless data compression by large models**（Nature Machine Intelligence 2025）citeturn32view0turn33search0 | 文本 + 图像 + 音频 + 视频              | 无损                   | 大型生成模型族（LMCompress）                                 | 宣称把 JPEG-XL / FLAC / H.264 的压缩率“减半”，文本接近 zpaq 的三分之一；代码在 Code Oceanciteturn32view0 | Code Ocean citeturn32view0    | S      | 中高   |
| 2025 | **BOA Constrictor: A Mamba-based lossless compressor for High Energy Physics data**（后发表于 MLST 2026）citeturn24view0turn25search7 | 结构化科学 byte stream / HEP 数据      | 无损                   | Mamba + streaming range coder                                | ATLAS：**2.21× vs 1.69×**；CMS：**44.14× vs 27.14×**；但吞吐落后于高度优化 CPU 压缩器citeturn24view0 | GitHub citeturn25search0      | A-     | 中     |
| 2026 | **Nacrith: Neural Lossless Compression via Ensemble Context Modeling and High-Precision CDF Coding** citeturn21view0 | 文本，也扩展到一般二进制文件           | 无损                   | 135M Transformer LM + n-gram/context mixing + arithmetic coding | alice29 **0.918 bpb**；enwik8 **0.9389 bpb**，优于 FineZip 与 ts_zip；强工程优化citeturn21view0 | GitHub citeturn22search0      | A      | 中     |
| 2026 | **StateSMix: Online Lossless Compression via Mamba State Space Models and Sparse N-gram Context Mixing** citeturn23view0 | 文本                                   | 无损                   | 在线训练 Mamba-style SSM + sparse n-gram + arithmetic coding | enwik8 10MB 上 **2.162 bpb**，比 xz -9e 好 **0.7%**；纯 C + AVX2 + OpenMP 实现citeturn23view0 | GitHub citeturn22search3      | A-     | 中     |

与本文“相关但不纳入主语料”的近邻方向主要有两类。第一类是 **prompt/context compression**，例如 LLMLingua、LongLLMLingua、tool-use context compression；它们追求的是**LLM 推理成本下降**而不是原始数据源编码意义上的压缩率。第二类是“把 compression 当作 evaluation protocol”而不是压缩器本身，例如把 lossless compression 作为 time-series 模型评测基准的工作。citeturn8search0turn8search4turn8search1turn28search3

## 按方向详细综述

### 文本压缩

如果只看 **真正无损文本压缩**，2022 年后的主线可以概括为三代。

第一代是 **在线 Transformer 压缩器**。TRACE 的核心不是“大模型”，而是**把 Transformer 做轻**，使它能在压缩任务里承担在线概率估计器的角色。它使用单层 Transformer、byte-grouping、shared-FFN 和更新控制器，把“深度模型压缩率高但太慢”的老问题，推进到了“压缩率不明显掉、速度显著提升”的工程平衡点。这类系统的优点是**自包含、无需预训练、适合一般 byte stream**；缺点是与后来的 LLM 相比，模型先验弱，压缩上限不够高。citeturn20search0turn20search1

第二代是 **Transformer/LLM 作为强先验文本模型**。GeneFormer 在基因序列上证明了 Transformer 的长程上下文建模确实能转化为更低 bitrate；LLMZip 则把这个逻辑搬到自然语言文本上，显式使用 LLaMA-7B 预测 token 概率，再交给 arithmetic coding 或 rank coding。LLMZip 的结果极具冲击力，但其最重要的局限也非常明显：实验只覆盖 1M text8 片段和 100K token 级别的新书文本，作者明确提示 text8 与 LLaMA 预训练语料可能重合，并且无法在全 100MB text8 上完成运行，因此这更像“可能上限”的展示，而不是可直接部署的实用压缩器。citeturn29search2turn29search6turn10view0

第三代是 **追求“可用性”的 LLM 压缩系统**。FineZip 的贡献不在于再次证明“LLM 压得更小”，而在于指出：如果使用 PEFT/LoRA 式“在线记忆”让模型适应当前文档，又用 dynamic context 让压缩过程可并行批处理，那么 LLM 压缩可以在速度上迈出决定性一步。Nacrith 进一步把**高精度 CDF、n-gram mixing、online bias correction、llama.cpp、并行 GPU、滑窗 KV cache**都纳入系统设计，说明 2026 年的文本压缩研究已经明显从“证明 LLM 能压缩”转向“把 LLM 压缩做成一个工程系统”。citeturn12view0turn21view0turn22search0

我的综合判断是：**无损文本压缩已经是 LLM 最有说服力的“非生成式”应用之一**，但目前最大的短板仍然不是压缩率，而是**吞吐、内存、模型分发、解码延迟和可复现 benchmark**。这也是为什么 FineZip、Nacrith 这类 work 的贡献里，工程优化往往和模型设计同样重要。citeturn12view0turn21view0

### LLM-based compression 与 Transformer / attention-based compression

这条线里，**Language Modeling Is Compression** 是方法论上的分水岭。它没有只把“压缩”当成应用，而是把“预测 = 压缩”的等价关系重新放到 foundation model 的规模化背景下讨论，进而分析 scaling law、tokenization、in-context learning 与压缩之间的联系。它最关键的启发是：**压缩不只是 downstream application，它还是评价模型是否真正建模了分布的自然指标**。citeturn16view0turn17search3

**Compression via Pre-trained Transformers** 则把这个视角从“大模型能压”推进为“多小的模型就足够有竞争力”。这篇 ICML 2025 论文的重要性很高，因为它明确考虑了**模型体积本身**，试图寻找预训练 Transformer 在真实压缩率上的 sweet spot。论文表明，百万级参数的小模型在原始 byte 级文本/图像/音频上，已经能在计入参数体积后超过 gzip/LZMA2，至少在某些模态上还优于 FLAC、JPEG-XL 这类传统或领域压缩器。换言之，foundation model 作为压缩器不一定非要走向越来越大。citeturn18view0turn19search1

**LMCompress** 更进一步，把“理解就是压缩”直接做成了一个极具传播力的结论。它在 Nature Machine Intelligence 上宣称在文本、图像、音频、视频四种模态上刷新 lossless compression 记录，并提供 Code Ocean 代码。这篇论文的影响力很大，但对于本文主题，应该做两点冷静区分：第一，它是**通用大模型驱动的 universal compression**，不是只针对文本；第二，从可公开访问内容看，最容易复核的是摘要级 claims，而**完整 throughput、计算成本、部署细节**并不像摘要中的压缩率那样容易核验，因此更适合作为“范式性信号”而非“现成立即可替代传统压缩器的工业方案”。citeturn32view0turn33search0

至于 **AlphaZip** 这类工作，我会把它放在“探索性、低到中可信度”层级：它证明了 rank sequence + 传统压缩器这个思路可以在小型 GPT-2、蒸馏模型、Brotli 上跑通，并且在小规模文学语料上确有增益；但它的 benchmark 与 enwik8/text8/Canterbury 这类经典压缩基准并不对齐，因此更像方法试验，而不是已稳定成立的 SOTA 证据。citeturn14view0turn15view1

### SSM-based compression

就本次纳入语料看，**SSM/Mamba 直接用于压缩器**的公开论文直到 2025–2026 才开始明显冒头。这意味着 SSM 路线目前仍是“新兴而非成熟主线”。citeturn24view0turn23view0

**BOA Constrictor** 的意义在于：它把 Mamba 作为 next-byte predictor，用于 **HEP 这种高度结构化、长程相关强、工业/科研 I/O 压力巨大的科学数据**。相比 LZMA-9，它在 ATLAS 和 CMS 数据上都给出了可观的压缩比提升，但作者也坦率指出 prototype 的吞吐还落后于 ZLIB 等高度优化实现。这说明 SSM 在**建模结构化长序列**方面很有潜力，但系统软件层面的优化仍然是门槛。citeturn24view0turn25search7

**StateSMix** 则把 Mamba-style SSM 拉回到更接近传统文本压缩 benchmark 的场景：它不依赖预训练权重，而是在压缩时对目标文件在线逐 token 训练，并用 sparse n-gram context mixing 做精确记忆补偿。这个设计特别值得注意，因为它与 LLMZip / FineZip / Nacrith 的“强预训练先验”路线正好相反：它试图证明**一个很小、在线适配、线性复杂度的 SSM**，在长上下文压缩中也能打出性价比。当前结果还算有前景，但与最强 LLM 压缩器相比，压缩率优势尚未成立；它更像是 SSM 在压缩方向上的“proof of concept”。citeturn23view0turn22search3

我的判断是：SSM 很可能会在未来几年里成为压缩研究的重要竞争者，尤其在**长序列、流式、在线、自适应**场景；但截至 2026 年中，Transformer/LLM 仍然是结果更强、证据更连续、系统更成熟的路线。citeturn20search0turn10view0turn12view0turn21view0turn23view0

### 浮点数、科学数据与时间序列压缩

这个方向与文本压缩最大的不同在于：它并不主要追求 bit-perfect universal coding，而是常常围绕 **error bound、PSNR、科学可用性、随机访问、I/O 吞吐** 来优化。citeturn36view1turn37view0turn39view0turn36view0

**HAE** 代表了“端到端 learned latent representation”在科学网格数据上的路线。它用 hierarchical autoencoder 学到紧凑表示，并在多个 benchmark 以及 CESM 气候数据上展示非常高的 compression ratio。其优点是压缩率高、适合块状场数据；局限是它更像“面向特定科学字段的 learned lossy codec”，而不是那种可直接替代通用科学压缩器的标准接口。citeturn36view1turn38search2

**SRN-SZ** 和 **NeurLZ** 代表另一条更有工程现实感的路线：不完全抛弃 SZ/ZFP 这类科学压缩传统，而是用神经网络去**修复残差、提升难压数据上的质量/码率表现、或在线适应数据场变化**。SRN-SZ 用 super-resolution network 改善 hard-to-compress datasets 的重建质量，在相同 error bound / PSNR 下给出非常大的 CR 改善；NeurLZ 则更进一步强调 **online learning、cross-field learning、error regulation**，这和 HPC 实际工作流非常契合。citeturn37view0turn39view0

**Deep Dict** 属于时间序列中的“真正 learned compressor”代表。它引入 Bernoulli Transformer Autoencoder 与 distortion constraint，并设计 QEL loss 处理时间序列压缩里的误差约束问题。与许多只做表示学习的时序论文不同，Deep Dict 明确以 compression ratio 为主评价目标，并在 10 个数据集上给出对 SOTA lossy compressor 的收益。citeturn36view0

至于**真正的神经 lossless floating-point compression**，在 2022–2026 的公开文献里仍然显得稀缺。即便 2025 年仍能看到专门讨论 floating-point lossless compression 的论文，很多也还是以**非神经变换**或传统压缩为主，而不是用神经网络学习概率模型来直接完成 lossless float compression。换句话说，这一块仍然是非常明显的研究空白。citeturn4search7turn36view1turn37view0turn39view0

### 数字、表格与结构化数据压缩

这一子方向目前最值得注意的工作是 **DeepMapping**。它的思想与文本压缩完全不同：与其做自回归概率分布估计，不如把表格压缩成一个会“记住” key-value mapping 的多任务神经网络，再用轻量辅助结构修复网络不能完美记住的样本。这样做的好处是，压缩与索引/查找被合并到了一个结构里，因此它同时优化**存储成本、查询延迟和运行时内存**。citeturn37view1turn30search9

这类方法的优点是非常适合数据库/嵌入式 key-value 场景；但局限也很鲜明。它不擅长处理像自然语言那样开放式、长上下文、强组合泛化的数据，也不天然提供“流式 source coding benchmark”里常见的 bpb/bpc 可比性。因此，DeepMapping 更像是**数据库系统里的 learned compression**，而不是传统压缩社区意义上的“统一无损序列压缩器”。citeturn37view1

值得补充的是，**Training LLMs over Neurally Compressed Text** 从另一角度指出了“结构化压缩表示”的重要性：如果压缩输出虽然熵低，但对后续模型来说不可学习，那么它未必是好的“semantic tokenization”。这对未来将表格、数值和文本混合建模的统一压缩框架非常关键。citeturn35view0

### Neural entropy coding 与传统编码器的结合

在本次主语料里，几乎所有表现强的**无损**系统都离不开一个共同模板：

**神经模型负责更准确的条件概率估计；传统熵编码器负责把概率真正变成 bitstream。**

这在 TRACE、GeneFormer、LLMZip、BOA、Nacrith、StateSMix 里都非常明显：Transformer/SSM/LLM 并不是单独完成“压缩”，而是把“预测的 log-loss”转化成“接近最优码长”的条件分布，再由 arithmetic/range coding 落地。这个事实很重要，因为它意味着“神经压缩”在很多时候并不是要替代信息论，而是把**更强的上下文建模能力**嫁接到**成熟的 entropy coder**上。citeturn20search0turn29search6turn10view0turn24view0turn21view0turn23view0

相反，完全不依赖熵编码、只把 rank 序列交给 gzip/bzip2/Brotli 的方法——如 AlphaZip、FineZip 的 rank-based 版本——更适合看成一种工程折中：实现简单、方便复用传统工具链，但理论上通常不如直接 feeding probability distribution 给 arithmetic coder 那么接近交叉熵极限。FineZip 自己的表格也说明，带 arithmetic coding 的版本压缩率更强，但时间成本极其高昂。citeturn12view0turn14view0turn15view1

我的综合判断是：未来几年里，**CDF 精度、量化误差、token vocabulary 的 probability floor、在线校准、context mixing** 这些 seemingly “old-school compression engineering” 的问题，会和大模型本身同样重要。Nacrith 的 24-bit CDF、online bias head 和 confidence-based skip，正说明研究前沿已经在“model quality”之外，重新重视“coder quality”。citeturn21view0

## 方法谱系图

如果用一条主线来描述这个领域近四年的技术演化，我会写成：

**传统 entropy coding**  
→ **在线神经概率模型**（RNN/轻量 Transformer 预测 next symbol）  
→ **Transformer-based general-purpose compressor**（TRACE）  
→ **domain-specific Transformer compressor**（GeneFormer）  
→ **frozen LLM as predictor + arithmetic/rank coding**（LLMZip）  
→ **压缩即建模 / foundation model as compressor**（Language Modeling Is Compression）  
→ **兼顾模型体积的 byte-level pretrained transformer compressor**（Compression via Pre-trained Transformers）  
→ **工程化实用 LLM compressor**（FineZip、Nacrith）  
→ **面向长序列低复杂度的 SSM/Mamba compressor**（BOA、StateSMix）。citeturn20search0turn29search2turn10view0turn16view0turn18view0turn12view0turn21view0turn24view0turn23view0

科学数据方向则是另一条相对独立但逐渐融合的支线：

**传统科学有损压缩器**（SZ/ZFP/FPZIP 系）  
→ **autoencoder / INR 式 learned latent compression**（HAE、KD-INR）  
→ **super-resolution / residual enhancement hybrid**（SRN-SZ）  
→ **在线学习增强传统科学压缩器**（NeurLZ）。citeturn36view1turn26search9turn37view0turn39view0

而表格/结构化数据则是一条数据库社区路线：

**dictionary/index + lossless table storage**  
→ **learned mapping / neural memorization + auxiliary correction**（DeepMapping）。citeturn37view1

## 关键论文深读

**TRACE（WWW 2022）** 是 2022 年后这条文献线非常重要的起点。它没有走“大模型”路线，而是把 Transformer 改造成适合压缩任务的轻量在线结构，用单层 transformer、byte-grouping、shared-FFN 和控制参数更新开销的机制，来解决深度压缩器“压得好但太慢”的老问题。它最重要的历史作用，不是单点 SOTA，而是证明 Transformer 在 lossless general-purpose compression 中可以成为一个现实可用的在线概率估计器。citeturn20search0turn20search4turn20search1

**GeneFormer（2022）** 说明 Transformer 压缩并不限于自然语言。该文把 Transformer-XL 变体、latent array 与 dynamic arithmetic coding 用到基因序列压缩上，相对先前学习式基因压缩器节省了 29.7% bitrate，并提高了解码速度。它的重要启发是：只要序列存在强长程依赖，Attention/Transformer 就有希望把“建模优势”直接转成“码长优势”。citeturn29search2turn29search6

**LLMZip（2023）** 是 LLM 压缩方向的标志性论文。它将 LLaMA-7B 作为 next-token predictor，用 SentencePiece tokenizer 产生 token，再用 arithmetic coding 或 rank coding 进行无损文本压缩；在 1M text8 片段上报告了 0.7101 bpc，并在 100K token 的 Gutenberg 新书上给出 0.8426 bpc。然而，它最值得认真阅读的其实是其**自我限制说明**：作者承认无法在全 100MB text8 上跑完，也承认 Wikipedia 预训练覆盖可能让 text8 结果偏乐观。因此它既是一个里程碑，也是一篇很好的“不要过度解读摘要数字”的案例。citeturn10view0turn9view0

**Language Modeling Is Compression（ICLR 2024）** 提供了整个领域最清晰的理论框架之一。它把“模型越会预测，压缩越好”推广到 foundation model scale，并且通过 Chinchilla 70B 在图像 patch 和音频 sample 上打败 PNG/FLAC 的案例，说明“语言模型”其实更接近“通用序列模型”。对后续研究来说，这篇论文最重要的遗产是两个：其一，把 compression 重新定义成一种模型评测方法；其二，提醒大家要把**模型权重大小**也纳入真实压缩率计算。citeturn16view0turn17search3turn17search0

**FineZip（2024）** 是最具“工程味”的 LLM 文本压缩论文之一。它把 LoRA 式 parameter-efficient fine-tuning 当作在线记忆模块，让预训练 Llama-3 8B 适配要压的语料；再用 dynamic context 允许更高程度的并行压缩。结果上，它把 10MB 文本的压缩时间从 LLMZip 的 9.5 天缩短到约 4 小时，54× 的提速几乎就是它的核心贡献。它同时也很诚实地表明：即便如此，LLM 仍然远谈不上“大规模文本压缩的现实替代品”。citeturn11view0turn12view0

**Compression via Pre-trained Transformers（ICML 2025）** 把研究问题改写成了“多小的 Transformer 才足够当压缩器”。这篇工作用 165GB 原始 byte 数据训练小型 vanilla Transformer，并在 OOD 文本/图像/音频上评估；结果显示，相对小的模型在计入参数体积后，仍可超过 gzip/LZMA2，甚至某些领域压缩器。它对未来研究最重要的启示是：foundation model 压缩并不一定非要更大，反而可能需要专门围绕**模型大小—预测质量—可部署性**做 rate-cost trade-off。citeturn18view0turn19search1

**DeepMapping（ICDE 2024）** 是结构化数据方向最值得重视的一篇。它不走 sequence modeling，而是将表格拆成多个 key-value 映射，再用多任务神经网络存储这些映射关系，并用辅助结构纠错，还支持插入、删除、更新。它的贡献在数据库场景里非常实在：压缩不再只追求最小码长，而是同时追求高效 lookup。它与文本压缩研究的关系在于：都在利用神经网络的“记忆/建模能力”，但 DeepMapping 更像 learned storage structure，而不是经典 source coder。citeturn37view1turn30search3turn30search9

**HAE（2023）** 是科学数据 learned compression 的代表作。它展示了 hierarchical autoencoder 在大规模高分辨率科学数据上的潜力，尤其是对 climate/simulation 网格的高 compression ratio。其优势在于码率非常激进；局限在于结果更多以“科学分析是否仍可用”来定义，而不是以文本压缩那样更统一的 bpb/bpc 来定义。这是科学数据压缩与文本压缩评价体系差异的一个典型样本。citeturn36view1turn38search2

**SRN-SZ（2023）** 非常典型地体现了“不是取代传统科学压缩器，而是增强它们”。它把 super-resolution network HAT 引入 scientific error-bounded lossy compression，在难压数据上以同误差界或同 PSNR 为约束，给出 75%/80% 量级的压缩改进。这条线的价值很高，因为 HPC 用户真正关心的往往不是“是否纯神经”，而是“同样误差限制下我是否能减少 I/O 和存储”。citeturn37view0

**NeurLZ（ICS 2025）** 把这种 hybrid 思路又往前推进了一步。它强调 compression-time online learning、cross-field learning 和 error regulation，目标是让神经模块在压缩时实时适应 residual pattern，而不是离线训练一个大模型再硬套到所有数据场。它非常值得注意，因为这可能是未来 HPC 压缩最现实的神经化路径：不是训练一个庞大的通用神经 codec，而是让一个轻量模型在压缩过程中边学边修复传统压缩的盲点。citeturn39view0

**Nacrith（2026）** 是目前文本无损压缩里最“系统工程化”的 work 之一。它用 135M 参数的 SmolLM2-135M，加上 token-level n-gram、本地在线校准头、高精度 24-bit CDF、confidence-based skip、llama.cpp 与多 GPU 并行，在 alice29 和 enwik8 上都给出了非常强的结果，并明确把一般二进制文件也纳入格式设计。这篇工作最有意思的地方在于：它表明“下一代 LLM 压缩器”的关键不只是更大的模型，而是**更好的 coder、局部上下文混合、在线校准与推理系统**。citeturn21view0turn22search0

**StateSMix（2026）** 是目前我看到最直接、最有代表性的 SSM 文本压缩器。它在线训练一个很小的 Mamba-style SSM，并用 sparse n-gram 表做 context mixing，在纯 CPU / 纯 C 环境中实现对 xz 的小幅超越。它还远不是 SSM 压缩路线的终局，但它已经把一个非常关键的问题提出来了：**如果我们真正关心在线压缩和长序列处理，难道不该重新考虑 attention 之外的 backbone？** 从这个意义上说，StateSMix 的研究价值很可能大于它当前的绝对结果。citeturn23view0turn22search3

## 研究空白与未来方向

当前最明显的空白，是 **“高压缩率”与“高吞吐/低成本”之间的鸿沟**。LLMZip、FineZip、LMCompress、Nacrith 都告诉我们：大模型确实能压，但系统开销极大；即使最优化的实现，也很难与 gzip/zstd/xz 这类传统工具的工程成熟度相比。未来最值得做的实验，不是再单纯提高 0.01 bpb，而是做 **compression ratio × throughput × memory × model-size × energy** 的系统级 Pareto 分析。citeturn10view0turn12view0turn32view0turn21view0

第二个空白是 **tokenization / representation design**。LLMZip、GeneFormer、Language Modeling Is Compression 与 Training LLMs over Neurally Compressed Text 都表明：tokenizer 不是中性组件，它直接决定上下文长度、概率离散化方式、CDF 精度需求与 learnability。未来很值得系统比较 **byte-level、BPE、equal-info windows、domain-aware tokenization** 在压缩率与学习性上的双重效果。citeturn10view0turn29search6turn16view0turn35view0

第三个空白是 **lossless floating-point / numerical array compression 的神经化仍很弱**。当前更成熟的是有损 / error-bounded scientific compression；如果未来要做真正强的神经 lossless float compressor，至少需要解决 typed data 的位级结构、跨元素统计依赖、数值稳定性与编码成本之间的张力。一个可操作的实验方向是：用 typed tokenization 把 exponent / mantissa / sign 分开建模，再让小型 Transformer 或 SSM 只负责最难预测的部件，同时保留传统字节变换和 entropy coder。citeturn36view1turn37view0turn39view0turn4search7

第四个空白是 **SSM 是否真的比 Attention 更适合压缩**。理论上，压缩器需要长上下文、低延迟、在线更新和稳定流式推理，这几乎是 SSM 的理想战场；但目前证据主要来自 2025–2026 的少数 preprint/prototype。一个非常值得做的基准，是在 **enwik8 / large text benchmark / structured binary logs / HPC streams** 上，对比同等参数预算下的 Transformer、Mamba、Hyena-like 模型，并统一报告 bpb、吞吐、内存和 warm-up 代价。citeturn23view0turn24view0

第五个空白是 **通用压缩器与领域压缩器的边界**。LMCompress、Language Modeling Is Compression 和 Compression via Pre-trained Transformers 代表“universal model as compressor”；SRN-SZ、NeurLZ、Deep Dict、DeepMapping、BOA 则说明 domain structure 非常值钱。未来最可能取得实用突破的，未必是“一个模型压万物”，而是 **通用先验 + 领域适配层 + 传统 coder** 的分层系统。citeturn16view0turn18view0turn32view0turn37view0turn39view0turn36view0turn37view1turn24view0

第六个空白是 **benchmark 与复现文化**。文本压缩论文常常用不同 tokenizer、不同 sample size、不同语料切片，很难横向公平比较。一个更好的未来方向，是建立类似“压缩版 MMLU/HELM”的统一框架：固定输入格式、固定是否计入模型权重、固定训练语料不重叠规则、统一报告 bpb/吞吐/显存/能耗/解码延迟。没有这个层面的标准化，很多“刷新记录”的结果都很难真正进入工业实践。citeturn10view0turn12view0turn18view0turn21view0

### 开放问题与本综述局限

这份综述已经尽量使用原始论文页、会议页和作者代码页来支撑结论，但仍有三点局限需要明确。其一，部分 2025–2026 前沿工作仍是 **arXiv 预印本**，尤其是 SSM/LLM 工程化压缩器，因此我把它们更多当作“研究信号”而不是已经尘埃落定的共识。citeturn21view0turn23view0turn24view0

其二，少数论文的公开网页只能稳定访问到**摘要级数字**，不能直接看到完整表格与所有基线设置；这时我只引用了公开页面明确写出的结果，没有扩展推断。其三，像 prompt compression、context compression、compressed-text learning 这类邻接方向非常重要，但它们与“真正的数据压缩”之间存在定义边界；我在本文中做了严格区分，因此没有把它们混进主语料。citeturn35view0turn8search0turn8search4turn8search1

综合起来，我的判断是：**2022 年以来，“大模型做压缩”已经从概念验证进入系统化探索阶段；但真正距离工业就绪最近的，不是纯粹更大的模型，而是“中等规模模型 + 更好的 entropy coding + 更强的在线适配 + 更扎实的工程优化 + 更统一的 benchmark”。** 文本方向已经初步验证了这一点；科学数据、结构化数据和 SSM 路线，接下来很可能会成为真正拉开差距的战场。citeturn12view0turn18view0turn39view0turn37view1turn23view0
