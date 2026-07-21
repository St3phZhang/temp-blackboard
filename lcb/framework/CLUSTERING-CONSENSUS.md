# PHASE 2 Step 2+3: Sub-Module Clustering Consensus

> **优先级 banner(ADR-0003)**:本文仅作为 12-slot taxonomy / clustering consensus 的**历史依据**;其 (b)/(d) 中把 S7/S8/S9 描述为固定顺序 stage 的视角**已过时**。encode 层已重构为逐列自路由的统一算子层(S5 入口路由 + encoder 池 + 算子链/DAG),**不再**以本文作为 S7/S8/S9 执行顺序或 encode pipeline 的 authority。凡涉及 encode 层表达、算子链/DAG、`ColumnDescriptor.encode_mode/encode_ops`,一律以 `UNIFIED-FRAMEWORK.md` + `docs/adr/0003-encode-layer-operator-chain.md` 为准。本文保留仅因 12-slot 分类共识仍被引用。

本文把两份独立 clustering analysis 合并为一个共识版本，并把高共享度簇细化成可供后续 unified-framework design 使用的 sub-framework data contract。本文只定义 sub-module clustering、slot 边界、统一输入输出契约与 variant points；不设计端到端统一框架。

## (a) Final sub-module slot list

两位 analyst 的 11-slot 结论高度收敛，但对旧 S4 的解释不同。本文将两者拆开：`metadata/regex pre-extractor` 是 dataset-specific 先验替换模块，`variable typer` 是把变量列判为 numeric/string/dict 的策略模块。因此最终为 **12 个 slots**。为避免旧编号混淆：旧 S6/S7/S8/S10 分别对应本文 **S7/S8/S9/S11**。

| Slot | Name | 7-stage 位置 | Definition | Why a distinct slot |
|---|---|---|---|---|
| S1 | Line framing / header split | stage 1 前段 | 把 raw bytes/lines 规范成 `Record{line_id, headers, body}`，可包含 multiline merge、timestamp/header 剥离、`log_format` 命名捕获。 | 它决定一条 record 的边界和 header/body 分离方式；CLP 的 timestamp boundary、LogReducer/LogShrink 的 `head.format`、LogBlock/LogGzip/LogZip 的 `log_format` capture、LogLite 的 newline-only framing 是不同输入契约（clp §1；logreducer §2；logshrink §1；logblock §1；loggzip §1；logzip §0；loglite §1）。 |
| S2 | Tokenizer / delimiter splitter | stage 1 | 把 body 或 content 切成 token/delimiter sequence，通常保留 delimiter token。 | delimiter 规则可独立替换，且被 template miner 与 variable typer 共同消费；DeLog 固定 delimiter table、Denum/LogShrink delimiter mining、LogGrep 固定 `" \t:=,"`、LogZip regex split 与 CLP variable-boundary tokenization 是同一接口的不同 policy（delog §1；denum §2；logshrink §2；loggrep §1；logzip §2；clp §1）。 |
| S3 | Template / pattern miner | stage 1 主体 | 从 token stream 或 body 产出 `template_id/pattern_id + ordered variables`。 | 这是分歧最大的模块；LengthParser、signature synthesis、numeric stripping、URT/skeleton、CLP variable schema、LogGzip NCD parser 都可以共享一个输出面但内部机制不同（logreducer §1；loggrep §1；logshrink §1；delog §1/§3；denum §1；lognexus §1/§2；logfold §2；clp §1；loggzip §3）。 |
| S4 | Metadata / regex pre-extractor | stage 1/2 optional | 用 dataset-specific regex 把 IP/timestamp/date/size/PID 等整体替换为 placeholder tag，并输出 `tag -> extracted values`。 | 这是先验知识注入模块，不等同于 tokenizer 或 variable typing；DeLog、Denum、LogNexus 共享 Loghub 16-dataset regex table lineage，CLP 另有 timestamp-only known-pattern table（delog §1.1；denum §1；lognexus §1；clp §1）。 |
| S5 | Variable typer | stage 2/3 前 | 对列或变量组判定 `numeric`、`string`、`dict`、`all_same`、`raw/fallback`，并给后续 S7/S8/S9 选择路径。 | Analyst B 识别的共同决策骨架不应被折进 S4；它消费 S3/S4/S6 的变量列，输出编码策略。CLP int/float/dict、DeLog ALLSAME/NUMERIC_DELTA/MAPPING、Denum tag exceptions、LogReducer `basic.rule`、LogShrink numeric/string split、LogNexus placeholder class、LogGrep real/nominal threshold 都是变量分类策略（clp §2；delog §3；denum §2；logreducer §2；logshrink §2/§3；lognexus §3；loggrep §2）。 |
| S6 | Column / field organizer | stage 2 -> stage 4 | 把 row-wise variables 转置为 column streams，核心公共面是 `map<(template_id, var_position), vector<value>>`，也允许 tag-based 或 capsule-based key。 | S7 numeric、S8 dictionary、S9 delta 的共同输入面是列；LogReducer/LogShrink/LogZip/LogNexus per-template variables、Denum/DeLog tag groups、CLP three-column segment、LogGrep Capsule columns、LogBlock direct transpose 都是列组织变体（logreducer §2/§4；logshrink §4；logzip §2/§4；lognexus §3/§4；denum §4；delog §4；clp §4；loggrep §3；logblock §2/§4）。 |
| S7 | Numeric encoder | stage 3 | 把 integer column 编为 compact bytes，主簇是 ZigZag + 7-bit little-endian varint，常与 delta/count-prefix 组合。 | 这是最强共享点；DeLog、Denum、LogNexus、LogReducer、LogShrink、LogFold 近字节级同构，variant 主要是 ZigZag on/off、count prefix、plain fallback（delog §3；denum §3；lognexus §3；logreducer §3；logshrink §3；logfold §3）。 |
| S8 | String dictionary | stage 3/4 | 把 strings/contents/nominal values 编成 `mapping id->value + id_stream`，ID stream 通常复用 S7。 | value->ID + mapping + ID stream 在 Denum、DeLog、LogNexus、LogShrink、LogZip 与 CLP 中反复出现，且与 S7 低层编码耦合（denum §3/§4；delog §3/§4；lognexus §3；logshrink §3；logzip §3；clp §3/§4）。 |
| S9 | Delta / correlation transform | stage 3 | 对 numeric column 先做 `first value + successive deltas`，或 LogReducer 式多列 residual，再交给 S7。 | 它是 S7 之前的可选 transform；Denum、DeLog、LogNexus、LogReducer、LogShrink、LogBlock、LogFold 都有相邻差分或动态 delta，LogReducer 还有唯一多列 correlation residual（denum §2/§3；delog §3；lognexus §3；logreducer §3；logshrink §3；logblock §3；logfold §3）。 |
| S10 | Persistence container | stage 4 | 把 encoded columns、dicts、row-order stream、metadata、failed/outlier streams 组织成 chunk directory、segment、CapsuleBox 或 bitset。 | 它决定 archive 前的 wire shape；主干多文件目录、CLP segment+SQLite、LogGrep CapsuleBox、LogLite bitset、LogBlock/LogGzip single text/CSV 是非同构容器（denum §4；delog §4；logreducer §4；logshrink §4；lognexus §4；logzip §4；clp §4；loggrep §4；loglite §4；logblock §4；loggzip §4）。 |
| S11 | Backend archiver | stage 5 | 把 chunk directory 或 byte streams 交给 general compressor，默认 join point 是 tar/LZMA-family backend。 | 绝大多数算法最终合流到 tar+LZMA/xz/gzip/bzip2/lz4/zstd/7z 或 LZMA family；LogLite 只以 external cascade 形式相邻，LogGzip 的 gzip 只是 NCD oracle（delog §5；denum §5；logreducer §5；logshrink §5；lognexus §5；logfold §5；logzip §5；logblock §5；clp §5；loggrep §5；loglite §5；loggzip §5）。 |
| S12 | Searchable index | stage 7 optional | 在压缩态上执行 query/search，输出 matching rows/bitmap，依赖 dictionary/index/stamp metadata。 | 只有 CLP 和 LogGrep 具备压缩态检索；其他 10 个需要完整解压或无存储压缩语义（clp §7；loggrep §7；delog §7；denum §7；logreducer §7；logshrink §7；lognexus §7；logfold §7；logzip §7；logblock §7；loglite §7；loggzip §7）。 |

## (b) Per-slot consensus clustering table

### S1 Line framing / header split

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Header/multiline boundary | clp, logreducer, logshrink | 根据 timestamp/header schema 切 record 与 body。CLP 用 known timestamp patterns 找 message boundary；LogReducer/LogShrink 用 `headLength/isMulti/headRegex` 和 `head.format` formalize message（clp §1；logreducer §2；logshrink §1）。 | CLP 保留 prefix/suffix 与 timestamp 语义；LogReducer/LogShrink 训练 header schema 并输出 `Head*.head`（clp §1；logreducer §2/§4；logshrink §1/§4）。 |
| `log_format` named capture | logblock, loggzip, logzip, logreducer, logshrink | 用 `<Header>` / `<Header:N>` 风格 schema 生成 regex capture 或 header format，B 识别了 A 未单列的 shared code lineage（logblock §1；loggzip §1；logzip §0；logreducer §2；logshrink §1）。 | LogBlock/LogGzip 偏 DataFrame 切列；LogReducer/LogShrink 接入 header/multiline restore；LogZip 用 LogLoader 生成格式（logblock §1；loggzip §1；logzip §0；logreducer §2；logshrink §1）。 |
| Chunk-of-lines with no semantic header | delog, denum, logfold, lognexus, logzip | 按 block/chunk 读行，header 不是独立语义，后续由 regex/tokenizer/miner 处理（delog §0；denum §0；logfold §1；lognexus §0；logzip §0）。 | DeLog/Denum/LogNexus 随后进入 S4 regex pre-extractor；LogFold 保留 holistic entry 后 tokenize；LogZip 也有 log_format loader 分支（delog §1；denum §1；lognexus §1；logfold §1；logzip §0）。 |
| Byte/line opaque | loglite | 只按 newline 切行并处理 byte-level residual，无 header/template 语义（loglite §1）。 | 整条上游不接入 S2-S10 主干，只可能在 S11 external cascade 相邻（loglite §1/§5）。 |

### S2 Tokenizer / delimiter splitter

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Fixed delimiter set, keep delimiters | delog, loggrep, loggzip, logzip, clp | 用固定 delimiter set 切 token，delimiter 自身可保留用于对齐 template 或 query（delog §1；loggrep §1；loggzip §1；logzip §2；clp §1）。 | DeLog 不把 `.`, `-`, `/`, `<`, `>` 当 delimiter；LogGrep 用 `" \t:=,"`；CLP 按 variable schema boundary 而非纯 delimiter 规则（delog §1；loggrep §1；clp §1）。 |
| Mined delimiter set | denum, logshrink | 从样本/候选字符中挖 high-frequency special chars，再 split 并保留 delimiter（denum §2；logshrink §2）。 | Denum fallback 到 space；LogShrink 使用 delimiter-level LCS/pattern miner，不只是 split（denum §2；logshrink §2）。 |
| Skeleton/sub-token tokenizer | logfold, lognexus | 按 alphanumeric / non-alphanumeric skeleton 与 sub-token matrix 组织结构（logfold §2；lognexus §1/§2）。 | LogFold 偏 FP-Growth critical-position；LogNexus 偏 URT/pathID（logfold §2；lognexus §1/§2）。 |
| No tokenizer | loglite | 整行 bytes 进入 same-length window + XOR/RLE（loglite §1/§3）。 | 不产生 token stream。 |

### S3 Template / pattern miner

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| LogReducer length-bucket exact-match parser | logreducer, logshrink, loggrep | `LengthSearch`/`LengthParser` 按 token length bucket，`templateNode::matchMatch` 对非 wildcard token exact match，是唯一明确共享代码谱系的 template miner（logreducer §1；logshrink §1；loggrep §1/§8）。 | LogReducer 训练期 Drain-like prefix tree、压缩期 exact match；LogShrink 改编 LogReducer parser；LogGrep 静态模板头共享后继续 runtime pattern branch（logreducer §1；logshrink §1；loggrep §1/§2）。 |
| Signature / numeric stripping without heavyweight parser | delog, denum | 不构造传统 EventTemplate；DeLog 单遍合成 pattern signature，Denum 抽 numeric tokens 后用 logs-without-numbers 字典承担 template-like 角色（delog §1/§3；denum §1/§2）。 | DeLog 以 context/length/special-char 生成 `<IDX|LEN|CTX|STR>`；Denum 以 tag groups 与 numeric variable extraction 为主（delog §3；denum §2）。 |
| Skeleton / URT structural miner | logfold, lognexus | 基于 delimiter skeleton、sub-token matrix 或 structural tree 挖结构与变量（logfold §2；lognexus §1/§2）。 | LogFold 是 paper-only structured token + FP-Growth；LogNexus 是 URT pathID 与 variable subtree（logfold §2；lognexus §1/§2）。 |
| Drain/LenMa/NCD-like parser | logzip, loggzip, logreducer training | 相似度阈值聚类或 prefix tree 训练模板（logzip §1/§2；loggzip §1/§3；logreducer §1）。 | LogGzip 用 gzip-NCD 作为 parser similarity，输出 CSV，不是 storage compressor（loggzip §3/§4/§5）。 |
| CLP variable-schema miner | clp | 用 variable schema 把 token 归为 logtype text 与 variables，生成 logtype dictionary（clp §1/§3）。 | 与 trunk parser 输出相似，但 downstream S7/S10/S12 自成 searchable storage branch（clp §2/§4/§7）。 |
| No template miner | loglite | N/A（loglite §1）。 | 无法接入 `template_id + variables` 面。 |

### S4 Metadata / regex pre-extractor

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| 16-dataset regex table + placeholder tags | delog, denum, lognexus | 对 Loghub-style datasets 硬编码 IP/timestamp/date/size/core/PID regex，替换为 `<I>/<T>/...` 并输出 tag values；这是 A 的 distinct S4，本文保留为 optional trunk sub-module（delog §1.1；denum §1；lognexus §1）。 | tag 名称与存储规整不同：DeLog normal/fast raw-vs-digits、Denum C++/Python IP padding 差异、LogNexus tag naming 差异，但输入输出同构（delog §1.1/§2；denum §1/§2；lognexus §1/§3）。 |
| Timestamp-only known pattern | clp | 30+ `TimestampPattern` 用于 line/body boundary 与 timestamp extraction（clp §1）。 | 只覆盖 timestamp，不是通用 IP/size regex table（clp §1）。 |
| No regex pre-extractor | logblock, loggrep, logreducer, logshrink, logfold, loglite, loggzip, logzip | 不使用 16-dataset pre-extraction table；有的使用 log_format 或 parser 规则代替（logblock §1；loggrep §1；logreducer §2；logshrink §1；logfold §1；loglite §1；loggzip §1；logzip §0）。 | `log_format` header capture 属 S1，不归入 S4。 |

### S5 Variable typer

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Numeric/string/dict decision tree | clp, delog, denum, logreducer, logshrink, lognexus | 尝试数值化或按 rule/tag 判 numeric，否则 string/dict；输出后续 encoder route（clp §2；delog §3；denum §2；logreducer §2；logshrink §2/§3；lognexus §3）。 | CLP 有 int/float/dictionary；DeLog 有 ALLSAME/NUMERIC_DELTA/NUMERIC_NODELTA/MAPPING；Denum 使用 tag exception set；LogReducer 由 `basic.rule` 固化数值/字符串列（clp §2；delog §3；denum §2；logreducer §2）。 |
| Low-cardinality dictionary decision | logblock, logshrink, loggrep | 低 unique ratio 或 nominal values 进入 dictionary/capsule dict（logblock §3；logshrink §3；loggrep §2）。 | LogGrep real/nominal threshold 是 runtime-pattern 分叉；LogBlock 是 column-local H3 dict，不是 global dict（loggrep §2；logblock §3）。 |
| No variable typer | loglite, loggzip | 无 typed variable streams 或仅 parser CSV（loglite §2；loggzip §2/§4）。 | 不接入 S7/S8/S9 主干。 |

### S6 Column / field organizer

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Per-template/position columns | logreducer, logshrink, lognexus, logzip | 把同 template/event/pathID 的同位 variables 连续存储，逻辑 key 是 `(template_id, var_position[, sub_position])`（logreducer §2/§4；logshrink §4；lognexus §3/§4；logzip §2/§4）。 | 文件命名不同：`E<eid>_<col>.dat/.str`、`var_list_E<eid>_<pos>`、`MegaVariables_col_*`、`{eid}_{para}_{sub}.csv`（logreducer §4；logshrink §4；lognexus §4；logzip §4）。 |
| Tag/signature grouped columns | denum, delog | 按 regex/numeric/signature tag 聚合 values，形成 `map<tag, vector<value>>`（denum §1/§4；delog §1/§4）。 | Denum numeric `.bin` 与 dictionary ids/mapping 分开；DeLog 合并同 encoding method 的 groups 并用 index offsets（denum §4；delog §4）。 |
| Search branch columns | clp, loggrep | CLP 保存 timestamp/logtype/variables columnar segment；LogGrep 把 variable vector 拆成 `.var/.svar/.dic/.entry` Capsule columns（clp §4；loggrep §3/§4）。 | 二者都列化，但输出为了 S12 search index 额外保留 segment/stamp metadata（clp §4/§7；loggrep §3/§7）。 |
| Direct transpose | logblock | 按 `log_format` 列转置，列内 `||` join、列间 newline（logblock §2/§4）。 | 无 template_id，key 是 header/content field name 或 column ordinal。 |
| No column organizer | loglite, loggzip | LogLite 是 sequential bitset；LogGzip 是明文 CSV parser output（loglite §4；loggzip §4）。 | outlier。 |

### S7 Numeric encoder

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| ZigZag + 7-bit little-endian varint | delog, denum, lognexus, logreducer, logshrink, logfold | `zigzag(n)=(n<<1)^(n>>63)` 或等价 signed mapping；每 byte low 7 bits payload，MSB=1 continuation，final MSB=0（delog §3；denum §3；lognexus §3；logreducer §3；logshrink §3；logfold §3）。 | LogReducer `.dat/.eid/.head` 有 `uvarint(count)` prefix；LogNexus 有 unsigned no-ZigZag path；LogReducer non-Z mode 是 plaintext fallback；LogFold 证据主要 paper-level（logreducer §3；lognexus §3；logfold §3）。 |
| Fixed/custom numeric encoding | clp | int/float 存 custom 64-bit `encoded_variable`，不是 varint stream（clp §2）。 | search branch 自有 encoding 与 dictionary/segindex 耦合（clp §2/§4/§7）。 |
| No dedicated numeric encoder | logblock, loggrep, loglite, loggzip, logzip | 数值以 string/padded bytes/base64/byte residual/CSV 方式交给后端或 runtime capsule（logblock §3；loggrep §3/§5；loglite §3；loggzip §4；logzip §3）。 | LogGrep 的 `.entry` 是 padded decimal/bytes，不走 S7 varint（loggrep §3）。 |

### S8 String dictionary

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Incremental ID + mapping + S7 ID stream | denum, delog, lognexus, logshrink | first-occurrence IDs, mapping ordered by ID, id stream encoded by elastic varint or equivalent（denum §3/§4；delog §3/§4；lognexus §3；logshrink §3）。 | ID start usually 1；DeLog line-number implicit mapping; LogShrink has dict-index diff stream; LogNexus prefix-coded string files（delog §4；logshrink §3；lognexus §3）。 |
| Structured storage dictionaries | clp, logzip, loggrep, logblock | CLP logtype/var dict + segindex；LogZip template/parameter mapping; LogGrep nominal Capsule dict; LogBlock column-local H3 dict（clp §3/§4；logzip §3；loggrep §2/§3；logblock §3）。 | These share value->ID idea but differ in search metadata, base-64 ParaID, capsule layout, or inline header（clp §7；logzip §3；loggrep §3/§7；logblock §3）。 |
| No persistent string dict | loglite, loggzip, logfold | LogLite byte residual; LogGzip md5/cache is parser aid not storage dict; LogFold has conceptual token/template dict but no source-backed persistent shared shape in current docs（loglite §3/§4；loggzip §3/§4；logfold §3）。 | N/A or paper-only. |

### S9 Delta / correlation transform

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| First value + successive deltas | denum, delog, lognexus, logreducer, logshrink, logblock, logfold | Store first raw value and successive differences, then encode or backend-compress（denum §2/§3；delog §3；lognexus §3；logreducer §3；logshrink §3；logblock §3；logfold §3）。 | Denum skips `<I>/<a>/<b>/<c>` in C++; DeLog skips IP/high-variance groups; LogNexus has IP/short-number exceptions; LogBlock H2 has source-noted off-by-one issue（denum §2/§8；delog §3；lognexus §3；logblock §3）。 |
| Multi-column / ID-grouped residual | logreducer | `do/up/doi/upi/diff2/diff3` generate adjacent, ID-grouped, two/three-column residuals with entropy selection and `corre_t=600` threshold（logreducer §3）。 | Unique superset of delta contract; restore path limitation is noted in algorithm doc and must be a variant constraint（logreducer §3/§6/§8）。 |
| No delta | clp, loggrep, loglite, loggzip, logzip | No S9 numeric transform in storage path（clp §2；loggrep §3；loglite §3；loggzip §4；logzip §3）。 | CLP has different typed numeric representation; LogGrep relies on runtime pattern/padding rather than numeric delta（clp §2；loggrep §3）。 |

### S10 Persistence container

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Multi-file chunk directory | denum, delog, logreducer, logshrink, lognexus, logzip | typed column files + dictionary/mapping + row-order id + index/relations/failed streams, then backend packed（denum §4；delog §4；logreducer §4；logshrink §4；lognexus §4；logzip §4）。 | Each has different file naming and metadata grammar; common abstraction is named typed stream set plus row order stream（denum §4；delog §4；logreducer §4；logshrink §4；lognexus §4；logzip §4）。 |
| Searchable single/compound container | clp, loggrep | CLP segment + dictionaries + segindex + SQLite metadata; LogGrep CapsuleBox with compressed meta directory and per-Capsule offsets（clp §4；loggrep §4）。 | These are S12-aware containers, not just archives（clp §7；loggrep §7）。 |
| Single file/bitset container | logblock, loglite, loggzip | LogBlock column-oriented preprocessed text; LogLite dynamic bitset; LogGzip plaintext CSV（logblock §4；loglite §4；loggzip §4）。 | LogGzip is not a compressed storage container; LogLite is byte-level outlier（loggzip §4/§5；loglite §4）。 |

### S11 Backend archiver

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| tar/7z + LZMA/xz-family default | delog, denum, logfold, lognexus, logreducer, logshrink | Pack structured files and apply LZMA/xz or 7z LZMA; DeLog additionally supports gzip/bzip2/lz4/zstd filters（delog §5；denum §5；logfold §5；lognexus §5；logreducer §5；logshrink §5）。 | Implemented by libarchive, `tar -cJf`, or `7za -m0=LZMA`; contract remains directory -> archive（delog §5；denum §5；logreducer §5）。 |
| tar/gzip/bzip2/deflate/lz4 variants | logblock, logzip | Same backend idea but different kernel set; LogZip lzma path is dead/non-mainline per algorithm doc（logblock §5；logzip §5）。 | Shallow branch joins at S11. |
| Search branch backend | clp, loggrep | CLP uses zstd/passthrough-style stream backend; LogGrep uses per-Capsule LZMA with padding for query locality（clp §5；loggrep §5）。 | LogGrep does not pack whole chunk directory; it compresses each Capsule independently（loggrep §5）。 |
| External cascade or non-storage oracle | loglite, loggzip | LogLite core is XOR/RLE and variants may cascade to zstd/lzma; LogGzip uses gzip length only for NCD, not as final storage backend（loglite §5；loggzip §5）。 | LogLite joins S11 only as external byte stream; LogGzip does not truly join. |

### S12 Searchable index

| Cluster | Algorithms | Shared technique | Variant points / outliers |
|---|---|---|---|
| Compressed-state search | clp, loggrep | Query encoded against compressed metadata, prune candidates, selectively decode/match, return rows/bitmap（clp §7；loggrep §7）。 | CLP uses dictionary + segment index + encoded-column matching; LogGrep uses static/runtime patterns, stamp pruning, fixed-length BM, Capsule bitmaps（clp §7；loggrep §7）。 |
| No compressed search | delog, denum, logblock, logfold, loggzip, loglite, lognexus, logreducer, logshrink, logzip | Full decompression or no storage compressor required before search（delog §7；denum §7；logblock §7；logfold §7；loggzip §7；loglite §7；lognexus §7；logreducer §7；logshrink §7；logzip §7）。 | S12 is optional branch, not trunk default. |

## (c) Trunk vs branch vs outlier map

**Pure trunk / trunk-dominant members**: `denum`, `delog`, `logreducer`, `logshrink`, `lognexus`, `logfold`。共同 backbone 是 `S3 parser/signature or S4 regex pre-extractor -> S6 column organizer -> S5 variable typer -> S9 delta + S7 varint + S8 dict -> S10 chunk directory -> S11 tar/LZMA-family backend`。这六个在 S7/S8/S9/S10/S11 上共享最多；差异主要集中在 S3 miner 和 S4/S5 policy（denum §1-§5；delog §1-§5；logreducer §1-§5；logshrink §1-§5；lognexus §1-§5；logfold §1-§5）。

**Searchable branch**: `clp` 与 `loggrep`。二者都保留 `S12 searchable index`，但分叉位置不同：`loggrep` 在 S3 静态模板头与 LogReducer cluster 合流，随后在 runtime pattern、Capsule、per-Capsule LZMA、stamp/BM search 上分叉（loggrep §1-§7）；`clp` 从 S1/S3 variable schema、S7 fixed/custom numeric、S8 dual dictionary、S10 segment+SQLite 到 S12 segindex search 都自成一支，只在 dictionary idea 和 backend compression family 上接近主干（clp §1-§7）。

**Shallow branch**: `logblock`。它使用 S1 `log_format`、S6 direct transpose、S9 H2/H3 heuristics、S10 single preprocessed text，并在 S11 gzip/deflate/lz4 汇合；它不共享 S7 varint trunk，也没有 S12 search（logblock §1-§5/§7）。

**Outlier joining only at S11 as external cascade**: `loglite`。它的上游是 same-length byte window + SIMD XOR + RLE/bitset，没有 S2/S3/S5/S6/S7/S8/S9 主干语义；只有 BZ/BL variants 把 output byte stream 交给 zstd/lzma，因而仅在 S11 概念上相邻（loglite §1/§3/§4/§5）。

**Reference-only / non-storage outlier**: `loggzip`。它是 parser + gzip-NCD similarity experiment，S10 是 plaintext CSV，S11 的 gzip 只作为 NCD length oracle，不产生可统一的 storage-compression artifact；纳入 clustering 的价值主要是 S3 NCD/LenMa-style parser 对比（loggzip §1/§3/§4/§5）。

## (d) Intra-cluster sub-framework

### S7 Numeric encoder: ZigZag + 7-bit varint

**Unified data contract**

```text
EncodedInts = bytes

encode_int(value: int64, spec: NumericSpec) -> bytes
decode_int(stream: ByteCursor, spec: NumericSpec) -> int64

encode_column(values: vector<int64>, spec: NumericColumnSpec) -> EncodedInts
decode_column(bytes: EncodedInts, spec: NumericColumnSpec) -> vector<int64>

NumericSpec {
  signed_mode: "zigzag64" | "unsigned"
  byte_order: "little_7bit"
  continuation_bit: "msb_1_more_0_stop"
}

NumericColumnSpec {
  int_spec: NumericSpec
  count_prefix: bool
  count_prefix_mode: "uvarint" | "none"
  plaintext_fallback: bool
}
```

**Variant points**

| Flag | Values | Meaning |
|---|---|---|
| `signed_mode` | `zigzag64`, `unsigned` | signed values use `(n<<1)^(n>>63)`; unsigned bypasses ZigZag. |
| `count_prefix` | `true`, `false` | column-level count before payload. |
| `continuation_bit` | fixed to `msb_1_more_0_stop` for source-compatible shared trunk | multiple docs warn paper prose may describe opposite convention; source-compatible shared wire uses MSB 1 as continuation（delog §3；denum §3；logreducer §3；lognexus §3）。 |
| `plaintext_fallback` | `true`, `false` | allow `ASCII decimal + newline` for LogReducer non-Z mode（logreducer §3）。 |

**Algorithm mapping**

| Algorithms | Flags |
|---|---|
| delog | `signed_mode=zigzag64`, `count_prefix=false`, `plaintext_fallback=false`; used for template IDs, numeric variables, ALLSAME values（delog §3/§4）。 |
| denum | `signed_mode=zigzag64`, `count_prefix=false`, `plaintext_fallback=false`; used for numeric tag `.bin` and dictionary IDs（denum §3/§4）。 |
| logreducer | `signed_mode=zigzag64`, `count_prefix=true`, `count_prefix_mode=uvarint`, `plaintext_fallback=true` for `-E NE/P`; used for `.dat/.eid/.head`（logreducer §3/§8）。 |
| logshrink | `signed_mode=zigzag64`, implementation delegated to LogReducer-derived Elastic path; count-prefix behavior follows its `.ee`/parser path（logshrink §3）。 |
| lognexus | `signed_mode=zigzag64` for signed values, `unsigned` for unsigned tag/IP paths; count prefix per concrete column writer（lognexus §3）。 |
| logfold | `signed_mode=zigzag64`, dynamic delta + elastic paper-level path（logfold §3）。 |

### S8 String dictionary: value -> ID + mapping + ID stream

**Unified data contract**

```text
DictionaryResult {
  mapping: vector<string>        # indexable by dictionary ID after applying id_base
  id_stream: vector<uint64> | EncodedInts
  scope: DictScope
  ordering: DictOrdering
}

encode_dictionary(values: vector<string>, spec: DictSpec) -> DictionaryResult
decode_dictionary(result: DictionaryResult, encoded_ids: EncodedInts, int_spec: NumericColumnSpec) -> vector<string>

DictSpec {
  id_base: 0 | 1
  ordering: "first_occurrence" | "sorted_by_id" | "pattern_segment_order"
  scope: "chunk" | "template" | "tag" | "segment" | "capsule"
  id_encoding: "S7" | "base64" | "plaintext" | "fixed_width_padded"
  include_segment_index: bool
}
```

**Variant points and mapping**

| Algorithms | Dict flags |
|---|---|
| denum | `id_base=1`, `ordering=first_occurrence/sorted_by_id`, `scope=chunk/tag stream`, `id_encoding=S7`, no search segment index（denum §3/§4）。 |
| delog | `id_base=1` with empty/zero special case for templates, first-occurrence mapping, `scope=chunk/tag`, `id_encoding=S7`, offsets in `index.fast/normal.csv`（delog §3/§4）。 |
| lognexus | `scope=variable column`, mapping in `_var_STRING.txt`, ID column in `MegaVariables_col_*`, prefix/string-specific encoding variants（lognexus §3/§4）。 |
| logshrink | global `header.dict/var.dict`, dict-index diff stream, `id_encoding=S7-derived`, scope global/chunk hybrid（logshrink §3/§4）。 |
| logzip | `template_mapping.json` and `parameter_mapping.json`, `id_encoding=base64`, scope template/parameter object; ParaID global-dedup bug is a variant caveat（logzip §3）。 |
| clp | `scope=segment`, `id_encoding=fixed/structured`, `include_segment_index=true`, split into `logtype.dict` and `var.dict`（clp §3/§4/§7）。 |
| loggrep | `scope=capsule`, nominal values create `.dic` + `.entry`; `id_encoding=fixed_width_padded`, ordering by dictionary entry/pattern segment（loggrep §2/§3）。 |
| logblock | `scope=column`, inline low-cardinality H3 dictionary, no global mapping contract（logblock §3）。 |

### S9 Delta / correlation transform

**Unified data contract**

```text
DeltaResult {
  values: vector<int64>
  transform: TransformDescriptor
}

delta_transform(values: vector<int64>, spec: DeltaSpec) -> DeltaResult
delta_inverse(result: DeltaResult) -> vector<int64>

correlation_transform(columns: map<ColumnKey, vector<int64>>, spec: CorrelationSpec) -> map<DerivedColumnKey, DeltaResult>

DeltaSpec {
  mode: "none" | "successive_delta"
  first_value_mode: "raw_first"
  skip_policy: set<TagPattern | ColumnPredicate>
}

CorrelationSpec {
  mode: "none" | "logreducer_rule"
  allowed_rules: set{"direct", "do", "up", "doi", "upi", "diff2", "diff3"}
  min_rows: int
  id_columns: vector<ColumnKey>
}
```

**Variant points and mapping**

| Algorithms | Transform flags |
|---|---|
| denum | `mode=successive_delta`, `skip_policy={<I>, <a>, <b>, <c>}` in C++ path; Python exceptions differ and must be captured as implementation profile（denum §2/§8）。 |
| delog | `mode=successive_delta` for numeric groups, plus `ALLSAME` shortcut; skips IP/high-variance or dictionary-forced tags（delog §3）。 |
| lognexus | `mode=successive_delta`, skip IP/short-number placeholders and unsigned paths where configured（lognexus §3）。 |
| logreducer | header columns use `successive_delta` via `IntEncoder(diff=true)`; variables may use `CorrelationSpec{mode=logreducer_rule, min_rows=600, allowed_rules=direct/do/up/doi/upi/diff2/diff3}`（logreducer §3/§8）。 |
| logshrink | `mode=successive_delta` / `diff_matcher` for selected columns（logshrink §3）。 |
| logblock | `mode=successive_delta` for H2 but mark `wire_compatibility=buggy_off_by_one` if reproducing source behavior（logblock §3）。 |
| logfold | dynamic selection of delta + elastic paper-level path（logfold §3）。 |

### S11 Backend archiver

**Unified data contract**

```text
ArchiveResult {
  path: string
  kernel: BackendKernel
  format: BackendFormat
  size_bytes: uint64
}

pack(input: BackendInput, spec: BackendSpec) -> ArchiveResult
unpack(archive: ArchiveResult, spec: BackendSpec) -> BackendInput

BackendInput = ChunkDirectory | NamedByteStreams | SingleByteStream

BackendSpec {
  format: "tar" | "7z" | "capsulebox" | "raw_stream"
  kernel: "xz" | "lzma" | "gzip" | "bzip2" | "lz4" | "zstd" | "deflate" | "none"
  compression_scope: "whole_directory" | "per_capsule" | "external_cascade" | "oracle_only"
}
```

**Variant points and mapping**

| Algorithms | Backend flags |
|---|---|
| delog | `format=tar`, `kernel=xz/gzip/bzip2/lz4/zstd/none`, `compression_scope=whole_directory`, libarchive PAX tar（delog §5）。 |
| denum | `format=tar`, `kernel=xz`, `compression_scope=whole_directory`, command `tar -cJf`（denum §5）。 |
| logreducer, logshrink | `format=7z`, `kernel=lzma`, `compression_scope=whole_directory/per-block directory`（logreducer §5；logshrink §5）。 |
| lognexus, logfold | `format=tar`, `kernel=xz/lzma`, `compression_scope=whole_directory`（lognexus §5；logfold §5）。 |
| logzip | `format=tar`, `kernel=gzip/bzip2`, note lzma path not mainline（logzip §5）。 |
| logblock | `format=single file -> backend`, `kernel=gzip/deflate/lz4`, shallow branch（logblock §5）。 |
| clp | `format=segment archive/storage`, `kernel=zstd/passthrough`, searchable branch（clp §5）。 |
| loggrep | `format=capsulebox`, `kernel=lzma`, `compression_scope=per_capsule`, level/dict parameters fixed by code（loggrep §5）。 |
| loglite | `format=raw_stream`, `kernel=zstd/lzma` only for external BZ/BL cascade, not core（loglite §5）。 |
| loggzip | `compression_scope=oracle_only`; gzip length is NCD signal, not storage output（loggzip §5）。 |

### S3 LogReducer-parser cluster

**Unified data contract**

```text
TemplateMatch {
  template_id: int
  template_tokens: vector<Token>
  variable_positions: vector<int>
  variable_values: vector<string>
  status: "matched" | "failed"
}

train_length_bucket_parser(samples: vector<TokenSeq>, spec: ParserSpec) -> TemplatePool
match_template(tokens: TokenSeq, pool: TemplatePool, spec: ParserSpec) -> TemplateMatch

ParserSpec {
  bucket_key: "token_count"
  wildcard_token: "<*>"
  training_match: "similarity_merge" | "external_templates"
  compression_match: "exact_non_wildcard"
  similarity_threshold: float?
  delimiter_policy_ref: TokenizerSpec
}
```

**Variant points and mapping**

| Algorithms | Parser flags |
|---|---|
| logreducer | `training_match=similarity_merge` via Drain-like tree; `compression_match=exact_non_wildcard`; templates from `template.col`; failures to `load_failed.log/match_failed.log`（logreducer §1/§4）。 |
| logshrink | `training_match=LogReducer-derived/THULR`, `compression_match=exact_non_wildcard`, plus LogShrink-specific downstream commonality/variability pipeline（logshrink §1/§4）。 |
| loggrep | sampling parser in `LengthParser`, `training_match=similarity_merge`, full match to `VarArray`; then runtime pattern branch starts from `variable_mapping`（loggrep §1/§2）。 |

### S4 Regex pre-extractor

**Unified data contract**

```text
RegexExtractionResult {
  masked_line: string
  extracted: map<Tag, vector<ExtractedValue>>
  tag_metadata: map<Tag, RegexTagMeta>
}

pre_extract(line: string, table: RegexTagTable, spec: RegexExtractSpec) -> RegexExtractionResult

RegexTagTable = map<DatasetName, vector<{pattern: regex, tag: Tag, value_policy: ValuePolicy}>>

RegexExtractSpec {
  unknown_dataset_policy: "empty" | "error"
  value_policy: "raw" | "digits" | "ip_pad3" | "dataset_specific"
  application_order: "source_order"
}
```

**Variant points and mapping**

| Algorithms | Regex flags |
|---|---|
| delog | `unknown_dataset_policy=empty/warn`, `application_order=source_order`, `value_policy=raw` in fast mode and digits/dataset-specific in normal mode（delog §1.1/§2）。 |
| denum | `unknown_dataset_policy=error` in C++ main path, `value_policy=digits` for many special numeric tokens; Python path has global regex behavior and IP pad3 variant（denum §1/§2/§8）。 |
| lognexus | `application_order=source_order`, similar Loghub placeholder extraction with LogNexus-specific tags and HealthApp special case（lognexus §1/§3）。 |
| clp | not the 16-dataset table; can be represented as `RegexTagTable{timestamp_only}` for S1/S4 boundary compatibility（clp §1）。 |

### S5 Variable typer

**Unified data contract**

```text
TypedColumn {
  key: ColumnKey
  raw_values: vector<string>
  type: "numeric" | "string" | "dict" | "all_same" | "raw" | "runtime_real" | "runtime_nominal"
  numeric_values?: vector<int64>
  encoding_route: vector<SlotRef>
  policy_trace: map<string, string>
}

type_column(column: Column<string>, spec: TypeSpec) -> TypedColumn

TypeSpec {
  numeric_predicate: "ascii_digits" | "int64_safe" | "float_or_int" | "tag_rule"
  dict_predicate: "unique_ratio" | "always" | "never"
  unique_threshold: float?
  all_same_enabled: bool
  tag_exception_policy: map<TagPattern, TypeOverride>
  runtime_pattern_enabled: bool
}
```

**Variant points and mapping**

| Algorithms | Type flags |
|---|---|
| clp | `numeric_predicate=float_or_int`, route int/float to CLP encoded variable, strings to logtype/var dict（clp §2/§3）。 |
| delog | `all_same_enabled=true`, numeric safe-length predicate, tag overrides for dictionary-forced tags and IP/high-variance skip-delta groups; routes to S8/S9/S7（delog §3）。 |
| denum | `numeric_predicate=tag_rule`, tag exceptions for no-delta short/IP groups, numeric variables as string dict values（denum §2/§3）。 |
| logreducer | `numeric_predicate=int32_non_float` via `basic.rule`; string columns direct `.str`; numeric columns may route through correlation/delta/S7（logreducer §2/§3）。 |
| logshrink | `numeric_predicate=isnumeric`, `dict_predicate=unique_ratio` for variable dictionaries（logshrink §2/§3）。 |
| lognexus | placeholder/sub-token classifier routes numeric/string/IP classes to corresponding encoders（lognexus §3）。 |
| loggrep | `runtime_pattern_enabled=true`, `unique_threshold=0.5`, output `runtime_real` or `runtime_nominal` before Capsule encoder（loggrep §2）。 |
| logblock | `dict_predicate=unique_ratio`, H2/H3 heuristics per transposed column（logblock §3）。 |

### S6 Column organizer

**Unified data contract**

```text
ColumnKey {
  template_id?: int | string
  tag?: string
  var_position?: int
  sub_position?: int
  field_name?: string
  branch: "trunk" | "search" | "direct_transpose" | "tag_group"
}

ColumnStore {
  columns: map<ColumnKey, vector<Value>>
  row_order: vector<RowKey | TemplateID | EventID>
  failed_streams: map<string, vector<RawRecord>>
  metadata: map<string, string>
}

organize_columns(records: vector<ParsedRecord>, spec: ColumnSpec) -> ColumnStore

ColumnSpec {
  key_mode: "template_position" | "tag" | "field" | "capsule" | "segment_three_column"
  preserve_row_order: bool
  include_failed_streams: bool
}
```

**Variant points and mapping**

| Algorithms | Column flags |
|---|---|
| logreducer, logshrink | `key_mode=template_position`, `preserve_row_order=true` via `Eid.eid`, `include_failed_streams=true`（logreducer §2/§4；logshrink §4）。 |
| lognexus, logzip | `key_mode=template_position`, pathID/EventID and sub-parameter keys（lognexus §3/§4；logzip §2/§4）。 |
| denum, delog | `key_mode=tag`, row order through dictionary ID streams/templates; tag groups feed S5/S7/S8/S9（denum §1/§4；delog §1/§4）。 |
| clp | `key_mode=segment_three_column`, columns timestamp/logtype/variables, search metadata attached（clp §4）。 |
| loggrep | `key_mode=capsule`, variable tag packs `Eid` and variable position, then runtime sub-position/capsule type（loggrep §1/§3/§4）。 |
| logblock | `key_mode=field`, direct transpose of log_format columns（logblock §2/§4）。 |

## (e) Cross-slot coupling notes

1. **S7 is the low-level primitive for S8 and S9 outputs**. Dictionary ID streams from S8 and delta/residual numeric values from S9 can both be represented as `EncodedInts`; this is why unifying S7 first unlocks the largest downstream reuse（denum §3；delog §3；logreducer §3；logshrink §3；lognexus §3）。

2. **S6 column organizer is the common input face for S5/S7/S8/S9**. The strongest cross-algorithm boundary is not a parser wrapper but `ColumnStore.columns: map<ColumnKey, vector<Value>>`; once columns are keyed by `(template_id, var_position)`, `tag`, `field`, or `capsule`, S5 can type them and route numeric values to `S9 -> S7`, string values to `S8 -> S7`, or search values to branch-specific Capsule/segment logic（logreducer §2/§4；denum §4；delog §4；clp §4；loggrep §3/§4）。

3. **S4 and S5 must remain separate**. S4 injects dataset-specific prior knowledge and returns masked lines plus extracted tag values; S5 classifies already-extracted variables/columns into encoding routes. Folding them together would hide the fact that DeLog/Denum/LogNexus share the 16-dataset regex table while LogReducer/LogShrink/CLP still share variable typing without that table（delog §1.1/§3；denum §1/§2；lognexus §1/§3；logreducer §2；logshrink §2；clp §2）。

4. **S11 is the universal join point, with one exception**. Trunk algorithms converge on directory/archive backends; LogBlock joins shallowly through general compressors; CLP/LogGrep join the compression-family concept but retain search-aware containers; LogLite joins only as external cascade; LogGzip does not truly join because gzip is only an NCD oracle（delog §5；denum §5；logreducer §5；loggrep §5；clp §5；loglite §5；loggzip §5）。

5. **S12 must stay optional**. Only CLP and LogGrep expose `query -> matching rows/bitmap` over compressed data; forcing S12 into the trunk would distort the other 10 algorithms, whose indices are decompression metadata rather than user-visible search indexes（clp §7；loggrep §7；delog §7；denum §7；logreducer §7；logshrink §7；lognexus §7；logfold §7；logzip §7；logblock §7；loglite §7；loggzip §7）。
