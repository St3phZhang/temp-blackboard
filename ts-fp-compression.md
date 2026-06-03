下面是按 **2022 年至 2026 年 5 月可检索公开资料**整理的一版系统综述。我把论文分成四类：**时序/浮点专用压缩**、**FastLanes / lightweight columnar compression**、**科学计算浮点有损压缩**、**边缘相关工作**。由于 Google Scholar 的完整引用图谱无法在这里稳定抓取，我主要用 **VLDB/PACMMOD/SIGMOD 官方论文页、PDF、DBLP、arXiv、GitHub、论文 related work** 交叉验证；下面标注“不确定”的地方表示我没有在可访问正文中确认到完整实验细节。

---

## 一、核心论文清单

### 1. Chimp: Efficient Lossless Floating Point Compression for Time Series Databases

**论文信息**

* 标题：*Chimp: Efficient Lossless Floating Point Compression for Time Series Databases*
* 作者：Panos Liakos, Katia Papakonstantinopoulou, Yannis Kotidis
* 年份：2022
* Venue：PVLDB 2022
* 论文链接：官方 PVLDB/PDF 可检索；论文页给出代码链接。
* 代码或项目链接：论文中给出 GitHub 链接。

**摘要翻译**

该论文关注时序数据库中的浮点数 lossless compression。传统通用压缩算法虽然压缩率较好，但压缩/解压开销高，不适合需要高吞吐写入和快速访问的时序场景。Chimp 提出一种面向浮点时间序列的流式压缩算法，在保持快速访问能力的同时，相比已有流式方法显著降低存储空间。论文报告称，Chimp 平均只需要当时最先进 streaming 方法约一半的空间，并能保持较好的压缩和访问速度。

**核心 Idea**

* 解决的问题：Gorilla 类 XOR-based 浮点时序压缩在某些数据分布下压缩率不足。
* 技术设计：仍以连续浮点值的 XOR 结果为核心，但改进 leading zero / trailing zero 的编码策略。
* 新颖点：对 Gorilla 的编码决策进行重新设计，使常见浮点时间序列中的 XOR pattern 得到更短表示。
* 与技术关键词关系：强相关于 **lossless floating-point compression、time-series compression、XOR encoding、bit-level encoding、streaming compression**。
* 不属于 FastLanes 风格的 SIMD 列式压缩；更偏时序数据库写入路径上的 streaming codec。

**针对的数据类型与能力**

* 数据类型：floating-point time-series data、sensor / IoT data、database time-series columns。
* 支持：lossless compression、streaming compression、快速顺序访问。
* 不强调：SIMD/vectorized decompression、compressed query execution、随机点访问。
* 查询处理：主要通过更快解压和较小存储间接提升查询性能。

**实验设置**

* 数据集：论文使用多个真实时间序列数据集，覆盖传感器、监控、科学/测量类浮点序列。
* 数据规模：公开正文摘要中强调“大规模时序数据库”场景，但不同数据集具体规模需看论文实验表。
* 数据类型：double / floating-point time series。
* 数据性质：以真实数据为主。

**对比 Baseline**

* Gorilla
* FPC
* 通用压缩算法，如 LZ4、Zstd、Snappy 等
* 后续论文中，Chimp/Chimp128 常作为 Elf、ALP、Serf 的核心 baseline。

**实验表现**

* 论文报告 Chimp 在平均空间占用上约为此前 SOTA streaming 方法的一半，并在压缩/访问时间上优于或接近已有方法。
* 优势：lossless、streaming、实现相对简单、适合 TSDB。
* 劣势：主要针对浮点时序；不利用 SIMD/vectorized execution；对列式分析查询和随机访问支持有限。

---

### 2. Frequency Domain Data Encoding in Apache IoTDB

**论文信息**

* 标题：*Frequency Domain Data Encoding in Apache IoTDB*
* 作者：Apache IoTDB 相关团队
* 年份：2022
* Venue：PVLDB 2022
* 论文链接：PVLDB/PDF。([VLDB][1])
* 代码或项目链接：与 Apache IoTDB / TsFile 生态相关。

**摘要翻译**

该论文研究如何在 Apache IoTDB 中对频域数据进行高效编码。许多时序数据经过 FFT 或类似变换后会变成高精度、分布偏斜的频域浮点数据，传统 Gorilla、差分编码或固定宽度 bit-packing 不一定适合。论文提出 Descend，通过量化和 descending bit-packing 对频域数据进行编码，也可用于时域数据的 lossy compression，并在 IoTDB 中落地。([VLDB][1])

**核心 Idea**

* 解决的问题：IoTDB 中高精度频域浮点数据难以用传统 TSDB 编码高效压缩。
* 技术设计：结合频域分析、量化、descending bit-packing。
* 新颖点：不是单纯在时间域上做 delta/XOR，而是针对 frequency-domain representation 的位宽分布设计编码。
* 关联技术：**lossy compression、bit-packing、quantization、time-series、IoTDB/TsFile**。
* 与 FastLanes 的关系：都关注 bit-packing 和轻量编码，但 Descend 是 IoTDB/频域时序专用，不是通用 SIMD 列式布局。

**针对的数据类型与能力**

* 数据类型：frequency-domain time-series data、floating-point data、IoT sensor data。
* 支持：lossy compression；也可直接编码频域数据。
* Streaming：适合 IoTDB 数据写入/存储路径。
* SIMD/vectorization：不是主要贡献。
* Query over compressed data：主要在 IoTDB 编码层面降低存储和 I/O。

**实验设置**

* 数据集：IoTDB 场景下的时序数据，包括频域数据和时域数据。
* 指标：compression ratio、encoding/decoding throughput、与 GZIP/LZ4/Snappy 等组合后的表现。
* 数据性质：真实 IoT/时序数据与特定频域变换数据。([VLDB][1])

**对比 Baseline**

* Gorilla
* 差分编码
* 固定宽度 bit-packing
* Buff
* GZIP / LZ4 / Snappy 等通用压缩器。([VLDB][1])

**实验表现**

* Descend 在多数数据集上取得最高或接近最高压缩率，但在噪声较强的数据上表现下降。
* Gorilla 解码最快，但压缩率通常低于 Descend。
* Descend 与 GZIP/LZ4/Snappy 组合可进一步提高压缩率，但会牺牲速度。([VLDB][1])

---

### 3. FastLanes: A New Portable SIMD-Friendly Compression Layout for Scalar and SIMD Cores

**论文信息**

* 标题：*FastLanes: A New Portable SIMD-Friendly Compression Layout for Scalar and SIMD Cores*
* 作者：Azim Afroozeh 等
* 年份：2023
* Venue：PVLDB 2023
* 论文链接：PVLDB/PDF。([VLDB][2])
* 代码或项目链接：FastLanes GitHub 项目。([GitHub][3])

**摘要翻译**

FastLanes 关注现代列式数据库和大数据文件格式中的 lightweight compression。论文指出，Parquet、ORC 等系统广泛使用 dictionary、frame-of-reference、delta、RLE 等轻量编码，但这些编码在不同 SIMD 指令集和架构上难以同时获得高吞吐与可移植性。FastLanes 提出一种新的压缩数据布局，通过统一转置布局和“虚拟 1024-bit SIMD register”思想，让普通 scalar C++ 代码也能被编译器自动向量化，并在多种轻量编码上达到超过每周期 40 个值的解码吞吐。([VLDB][2])

**核心 Idea**

* 解决的问题：列式压缩算法本身轻量，但内存布局不适合自动 SIMD/vectorization。
* 技术设计：Unified Transposed Layout；将数据组织成 SIMD-friendly lanes。
* 新颖点：不是发明一个新的单一 codec，而是提出一种 **compression layout**，让 DICT、FOR、DELTA、RLE 等既有编码自动获得数据并行性。
* 关联技术：**SIMD、vectorized decompression、bit-packing、FOR、DELTA、RLE、dictionary encoding、columnar data**。
* 与浮点压缩关系：FastLanes 本身主要是通用列式压缩布局，不是专门为浮点时序设计；但 ALP/FastLanes File Format 后续把这种思想扩展到 double/float 和文件格式层。

**针对的数据类型与能力**

* 数据类型：database columns、integer columns、dictionary-encoded values、columnar data。
* 支持：lossless compression、SIMD/vectorized decompression、random-access-friendly block/vector layout。
* Streaming：不是主要目标。
* Query over compressed data：为向量化扫描和列式执行提供基础。

**实验设置**

* 数据集：论文对数据库列、Parquet/ORC 类列式编码进行评测。
* 指标：decompression throughput、values per cycle、不同 CPU/SIMD 架构上的可移植性。
* 数据性质：数据库列式数据为主。

**对比 Baseline**

* Parquet / ORC 中的 lightweight compression
* 传统 scalar 解码
* 手写 SIMD / 平台相关 SIMD 实现
* DICT / FOR / DELTA / RLE 类编码实现。([VLDB][2])

**实验表现**

* 论文报告在多个轻量编码上达到超过 40 values/cycle 的吞吐，并显著加速 Parquet/ORC 风格的列式编码。([VLDB][2])
* 优势：高吞吐、可移植、适合列式数据库执行引擎。
* 劣势：核心贡献是布局和向量化，不是针对浮点数语义或时序相关性的压缩算法。

---

### 4. Elf: Erasing-based Lossless Floating-point Compression

**论文信息**

* 标题：*Elf: Erasing-based Lossless Floating-point Compression*
* 作者：Yuyao 等
* 年份：2023
* Venue：PVLDB 2023
* 论文链接：PVLDB/PDF。
* 代码或项目链接：未在我检索到的摘要页中可靠确认。

**摘要翻译**

Elf 研究 lossless floating-point compression，目标是同时适用于 time-series 和非 time-series 的浮点数据。论文发现，若在 XOR 编码前将浮点数尾部若干位“擦除”为 0，可以增加 XOR 结果中的 trailing zeros，从而提高压缩率；关键挑战是如何在不丢失原值的情况下恢复这些被擦除的信息。Elf 给出数学条件和编码方案，使这种“擦除”仍然保持 lossless，并以 O(N) 时间和 O(1) 内存完成压缩。

**核心 Idea**

* 解决的问题：Gorilla/Chimp 类方法依赖相邻浮点 XOR pattern，但 trailing zero 不够多时编码效率下降。
* 技术设计：先对浮点数低位做可恢复的 erasing，再进行 XOR 编码。
* 新颖点：看似“清零尾部位”会有损，Elf 通过数学恢复条件让它保持 lossless。
* 关联技术：**lossless floating-point compression、XOR encoding、leading/trailing zeros、time-series / non-time-series floating data**。
* 与 SIMD/FastLanes：不以 SIMD 为核心；更接近 Chimp/Patas/Gorilla 传统 bit-level streaming 浮点压缩。

**针对的数据类型与能力**

* 数据类型：floating-point data、time-series data、non-time-series floating-point columns。
* 支持：lossless compression、streaming-like sequential compression。
* Random access：不作为核心卖点。
* SIMD/vectorization：不是主要贡献。
* Query over compressed data：主要依赖解压。

**实验设置**

* 数据集：论文使用 22 个数据集，覆盖 time-series 与 non-time-series 浮点数据。
* 指标：compression ratio、compression/decompression time、memory footprint。
* 数据性质：真实数据集为主，覆盖多个浮点场景。

**对比 Baseline**

* Gorilla
* FPC
* Chimp / Chimp128
* LZ4 / Zstd / Snappy
* 其他 floating-point compression 方法。

**实验表现**

* 在 time-series 数据上，Elf 相比 Gorilla/FPC 平均相对提升约 51%，相比 Chimp 约 47%，相比 Chimp128 约 12%。
* 相比 LZ4、Zstd、Snappy，在 time-series 数据上的平均相对提升分别约为 30.2%、7.5%、27.5%。
* Elf 的内存占用为 O(1)，而 Chimp128 需要保存额外窗口状态。
* 劣势：后续 ALP 论文指出 Elf 虽然压缩率强，但速度相对 Chimp128 慢不少。([ir.cwi.nl][4])

---

### 5. Sim-Piece: Highly Accurate Piecewise Linear Approximation through Similar Segment Merging

**论文信息**

* 标题：*Sim-Piece: Highly Accurate Piecewise Linear Approximation through Similar Segment Merging*
* 作者：Yiran 等
* 年份：2023
* Venue：PVLDB 2023
* 论文链接：PVLDB/PDF。([VLDB][5])
* 代码或项目链接：论文中给出项目链接。([VLDB][5])

**摘要翻译**

Sim-Piece 研究 time-series lossy compression 中的 piecewise linear approximation。传统 PLA 方法用线段近似时序，并保证误差界限，但压缩率与精度之间存在明显权衡。Sim-Piece 发现许多线段具有相似形状，因此可以合并相似 segment，用更少参数表示多个线性片段。论文报告相对传统 PLA 方法平均压缩率提升超过 2 倍，同时保持较高精度。([VLDB][5])

**核心 Idea**

* 解决的问题：传统 PLA 对每段线都单独编码，无法利用相似线段之间的冗余。
* 技术设计：识别并合并相似 line segments；对起点和斜率候选进行组织；用分组方式减少编码成本。
* 新颖点：从“每段独立拟合”转向“相似片段共享表示”。
* 关联技术：**lossy time-series compression、piecewise linear approximation、error-bounded compression**。
* 与 SIMD/FastLanes：关系较弱；主要是算法层面的 lossy approximation。

**针对的数据类型与能力**

* 数据类型：time-series data、sensor data、trajectory/measurement-like sequences。
* 支持：lossy compression、error-bounded approximation。
* Streaming：部分 PLA baseline 支持 online，但 Sim-Piece 更偏压缩算法评估。
* Random access：可通过 segment index 支持近似访问，但不是核心亮点。
* SIMD/vectorization：不是主要贡献。

**实验设置**

* 数据集：论文使用多个时序数据集，并与 PLA 系列方法比较。
* Baseline 实现包括 PMC-MR、Swing、Slide、Mixed 等。([VLDB][5])
* 指标：compression ratio、误差约束满足情况、压缩时间。

**对比 Baseline**

* PMC-MR
* Swing
* Slide
* Mixed
* 传统 PLA 方案。([VLDB][5])

**实验表现**

* 论文报告平均压缩率相比传统 PLA 提升超过 2 倍。
* 优势：在严格误差阈值下仍能提升压缩率。
* 劣势：这是 lossy approximation，不适合需要 bit-exact 恢复的数据库列或金融/科学精确数据。([VLDB][5])

---

### 6. BtrBlocks: Efficient Columnar Compression for Data Lakes

**论文信息**

* 标题：*BtrBlocks: Efficient Columnar Compression for Data Lakes*
* 作者：Maximilian Kuschewski, David Sauerwein, Adnan Alhomssi, Viktor Leis
* 年份：2023
* Venue：PACMMOD / SIGMOD 2023
* 论文链接：ACM / PACMMOD 页面。([ACM DL Next][6])
* 代码或项目链接：GitHub 项目可检索。([GitHub][7])

**摘要翻译**

BtrBlocks 面向 data lake 中的列式存储压缩，目标是在保持高解压吞吐的同时获得较好的压缩率。它采用多种 lightweight encoding，并针对列数据选择合适编码方式。它不是专门针对时序或浮点数据的算法，而是通用列式压缩框架。([Department of Computer Science][8])

**核心 Idea**

* 解决的问题：Parquet/ORC 等 data lake 格式压缩选择有限，通用压缩器不利于快速分析查询。
* 技术设计：对列数据采用轻量编码组合；根据数据特征选择编码。
* 新颖点：系统性地把 lightweight encoding 用于开放 data lake 格式。
* 关联技术：**dictionary encoding、RLE、frame-of-reference、delta-like encoding、columnar compression**。
* 与 FastLanes：BtrBlocks 是 FastLanes File Format 的重要对照对象；FastLanes 后续认为 BtrBlocks 支持 cascading lightweight compression，但缺少 FastLanes 的 data-parallel layout。([VLDB][9])

**针对的数据类型与能力**

* 数据类型：database columns、data lake columnar data、integer/string/floating columns。
* 支持：lossless compression、列式扫描。
* Random access：列块级访问。
* SIMD/vectorization：不是核心贡献。
* Query over compressed data：通过列式格式和轻量编码改善查询路径。

**实验设置**

* 数据集：data lake / analytical columnar workloads。
* 指标：compression ratio、compression/decompression speed、query processing 相关开销。
* 具体数据集细节需要查阅原文实验表。

**对比 Baseline**

* Parquet
* ORC
* 通用压缩器
* 轻量列式编码组合
* 后续 FastLanes File Format 将其作为相关系统比较。([VLDB][9])

**实验表现**

* 公开摘要强调 BtrBlocks 兼顾高解压吞吐与较好压缩率。([Department of Computer Science][8])
* 优势：通用列式场景强。
* 劣势：不专门解决浮点数二进制表示问题；SIMD/data-parallel layout 不如 FastLanes 系列突出。

---

### 7. LeCo: Lightweight Compression via Learning Serial Correlations

**论文信息**

* 标题：*LeCo: Lightweight Compression via Learning Serial Correlations*
* 作者：Yihao Liu, Xinyu Zeng, Huanchen Zhang
* 年份：2024
* Venue：PACMMOD / SIGMOD 2024
* 论文链接：arXiv / ACM 可检索。([arXiv][10])
* 代码或项目链接：GitHub 项目。([GitHub][11])

**摘要翻译**

LeCo 研究 column store 中的 lightweight compression。论文认为，Frame-of-Reference、Delta、Run-Length Encoding 等传统方法都可以看作是在利用列值之间的 serial correlation；LeCo 用轻量学习模型捕捉这种相关性，从而在压缩率和随机访问性能之间取得更好的 Pareto 权衡。([arXiv][10])

**核心 Idea**

* 解决的问题：传统 FOR/Delta/RLE 只捕捉简单相关性，面对复杂但可预测的序列时压缩率不足。
* 技术设计：学习序列相关性，预测值或局部模式，再压缩 residual。
* 新颖点：把 classical lightweight encoding 统一视为 serial correlation modeling 的特例。
* 关联技术：**learned compression、FOR、delta encoding、RLE、random access、column store compression**。
* 与 FastLanes：方法目标相似，都是列式 lightweight compression；但 LeCo 强调 learned correlation，而 FastLanes 强调 SIMD-friendly layout。

**针对的数据类型与能力**

* 数据类型：integer / numeric database columns、columnar data、序列化列值。
* 支持：lossless compression、random access。
* SIMD/vectorization：不是主贡献。
* Query over compressed data：可用于 Arrow、RocksDB 等系统路径。
* Floating-point：不是核心方向，但可作为通用 numeric columns 的相关工作。

**实验设置**

* 数据集：3 个 synthetic 数据集和 6 个 real 数据集。
* 系统集成：论文摘要提到在 Apache Arrow 上获得 5.2× 查询加速，在 RocksDB 上提升约 16% 吞吐。([arXiv][10])

**对比 Baseline**

* FOR
* Delta
* RLE
* 通用 lightweight column compression
* Arrow / RocksDB 相关压缩路径。([arXiv][10])

**实验表现**

* 优势：在压缩率和 random access 上取得 Pareto 改进。
* 劣势：模型学习和元数据开销需要控制；不直接解决 IEEE floating-point 表示问题。([arXiv][10])

---

### 8. ALP: Adaptive Lossless floating-Point Compression

**论文信息**

* 标题：*ALP: Adaptive Lossless floating-Point Compression*
* 作者：Azim Afroozeh 等
* 年份：2024
* Venue：SIGMOD / PACMMOD 2024
* 论文链接：官方 PDF。([ir.cwi.nl][4])
* 代码或项目链接：ALP GitHub 项目；仓库说明其为 SIGMOD 2024 work。([GitHub][12])

**摘要翻译**

ALP 面向数据库中的 double-precision floating-point 列。论文观察到，许多分析数据中的浮点数本质上来自 decimal values，只是被 IEEE floating-point 表示引入了二进制舍入问题。ALP 用 adaptive sampling 判断每个 row group/vector 是否适合 decimal-like encoding；若适合，则把 double losslessly 转换为 integer，再用 bit-packing/FOR 等轻量编码；若不适合，则使用 front-bit compression。ALP 的实现依赖 FastLanes 风格的向量化执行，使 scalar code 能自动向量化。([ir.cwi.nl][4])

**核心 Idea**

* 解决的问题：数据库浮点列常被通用压缩器或 Patas/Elf/Chimp 类方法处理，但要么慢，要么不适合向量化扫描。
* 技术设计：两条路径：enhanced PseudoDecimals + vectorized front-bit compression。
* 新颖点：利用“很多 double 实际来自 decimal”的事实，把 lossless float compression 转换为 integer compression。
* 关联技术：**lossless floating-point compression、SIMD/vectorized execution、FastLanes、bit-packing、frame-of-reference、predicate pushdown**。
* 论文明确使用 FastLanes 的 Fused Frame-Of-Reference encoding。([ir.cwi.nl][4])

**针对的数据类型与能力**

* 数据类型：floating-point database columns、BI 数据、sensor/time-series/scientific-style numeric columns。
* 支持：lossless compression、SIMD/vectorized decompression、random/vector-level access、predicate pushdown。
* Streaming：不是主要目标。
* Query over compressed data：支持 vector-level skip 和 predicate pushdown 思路。([ir.cwi.nl][4])

**实验设置**

* 数据集：论文分析 30 个数据集，其中 18 个来自 Elf/Chimp，12 个来自 Public BI Benchmark / PDE 相关数据。([ir.cwi.nl][4])
* 数据类型：真实 double columns，包括分析数据、传感器/时序类、公开 BI 数据。
* 指标：compression ratio、compression/decompression throughput、query-related benefits。

**对比 Baseline**

* Chimp / Chimp128
* Elf
* Patas
* Zstd / Snappy / LZ4
* BtrBlocks / PseudoDecimal Encoding
* FastLanes integer codecs。([ir.cwi.nl][4])

**实验表现**

* ALP 论文声称在相关维度上优于竞争方法；其压缩率只被较重的通用压缩器接近，但那些方法通常较慢且 block-based。
* 论文还指出，当级联 Dictionary/RLE 时，ALP 可进一步改善压缩率，例如文中提到 ALP cascading 后相较 zstd 的空间表现。([ir.cwi.nl][4])
* 优势：这是 2022 年后最值得重点读的 **floating-point database compression + FastLanes-style vectorization** 论文。
* 劣势：主要适合数据库列中的 double；对纯 streaming IoT 场景不如 Chimp/Elf/Serf 直接。

---

### 9. FCBench: Cross-domain Benchmarking of Lossless Floating-point Compressors

**论文信息**

* 标题：*FCBench: Cross-domain Benchmarking of Lossless Floating-point Compressors*
* 作者：Shouyi Yin 等
* 年份：2024
* Venue：PVLDB 2024
* 论文链接：PVLDB/PDF。
* 代码或项目链接：论文中给出代码链接。

**摘要翻译**

FCBench 不是提出单一新压缩算法，而是系统比较 lossless floating-point compressors。论文指出数据库和 HPC/scientific computing 社区长期各自评估压缩器，指标和数据集不统一。FCBench 评测 8 个 CPU 方法和 5 个 GPU 方法，使用 33 个真实数据集，并用 roofline 等方法分析压缩器在不同硬件和数据分布下的瓶颈。

**核心 Idea**

* 解决的问题：lossless floating-point compression 缺少跨数据库/HPC/时序/观测数据的统一 benchmark。
* 技术设计：统一实现、统一数据集、统一指标；比较 CPU/GPU 压缩器。
* 新颖点：把 DB-oriented 和 HPC-oriented 浮点压缩器放在同一框架评估。
* 关联技术：**lossless floating-point compression、CPU/GPU、HPC data、time-series、database data、roofline analysis**。
* 与 FastLanes：可作为评估 ALP/FastLanes-style 方法在更广泛浮点数据上的参照。

**针对的数据类型与能力**

* 数据类型：HPC scientific data、time-series、observation data、database transaction data。
* 支持：视具体压缩器而定；FCBench 本身是 benchmark。
* SIMD/GPU：覆盖 CPU 与 GPU 压缩方法。

**实验设置**

* 数据集：33 个真实数据集，覆盖 HPC、time-series、observation、database transaction 四类。
* 方法：8 个 CPU compressors、5 个 GPU compressors。
* 指标：compression ratio、compression/decompression throughput、block/page size、数据库查询额外开销、roofline 分析。

**对比 Baseline**

* 多个 CPU/GPU lossless floating-point compressors。
* 包括数据库侧和 HPC 侧方法。
* 论文还讨论 Buff 等面向低精度 IoT/server monitoring 的方法。

**实验表现**

* 主要结论不是“某一算法绝对最好”，而是压缩率、吞吐、硬件利用率和数据域之间存在明显 trade-off。
* 优势：非常适合作为选型和复现实验的入口。
* 劣势：不是新 codec，不能直接替代 ALP/Elf/Chimp。

---

### 10. CompressIoTDB: Improving Time Series Data Compression in Apache IoTDB

**论文信息**

* 标题：*Improving Time Series Data Compression in Apache IoTDB*
* 作者：Apache IoTDB / 相关研究团队
* 年份：2025
* Venue：PVLDB 2025
* 论文链接：PVLDB/PDF。
* 代码或项目链接：论文中给出项目链接。

**摘要翻译**

该论文关注 IoTDB 中压缩与查询执行之间的矛盾。时序数据库通常通过压缩显著降低磁盘和内存占用，但查询时若必须完整解压，会带来额外延迟和资源消耗。论文提出 CompressIoTDB 框架，在编码层和查询执行层引入 compressed-domain processing，使过滤、聚合、窗口等操作尽量在压缩表示上完成。实验显示平均吞吐提升 53.4%，内存占用降低 20%。

**核心 Idea**

* 解决的问题：TSDB 压缩节省空间，但查询时全量解压带来性能损失。
* 技术设计：CompColumn 抽象；在 RLE、Dictionary、Ts_2Diff 等编码上执行过滤、聚合、窗口等操作。
* 新颖点：从“更好的 codec”转向“compressed query processing”。
* 关联技术：**time-series database、compressed-domain query processing、RLE、dictionary、delta encoding、bit-packing**。
* 与 FastLanes：理念上相似，都服务于压缩数据上的高性能执行；但 CompressIoTDB 是 TSDB/IoTDB 专用，FastLanes 是列式 vectorized layout。

**针对的数据类型与能力**

* 数据类型：time-series data、IoT sensor data、industrial telemetry。
* 支持：lossless encoding 路径、query over compressed data、streaming TSDB 写入。
* SIMD/vectorization：不是核心。
* Random access：由 TsFile/IoTDB 存储结构支持。

**实验设置**

* 数据集：IoT-benchmark 和 5 个真实数据集。
* 真实场景：论文提到铁路系统每天约 3000 亿点、未压缩约 5TB，TsFile 可减少最多 95% 空间。
* 指标：throughput、memory、query latency、compression effect。

**对比 Baseline**

* Apache IoTDB 原始执行路径
* RLE / Dictionary / Ts_2Diff 等编码
* Gorilla 类组合编码作为背景。

**实验表现**

* 平均吞吐提升 53.4%，内存占用降低 20%。
* 优势：对系统查询路径价值大。
* 劣势：不一定提出通用浮点 codec；与 IoTDB/TsFile 绑定较深。

---

### 11. Beyond Compression: The Comprehensive Evaluation of Lossless Floating-Point Compression

**论文信息**

* 标题：*Beyond Compression: The Comprehensive Evaluation of Lossless Floating-Point Compression*
* 作者：Ronald T. H. van der Weerdt 等
* 年份：2025
* Venue：PVLDB 2025
* 论文链接：PVLDB/PDF。
* 代码或项目链接：论文中给出 Rust library / 代码链接。

**摘要翻译**

该论文系统评估 lossless floating-point compression，不只看压缩率和速度，还看压缩状态下的查询执行、in-situ queries，以及机器学习场景中的 kNN/RAG 类工作负载。论文指出没有一种方法在所有维度上最优，实际系统需要在压缩率、随机访问、查询执行、ML workload 和工程复杂度之间权衡。

**核心 Idea**

* 解决的问题：已有浮点压缩论文通常只报告压缩率/速度，缺少 end-to-end 查询与 ML 任务评估。
* 技术设计：统一 Rust library；覆盖数据库查询、time-series、ML embedding、RAG/kNN 等场景。
* 新颖点：从“压缩器 benchmark”扩展到“压缩后能否高效计算”。
* 关联技术：**lossless floating-point compression、in-situ query、ML embeddings、time-series、database workloads**。
* 与 FastLanes：非常适合评估 ALP/FastLanes-style 浮点列压缩在现代分析与 ML 场景中的真实价值。

**针对的数据类型与能力**

* 数据类型：time-series、ML embeddings、TPC-H numeric columns。
* 支持：取决于被测压缩器；论文关注 compressed-domain / in-situ task behavior。
* SIMD/vectorization：评估中涉及实现差异，但不是单一方法贡献。
* Query over compressed data：核心评估点之一。

**实验设置**

* 数据集：UCR Archive 2018 的 128 个数据集、Arxiv-Embed、Airbnb-Embed、TPC-H。
* 指标：compression ratio、compression/decompression speed、query time、ML task performance、随机访问/任务执行开销。

**对比 Baseline**

* Gorilla
* Buff
* Chimp
* Elf
* ALP
* 多种 floating-point compressors。

**实验表现**

* 论文结论是没有单一方法统治所有 workload；压缩率最高的方法不一定查询最快，适合时序的方法不一定适合 embedding。
* 优势：为 2025 年以后选型提供最系统基准。
* 劣势：它是评测论文，不提出新的核心 codec。

---

### 12. FastLanes: A Data-Parallel Encoded File Format

**论文信息**

* 标题：*FastLanes: A Data-Parallel Encoded File Format*
* 作者：Azim Afroozeh 等
* 年份：2025
* Venue：PVLDB 2025
* 论文链接：PVLDB/PDF。([VLDB][9])
* 代码或项目链接：论文中给出代码链接。([VLDB][9])

**摘要翻译**

该论文把 FastLanes 从 SIMD-friendly compression layout 推进到完整文件格式。它针对 Parquet 等格式中依赖 Snappy/Zstd 等通用压缩器的问题，提出一种 data-parallel encoded file format：使用 flexible expression encoding、multi-column compression、小 batch cache-friendly 布局，并提供 partial decompression 和 compressed query API。论文报告 FastLanes 在压缩率上优于 Parquet，并加速解压。([VLDB][9])

**核心 Idea**

* 解决的问题：Parquet/ORC 等文件格式对现代 SIMD/GPU 和 compressed execution 支持不足。
* 技术设计：以 data-parallel encoding 为文件格式核心，而不是把 generic byte compression 作为最后一步。
* 新颖点：从 codec/layout 扩展为 end-to-end file format。
* 关联技术：**SIMD/GPU-friendly compression、columnar file format、partial decompression、compressed query、multi-column compression**。
* 与 FastLanes 2023：2023 论文是 layout；2025 论文是 file format 和 query API。

**针对的数据类型与能力**

* 数据类型：columnar data、database columns、data lake 文件。
* 支持：lossless compression、SIMD/vectorized decompression、partial decompression、query processing over compressed data。
* Streaming：不是主目标。
* Random access：通过文件格式和小 batch 支持。

**实验设置**

* 数据集：列式分析数据与文件格式 benchmark。
* 对比：Parquet、BtrBlocks 等。
* 指标：compression ratio、decompression speed、query performance、portability。([VLDB][9])

**对比 Baseline**

* Parquet
* BtrBlocks
* 传统 heavyweight generic compression
* Lightweight cascading compression。([VLDB][9])

**实验表现**

* 论文摘要报告相较 Parquet 有更好的压缩率，并能加速解压。
* 优势：目前最直接的 FastLanes-style 系统论文。
* 劣势：对 time-series / floating-point 专用模式不是核心，但可与 ALP 等方法组合。

---

### 13. Serf: Streaming Error-Bounded Floating-Point Compression

**论文信息**

* 标题：*Serf: Streaming Error-Bounded Floating-Point Compression*
* 作者：Ruiyuan Li, Zechao Chen, Ruyun Lu, Xiaolong Xu, Guangchao Yang, Chao Chen, Jie Bao, Yu Zheng
* 年份：2025
* Venue：PACMMOD / SIGMOD 2025
* 论文链接：SIGMOD 2025 论文列表、DBLP、作者 PDF。([SIGMOD 2025][13])
* 代码或项目链接：GitHub 项目。([GitHub][14])

**摘要翻译**

Serf 研究 streaming 场景下的 error-bounded floating-point compression。已有方法要么是 batch lossy compression，会引入较长延迟；要么是 streaming lossless compression，当允许误差时压缩率不够理想。Serf 提出两个版本：Serf-Qt 使用量化和 Elias gamma coding；Serf-XOR 则把 XOR-based 浮点压缩扩展到 lossy/error-bounded 场景，通过数据偏移和近似值搜索增加 XOR 结果的 leading/trailing zeros。([Kangry][15])

**核心 Idea**

* 解决的问题：IoT/传感器流式传输中，既希望允许误差提升压缩率，又不能等待大批量 batch。
* 技术设计：Serf-Qt 量化浮点为整数；Serf-XOR 在误差界内寻找更易压缩的近似值。
* 新颖点：把 XOR-based streaming float compression 从 lossless 推向 error-bounded lossy。
* 关联技术：**streaming compression、lossy floating-point compression、time-series、XOR encoding、quantization、Elias gamma coding**。
* 与 FastLanes：关系较弱；Serf 是流式浮点时序算法，不是列式 SIMD layout。

**针对的数据类型与能力**

* 数据类型：floating-point time-series、sensor/IoT streaming data。
* 支持：lossy/error-bounded compression、streaming compression。
* Random access：不是核心。
* SIMD/vectorization：不是主贡献。
* Query over compressed data：主要面向传输/存储压缩。

**实验设置**

* 数据集：13 个不同领域的 time-series 数据集；每个数据集中随机抽取最多 100,000 条连续记录的设置在公开材料中可见。([CSDN][16])
* 指标：compression ratio、compression time、decompression time、传输原型系统表现。
* 实验系统：论文还构建了真实开发板上的 streaming transmission prototype。([Kangry][15])

**对比 Baseline**

* Batch lossless：LZ77、Zstd、ALP、Snappy
* Batch lossy：SZ2、Machete、Sim-Piece、SZ_ADT、Sprintz、HIRE、Buff
* Streaming lossless：Deflate、LZ4、FPC、Gorilla、Chimp128、Elf。([CSDN][16])

**实验表现**

* 论文摘要称 Serf-Qt 和 Serf-XOR 在 13 个数据集、17 个竞争方法上表现出较好的压缩率和高效率。([Kangry][15])
* 海报材料显示 Serf-XOR 在部分设置下相较 Serf-Qt 压缩率更好，并在传输场景中表现突出。([Kangry][17])
* 优势：2025 年最值得关注的 streaming lossy floating-point time-series compression。
* 劣势：error-bounded lossy，不适合 bit-exact 场景。

---

## 二、科学计算 / HPC 浮点有损压缩相关论文

这些论文和用户主题相关，但重点不在数据库列式或时序数据库，而在 **multi-dimensional scientific arrays**、GPU、error-bounded lossy compression。建议作为“浮点压缩大背景”阅读。

### 14. SZ3 / Modular Error-Bounded Lossy Compression Framework

* 方向：scientific data compression、error-bounded lossy compression。
* 核心：把 prediction-based lossy compressor 模块化，便于组合预测器、量化器、编码器等组件。
* 数据类型：floating-point scientific arrays、simulation data。
* 相关性：强相关于浮点压缩，但不是 time-series/database/FastLanes-style。
* 论文/项目页面显示 SZ3 是一个 modular framework for composing prediction-based error-bounded lossy compressors。([anl.gov][18])

### 15. cuSZp / cuSZ-i 系列 GPU Error-Bounded Lossy Compression

* 方向：GPU scientific lossy compression。
* 核心：面向 GPU 的 ultra-fast error-bounded lossy compression，支持 single/double floating-point arrays。
* 数据类型：scientific arrays、HPC simulation data。
* 相关性：与 floating-point compression 强相关；与 FastLanes 的共同点是硬件并行化，但一个偏 GPU scientific array，一个偏 CPU/SIMD columnar database。
* cuSZp 被标注为 SC 2023 工作，项目页说明其把 compression/decompression 融合在 CUDA kernel 中。([FZ][19])

### 16. HPEZ / QoZ 2.0

* 方向：high-ratio scientific lossy compression。
* 核心：通过新的 interpolation 和 quality-metric auto-tuning 改善压缩质量与速度。
* 数据类型：scientific floating-point datasets。
* 相关性：浮点有损压缩相关，但不是数据库/TSDB 场景。([arXiv][20])

### 17. MGARD 及其 2024 综述/框架论文

* 方向：multi-grid error-controlled lossy compression。
* 核心：将 scientific data 分解、截断精度、再做 lossless encoding；强调误差控制。
* 数据类型：multi-dimensional scientific data。
* 相关性：浮点压缩相关，但不直接处理 time-series database 或 FastLanes-style vectorized columnar compression。([arXiv][21])

### 18. Error-Bounded Lossy Compression Survey

* 标题：*A Survey on Error-Bounded Lossy Compression for Scientific Datasets*
* 年份：2024 arXiv
* 价值：适合作为 SZ/ZFP/MGARD/CPU-GPU scientific compression 的总入口。
* 摘要显示该 survey 总结 6 类 compression models、10+ 组件、10+ SOTA compressors 和 10+ 应用。([arXiv][22])

---

## 三、边缘相关论文 / 系统

### A. Patas

* Patas 很重要，但我没有找到独立正式论文；它更像 DuckDB 中的实现/PR 和后续论文中的 baseline。
* ALP 论文 related work 把 Patas 作为 Chimp 的变体引用，DuckDB 文档也列出 Chimp、Patas、ALP 作为支持的浮点压缩方法。([ir.cwi.nl][4])
* 相关性：强相关于 floating-point compression baseline，但不是单独学术论文。

### B. Mix-Piece

* 标题：*Flexible Grouping of Linear Segments for Highly Accurate Lossy Compression of Time Series Data*
* 方向：Sim-Piece 的扩展/后续，面向 time-series lossy PLA。
* 公开摘要称其优于 Sim-Piece 和先前 PLA 方法，平均压缩率相对 PLA 超过 2 倍。([Springer][23])
* 相关性：时序有损压缩相关；与浮点 bit-level / FastLanes 关系弱。

### C. NPAC / Adaptive Encoding Strategies 等 IEEE IoT 类工作

* 这些工作常关注 IoT 浮点序列压缩，但部分 venue 不属于用户指定重点的 CCF A 数据库/系统顶会。
* 可作为补充阅读，不建议放在主线。([Springer][24])

---

## 四、总览表

|        年份 | 论文                                             | 方向                                     | 是否 lossy | SIMD / vectorization | 数据类型                          | 主要 baseline                                 | 主要贡献                                              | 实验表现简述                                  |
| --------: | ---------------------------------------------- | -------------------------------------- | -------- | -------------------- | ----------------------------- | ------------------------------------------- | ------------------------------------------------- | --------------------------------------- |
|      2022 | Chimp                                          | 浮点时序 lossless                          | 否        | 否                    | floating-point time-series    | Gorilla, FPC, LZ4/Zstd/Snappy               | 改进 XOR-based streaming float compression          | 平均空间约为此前 SOTA streaming 方法一半；速度保持较好     |
|      2022 | Frequency Domain Data Encoding in Apache IoTDB | IoTDB 频域/时序压缩                          | 是        | 否                    | frequency-domain TS, IoT data | Gorilla, Buff, GZIP/LZ4/Snappy              | Descend：量化 + descending bit-packing               | 多数数据集压缩率高；Gorilla 解码更快                  |
|      2023 | FastLanes Compression Layout                   | SIMD-friendly 列式压缩布局                   | 否        | 是                    | database columns              | Parquet/ORC style codecs                    | Unified Transposed Layout + virtual 1024-bit SIMD | 多种 lightweight codec 超过 40 values/cycle |
|      2023 | Elf                                            | 浮点 lossless                            | 否        | 否                    | time-series + general FP      | Gorilla, FPC, Chimp, LZ4/Zstd/Snappy        | erasing trailing bits while remaining lossless    | 相比 Chimp/Chimp128 压缩率提升，但速度不是最强         |
|      2023 | Sim-Piece                                      | 时序 lossy PLA                           | 是        | 否                    | time-series                   | PMC-MR, Swing, Slide, Mixed                 | 相似线段合并                                            | 平均压缩率较传统 PLA 提升超过 2×                    |
|      2023 | BtrBlocks                                      | data lake 列式压缩                         | 否        | 部分/非核心               | columnar data                 | Parquet, ORC, generic compression           | 多 lightweight encoding 组合                         | 兼顾压缩率和解压吞吐                              |
|      2024 | LeCo                                           | learned lightweight column compression | 否        | 否                    | numeric columns               | FOR, Delta, RLE                             | 学习 serial correlation                             | Arrow 查询 5.2× 加速，RocksDB 吞吐约 +16%       |
|      2024 | ALP                                            | 数据库浮点 lossless                         | 否        | 是                    | double columns                | Chimp, Elf, Patas, Zstd, Snappy             | decimal-like double 转 integer + FastLanes         | 在压缩率、速度、查询友好性上综合强                       |
|      2024 | FCBench                                        | lossless FP benchmark                  | 视方法      | 覆盖 CPU/GPU           | HPC, TS, DB, observation      | 多 CPU/GPU compressors                       | 跨领域统一评测                                           | 无单一最优；揭示数据域和硬件 trade-off                |
|      2025 | CompressIoTDB                                  | TSDB compressed query                  | 否/编码层    | 否                    | IoT time-series               | IoTDB 原路径, RLE, Dict, Ts_2Diff              | 压缩域过滤/聚合/窗口                                       | 平均吞吐 +53.4%，内存 -20%                     |
|      2025 | Beyond Compression                             | lossless FP evaluation                 | 否        | 视方法                  | TS, embeddings, TPC-H         | Gorilla, Buff, Chimp, Elf, ALP              | 评估 in-situ query / ML workload                    | 无单一赢家；任务相关性强                            |
|      2025 | FastLanes File Format                          | data-parallel encoded file format      | 否        | 是                    | columnar data                 | Parquet, BtrBlocks                          | 文件格式级 compressed execution                        | 压缩率优于 Parquet，并加速解压                     |
|      2025 | Serf                                           | streaming error-bounded FP TS          | 是        | 否                    | streaming FP time-series      | ALP, SZ2, Sim-Piece, Gorilla, Chimp128, Elf | Serf-Qt + Serf-XOR                                | 13 数据集、17 baseline 上表现强                 |
| 2023–2024 | cuSZp / cuSZ-i                                 | GPU scientific lossy                   | 是        | GPU 并行               | scientific FP arrays          | SZ/ZFP/GPU compressors                      | GPU error-bounded compression                     | 极高吞吐，适合 HPC                             |
|      2024 | SZ3 / HPEZ / MGARD survey                      | scientific lossy                       | 是        | 部分支持                 | scientific arrays             | SZ, ZFP, MGARD                              | 模块化/插值/多重网格误差控制                                   | 更偏 HPC，不是 TSDB/columnar 主线              |

---

## 五、2022 年以来的主要技术趋势

**1. 从“压缩率优先”转向“压缩 + 查询执行 + 硬件友好”**

早期 Gorilla/Chimp/Elf 主要关注 float/time-series 的压缩率和流式速度；FastLanes、ALP、CompressIoTDB、Beyond Compression 则明显转向 compressed execution、vectorized decompression、predicate pushdown、in-situ query。换句话说，数据库系统里“能不能快查”越来越重要，而不是只看 compressed size。([VLDB][2])

**2. 浮点压缩分成两条路线**

一条是 **bit-level/XOR streaming 路线**：Gorilla → Chimp → Elf → Serf，适合 time-series、IoT、telemetry、流式传输。另一条是 **columnar/vectorized database 路线**：BtrBlocks → FastLanes → ALP → FastLanes File Format，适合分析数据库、data lake、OLAP。

**3. Lossy compression 在时序和科学计算里更活跃**

Sim-Piece、Serf、SZ3、cuSZp、MGARD/HPEZ 都说明：只要业务允许误差界限，压缩率和传输/存储效率可以显著提升。但数据库列压缩和金融/精确科学数据仍大量需要 lossless。([VLDB][5])

**4. SIMD / vectorization 成为列式压缩的核心竞争点**

FastLanes 的关键贡献不是单个 codec，而是把数据布局改成编译器和 SIMD 友好的形态。ALP 继承这一方向，把 floating-point compression 也接入 vectorized lightweight compression。([VLDB][2])

---

## 六、FastLanes 与这些工作的关系

FastLanes 可以理解为这一领域的“系统化分水岭”：

* 对 Gorilla/Chimp/Elf/Serf：FastLanes 不直接竞争流式时序 codec；这些方法优化的是相邻浮点值之间的 bit-level redundancy。
* 对 ALP：ALP 是 FastLanes-style 思想在 **double column compression** 上最直接的延伸，尤其是把 double 转成 integer 后接入 FastLanes 的 bit-packing/FOR 路径。
* 对 BtrBlocks/Parquet/ORC：FastLanes 的目标是证明列式文件格式不应只依赖 generic byte compression，而应在编码布局层面天然支持 data parallelism。
* 对 CompressIoTDB：两者都关注 compressed data 上的执行，但一个偏 OLAP columnar/vectorized file format，一个偏 TSDB compressed query operator。

最简洁地说：**FastLanes 是“compression layout / file format / vectorized execution”路线；Chimp/Elf/Serf 是“floating-point time-series codec”路线；SZ/ZFP/MGARD/cuSZp 是“scientific error-bounded lossy array compression”路线。**

---

## 七、最值得深入阅读的论文

1. **ALP 2024**：如果你关心 floating-point compression + database + FastLanes，这是第一优先级。
2. **FastLanes 2023 + FastLanes File Format 2025**：理解 SIMD-friendly compression layout 和文件格式演进的核心。
3. **Chimp 2022 + Elf 2023 + Serf 2025**：构成 float time-series compression 的主线，分别代表 lossless streaming、erasing-based lossless、error-bounded streaming lossy。
4. **Beyond Compression 2025 + FCBench 2024**：做选型、复现实验、设计 benchmark 时非常重要。
5. **CompressIoTDB 2025**：如果研究目标是 TSDB 查询性能，而不是单独 codec，非常值得读。
6. **Sim-Piece / Mix-Piece**：如果关注 lossy time-series approximation，这条线值得读；否则可作为补充。

---

## 八、仍然开放的研究问题

**1. 能否统一 streaming time-series codec 与 vectorized columnar codec？**
Chimp/Elf/Serf 擅长流式时序，FastLanes/ALP 擅长列式向量化。一个开放问题是：能否设计同时支持 streaming ingest、SIMD decompression、random access 和 compressed query 的统一浮点压缩格式。

**2. Lossless 与 error-bounded lossy 能否自适应切换？**
现实系统中，有些列必须 lossless，有些 telemetry 可以允许误差。未来系统可能需要 per-column、per-segment、per-query 的 error policy，而不是固定 codec。

**3. Query over compressed floating-point data 仍不成熟**
CompressIoTDB 和 FastLanes File Format 已经往 compressed execution 前进，但对复杂谓词、join、window、UDF、ML embedding search 等场景，压缩表示上的执行还远未统一。

**4. 浮点语义仍然难处理**
ALP 利用 decimal-like double 是很强的 insight，但并非所有浮点数据都有 decimal origin。科学计算数据、embedding、噪声传感器数据的分布差异巨大，仍需要更细的数据画像和 codec selection。

**5. Benchmark 需要跨域统一**
FCBench 和 Beyond Compression 都指出没有单一赢家。未来 benchmark 需要同时覆盖 DB columns、time-series、scientific arrays、embeddings、streaming transmission、compressed query，而不是只报告 compression ratio。

**6. GPU/SIMD/异构硬件下的统一抽象仍缺失**
FastLanes 偏 CPU SIMD 和可移植 scalar auto-vectorization，cuSZp 偏 GPU scientific compression。如何为 CPU SIMD、GPU、TPU/NPU 设计统一 compressed layout，是一个很有潜力的系统研究方向。

[1]: https://www.vldb.org/pvldb/vol16/p282-song.pdf "Frequency Domain Data Encoding in Apache IoTDB"
[2]: https://www.vldb.org/pvldb/vol16/p2132-afroozeh.pdf "Untitled"
[3]: https://github.com/cwida/FastLanes?utm_source=chatgpt.com "GitHub - cwida/FastLanes: Next-Gen Big Data File Format"
[4]: https://ir.cwi.nl/pub/33334/33334.pdf "ALP: Adaptive Lossless floating-Point Compression"
[5]: https://www.vldb.org/pvldb/vol16/p1910-liakos.pdf "Sim-Piece: Highly Accurate Piecewise Linear Approximation through Similar Segment Merging"
[6]: https://dlnext.acm.org/doi/pdf/10.1145/3589263?utm_source=chatgpt.com "BtrBlocks: Efficient Columnar Compression for Data Lakes"
[7]: https://github.com/maxi-k/btrblocks?utm_source=chatgpt.com "BtrBlocks - Efficient Columnar Compression for Data Lakes"
[8]: https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/btrblocks.pdf?utm_source=chatgpt.com "BtrBlocks: Efficient Columnar Compression for Data Lakes"
[9]: https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf "Untitled"
[10]: https://arxiv.org/abs/2306.15374?utm_source=chatgpt.com "LeCo: Lightweight Compression via Learning Serial Correlations"
[11]: https://github.com/yhliu918/Learn-to-Compress?utm_source=chatgpt.com "Leco: Learn-to-Compress - GitHub"
[12]: https://github.com/cwida/ALP?utm_source=chatgpt.com "ALP: Adaptive Lossless Floating-Point Compression - GitHub"
[13]: https://2025.sigmod.org/sigmod_papers.shtml?utm_source=chatgpt.com "The 2025 ACM SIGMOD/PODS Conference: Berlin, Germany - 2025.sigmod.org"
[14]: https://github.com/Spatio-Temporal-Lab/Serf?utm_source=chatgpt.com "GitHub - Spatio-Temporal-Lab/Serf"
[15]: https://kangry.net/paper/Serf_SIGMOD2025.pdf?utm_source=chatgpt.com "Serf: Streaming Error-Bounded Floating-Point Compression"
[16]: https://blog.csdn.net/kangrydotnet/article/details/147052428?utm_source=chatgpt.com "SIGMOD 2025 | Serf: 流式误差有界浮点压缩（附论文和源码 ..."
[17]: https://kangry.net/paper/Serf_poster_SIGMOD2025.pdf?utm_source=chatgpt.com "Serf: Streaming Error-Bounded Floating-Point Compression @SIGMOD 2025"
[18]: https://www.anl.gov/argonne-scientific-publications/pub/173407?utm_source=chatgpt.com "SZ3: A Modular Framework for Composing Prediction-Based Error-Bounded ..."
[19]: https://fzframework.org/blog/2023/11/25/paper-cuszp-an-ultra-fast-gpu-error-bounded-lossy-compression-framework-with-optimized-end-to-end-performance/?utm_source=chatgpt.com "Paper: cuSZp: An Ultra-fast GPU Error-bounded Lossy Compression ..."
[20]: https://arxiv.org/html/2311.12133v3?utm_source=chatgpt.com "High-performance Effective Scientific Error-bounded Lossy Compression ..."
[21]: https://arxiv.org/html/2401.05994?utm_source=chatgpt.com "MGARD: A multigrid framework for high-performance, error-controlled ..."
[22]: https://arxiv.org/abs/2404.02840?utm_source=chatgpt.com "A Survey on Error-Bounded Lossy Compression for Scientific Datasets"
[23]: https://link.springer.com/article/10.1007/s00778-024-00862-z?utm_source=chatgpt.com "Flexible grouping of linear segments for highly accurate lossy ..."
[24]: https://link.springer.com/article/10.1007/s11280-025-01382-8?utm_source=chatgpt.com "NPAC: numeric pattern aware compression algorithm for ... - Springer"
下面是继续查漏后的补充结果。结论先说：上一版确实漏掉了几篇应进入主线的论文，尤其是 **Camel、DeXOR、NeaTS、G-ALP、Accelerating GPU Data Processing using FastLanes Compression、HPEZ/QoZ 2.0、cuSZ-i、ASPLOS 2025 CPU/GPU lossless FP compression**。另外，**F3 / Vortex / Bullion / Functional Decomposition of Storage Formats** 更像“列式格式与压缩生态”相关工作，建议放入边缘相关或系统背景部分。

---

## A. 应补入“核心论文”的遗漏项

### 1. Camel: Efficient Compression of Floating-Point Time Series

这是上一版最大的遗漏之一。Camel 发表在 **Proc. ACM Manag. Data Vol. 2, Issue 6 / SIGMOD 2025 轮次**，DBLP 记录为 Yuanyuan Yao, Lu Chen, Ziquan Fang, Yunjun Gao, Christian S. Jensen, Tianyi Li。它专门面向 **floating-point time series compression**，和 Chimp、Elf、Serf、DeXOR 属于同一主线。([DBLP][1])

**核心 Idea：**

* 不再把 IEEE double 的整数部分与小数部分整体一起 XOR，而是分别处理 integer part 和 decimal part。
* 不固定使用前一个值做 XOR reference，而是寻找更有利于压缩的 reference value。
* 同时提供 compressed data indexing，用于在压缩数据上支持更高效查询。
* 实验比较了 **11 个 lossless 方法、6 个 lossy 方法、22 个公开数据集和 3 个 AliCloud 工业数据集**；ACM 页面摘要称其在压缩率和效率上都能超过已有方法，并且对 time-series 与 non-time-series 数据都有较好表现。([ACM Digital Library][2])
* 代码已公开，GitHub 仓库显示包含 Camel compressor/decompressor 以及多种 XOR-based baseline 实现。([GitHub][3])

**建议归类：**
主线核心论文；放在 **Elf 之后、Serf / DeXOR 之前**。它是 **lossless floating-point time-series compression** 的关键新增节点。

---

### 2. DeXOR: Enabling XOR in Decimal Space for Streaming Lossless Compression of Floating-point Data

DeXOR 是 **2026 PVLDB / arXiv 2026** 工作，当前已经能检索到 PVLDB Vol.19 PDF 和 arXiv 条目。作者包括 Chuanyi Lv, Huan Li, Dingyu Yang, Zhongle Xie, Lu Chen, Christian S. Jensen。([arXiv][4])

**核心 Idea：**

* 目标是 streaming lossless compression of floating-point data。
* 关键思想是把 XOR 操作搬到 **decimal space**，编码 decimal-space 的 longest common prefixes/suffixes。
* 它试图同时利用相邻值的 smoothness 和浮点数 decimal redundancy，避免 Elf / Camel 类方法在格式转换和连续平滑性利用之间的干扰。
* 技术上包括 scaled truncation、error-tolerant rounding、decimal XOR 的 bit management，以及针对 exponent 的 exception handler。
* arXiv 摘要称其在 22 个数据集上相较 SOTA 方法压缩率高 15%，解压速度快 20%，压缩速度保持竞争力。([arXiv][4])
* 代码和数据也已公开在 GitHub。([GitHub][5])

**建议归类：**
主线核心论文；放在 **Camel 之后**。这是 2026 年目前最值得关注的 **streaming lossless FP time-series compression** 新工作。

---

### 3. NeaTS: Learned Compression of Nonlinear Time Series with Random Access

NeaTS 是 **ICDE 2025** 论文，DBLP 记录为 Andrea Guerra, Giorgio Vinciguerra, Antonio Boffa, Paolo Ferragina, ICDE 2025: 1579–1592。它不是纯浮点 bit-level codec，而是 **time-series compression + random access + learned/nonlinear approximation** 方向的重要补充。([DBLP][6])

**核心 Idea：**

* 用一组不同形状的 nonlinear functions 近似 time series，并通过 partitioning algorithm 选择最省空间的分段方式。
* residuals 是 bounded 的：保留 residual 可实现 lossless reconstruction；丢弃 residual 可得到 error-bounded lossy representation。
* 明确支持 random access，这是很多 Gorilla/Chimp 类 streaming codec 的短板。
* arXiv 摘要称，其相较使用线性/非线性函数的 SOTA lossy compressors 压缩率最多提升 14%；相较 lossless compressors，它同时提供接近或更优的压缩率、更快解压和数量级更高效的 random access。([arXiv][7])
* 代码已公开。([GitHub][8])

**建议归类：**
主线补充论文；放在 **time-series compression / random access** 小节，而不是 floating-point bit-level 小节。它和 Sim-Piece/Mix-Piece 更接近，但比传统 PLA 更强调随机访问和 lossless/lossy 双模式。

---

### 4. G-ALP: Rethinking Light-weight Encodings for GPUs

这是 FastLanes/ALP 线上的重要遗漏。G-ALP 是 **DaMoN 2025** 工作，DBLP 记录作者为 Sven Hepkema, Azim Afroozeh, Charlotte Felius, Peter Boncz, Stefan Manegold。它是 **ALP 的 GPU-optimized 版本**，直接属于 FastLanes-style lightweight compression 生态。([DBLP][9])

**核心 Idea：**

* 将 ALP 的 floating-point lightweight compression 迁移到 GPU-friendly / fully data-parallel decoding。
* 重点优化 exception patching，因为 ALP 中 exception 通常只占小比例，但在 GPU 上会带来 warp divergence 和非规则访存问题。
* 面向 OLAP workload，强调 high decompression throughput、fine-grained access、kernel-agnostic decoding API。
* CWI 摘要明确称 G-ALP 是 ALP 的 GPU-optimized version，目标是 floating-point compression。([CWI][10])
* GitHub 仓库显示它是 FastLanes project 的一部分。([GitHub][11])

**建议归类：**
FastLanes 直接相关核心论文；放在 **ALP 之后、FastLanes File Format 之前/之后均可**。虽然 DaMoN 是 workshop，不是 CCF A 主会，但因为它是 FastLanes/ALP 作者线的直接延伸，应该保留。

---

### 5. Accelerating GPU Data Processing using FastLanes Compression

这篇是 **DaMoN 2024**，作者 Azim Afroozeh, Lotte Felius, Peter Boncz。它比 G-ALP 更早，把 FastLanes 的 data-parallel bit-packing / encoding 放到 GPU query processing 中评估。([CWI][12])

**核心 Idea：**

* 论证 compression 对 GPU data processing 可以是 win-win：既能让 GPU global memory 放下更多数据，也能加快查询处理。
* 将 FastLanes 的 fully data-parallel bit-packing 与 encodings 微基准测试在 NVIDIA T4 / V100 上，并集成到 Crystal GPU query processing prototype。
* 与 2023 FastLanes 论文关系：2023 是 CPU/SIMD-friendly layout；这篇是 GPU query processing 扩展。([CWI][12])

**建议归类：**
FastLanes 直接相关；主表中可作为 **FastLanes GPU extension**，但由于是 workshop，可标注“重要系统扩展，非主会”。

---

### 6. HPEZ / QoZ 2.0: High-performance Effective Scientific Error-bounded Lossy Compression with Auto-tuned Multi-component Interpolation

上一版只把它作为科学计算背景一笔带过，建议提升为“科学计算浮点有损压缩核心补充”。它发表于 **Proc. ACM Manag. Data 2024**，CV/项目页显示标题为 *High-performance Effective Scientific Error-bounded Lossy Compression with Auto-tuned Multi-component Interpolation*。([Jinyang's Homepage][13])

**核心 Idea：**

* 面向 scientific floating-point datasets 的 error-bounded lossy compression。
* 通过 multi-component interpolation 和 quality-metric-driven auto-tuning，在压缩质量、压缩率、速度之间取得更好平衡。
* arXiv 页面称 HPEZ / QoZ 2.0 在压缩质量上明显优于已有 high-performance compressors，同时比 high-ratio compressors 快很多。([arXiv][14])

**建议归类：**
科学计算 floating-point lossy compression 核心论文；和 SZ3、ZFP、MGARD、cuSZ-i 放一起。

---

### 7. cuSZ-i: High-Ratio Scientific Lossy Compression on GPUs with Optimized Multi-Level Interpolation

上一版提到了 cuSZp，但漏掉了更近的 cuSZ-i。cuSZ-i 是 **SC 2024** 论文，属于 HPC/科学数据压缩方向的 CCF A 级别主线。([阿贡领导计算设施][15])

**核心 Idea：**

* 面向 GPU 的 error-bounded scientific lossy compression。
* 用 GPU-optimized multi-level interpolation 改善压缩率和重建质量。
* 优化 Huffman encoding，并集成 NVIDIA Bitcomp-lossless 作为进一步提升压缩率的模块。
* IEEE/SC 页面摘要称其在相同 error bound 下，相比第二好的 GPU-based lossy compressor 压缩率优势达 476%。([IEEE Computer Society][16])

**建议归类：**
科学计算 / GPU floating-point lossy compression 核心论文。它不属于 TSDB/FastLanes，但在“floating-point compression”大综述里不能漏。

---

### 8. Efficient Lossless Compression of Scientific Floating-Point Data on CPUs and GPUs

这是 **ASPLOS 2025** 论文，作者 Noushin Azami, Alex Fallin, Martin Burtscher。它不是 database paper，但 ASPLOS 是系统/体系结构顶会，且主题是 **scientific floating-point data lossless compression**，应纳入。([userweb.cs.txstate.edu][17])

**核心 Idea：**

* 提出 4 个新的 lossless compression algorithms，用于 single-precision 和 double-precision scientific floating-point data。
* 强调 CPU 与 GPU 兼容性，以及不能让 compression 成为新的 I/O/transfer bottleneck。
* 适合放在 ndzip-gpu、FCBench、cuSZ-i 附近，作为 HPC/scientific lossless FP compression 线索。([ACM Digital Library][18])

**建议归类：**
科学计算 lossless floating-point compression 核心补充；和数据库浮点列压缩不同，它更关注 scientific arrays / CPU-GPU 数据传输。

---

### 9. Everything You Always Wanted to Know About Storage Compressibility of Pre-Trained ML Models but Were Afraid to Ask

这是 **PVLDB 2024**，上一版没有纳入。它不是时序，也不是数据库列压缩，但它提出了面向 **pre-trained model floating-point parameters** 的 error-bounded lossy floating-point compression 方法 Elf / Elves，值得放入“边缘但重要的浮点压缩”部分。

**核心 Idea：**

* 研究 PTM / Hugging Face 模型文件的 storage compressibility。
* 发现传统 dedup、similarity detection、dictionary compression 对 PTM 文件效果有限。
* 提出 Exponent-Less Floating-point encoding，即把大量落在 (-1, 1) 的模型参数映射到 [1, 2)，从而去掉 common exponent field。
* 论文报告 Elves 总体压缩率 1.52×，分别比 zstd、SZ3、uniform quantization 高 1.31×、1.32×、1.29×，且模型精度损失可忽略。

**建议归类：**
边缘相关但重要。它和 Elf: Erasing-based Lossless Floating-point Compression 名字相撞，但不是同一篇，也不是同一种 Elf；需要在综述中明确区分。

---

## B. 应加入“FastLanes / 列式格式生态”的相关论文或系统

### 10. F3: The Open-Source Data File Format for the Future

F3 是 **Proc. ACM Manag. Data 2025 / SIGMOD 2026 轮次** 文件格式论文。它不是 FastLanes 论文，但和 FastLanes File Format、BtrBlocks、Vortex 属于同一代“后 Parquet”列式文件格式讨论。([DBLP][19])

**为什么相关：**

* F3 关注 metadata、physical grouping layout、encoding-agnostic API。
* 它明确提到支持 cascading compression 和 vectorized decoding，并把编码方法作为 plug-in 通过 Wasm 嵌入到文件中。([清华大学交叉信息研究院][20])
* 它的主贡献不是新 codec，而是让文件格式可扩展、可互操作、可承载未来压缩/编码方法。

**建议归类：**
边缘相关 / 文件格式生态；和 FastLanes File Format、BtrBlocks 并列比较。

---

### 11. Vortex

Vortex 目前更像开源系统/格式，而不是单篇顶会论文，但它与 BtrBlocks / FastLanes / ALP / FSST 的关系非常紧密。官方仓库称其为 next-generation columnar file format and toolkit，支持 compressed Apache Arrow arrays、pluggable encoding、compression strategy、layout strategy。([GitHub][21])

**为什么相关：**

* Vortex 文档/生态强调 cascading compression、late decompression、compute on compressed data。
* 相关博客称其 cascading compression 框架受 BtrBlocks 启发，并在 TPC-H SF10 上相对 Parquet+ZSTD 文件小 38%、解压快 10–25×。([Spiral][22])
* DuckDB 也将 Vortex 描述为关注 lightweight compression encodings、late decompression、compressed compute 的 Parquet 替代方向。([DuckDB][23])

**建议归类：**
不要当作学术论文主条目；放入“系统/工业生态补充”，用于理解 FastLanes-style ideas 的落地趋势。

---

### 12. Towards Functional Decomposition of Storage Formats

这是 **CIDR 2025** 论文，作者包括 Martin Prammer, Xinyu Zeng, Ruijun Meng, Wes McKinney, Huanchen Zhang, Andrew Pavlo, Jignesh Patel。它不是压缩算法，但直接讨论列式格式中 **compression block size 与 search acceleration metadata block size 的冲突**。([VLDB][24])

**为什么相关：**

* 论文主张把 storage layer 和 search acceleration layer 分离。
* 动机是 row-skipping metadata 不应被迫使用和 compression blocks 相同的 partition size。
* 这对 FastLanes File Format / F3 / Parquet / ORC / BtrBlocks 的设计讨论有背景价值。([VLDB][24])

**建议归类：**
边缘相关；放在“文件格式与压缩粒度 trade-off”部分。

---

### 13. Bullion: A Column Store for Machine Learning

Bullion 是 **CIDR 2025**，面向 ML workloads 的 column store。它不是传统数据库列压缩论文，但涉及 optimized encoding scheme for sparse features、feature quantization in storage、cascading encoding framework。([VLDB][25])

**建议归类：**
边缘相关；适合放在“columnar storage for AI / ML data”部分。它与 floating-point compression 的联系主要在 ML feature / embedding / quantization，而不是 time-series floating-point codec。

---

## C. 应放入“边缘相关 / 不建议主线展开”的论文

### 14. NULLS!: Revisiting Null Representation in Modern Columnar Formats

DaMoN 2024 workshop。它讨论 Parquet/ORC 等格式中 NULL representation 对 compression ratio、decoding speed、data distribution、SIMD ISA 的影响，并提出 SmartNull。([清华大学交叉信息研究院][26])

**建议：**
只作为列式格式压缩细节补充，不进主线。

---

### 15. Floating-Point Data Transformation for Lossless Compression

arXiv 2025，提出 Typed Data Transformation，把相关 byte 分组以帮助 generic compressors 更好利用 floating-point 表示中的相关性。([arXiv][27])

**建议：**
可放入“arXiv / generic compressor preprocessing”部分。它和 FastLanes/ALP 有思想上的邻近性，但目前缺少顶会验证。

---

### 16. Falcon / GPU-Based Floating-point Adaptive Lossless Compression

这是 2025 arXiv / GitHub 项目，面向 GPU-accelerated lossless compression of floating-point time-series data。GitHub 描述其三点创新为 asynchronous pipeline、precise float-to-integer conversion、adaptive sparse bit-plane encoding。([arXiv][28])

**建议：**
由于目前主要是 arXiv/代码项目，放在“arXiv 预印本、待观察”部分。若后续进入 SIGMOD/VLDB/ICDE/FAST/SC，应提升到核心。

---

### 17. AFC / ACTF / NPAC 等普通期刊或非重点会议论文

检索到若干 2023–2025 的浮点时序压缩论文，如 AFC、ACTF、NPAC。AFC 对比 Gorilla、FPC、TSXor、Chimp，并称压缩率至少提升 20%；NPAC 面向 floating-point time-series 的 numeric pattern aware compression。([科学直达][29])

**建议：**
因为用户要求聚焦 CCF A 和高质量 arXiv，这类论文可以列入“边缘相关 / 暂不主读”，除非后续需要做非常完整的算法谱系。

---

## D. 需要从上一版“修正定位”的条目

### Patas

仍然建议把 Patas 作为 **baseline / system implementation**，不要当作独立论文主条目。它在 ALP、FCBench、Beyond Compression 等论文中频繁作为 baseline 出现，但我这轮仍未确认到独立顶会论文。

### TSXor

TSXor 很重要，但它是 **2021 SPIRE**，不在 2022 年以来范围内。它应该放入“历史 baseline”而不是主表。TSXor 的核心是用 window/cache 选择近邻 reference，论文称相较当时 SOTA 可达最高 3× 更好压缩和 2× 更快解压。([jermp.github.io][30])

### Buff

Buff 也应作为历史 baseline/related work，不应被写成 2022 后主线新论文。它在 Elf、ALP、FCBench、PTM compressibility 等论文里都作为重要对照出现，但不属于这次“2022 年至今新增论文”的核心列表。

---

## E. 更新后的“补充总览表”

|        年份 | 论文 / 系统                                                               | 应否补入主线         | 方向                                           | Lossy              | SIMD/GPU/vectorization | 数据类型                      | 备注                                      |
| --------: | --------------------------------------------------------------------- | -------------- | -------------------------------------------- | ------------------ | ---------------------- | ------------------------- | --------------------------------------- |
| 2024/2025 | Camel                                                                 | 是              | floating-point time-series compression       | 否                  | 不主打 SIMD               | FP time-series, non-TS FP | Chimp/Elf 后的重要 lossless 方法              |
| 2025/2026 | DeXOR                                                                 | 是              | streaming lossless FP compression            | 否                  | 不主打 SIMD               | streaming FP data         | decimal-space XOR，PVLDB 2026/arXiv 2026 |
|      2025 | NeaTS                                                                 | 是              | time-series compression + random access      | 可 lossless / lossy | 否                      | time-series               | ICDE 2025，非线性函数近似 + residual            |
|      2024 | Accelerating GPU Data Processing using FastLanes Compression          | 是，FastLanes 补充 | FastLanes on GPU                             | 否                  | GPU                    | columnar data             | DaMoN 2024，FastLanes GPU extension      |
|      2025 | G-ALP                                                                 | 是，FastLanes 补充 | GPU ALP / lightweight FP encoding            | 否                  | GPU                    | FP database columns       | DaMoN 2025，ALP 的 GPU 化                  |
|      2024 | HPEZ / QoZ 2.0                                                        | 是，科学计算线        | scientific lossy compression                 | 是                  | 部分并行                   | scientific FP arrays      | PACMMOD 2024，auto-tuned interpolation   |
|      2024 | cuSZ-i                                                                | 是，科学计算线        | GPU scientific lossy compression             | 是                  | GPU                    | scientific FP arrays      | SC 2024，multi-level interpolation       |
|      2025 | Efficient Lossless Compression of Scientific FP Data on CPUs and GPUs | 是，科学计算线        | scientific FP lossless                       | 否                  | CPU/GPU                | scientific FP data        | ASPLOS 2025                             |
|      2024 | PTM Compressibility / Elves                                           | 边缘但重要          | ML model FP parameter compression            | 是，error-bounded    | 并行友好                   | PTM float parameters      | PVLDB 2024，不是 TSDB/columnar             |
|      2025 | F3                                                                    | 边缘相关           | next-gen file format                         | 取决于 codec          | 支持 vectorized decoding | columnar files            | 文件格式，不是 codec                           |
|      2025 | Functional Decomposition of Storage Formats                           | 边缘相关           | storage format / compression block trade-off | 否                  | 间接相关                   | columnar files            | CIDR 2025                               |
|      2025 | Bullion                                                               | 边缘相关           | ML column store encoding                     | 包含 quantization    | 部分相关                   | sparse features, ML data  | CIDR 2025                               |
|      2025 | Falcon                                                                | arXiv 待观察      | GPU FP time-series lossless                  | 否                  | GPU                    | FP time-series            | 若进入主会应提升                                |
|      2025 | Floating-Point Data Transformation                                    | arXiv 待观察      | generic FP preprocessing                     | 否                  | CPU/GPU eval           | FP data                   | 辅助 generic compressors                  |

---

## F. 更新后的阅读优先级

第一优先级应该调整为：

1. **FastLanes 2023 + ALP 2024 + FastLanes File Format 2025 + G-ALP 2025**
   这是 FastLanes-style / vectorized lightweight compression 主线。

2. **Chimp 2022 + Elf 2023 + Camel 2025 + DeXOR 2026**
   这是 lossless floating-point time-series compression 主线。

3. **Serf 2025 + NeaTS 2025 + Sim-Piece / Mix-Piece**
   这是 time-series lossy / error-bounded / random access 主线。

4. **FCBench 2024 + Beyond Compression 2025**
   这是 benchmark / evaluation 主线，用来判断各 codec 的真实 trade-off。

5. **HPEZ/QoZ 2.0 + cuSZ-i + ASPLOS 2025 CPU/GPU lossless FP compression**
   这是 scientific floating-point compression 主线。

6. **BtrBlocks + F3 + Vortex + Functional Decomposition of Storage Formats**
   这是列式文件格式、cascading compression、compressed execution 生态主线。

---

## G. 最重要的查漏结论

上一版如果要修订，建议新增一个独立小节：

**“2024–2026 新一代 floating-point time-series compression”**

其中包含：

* Chimp 2022
* Elf 2023
* Camel 2025
* Serf 2025
* DeXOR 2026
* NeaTS 2025，作为 random access / learned nonlinear compression 分支

另一个新增小节：

**“FastLanes-style on GPU / post-Parquet file formats”**

其中包含：

* FastLanes 2023
* ALP 2024
* Accelerating GPU Data Processing using FastLanes Compression 2024
* FastLanes File Format 2025
* G-ALP 2025
* F3 2025/2026
* Vortex，作为工业系统补充

这样修订后，覆盖面会比上一版完整很多，尤其不会漏掉 2025–2026 这条非常活跃的 **Camel → DeXOR → G-ALP / GPU FP compression** 新线索。

[1]: https://dblp.org/rec/journals/pacmmod/YaoCFGJL24?utm_source=chatgpt.com "dblp: <i>Camel:</i> Efficient Compression of Floating-Point Time Series."
[2]: https://dl.acm.org/doi/epdf/10.1145/3698802?utm_source=chatgpt.com "Camel: Efficient Compression of Floating-Point Time Series"
[3]: https://github.com/yoyo185644/camel?utm_source=chatgpt.com "GitHub - yoyo185644/camel · GitHub"
[4]: https://arxiv.org/abs/2601.00695?utm_source=chatgpt.com "DeXOR: Enabling XOR in Decimal Space for Streaming Lossless Compression of Floating-point Data"
[5]: https://github.com/SuDIS-ZJU/DeXOR?utm_source=chatgpt.com "GitHub - SuDIS-ZJU/DeXOR"
[6]: https://dblp.org/rec/conf/icde/GuerraVBF25?utm_source=chatgpt.com "\"Learned Compression of Nonlinear Time Series with Random Access.\" - dblp"
[7]: https://arxiv.org/abs/2412.16266?utm_source=chatgpt.com "Learned Compression of Nonlinear Time Series With Random Access"
[8]: https://github.com/and-gue/NeaTS?utm_source=chatgpt.com "GitHub - and-gue/NeaTS"
[9]: https://dblp.org/rec/conf/damon/HepkemaAFBM25?utm_source=chatgpt.com "\"G-ALP: Rethinking Light-weight Encodings for GPUs.\" - dblp"
[10]: https://ir.cwi.nl/pub/35205/35205.pdf?utm_source=chatgpt.com "G-ALP: Rethinking Light-weight Encodings for GPUs"
[11]: https://github.com/cwida/FastLanesGpu-Damon2025?utm_source=chatgpt.com "GitHub - cwida/FastLanesGpu-Damon2025"
[12]: https://ir.cwi.nl/pub/34260/34260.pdf?utm_source=chatgpt.com "Accelerating GPU Data Processing using FastLanes Compression"
[13]: https://jliu-1.github.io/CV.pdf?utm_source=chatgpt.com "Jinyang Liu - GitHub Pages"
[14]: https://arxiv.org/html/2311.12133v3?utm_source=chatgpt.com "High-performance Effective Scientific Error-bounded Lossy Compression ..."
[15]: https://www.alcf.anl.gov/publications/cusz-i-high-ratio-scientific-lossy-compression-gpus-optimized-multi-level?utm_source=chatgpt.com "CUSZ-i: High-Ratio Scientific Lossy Compression on GPUs with Optimized ..."
[16]: https://www.computer.org/csdl/proceedings-article/sc/2024/529100a158/21HUVd8CAPC?utm_source=chatgpt.com "CUSZ-i: High-Ratio Scientific Lossy Compression on GPUs with Optimized ..."
[17]: https://userweb.cs.txstate.edu/~burtscher/LC/?utm_source=chatgpt.com "Martin Burtscher / LC Project - Texas State University"
[18]: https://dl.acm.org/doi/epdf/10.1145/3669940.3707280?utm_source=chatgpt.com "Efficient Lossless Compression of Scientific Floating-Point Data on ..."
[19]: https://dblp.org/pid/58/4127?utm_source=chatgpt.com "Andrew Pavlo - dblp"
[20]: https://people.iiis.tsinghua.edu.cn/~huanchen/publications/f3-sigmod26.pdf?utm_source=chatgpt.com "F3: The Open-Source Data File Format for the Future"
[21]: https://github.com/vortex-data/vortex?utm_source=chatgpt.com "GitHub - vortex-data/vortex: An extensible, state-of-the-art framework ..."
[22]: https://spiraldb.com/blog/cascading-compression-with-btrblocks?utm_source=chatgpt.com "Have your cake and decompress it too - spiraldb.com"
[23]: https://duckdb.org/2026/01/23/duckdb-vortex-extension?utm_source=chatgpt.com "Announcing Vortex Support in DuckDB – DuckDB"
[24]: https://www.vldb.org/cidrdb/papers/2025/p19-prammer.pdf?utm_source=chatgpt.com "Towards Functional Decomposition of Storage Formats"
[25]: https://www.vldb.org/cidrdb/papers/2025/p26-liao.pdf?utm_source=chatgpt.com "Bullion: A Column Store for Machine Learning - vldb.org"
[26]: https://people.iiis.tsinghua.edu.cn/~huanchen/publications/nulls-damon24.pdf?utm_source=chatgpt.com "NULLS! Revisiting Null Representation in Modern Columnar Formats"
[27]: https://arxiv.org/html/2506.18062v1?utm_source=chatgpt.com "Floating-Point Data Transformation for Lossless Compression"
[28]: https://arxiv.org/pdf/2511.04140v1?utm_source=chatgpt.com "GPU-Based Floating-point Adaptive Lossless Compression"
[29]: https://www.sciencedirect.com/science/article/pii/S0020025523014329?utm_source=chatgpt.com "AFC: An adaptive lossless floating-point compression algorithm in time ..."
[30]: https://jermp.github.io/assets/pdf/papers/SPIRE2021.pdf?utm_source=chatgpt.com "TSXor: A Simple Time Series Compression Algorithm"
我把 “test-series” 按 **time-series** 理解。下面是按三条支线整理的总表；同一篇论文如果横跨多条线，我会放在最主要的一条，避免重复。

---

## 1. Time-series compression 支线

|          年份 | 论文名                                                                                                            | 作者                                                                                                    | Venue                                                         |
| ----------: | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
|        2022 | **Chimp: Efficient Lossless Floating Point Compression for Time Series Databases**                             | Panos Liakos, Katia Papakonstantinopoulou, Yannis Kotidis                                             | PVLDB 2022 ([VLDB][1])                                        |
|        2022 | **Frequency Domain Data Encoding in Apache IoTDB**                                                             | Haoyu Wang, Shaoxu Song                                                                               | PVLDB 2022 ([VLDB][2])                                        |
|        2023 | **Sim-Piece: Highly Accurate Piecewise Linear Approximation through Similar Segment Merging**                  | Xenophon Kitsios, Panagiotis Liakos, Katia Papakonstantinopoulou, Yannis Kotidis                      | PVLDB 2023 ([VLDB][3])                                        |
|        2023 | **MOST: Model-based Compression for Time Series**                                                              | Zehai Yang, Shimin Chen                                                                               | PACMMOD / SIGMOD 2023 ([市民陈][4])                              |
|        2024 | **Machete: Efficient Lossy Floating-Point Compression Using Learned Quantization**                             | Yang Shi, Xiangyu Zou, Xinyu Chen, Sian Jin, Dingwen Tao, Deng Cai, Yufan Chen, Wen Xia               | DCC 2024 ([Hal Science][5])                                   |
|        2024 | **AdaEdge: A Resource-Efficient Adaptive Monitoring System for Time Series**                                   | Chunwei Liu, John Paparrizos, Aaron J. Elmore                                                         | ICDE 2024 ([Researchr][6])                                    |
|        2024 | **Scalable Model-Based Management of Correlated Dimensional Time Series in ModelarDB+**                        | Abduvoris Abduvakhobov, Søren Kejser Jensen, Torben Bach Pedersen, Christian Thomsen                  | PVLDB 2024 ([DBLP][7])                                        |
| 2024 / 2025 | **Flexible Grouping of Linear Segments for Highly Accurate Lossy Compression of Time Series Data / Mix-Piece** | Xenophon Kitsios, Panagiotis Liakos, Katia Papakonstantinopoulou, Yannis Kotidis                      | VLDB Journal ([Springer链接][8])                                |
|        2025 | **Camel: Efficient Compression of Floating-Point Time Series**                                                 | Yuanyuan Yao, Lu Chen, Ziquan Fang, Yunjun Gao, Christian S. Jensen, Tianyi Li                        | PACMMOD / SIGMOD 2025 ([2025 ACM SIGMOD/PODS Conference][9])  |
|        2025 | **Serf: Streaming Error-Bounded Floating-Point Compression**                                                   | Ruiyuan Li, Zechao Chen, Ruyun Lu, Xiaolong Xu, Guangchao Yang, Chao Chen, Jie Bao, Yu Zheng          | PACMMOD / SIGMOD 2025 ([2025 ACM SIGMOD/PODS Conference][10]) |
|        2025 | **NeaTS: Learned Compression of Nonlinear Time Series with Random Access**                                     | Andrea Guerra, Giorgio Vinciguerra, Antonio Boffa, Paolo Ferragina                                    | ICDE 2025 ([DBLP][11])                                        |
|        2025 | **Improving Time Series Data Compression in Apache IoTDB / CompressIoTDB**                                     | Yuxin Tang, Feng Zhang, Jiawei Guan, Yuan Tian, Xiangdong Huang, Chen Wang, Jianmin Wang, Xiaoyong Du | PVLDB 2025                                                    |
|        2025 | **SPARTAN: A Model-Based Semantic Compression System for Massive Time Series**                                 | Fan Yang, John Paparrizos                                                                             | PACMMOD / SIGMOD 2025 ([The Datum Organization][12])          |
|        2026 | **DeXOR: Enabling XOR in Decimal Space for Streaming Lossless Compression of Floating-point Data**             | Chuanyi Lv, Huan Li, Dingyu Yang, Zhongle Xie, Lu Chen, Christian S. Jensen                           | PVLDB 2026 / arXiv 2026 ([arXiv][13])                         |

---

## 2. Floating-point / scientific data compression 支线

|   年份 | 论文名                                                                                                                                      | 作者                                                                                                                                                               | Venue                                |
| ---: | ---------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| 2022 | **MDZ: An Efficient Error-Bounded Lossy Compressor for Molecular Dynamics**                                                              | Kai Zhao, Sheng Di, Danny Perez, Xin Liang, Zizhong Chen, Franck Cappello                                                                                        | ICDE 2022 ([LARO][14])               |
| 2023 | **Elf: Erasing-based Lossless Floating-point Compression**                                                                               | Ruiyuan Li, Dongjie Li, Yuyao Li, Chen Chen, Lu Chen, Christian S. Jensen, Yu Zheng                                                                              | PVLDB 2023 ([VLDB][15])              |
| 2024 | **ALP: Adaptive Lossless Floating-Point Compression**                                                                                    | Azim Afroozeh, Leonardo Kuffó, Peter Boncz                                                                                                                       | SIGMOD / PACMMOD 2024 ([GitHub][16]) |
| 2024 | **FCBench: Cross-domain Benchmarking of Lossless Floating-point Compressors**                                                            | Xinyu Chen, Jiannan Tian, Ian Beaver, Cynthia Freeman, Yan Yan, Jianguo Wang, Dingwen Tao                                                                        | PVLDB 2024 ([VLDB][17])              |
| 2024 | **High-performance Effective Scientific Error-bounded Lossy Compression with Auto-tuned Multi-component Interpolation / HPEZ / QoZ 2.0** | Jinyang Liu, Jiannan Tian, Sheng Di, Xiaodong Yu, Kai Zhao, Sian Jin, Dingwen Tao, Xin Liang, Franck Cappello                                                    | PACMMOD 2024 ([arXiv][18])           |
| 2024 | **cuSZ-i: High-Ratio Scientific Lossy Compression on GPUs with Optimized Multi-Level Interpolation**                                     | Jinyang Liu, Jiannan Tian, Sheng Di, Xiaodong Yu, Dingwen Tao, Xin Liang, Franck Cappello                                                                        | SC 2024 ([DBLP][19])                 |
| 2024 | **Everything You Always Wanted to Know About Storage Compressibility of Pre-Trained ML Models but Were Afraid to Ask**                   | Zhaoyuan Su, Ammar Ahmed, Zirui Wang, Ali Anwar, Yue Cheng                                                                                                       | PVLDB 2024 ([arXiv][20])             |
| 2025 | **Beyond Compression: The Comprehensive Evaluation of Lossless Floating-Point Compression**                                              | Kaisei Hishida, Chunwei Liu, John Paparrizos, Aaron J. Elmore                                                                                                    | PVLDB 2025 ([DBLP][21])              |
| 2025 | **Efficient Lossless Compression of Scientific Floating-Point Data on CPUs and GPUs**                                                    | Noushin Azami, Alex Fallin, Martin Burtscher                                                                                                                     | ASPLOS 2025 ([vusec][22])            |
| 2025 | **TspSZ: Topological Skeleton Preservation for Error-Bounded Lossy Compression of Scientific Data**                                      | Mingze Xia, Bei Wang, Yuxiao Li, Pu Jiao, Xin Liang, Hanqi Guo                                                                                                   | ICDE 2025 ([Mingze's Homepage][23])  |
| 2025 | **Pcodec: Better Compression for Columnar Data**                                                                                         | Martin Loncaric, Niels Jeppesen, Ben Zinberg                                                                                                                     | arXiv 2025 ([arXiv][24])             |
| 2025 | **Floating-Point Data Transformation for Lossless Compression**                                                                          | Samirasadat Jamalidinan, Kazem Cheshmi                                                                                                                           | arXiv 2025 ([Cool Papers][25])       |
| 2025 | **cuSZ-Hi: High-Ratio Scientific Lossy Compression on GPUs with Hardware-Optimized Interpolation**                                       | Shixun Wu, Jinwen Pan, Jinyang Liu, Jiannan Tian, Ziwei Qiu, Jiajun Huang, Kai Zhao, Xin Liang, Sheng Di, Zizhong Chen, Franck Cappello                          | SC 2025 / arXiv ([arXiv][26])        |
| 2025 | **STZ: An Efficient Error-controlled Lossy Compression Framework for Spatiotemporal Data**                                               | Daoce Wang, Pascal Grosset, Jesus Pulido, Jiannan Tian, Tushar M. Athawale, Jinda Jia, Baixi Sun, Boyuan Zhang, Sian Jin, Kai Zhao, James Ahrens, Fengguang Song | SC 2025 ([arXiv][27])                |

---

## 3. FastLanes-like / lightweight columnar compression 支线

|          年份 | 论文名                                                                                              | 作者                                                                                                    | Venue                                                                   |
| ----------: | ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
|        2023 | **The FastLanes Compression Layout: Decoding >100 Billion Integers per Second with Scalar Code** | Azim Afroozeh, Peter Boncz                                                                            | PVLDB 2023 ([VLDB][28])                                                 |
|        2023 | **BtrBlocks: Efficient Columnar Compression for Data Lakes**                                     | Maximilian Kuschewski, David Sauerwein, Adnan Alhomssi, Viktor Leis                                   | SIGMOD / PACMMOD 2023 ([Department of Computer Science][29])            |
|        2024 | **LeCo: Lightweight Compression via Learning Serial Correlations**                               | Yihao Liu, Xinyu Zeng, Huanchen Zhang                                                                 | SIGMOD / PACMMOD 2024 ([arXiv][30])                                     |
|        2024 | **Accelerating GPU Data Processing Using FastLanes Compression**                                 | Azim Afroozeh, Lotte Felius, Peter Boncz                                                              | DaMoN 2024 ([CWI][31])                                                  |
|        2025 | **The FastLanes File Format**                                                                    | Azim Afroozeh, Peter Boncz                                                                            | PVLDB 2025 ([VLDB][32])                                                 |
|        2025 | **G-ALP: Rethinking Light-weight Encodings for GPUs**                                            | Sven Hielke Hepkema, Azim Afroozeh, Charlotte Felius, Peter Boncz, Stefan Manegold                    | DaMoN 2025 ([DBLP][33])                                                 |
|        2025 | **Towards Functional Decomposition of Storage Formats**                                          | Martin Prammer, Xinyu Zeng, Ruijun Meng, Wes McKinney, Huanchen Zhang, Andrew Pavlo, Jignesh M. Patel | CIDR 2025 ([VLDB][34])                                                  |
| 2025 / 2026 | **F3: The Open-Source Data File Format for the Future**                                          | Xinyu Zeng, Ruijun Meng, Martin Prammer, Wes McKinney, Jignesh M. Patel, Andrew Pavlo, Huanchen Zhang | PACMMOD 2025 / SIGMOD 2026 round ([Carnegie Mellon Database Group][35]) |

---

## 进一步查漏补缺结果

在上一版基础上，这一轮我认为**应该新增到主清单**的主要是：

| 新增论文                                            | 应放入支线            | 原因                                                                            |
| ----------------------------------------------- | ---------------- | ----------------------------------------------------------------------------- |
| **MOST**                                        | time-series      | SIGMOD/PACMMOD 2023，model-based time-series compression，上一版遗漏。                |
| **Machete**                                     | time-series / FP | DCC 2024，面向 floating-point 的 learned quantization lossy compression。          |
| **AdaEdge**                                     | time-series      | ICDE 2024，偏 adaptive monitoring / edge compression，和纯 codec 不同，但应列入时序压缩生态。    |
| **ModelarDB+ wind-turbine paper**               | time-series      | PVLDB 2024，model-based management of correlated dimensional time series。      |
| **SPARTAN**                                     | time-series      | SIGMOD/PACMMOD 2025，model-based semantic compression for massive time series。 |
| **MDZ**                                         | FP / scientific  | ICDE 2022，科学计算 error-bounded lossy compression，上一版遗漏。                         |
| **TspSZ**                                       | FP / scientific  | ICDE 2025，topology-preserving scientific lossy compression。                   |
| **cuSZ-Hi / STZ**                               | FP / scientific  | SC 2025 新工作，属于 scientific floating-point compression 近年主线。                    |
| **Pcodec / Floating-Point Data Transformation** | FP / arXiv       | 不是 CCF A，但与 columnar/floating-point compression 相关，建议列入 arXiv 待观察。            |
| **F3 / Functional Decomposition**               | FastLanes-like   | 不是单一 codec，但对下一代 columnar file format 与 compressed execution 很相关。             |

我建议**暂不放进主表，只放边缘相关**的有：

| 论文 / 系统                                                                  | 原因                                                                                  |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| **Vortex**                                                               | 很相关，但目前更像开源 columnar format / toolkit，不是单篇学术论文主条目。                                  |
| **NULLS!: Revisiting Null Representation in Modern Columnar Formats**    | DaMoN 2024，聚焦 NULL representation，对压缩有影响但不是主 codec。                                 |
| **Bullion: A Column Store for Machine Learning**                         | CIDR 2025，涉及 ML 特征编码和 quantization，但不是压缩主线。                                         |
| **Adaptive Encoding Strategies for Lossless Floating-Point Compression** | IEEE IoT Journal 2025，主题相关，但不符合你“优先 CCF A / arXiv”的主筛选标准。                           |
| **AFC / ACTF / NPAC 等 IoT/普通期刊工作**                                       | 与 floating-point time-series compression 相关，但 venue 优先级较低，可作为 exhaustive survey 附录。 |

如果只保留最核心的阅读集，我会压缩成这 12 篇：**Chimp、Elf、Camel、Serf、DeXOR、NeaTS、FastLanes Layout、ALP、FastLanes File Format、G-ALP、FCBench、Beyond Compression**。

[1]: https://www.vldb.org/pvldb/vol15/p3058-liakos.pdf?utm_source=chatgpt.com "Chimp: Efficient Lossless Floating Point Compression for Time Series ..."
[2]: https://www.vldb.org/pvldb/vol16/p282-song.pdf "Frequency Domain Data Encoding in Apache IoTDB"
[3]: https://www.vldb.org/pvldb/vol16/p1910-liakos.pdf "Sim-Piece: Highly Accurate Piecewise Linear Approximation through Similar Segment Merging"
[4]: https://www.shimin-chen.com/papers/most-sigmod24.pdf "MOST: Model-Based Compression with Outlier Storage for Time Series Data"
[5]: https://hal.science/hal-04367319/document?utm_source=chatgpt.com "Machete: An Efficient Lossy Floating-Point Compressor Designed for Time ..."
[6]: https://researchr.org/publication/LiuPE24?utm_source=chatgpt.com "AdaEdge: A Dynamic Compression Selection Framework for Resource ..."
[7]: https://dblp.dagstuhl.de/rec/journals/pvldb/AbduvakhobovJPT24.html?utm_source=chatgpt.com "dblp: Scalable Model-Based Management of Massive High Frequency Wind ..."
[8]: https://link.springer.com/article/10.1007/s00778-024-00862-z "Flexible grouping of linear segments for highly accurate lossy compression of time series data | The VLDB Journal | Springer Nature Link"
[9]: https://2025.sigmod.org/toc-2-6.html?utm_source=chatgpt.com "Proceedings of the ACM on Management of Data: SIGMOD ... - 2025.sigmod.org"
[10]: https://2025.sigmod.org/sigmod_papers.shtml?utm_source=chatgpt.com "The 2025 ACM SIGMOD/PODS Conference: Berlin, Germany - Accepted Papers ..."
[11]: https://dblp.org/rec/conf/icde/GuerraVBF25?utm_source=chatgpt.com "\"Learned Compression of Nonlinear Time Series with Random Access.\" - dblp"
[12]: https://thedatumorg.github.io/DATUM/publications.html?utm_source=chatgpt.com "DATUM LAB"
[13]: https://arxiv.org/abs/2601.00695?utm_source=chatgpt.com "DeXOR: Enabling XOR in Decimal Space for Streaming Lossless Compression of Floating-point Data"
[14]: https://laro.lanl.gov/esploro/outputs/conferenceProceeding/MDZ-An-Efficient-Error-bounded-Lossy-Compressor/9916401273603761?utm_source=chatgpt.com "MDZ: An Efficient Error-bounded Lossy Compressor for Molecular Dynamics ..."
[15]: https://www.vldb.org/pvldb/vol16/p1763-li.pdf?utm_source=chatgpt.com "Elf: Erasing-based Lossless Floating-Point Compression"
[16]: https://github.com/cwida/ALP?utm_source=chatgpt.com "ALP: Adaptive Lossless Floating-Point Compression - GitHub"
[17]: https://www.vldb.org/pvldb/vol17/p1418-tao.pdf?utm_source=chatgpt.com "FCBench: Cross-Domain Benchmarking of Lossless Compression for Floating ..."
[18]: https://arxiv.org/abs/2311.12133?utm_source=chatgpt.com "High-performance Effective Scientific Error-bounded Lossy Compression with Auto-tuned Multi-component Interpolation"
[19]: https://dblp.org/rec/conf/sc/0003TWD0UHH0LTC24?utm_source=chatgpt.com "\"cuSZ-i: High-Ratio Scientific Lossy Compression on GPUs with ... - dblp"
[20]: https://arxiv.org/abs/2402.13429?utm_source=chatgpt.com "Everything You Always Wanted to Know About Storage Compressibility of Pre-Trained ML Models but Were Afraid to Ask"
[21]: https://dblp.org/rec/journals/pvldb/HishidaLPE25?utm_source=chatgpt.com "dblp: Beyond Compression: A Comprehensive Evaluation of Lossless ..."
[22]: https://download.vusec.net/asplos-eurosys-2025/talk/EAPKGT/?utm_source=chatgpt.com "Potpourri 1 :: ASPLOS/EuroSys 2025 Conference :: pretalx"
[23]: https://xty9501.github.io/publication/2025/1/1-tspsz-an-efficient-parallel-error-bounded-lossy-compressor-for-topological-skeleton-preservation?utm_source=chatgpt.com "TspSZ: An Efficient Parallel Error-Bounded Lossy Compressor for ..."
[24]: https://arxiv.org/abs/2502.06112?utm_source=chatgpt.com "Pcodec: Better Compression for Numerical Sequences"
[25]: https://papers.cool/arxiv/2506.18062?utm_source=chatgpt.com "Floating-Point Data Transformation for Lossless Compression | Cool ..."
[26]: https://arxiv.org/abs/2507.11165?utm_source=chatgpt.com "Boosting Scientific Error-Bounded Lossy Compression through Optimized Synergistic Lossy-Lossless Orchestration"
[27]: https://arxiv.org/abs/2509.01626?utm_source=chatgpt.com "STZ: A High Quality and High Speed Streaming Lossy Compression Framework for Scientific Data"
[28]: https://www.vldb.org/pvldb/vol16/p2132-afroozeh.pdf "Untitled"
[29]: https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/btrblocks.pdf?utm_source=chatgpt.com "BtrBlocks: Efficient Columnar Compression for Data Lakes"
[30]: https://arxiv.org/abs/2306.15374?utm_source=chatgpt.com "LeCo: Lightweight Compression via Learning Serial Correlations"
[31]: https://ir.cwi.nl/pub/34260/34260.pdf?utm_source=chatgpt.com "Accelerating GPU Data Processing using FastLanes Compression"
[32]: https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf?utm_source=chatgpt.com "The FastLanes File Format"
[33]: https://dblp.org/rec/conf/damon/HepkemaAFBM25?utm_source=chatgpt.com "\"G-ALP: Rethinking Light-weight Encodings for GPUs.\" - dblp"
[34]: https://www.vldb.org/cidrdb/papers/2025/p19-prammer.pdf?utm_source=chatgpt.com "Towards Functional Decomposition of Storage Formats"
[35]: https://db.cs.cmu.edu/projects/future-file-formats/?utm_source=chatgpt.com "Future File Formats - Carnegie Mellon Database Group"
下面是把 **AFC / ACTF / NPAC 这类非 CCF-A 主会、IoT/普通期刊/CCF-C/预印本**中与 **floating-point time-series compression** 相关的工作单独列出来。优先列 2022 年至今；个别 2020/2021 的经典边缘 baseline 我放在最后。

## 直接相关：floating-point time-series compression

|   年份 | 论文名                                                                                                                               | 作者                                                                                                     | Venue                                                                                                  |
| ---: | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| 2024 | **AFC: An adaptive lossless floating-point compression algorithm in time series database**                                        | Haoyuan Chen, Liang Liu, Jingwen Meng, Wanying Lu                                                      | *Information Sciences*, Vol. 654, Article 119847, 2024 ([科学直达][1])                                     |
| 2024 | **ACTF: An efficient lossless compression algorithm for time series floating point data**                                         | Weijie Wang, Wenhui Chen, Qinhon Lei, Zhe Li, Huihuang Zhao                                            | *Journal of King Saud University - Computer and Information Sciences*, 36(10):102246, 2024 ([科学直达][2]) |
| 2025 | **Adaptive Encoding Strategies for Lossless Floating-Point Compression** / **Elf***                                               | Zheng Li, Ruiyuan Li, Xiaolong Xu, Yi Wu, Chao Chen, Tong Liu, Jiaxing Shang, Yu Zheng                 | *IEEE Internet of Things Journal*, 2025 ([X-MOL][3])                                                   |
| 2025 | **NPAC: numeric pattern aware compression algorithm for floating-point time-series data**                                         | Wenjing Wang, Liang Liu, Kaibin Zhang, Keyue Yang, Lingwei Kuang, Ziyi Zheng, Jinan Wang, Junliang Cao | *World Wide Web*, 28, Article 69, 2025 ([Springer链接][4])                                               |
| 2025 | **Heuristic genetic algorithm parameter optimizer: Making lossless compression algorithms efficient and flexible** / **HGA-ACTF** | Weijie Wang, Wenhui Chen, Li Yan, Yanqing Yang, Huihuang Zhao                                          | *Expert Systems with Applications*, 272:126693, 2025 ([AbleSci][5])                                    |
| 2025 | **DDC: Efficient Dynamic-Dictionary-Based Compression on Floating Time Series Data**                                              | Keyue Yang, Dongping Wang, Shijie Li, Wenbin Zhai, Wenjing Wang, Ziyi Zheng, Jinan Wang, Liang Liu     | IEEE ISPA 2025, CCF C, pp. 1292–1299 ([wenbinzhai.github.io][6])                                       |
| 2026 | **Dolphin: an adaptive lossless compression algorithm for oscillating floating-point time series**                                | Wenhui Chen, Li Yan, Hoekyung Jung, Huihuang Zhao                                                      | *Journal of King Saud University Computer and Information Sciences*, 2026 ([Springer链接][7])            |
| 2026 | **Rabbit: Adaptive Lossless Compression for Floating-Point Time Series via Temporal Locality-Aware Dynamic Encoding**             | Qinhong Lei, Wenhui Chen, Yan Wang, Ya Guo                                                             | *Symmetry*, 18(4):558, 2026 ([MDPI][8])                                                                |
| 2026 | **Waved Zero Streaming: A time series compression framework with adaptive segmentation optimization** / **WZS**                   | Weijie Wang, Wenhui Chen, Huihuang Zhao, Yilin Zhang, Mugang Lin                                       | *Information Processing & Management*, 63(6):104697, 2026 / online metadata ([ir.specialsci.cn][9])    |

---

## 较弱相关：time-series compression，但不一定是 FP 专用

|   年份 | 论文名                                                                                                                        | 作者                                                          | Venue                                                                                         |
| ---: | -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 2023 | **HBC: Combining Lossy and Lossless Hybrid Bilayer Compression Framework on Time-Series Data**                             | Wanying Lu, Liang Liu, Wenbin Zhai, Haoyuan Chen, Yulei Liu | IEEE ISPA/BDCloud/SocialCom/SustainCom 2023, CCF C, pp. 670–679 ([wenbinzhai.github.io][6])   |
| 2026 | **Enhancing storage efficiency for cloud-based vehicle data based on advanced lossless compression algorithm** / **RTLAE** | 作者页/ScienceDirect 可检索，当前我只确认到题名与方法摘要                        | *Expert Systems with Applications*, 2026 online/in press 信息；偏车辆云端时序数据，不确认是 FP 专用 ([科学直达][10]) |

---

## 早于 2022、但常被这些论文引用的普通/边缘 baseline

|   年份 | 论文名                                                                                                  | 作者                                                                                                | Venue                                    |
| ---: | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| 2020 | **LFZip: Lossy Compression of Multivariate Floating-Point Time Series Data via Improved Prediction** | Shubham Chandak, Kedar Tatwawadi, Chengtao Wen, Lingyun Wang, Juan Aparicio Ojea, Tsachy Weissman | DCC 2020, pp. 342–351 ([Researchr][11])  |
| 2021 | **TSXor: A Simple Time Series Compression Algorithm**                                                | 需要再单独核作者版本；常作为 Gorilla/Chimp/AFC/ACTF baseline                                                    | SPIRE 2021；不在 2022+ 主范围，但 AFC/ACTF 等经常对比 |

---

## 快速判断

这些里面，和 **Chimp / Elf / Camel / DeXOR** 主线最接近的是：

1. **AFC**：自适应选择多种压缩策略，集成到 VictoriaMetrics 做 TSDB 场景验证。
2. **ACTF**：在 Chimp/Gorilla 基础上做编码类别扩展、特征表示优化。
3. **Elf***：Elf 的 journal/IoTJ 扩展版，优化 significand count、leading/trailing zeros、sharing condition。
4. **NPAC**：按窗口识别 numerical pattern，再两层决策选择压缩方案。
5. **DDC**：用动态字典捕捉 long-term similarity / periodic pattern。
6. **Dolphin / Rabbit / WZS**：2026 年同一类“继续围绕 XOR/branch/自适应策略”优化的普通期刊线。

相对来说，**HBC、RTLAE** 更像泛 time-series / IoT domain compression，不一定是 IEEE-754 floating-point codec 主线，适合作为边缘相关。

[1]: https://www.sciencedirect.com/science/article/pii/S0020025523014329 "AFC: An adaptive lossless floating-point compression algorithm in time series database - ScienceDirect"
[2]: https://www.sciencedirect.com/science/article/pii/S1319157824003355?utm_source=chatgpt.com "ACTF: An efficient lossless compression algorithm for time series ..."
[3]: https://www.x-mol.com/paper/1908591130915876864?utm_source=chatgpt.com "Adaptive Encoding Strategies for Lossless Floating-Point ..."
[4]: https://link.springer.com/content/pdf/10.1007/s11280-025-01382-8.pdf?utm_source=chatgpt.com "NPAC: numeric pattern aware compression algorithm for ... - Springer"
[5]: https://www.ablesci.com/scholar/paper?id=3JkVJlODE&utm_source=chatgpt.com "Heuristic genetic algorithm parameter optimizer: Making ..."
[6]: https://wenbinzhai.github.io/publications/?utm_source=chatgpt.com "Publications | Wenbin Zhai"
[7]: https://link.springer.com/article/10.1007/s44443-026-00553-5 "Dolphin: an adaptive lossless compression algorithm for oscillating floating-point time series | Journal of King Saud University Computer and Information Sciences | Springer Nature Link"
[8]: https://www.mdpi.com/2073-8994/18/4/558?utm_source=chatgpt.com "Rabbit: Adaptive Lossless Compression for Floating-Point Time Series ..."
[9]: https://ir.specialsci.cn/hunnd/index-collection?index_collection=SSCI&sort=time&utm_source=chatgpt.com "检索结果 · 湖南农业大学机构知识库"
[10]: https://www.sciencedirect.com/science/article/pii/S0957417425035328?utm_source=chatgpt.com "Enhancing storage efficiency for cloud-based vehicle data based on ..."
[11]: https://researchr.org/publication/ChandakTWWOW20?utm_source=chatgpt.com "LFZip: Lossy Compression of Multivariate Floating-Point Time Series ..."
