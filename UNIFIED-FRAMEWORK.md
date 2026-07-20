# Unified Log-Compression Framework

本文是 `docs/framework/_unified_framework_designA.md` 与 `docs/framework/_unified_framework_designB.md` 的融合版权威设计。设计依据是 `CLUSTERING-CONSENSUS.md` 的 12-slot 共识、per-slot data contract、variant flags、trunk/branch/outlier map；涉及实际物理文件名时，只采用 `docs/algorithms/*` 中已经钉住的算法事实。

核心结论必须先说清：本框架的统一点是 **shared data contract**，不是 wrapper/adapter。每个 slot 的实现必须直接消费同一个 input type、直接产出同一个 output type；算法差异只能写进 `Spec`、`Descriptor` 或 manifest。若某算法必须先生成私有结构，再用额外转换函数翻译成框架类型，它就没有 fit 该 slot：它要么是合法 branch，要么是 S11 external cascade，要么是 reference-only。这个判定沿用共识 (c) trunk/branch/outlier map 与共识 (e2) `ColumnStore`、(e1) `EncodedInts`、(e4) S11 join point、(e5) optional S12 的边界。

## 0. Wrapper 反模式判定规则

一个实现 fit 某个 slot，当且仅当它满足以下条件：

| 判定项 | Fit slot | Wrapper / outlier |
|---|---|---|
| 输入输出 | `Sn(input: FixedInType, spec: SnSpec) -> FixedOutType`，类型逐字一致 | 先产出算法私有类型，再另写转换器 |
| 差异表达 | 差异只在 `SnSpec`、`TransformDescriptor`、`ColumnDescriptor`、`ChunkManifest` 中 | 差异藏在算法名分支或私有目录解析器里 |
| 下游可见面 | 下游只读共享 IR，例如 `ColumnStore`、`EncodedInts`、`DictionaryResult`、`UnifiedChunk` | 下游需要知道 Denum/DeLog/LogReducer 的私有文件命名 |
| 不满足时 | 标成 branch / external cascade / reference-only | 强行塞进 trunk |

因此，`DenumAdapter -> DenumFiles`、`LogReducerAdapter -> LogReducerFiles`、`LogNexusAdapter -> LogNexusFiles` 不是本设计。真正的替换面是：

```text
Any S3 TemplateMiner     -> ParsedBatch
Any S6 ColumnOrganizer   -> ColumnStore
Any S7 NumericEncoder    -> EncodedInts
Any S8 StringDictionary  -> DictionaryResult
Any S9 NumericTransform  -> TransformStore
Any S10 ChunkPersistor   -> UnifiedChunk
```

这条规则直接来自共识 (a) S1-S12 slot list、共识 (c) pure trunk / searchable branch / shallow branch / outlier map、共识 (e) cross-slot coupling notes。算法文档也支持这个边界：LogLite 没有 S2-S9 语义列，只能是 S11 external cascade（loglite §1/§3/§4/§5；共识 S11）；LogGzip 的 gzip 是 NCD length oracle，落盘是 parser CSV，不是 storage compressor artifact（loggzip §3/§4/§5；共识 c/e4）。

## 1. Pipeline Stage Graph

### 1.1 线性主干 + fork/rejoin

主干是一条 typed pipeline。S4 与 S12 是可选或条件节点；进入 encode 段后，S7/S8/S9 不再是 pipeline 上按固定顺序排开的 stage，而是统一 encode layer 内的 operator-category labels。不开启某类 operator 时，只是同一 encode contract 上的 empty/none route，不是 wrapper。

```text
raw bytes
   │
   ▼
[S1] Line framing / header split       RawInput -> RecordBatch
   │                                   Record{line_id, headers, body, newline, status}
   ▼
[S2] Tokenizer / delimiter splitter     RecordBatch -> TokenizedBatch
   │                                   TokenSeq with delimiters preserved
   ├────────────► [S4] Regex pre-extractor (optional dataset prior)
   │                 RecordBatch/TokenizedBatch -> RegexExtractionResult
   │                 masked_batch + extracted columns ─────┐
   ▼                                                       │
[S3] Template / pattern miner           PipelineSpec selects original or S4-masked TokenizedBatch
   │                                   -> ParsedBatch{pattern_id, ordered variables}
   ▼
[S6] Column / field organizer           ParsedBatch + RegexExtractionResult?
   │                                   -> ColumnStore{columns,row_order,failed_streams}
   │                                   columns already split to algorithm-minimal encode units
   ▼
[S5] Variable typer / encode router     ColumnStore -> TypedColumnStore
   │                                   per-column type + EncodeRoute{slot_labels, plan_selector}
   ▼
[Encode layer] Encoder pool / planner   TypedColumnStore -> EncodedColumnSet fragments
   │
   ├── chain example (.var_dict):
   │      [S8] dict_lookup -> [S9] successive_delta -> [S7] zigzag_varint
   │
   ├── DAG example (LogReducer correlation):
   │      TransformGraph-backed [S9] correlation_residual -> [S7] numeric_codec
   │
   └── direct/raw/all_same route -> metadata / raw stream / single [S7] op
   │
   ▼
[S10] Persistence container             EncodedColumnSet + ChunkManifest
   │                                   -> UnifiedChunk (typed stream directory)
   ▼
[S11] Backend archiver                  UnifiedChunk | BranchChunk | ExternalByteStream
   │                                   -> ArchiveResult
   ▼
archive on disk
   ┊
   └┈┈┈► [S12] Searchable index (optional branch)
          UnifiedChunk + dict/segment/capsule metadata -> SearchIndexResult
```

依据：共识 S1-S12 定义了上述 slot；共识 (e3) 要求 S4 与 S5 分离；共识 (e1) 要求 S7 仍是 S8 ID stream 与 S9 residual 的低层 primitive；本次 encode-layer 重构只把 S7/S8/S9 从固定 stage 改成同一 encoder pool 的类别标签；共识 (e4) 把 S11 定为 universal join point；共识 (e5) 要求 S12 optional。

### 1.2 Mandatory-trunk 与 optional slots

| Slot | 主干状态 | 输入 -> 输出 | 说明 |
|---|---|---|---|
| S1 Line framing / header split | mandatory | `RawInput -> RecordBatch` | 所有成员至少需要 record boundary；LogReducer/LogShrink `head.format`、CLP timestamp boundary、LogBlock `log_format` 都落在这里（共识 S1；logreducer §2；logshrink §1；clp §1；logblock §1）。 |
| S2 Tokenizer / delimiter splitter | mandatory for semantic trunk | `RecordBatch -> TokenizedBatch` | DeLog fixed delimiters、Denum/LogShrink mined delimiters、LogGrep fixed delimiters、CLP variable-boundary tokenization、LogFold skeleton tokenizer 共享 token/delimiter 输出面（共识 S2）。 |
| S3 Template / pattern miner | mandatory for trunk, variant-heavy | `TokenizedBatch -> ParsedBatch` | 输入是 original S2 stream 还是 S4 `masked_batch` 由 `PipelineSpec.s3_input_stream` 声明；LengthParser、signature synthesis、numeric stripping、URT/skeleton、CLP variable schema 都必须产出 `pattern_id + ordered variables`（共识 S3/S4；denum §1/§2；delog §1/§4；lognexus §1）。 |
| S4 Metadata / regex pre-extractor | optional trunk | `RecordBatch | TokenizedBatch -> RegexExtractionResult` | DeLog/Denum/LogNexus 共享 16-dataset regex lineage，输出 first-class `masked_batch`、tag columns、application order 与 raw span backrefs；CLP 有 timestamp-only table；LogReducer/LogShrink 可关闭（共识 S4/e3）。 |
| S5 Variable typer | mandatory after S6 | `ColumnStore -> TypedColumnStore` | 对列判 `numeric/string/dict/all_same/raw/runtime_*`，并写 `encoding_route{slot_labels,plan_selector}`；它是 encode layer 的入口路由，不直接执行 S7/S8/S9（共识 S5）。 |
| S6 Column / field organizer | mandatory | `ParsedBatch + RegexExtractionResult? -> ColumnStore` | 这是主干脊柱；职责是把列拆到“该算法的最小 encode 单元”，后续 S5 与 encode layer 只看 `ColumnStore.columns`（共识 S6/e2）。 |
| S7 Numeric encoder | mandatory encode-layer category | `IntColumnBundle -> EncodedInts` | S7 现在是 encode layer 的 `numeric-codec` 类 operator label；ZigZag + 7-bit little-endian varint 是 trunk 最强共享点，plaintext、unsigned、fixed64 是 spec/operator 变体（共识 S7；logreducer §3；denum §3；delog §3；lognexus §3）。 |
| S8 String dictionary | conditional per column, mandatory capability | `StringColumnBundle -> DictionaryResult` | S8 现在是 `dictionary` 类 operator label；`dict_lookup` 等算子产出 value->ID + mapping + ID stream，ID stream 默认复用 S7；是否继续接 S9/S7 由算子链决定（共识 S8/e1）。 |
| S9 Delta / correlation transform | optional per column/graph | `TypedColumnStore -> TransformStore` | S9 现在是 `transform` 类 operator label；successive delta 与 LogReducer correlation residual 通过同一 contract 表达，可作为单列 chain 或跨列 `TransformGraph` DAG（共识 S9；logreducer §3）。 |
| S10 Persistence container | mandatory | `EncodedColumnSet + ChunkManifest -> UnifiedChunk` | archive 前 wire shape；主干多文件目录抽象为 typed stream directory（共识 S10）。 |
| S11 Backend archiver | mandatory join | `UnifiedChunk | BranchChunk | ExternalByteStream -> ArchiveResult` | tar/xz、7z/LZMA、zstd、per-capsule LZMA、external cascade 都在这里表达（共识 S11）。 |
| S12 Searchable index | optional branch | `UnifiedChunk + SearchIndexSpec -> SearchIndexResult` | 仅 CLP/LogGrep 具备压缩态 query/search；不能污染 trunk baseline（共识 S12/e5；clp §7；loggrep §7）。 |

### 1.3 Branch fork / rejoin

```text
Semantic trunk:
  S1 -> S2 -> S4? -> S3 -> S6 -> S5 -> EncodeLayer{per-column chain | cross-column DAG using S7/S8/S9 labels} -> S10 -> S11

Searchable branch:
  S1/S2/S3 -> S6 -> S5 -> branch encode/search variants -> S10(search-aware) -> S12 -> S11
  or S10(search-aware) -> S11 -> S12-open

LogBlock shallow branch:
  S1(log_format) -> S6(field direct transpose) -> S5(column heuristics) -> S10(single preprocessed text) -> S11

LogLite external cascade:
  raw line bytes -> LogLite byte stream -> S11 ExternalByteStream cascade

LogGzip reference-only:
  S3 parser/NCD reference -> plaintext CSV, no storage-compression chunk contract
```

Searchable branch 的 rejoin 是 S10/S11：CLP 的 segment + dictionaries + segindex 与 LogGrep 的 CapsuleBox 都可以作为 `UnifiedChunk.search_extensions` / branch typed streams，再由 S11 用 zstd/passthrough 或 per-capsule LZMA 归档（共识 c/S10/S11/S12；clp §4/§5/§7；loggrep §4/§5/§7）。LogLite 不产生 `Record`、`ColumnStore`、`DictionaryResult`，只接 S11 external cascade（共识 c/e4；loglite §4/§5）。LogGzip 是 reference-only，不生成可统一解压的 storage artifact（共识 c/e4；loggzip §4/§5）。

## 2. Core Unified Data Structures / IR

以下是唯一 canonical IR type set。字段名在全文一致使用；算法私有命名只能出现在 `physical_name`、`source_profile` 或物理映射表中。

### 2.1 Scalar, identity, and losslessness boundary

```text
LineId = uint64
ChunkId = string
TemplateId = string | uint64
PathId = string | uint64
ColumnId = string
Tag = string
SlotRef = "S1" | "S2" | "S3" | "S4" | "S5" | "S6" | "S7" | "S8" | "S9" | "S10" | "S11" | "S12"

Value = IntValue | FloatTextValue | StringValue | BytesValue | NullValue

PreservationProfile {
  byte_identity: bool
  line_order: "preserve" | "normalized" | "unknown"
  newline_policy: "preserve" | "normalize_lf" | "append_lf" | "drop" | "unknown"
  final_newline_policy: "preserve" | "force_present" | "force_absent" | "unknown"
  whitespace_policy: "preserve" | "token_separator_normalized" | "strip_or_relaxed" | "unknown"
  timestamp_precision: "raw" | "milliseconds" | "token_text" | "not_applicable" | "unknown"
  token_boundary_policy: "preserve_delimiters" | "token_stream_only" | "schema_reformatted" | "opaque"
  raw_line_available: bool
  non_utf8_policy: "preserve_bytes" | "reject" | "archive_fallback" | "unknown"
  strict_writer_allowed: bool
}

IntValue {
  raw_text?: string
  value: int64 | uint64
  numeric_domain: "signed" | "unsigned" | "timestamp" | "ip_packed" | "dictionary_id" | "path_id"
}

FloatTextValue {
  raw_text: string
  decimal_encoding?: bytes
  preservation: "raw_text" | "decimal_semantics"
}

StringValue {
  value: string
  preservation: "raw" | "normalized" | "token_level"
}

BytesValue { value: bytes }
NullValue { reason: "missing" | "not_applicable" | "masked" }
```

`PreservationProfile` 是显式合同，不是注释。算法文档显示 losslessness 边界不同：LogReducer/LogShrink 目标是文本重建并保留 failed streams（logreducer §6；logshrink §6）；LogNexus artifact 明确是 whitespace-separated token stream equality 而非 byte identity（lognexus §6/§8）；Denum 当前源码只能支持 normalized/whitespace-relaxed 口径（denum §6/§8）；CLP 是 UTF-8 text log 的 line-level reconstruction，timestamp precision 限到 milliseconds，非 UTF-8 走 archive fallback 或拒绝（clp §5）；LogBlock source H2 wire format 不可逆（logblock §3/§6）；LogGzip reference-only（共识 c/e4）。Manifest 必须声明完整 `PreservationProfile`，strict writer 只允许 `strict_writer_allowed=true` 的 chunk。

### 2.2 Record and token contracts

```text
RawInput {
  bytes: bytes
  source_name?: string
  log_format?: LogFormatSchema
}

Record {
  line_id: LineId
  chunk_id: ChunkId
  physical_span: { start_byte?: uint64, end_byte?: uint64 }
  headers: map<string, Value>
  body: string
  raw_line?: bytes
  newline: "lf" | "crlf" | "none" | "normalized" | "unknown"
  status: "normal" | "load_failed" | "match_failed" | "opaque"
}

RecordBatch {
  chunk_id: ChunkId
  records: vector<Record>
  framing: FramingDescriptor
  metadata: map<string, Value>
}

Token {
  text: string
  kind: "token" | "delimiter" | "placeholder" | "newline" | "static" | "variable_marker"
  source_span?: { record_line_id: LineId, start: uint32, end: uint32 }
}

TokenSeq {
  line_id: LineId
  tokens: vector<Token>
}

TokenizedRecord {
  line_id: LineId
  headers: map<string, Value>
  sequence: TokenSeq
}

TokenizedBatch {
  chunk_id: ChunkId
  sequences: vector<TokenSeq>
  tokenizer_spec: TokenizerSpec
}
```

依据：共识 S1 要求 `Record{line_id, headers, body}`；共识 S2 要求 token/delimiter sequence 且通常保留 delimiter。`newline` 与 `physical_span` 只记录 boundary contract，避免把 LogLite/LogGzip 这类字节级或 reference-only outlier 伪装成 semantic trunk。

### 2.3 Regex extraction and parsed records

```text
RegexExtractionResult {
  chunk_id: ChunkId
  masked_batch: TokenizedBatch
  extracted: map<ColumnKey, vector<Value>>
  tag_metadata: map<Tag, RegexTagMeta>
  application_order: vector<Tag>
  raw_span_map: vector<RawSpanBackref>
}

RawSpanBackref {
  line_id: LineId
  placeholder_token_index: uint32
  column_key: ColumnKey
  column_occurrence_index: uint64
  raw_span: { start_byte?: uint64, end_byte?: uint64, start_char?: uint32, end_char?: uint32 }
  raw_text_available: bool
}

PipelineSpec {
  s3_input_stream: "original_s2" | "s4_masked_batch"
  s4_enabled: bool
  preservation: PreservationProfile
  sampling?: SamplingPlan
  block?: BlockPlan
  execution?: ExecutionPlan
}

ParamResolution<T> {
  source_default: T
  experiment_override?: T
  effective: T
  override_status: "none" | "applied" | "not_applicable" | "rejected"
  provenance: {
    algorithm_profile: string
    source_evidence: vector<string>
    override_reason?: string
  }
}

SamplingPlan = ParamResolution<SamplingProfile>
BlockPlan = ParamResolution<BlockProfile>
ExecutionPlan = ParamResolution<ExecutionProfile>

SamplingProfile {
  phases: vector<SamplingPhase>
}

SamplingPhase {
  purpose: "none" | "template_training" | "delimiter_mining" | "runtime_pattern_mining" | "sequence_analyzer" | "encoding_probe" | "external_templates" | "parser_reference"
  unit: "line" | "sequence" | "value" | "byte" | "none"
  mode: "none" | "full_scan" | "external_templates" | "random_ratio" | "sample_rate" | "distinct_length_cap" | "first_n" | "minmax_ratio" | "reference_oracle"
  ratio?: float
  sample_rate?: float
  sample_size?: uint64
  min_samples?: uint64
  max_samples?: uint64
  max_iterations?: uint64
  window_size?: uint64
  n_candidates?: uint64
  contiguous_lines_per_sample?: uint64
  selection_threshold?: float
  random_seed?: uint64
  source_knob?: string
  overrideable: bool
  notes?: string
}

BlockProfile {
  unit: "line" | "byte" | "segment" | "file" | "line_window" | "none"
  size_lines?: uint64
  size_bytes?: uint64
  allowed_size_bytes?: vector<uint64>
  target_uncompressed_bytes?: uint64
  target_encoded_bytes?: uint64
  target_dictionary_bytes?: uint64
  window_size_lines?: uint64
  max_line_bytes?: uint64
  boundary_policy: "fixed_line_count" | "fixed_byte_window_drop_partial_line" | "segment_threshold" | "whole_file" | "line_window" | "none"
  archive_scope: "per_block" | "per_segment" | "whole_input" | "external_stream" | "reference_only"
  source_knob?: string
  overrideable: bool
  notes?: string
}

ExecutionProfile {
  mode: "single_threaded"
  block_processing: "sequential"
  parallelism_enabled: false
  block_order: "input_order"
  rationale: string
  source_parallel_knobs_ignored?: vector<string>
}

ParsedRecord {
  line_id: LineId
  pattern_id: TemplateId | PathId
  pattern_tokens: vector<Token>
  variables: vector<VariableOccurrence>
  parser_status: "matched" | "load_failed" | "match_failed" | "unparsed"
  parser_trace?: map<string, Value>
}

VariableOccurrence {
  position: uint32
  sub_position?: uint32
  tag?: Tag
  raw_value: string
  normalized_value?: Value
  source: "template_wildcard" | "regex_tag" | "numeric_token" | "urt_residual" | "field" | "capsule"
}

ParsedBatch {
  chunk_id: ChunkId
  records: vector<ParsedRecord>
  templates: map<TemplateId, TemplateDescriptor>
  failures: FailedStreams
  metadata: map<string, Value>
}
```

`sampling`、`block`、`execution` 三个 common-flow 字段全部 optional：`loggzip` 是 reference-only，`loglite` 是 external byte stream，若干算法没有 template sampling 或 storage block；强制默认会把 outlier 伪装成 trunk。三者都使用 `ParamResolution<T>` dual-constraint envelope：`source_default` 复现当前源码/论文 profile，`experiment_override` 是 controlled-variable 实验输入，`effective` 是 writer 实际使用的值。resolution precedence 必须完整写入 manifest：第一，无 override 时 `effective=source_default` 且 `override_status="none"`；第二，override 存在且与算法支持的 `unit` / `purpose` / `mode` 兼容时，`effective=experiment_override` 且 `override_status="applied"`；第三，算法根本没有该 phase/block 概念却被尝试 override 时，`override_status="not_applicable"`，例如给 LogGzip 施加 storage block，或给 LogLite 施加 semantic sampling；第四，概念存在但 override 的 `unit` / `purpose` / `mode` 不兼容时，`override_status="rejected"`，例如给 CLP segment block 强设 line block，或给 LogBlock byte chunk 强设 line count。实验策略也必须显式：`reject_incompatible` 让本次 run 失败，`skip_incompatible` 则把该算法从 controlled-variable subset 中剔除。任何情况下都不能 silently convert units。`execution` 本轮固定为 `ExecutionProfile{mode="single_threaded", block_processing="sequential", parallelism_enabled=false, block_order="input_order"}`；source thread/worker knobs 只进 provenance，block 仍是 compression data/archive unit，不是 scheduling contract。

Common-param enum evidence 摘要：

| enum surface | source-backed justification |
|---|---|
| `SamplingPhase.purpose="template_training"` | LogReducer `--SampleRate` + `sampler.py` contiguous samples；LogGrep static template `sampleRange=100`（logreducer §0/§8；loggrep §1/§8）。 |
| `SamplingPhase.purpose="delimiter_mining"` | Denum distinct-length sample cap 10、max 200 iterations、threshold `>10`、fallback space（denum §2）。 |
| `SamplingPhase.purpose="runtime_pattern_mining"` | LogGrep `Union(mbuf,temp,0.0001)` with `min(max(_tot*0.0001,1000),_tot)`（loggrep §2）。 |
| `SamplingPhase.purpose="sequence_analyzer"` | LogShrink `-S` sequence sampler with `sample_rate`、`h`、`n_candidates`（logshrink §3）。 |
| `SamplingPhase.purpose="encoding_probe"` | LogFold dynamic delta probes first ten values before route selection（logfold §3）。 |
| `SamplingPhase.purpose="external_templates"` | LogZip source consumes external templates file and lacks source-backed ISE sampling（logzip §0/§1/§5/§8）。 |
| `SamplingPhase.purpose="parser_reference"` | LogGzip gzip-NCD is parser/reference oracle, not storage sampling（loggzip §3/§5）。 |
| `SamplingPhase.purpose="none"` | CLP, LogBlock, LogLite, LogNexus have no framework sampling phase in the relevant sense（CONSISTENCY-CHECK §A.1）。DeLog belongs with `mode="full_scan"` evidence rather than `mode="none"` because it still scans the input as the sample. |
| `SamplingPhase.mode="full_scan"` | Algorithms that scan the whole available input rather than consuming a bounded sampling budget: DeLog performs a single pass over each block for signature/template construction, and LogGzip scans parser input through the full gzip-NCD template reference path（delog §1/§4；loggzip §1/§3/§5）。This differs from `mode="none"`: the whole input is the sample. |
| `SamplingPhase.mode="none"` | No sampling phase exists at all, so there is no scan-as-sample concept to tune; this is the source-default shape for CLP, LogBlock, LogLite, and LogNexus template-sampling absence（CONSISTENCY-CHECK §A.1；loglite §1/§2）。 |
| `SamplingPhase.mode` values `random_ratio`, `sample_rate`, `distinct_length_cap`, `minmax_ratio`, `first_n`, `external_templates`, `reference_oracle` | Respectively LogGrep static sampling, LogReducer/LogShrink sample rates, Denum delimiter mining, LogGrep runtime mining, LogFold probe, LogZip side input templates, LogGzip reference oracle（algorithm refs above）。 |
| `BlockProfile.unit="line"` / `boundary_policy="fixed_line_count"` | DeLog, Denum, LogReducer, LogShrink, LogNexus, LogFold use line-count block/chunk profiles, commonly 100K lines（respective §0）。 |
| `BlockProfile.unit="byte"` / `boundary_policy="fixed_byte_window_drop_partial_line"` | LogBlock byte chunks 16K/32K/64K/128K with dropped partial final line; LogGrep 64MB log blocks（logblock §0；loggrep §0）。 |
| `BlockProfile.unit="segment"` / `boundary_policy="segment_threshold"` | CLP segment thresholds for uncompressed, encoded, and dictionary sizes（clp §0/§4）。 |
| `BlockProfile.unit="file"` / `boundary_policy="whole_file"` | LogZip whole DataFrame/tmp CSV/JSON/tar flow has no block-size knob（logzip §0/§4/§5）。 |
| `BlockProfile.unit="line_window"` / `boundary_policy="line_window"` | LogLite same-length line window, `EACH_WINDOW_SIZE=8`, `MAX_LEN=10000`（loglite §1/§3/§4）。 |
| `BlockProfile.unit="none"` / `boundary_policy="none"` | LogGzip has no storage block; parser CSV/reference profile only（loggzip §4/§5/§6）。 |
| `BlockProfile.boundary_policy="none"` | No storage block boundary exists at all; this is only valid with `unit="none"`, for reference-only profiles such as LogGzip where the output is plaintext parser CSV and gzip length is an oracle rather than an archive boundary（loggzip §4/§5/§6；framework §6.3）。 |

S4 的输出是 canonical stream，不是给 S3 的旁路提示：`masked_batch` 是 S3 可以直接消费的 `TokenizedBatch`，`extracted` 是按 `ColumnKey` 对齐的 tag values，`raw_span_map` 把 masked placeholder token 回指到原始 raw span 与 extracted column occurrence。`PipelineSpec.s3_input_stream` 必须在 manifest 中声明 S3 挖 original S2 stream 还是 S4 masked stream；Denum 的 regex replacement before `variable_extract`、DeLog 的 modified lines with compact IDs、LogNexus 的 placeholder extraction 都必须通过该字段表达，不能由算法名分支决定（共识 S4；denum §1/§2；delog §1/§4；lognexus §1/§3）。S1/S2 reconstruction 使用 `raw_span_map` 与 `PreservationProfile` 决定是恢复 raw text、normalized token text 还是 token-stream equality。

S3 的共同输出面是 `template_id/pattern_id + ordered variables`（共识 S3）。LogNexus 的 `pathID` 与 URT residual、LogFold skeleton ID、LogReducer `Eid`、Denum `logs without numbers` dictionary ID 都是该 semantic role 的不同 `pattern_id` 表达；差异在 `TemplateSpec`、`ColumnSpec` 与 `PipelineSpec.s3_input_stream`，不是 wrapper。

### 2.4 ColumnStore: trunk 的核心 IR

```text
ColumnKey {
  column_id: ColumnId
  key_mode: "template_position" | "tag" | "field" | "path_residual" | "capsule" | "segment" | "direct_transpose"
  template_id?: TemplateId
  path_id?: PathId
  tag?: Tag
  var_position?: uint32
  sub_position?: uint32
  field_name?: string
  branch: "trunk" | "search" | "direct_transpose" | "external"
  semantic_role: "header" | "template_id" | "variable" | "metadata" | "row_order" | "failed" | "residual" | "search_payload"
}

Column<T = Value> {
  key: ColumnKey
  values: vector<T>
  row_refs: vector<LineId>
  occurrence_refs?: vector<CellRef>
  nulls?: Bitmap
  source_slot: SlotRef
  ordering: "line_order" | "template_occurrence_order" | "tag_occurrence_order" | "segment_order" | "capsule_order"
}

CellRef {
  line_id: LineId
  placeholder_index: uint32
  column_key: ColumnKey
  column_occurrence_index: uint64
}

PlaceholderBinding {
  slot_index: uint32
  placeholder_index: uint32
  column_key: ColumnKey
  source: "template_wildcard" | "regex_tag" | "signature_tag" | "path_residual" | "runtime_capsule" | "field"
  repeat_policy: "consume_next" | "reuse_same_value" | "literal_if_missing" | "failed_stream"
}

TemplateDescriptor {
  template_id: TemplateId | PathId
  tokens: vector<Token>
  placeholders: vector<PlaceholderBinding>
  grammar_ref?: DescriptorRef
}

OccurrenceStream {
  descriptor: OccurrenceStreamDescriptor
  cells: vector<CellRef>
}

RowOrderEntry {
  line_id: LineId
  pattern_id?: TemplateId | PathId
  status: "normal" | "load_failed" | "match_failed" | "opaque"
  occurrence_index?: uint64
}

FailedStreams {
  load_failed: vector<Record>
  match_failed: vector<Record>
  outliers: map<string, vector<Record | VariableOccurrence>>
}

ColumnStore {
  chunk_id: ChunkId
  columns: map<ColumnKey, Column>
  row_order: vector<RowOrderEntry>
  occurrence_stream?: OccurrenceStream
  failed_streams: FailedStreams
  dictionaries?: map<string, DictionaryResult>
  metadata: map<string, Value>
}
```

这是共识 (e2) 的 `ColumnStore.columns: map<ColumnKey, vector<Value>>` 的完整化。不同算法只改变 `ColumnKey.key_mode`：LogReducer/LogShrink 是 `(template_id,var_position)`；Denum/DeLog 是 tag group；LogNexus 是 `pathID + residual`；CLP 是 segment three-column；LogGrep 是 capsule；LogBlock 是 direct transpose（共识 S6；logreducer §4；denum §4；delog §4；lognexus §4；clp §4；loggrep §4；logblock §4）。

`row_order` 只回答“哪一行用哪个 pattern/status”，不回答“这一行第 N 个 placeholder 取哪个列的第几个值”。因此 `TemplateDescriptor.placeholders` 与 `OccurrenceStreamDescriptor`/decoded `OccurrenceStream.cells` 是 decode-critical：`template_position` 模式可由 `template_id + placeholder_index` 推导 `ColumnKey`，但 tag/signature/path-residual/capsule 模式必须显式 emit `CellRef{line_id, placeholder_index, column_key, column_occurrence_index}`。Denum 的 `allids.bin`/`variablesetids.bin`/tag `.bin`、DeLog 的 modified template + compact tag providers + `index.normal.csv` offset、LogReducer/LogShrink 的 event/template ID + per-template var position + failed sentinels 都必须落到这个 mapping，S6 `unorganize` 才能不写“consume next <I> value”这类算法分支（denum §4/§6；delog §4/§6；logreducer §4/§6；logshrink §4/§6）。

### 2.5 Typed columns and routing-is-data

```text
ColumnType = "numeric" | "string" | "dict" | "all_same" | "raw" | "runtime_real" | "runtime_nominal" | "opaque_bytes"

EncodeRoute {
  slot_labels: vector<SlotRef>
  plan_selector: string
}

TypedColumn {
  key: ColumnKey
  raw_values: vector<Value>
  type: ColumnType
  numeric_values?: vector<int64 | uint64>
  string_values?: vector<string>
  encoding_route: EncodeRoute
  type_trace: map<string, Value>
}

TypedColumnStore {
  chunk_id: ChunkId
  typed_columns: map<ColumnKey, TypedColumn>
  row_order: vector<RowOrderEntry>
  failed_streams: FailedStreams
  metadata: map<string, Value>
}
```

`EncodeRoute.slot_labels` 只记录 S5 判定后的兼容性标签 footprint（通常来自 `{S7,S8,S9}`），不是精确执行顺序；真正的 per-column 次序/图形表达在 `encode_ops` / `encode_graph_ref` 中落盘。这样 `encoding_route` 仍保持 routing-is-data，而 `.var_dict` 这类复合路径、LogReducer correlation 这类跨列 DAG 则不再被压扁成粗粒度 `SlotRef` 排列（共识 S5；delog §3；denum §2/§3；logshrink §3；loggrep §2/§3；clp §2）。

### 2.6 EncodedInts, dictionaries, transforms, encoded columns

```text
IntColumnBundle {
  key: ColumnKey
  values: vector<int64 | uint64>
  row_refs: vector<LineId>
}

StringColumnBundle {
  key: ColumnKey
  values: vector<string | bytes>
  row_refs: vector<LineId>
}

EncoderInputRef {
  kind: "column_values" | "op_output" | "graph_node" | "dict_mapping" | "literal"
  ref: string
}

EncoderOutputRef {
  ref: string
  semantics: "column_values" | "dictionary_id" | "delta" | "correlation_residual" | "encoded_bytes" | "raw_passthrough"
}

EncoderOp {
  op_id: string
  slot_label: "S7" | "S8" | "S9"
  category: "numeric_codec" | "dictionary" | "transform"
  operator: string
  inputs: vector<EncoderInputRef>
  params: map<string, Value>
  output: EncoderOutputRef
  artifact_ref?: string
  inverse_available: bool
}

EncodedInts {
  codec: CodecDescriptor
  bytes: bytes
  count?: uint64
  count_prefix: bool
  signed_mode: "zigzag64" | "unsigned" | "none"
  continuation_bit: "msb_1_more_0_stop"
  value_semantics: "raw" | "delta" | "correlation_residual" | "dictionary_id" | "path_id"
}

DictionaryResult {
  dict_id: string
  scope: "trunk_chunk" | "trunk_template" | "trunk_tag" | "trunk_column"
  id_base: 0 | 1
  ordering: "first_occurrence" | "sorted_by_id" | "pattern_segment_order" | "source_order"
  mapping: vector<string | bytes>
  id_stream: EncodedInts
  metadata: map<string, Value>
}

SearchDictionaryDescriptor {
  dict_id: string
  branch: "clp_segment" | "loggrep_capsule" | "logzip_base64" | "logblock_inline"
  mapping_stream: StreamName
  id_stream_descriptor?: IdStreamDescriptor
  segment_index?: SegmentIndexDescriptor
  capsule_ref?: string
  decode_contract: "branch_descriptor"
}

TransformDescriptor {
  transform_id: string
  mode: "none" | "successive_delta" | "transform_graph" | "dict_index_diff" | "mixed_parity" | "logblock_h2_source_compat"
  first_value_mode?: "raw_first"
  inverse_available: bool
  graph_ref?: TransformGraphRef
}

TransformGraphRef { transform_id: string }

TransformGraph {
  transform_id: string
  nodes: vector<TransformNode>
  edges: vector<TransformEdge>
  deps: TransformDeps
  coverage: vector<TransformCoverage>
  profile: "direct_only_lossless" | "correlation_graph_lossless" | "source_compat_nonrestorable"
  inverse_available: bool
}

TransformNode {
  node_id: string
  kind: "source_column" | "temporary_column" | "emitted_residual"
  column_key?: ColumnKey
  physical_name?: string
}

TransformEdge {
  edge_id: string
  kind: "direct" | "do" | "up" | "doi" | "upi" | "diff2" | "diff3"
  inputs: vector<string>
  output: string
}

TransformDeps {
  id_columns: vector<ColumnKey>
  row_grouping: "template_occurrence" | "id_grouped_previous_next" | "line_order"
}

TransformCoverage {
  original_column: ColumnKey
  reconstructable_from: vector<string>
  lossless: bool
}

DeltaResult {
  key: ColumnKey
  output_key: ColumnKey
  values: vector<int64 | uint64>
  descriptor: TransformDescriptor
}

TransformStore {
  chunk_id: ChunkId
  transforms: map<ColumnKey, DeltaResult>
  source_columns: TypedColumnStore
}

EncodedColumn {
  key: ColumnKey
  physical_name: string
  logical_type: "encoded_ints" | "dictionary" | "raw_string" | "raw_bytes" | "failed_stream" | "metadata"
  payload_ref: string
  encode_route: EncodeRoute
  encode_ops: vector<EncoderOp>
  encode_graph_ref?: TransformGraphRef
  persisted_output_ref?: string
  reader_contract: DecodeDescriptor
}

EncodedColumnSet {
  chunk_id: ChunkId
  columns: map<ColumnKey, EncodedColumn>
  dictionaries: map<string, DictionaryResult>
  row_order: EncodedColumn
  failed_streams: map<string, EncodedColumn>
  metadata: map<string, Value>
}
```

`DeltaResult` 名称用于对齐共识 S9；`TransformStore` 是多列/多结果容器。主干 S7 是所有 trunk integer stream 的唯一 gateway：S9 的 residual/delta `values` 与 S8 trunk dictionary IDs 落盘前都必须进入 `EncodedInts`（共识 e1）。`EncoderOp` 才是 encode 路径的权威表达：单列时 `encode_ops` 是线性 chain；跨列相关性时 `encode_ops + TransformGraph` 组成 DAG，chain 只是 DAG 的退化特例。本设计选择 **strict trunk**：`DictionaryResult.id_stream` 只允许 S7 `EncodedInts`；CLP segment dictionaries、LogGrep capsule dictionaries、LogZip base64 IDs、LogBlock inline dictionaries 不再伪装成 trunk `DictionaryResult`，而是进入 `SearchDictionaryDescriptor` 或 shallow branch descriptor。这样避免 `EncodedInts` 与 fixed-width/text/base64 ID stream 同名不同义（共识 S8/S12；clp §3/§4；loggrep §3/§4；logzip §3）。LogReducer 的 plaintext fallback 是 `CodecDescriptor{kind=plaintext_decimal}`，不是单独 slot（logreducer §3）。

LogReducer `num.rule` 不是一个 opaque parameter map，而是 `TransformGraph`：`direct/do/up/doi/upi/diff2/diff3` 都是有输入、输出和 coverage 的 graph edge，ID-grouped rules 依赖 string ID columns 与 row grouping；只有 `inverse_available=true` 且 `coverage.lossless=true` 的 graph 可以进入 strict-lossless chunk。S9 算子吃的是 `EncoderOp.inputs` 指定的上游值：可以是原始 numeric column，也可以是上游 dictionary IDs，因此 `.var_dict` 这类“先 dict 再 delta”的语义不再靠 `transform_ref` 旁挂猜测。当前 source restore 不读 `num.rule`、不能逆 residual-only storage，因此 source-compatible residual-only profile 必须标 `source_compat_nonrestorable`，strict writer 要拒绝或降级（logreducer §3/§6）。

### 2.7 UnifiedChunk: one on-disk chunk format

统一 chunk 是 S10 的 shared data storage structure。它是一个 typed stream directory，可以被 tar/xz、7z/LZMA、zstd、per-capsule LZMA 等 S11 backend 包装。A 的目录布局保留为 canonical physical layout；B 的 `Chunk = TypedStreamSet`、`ChunkHeader`、`TypedStream`、`ColumnDescriptor`、`DictDescriptor` 合并为目录中的 field-level manifest。

```text
UnifiedChunk/
  manifest.json
  header.json
  records/
    row_order.uvi
    failed_load.raw
    failed_match.raw
    outliers/<stream_id>.raw
  schema/
    templates.udict
    patterns.udict
    tags.map
    framing.json
    tokenizer.json
    type_routes.json
  columns/
    <stream_name>.uenc
  dictionaries/
    <dict_id>.umap
    <dict_id>.uids
  transforms/
    <transform_id>.json
  search/
    segment_index/*
    capsule_index/*
    query_metadata.json
  extensions/
    branch_payloads/*
```

```text
UnifiedChunk {
  root_path: string
  manifest: ChunkManifest
  header: ChunkHeader
  streams: map<StreamName, TypedStream>
}

ChunkManifest {
  format: "ULCF"
  version: 1
  chunk_id: ChunkId
  algorithm_profile: string
  pipeline: PipelineSpec
  common_param_profile: {
    source_profile_id: string
    experiment_override_id?: string
    override_status: map<"sampling"|"block"|"execution", string>
    compatibility_notes?: vector<string>
  }
  slots: map<SlotRef, SlotDescriptor>
  row_count: uint64
  preservation: PreservationProfile
  row_order_ref: StreamName
  occurrence_stream?: OccurrenceStreamDescriptor
  regex_extraction?: RegexExtractionDescriptor
  schema: SchemaManifest
  columns: vector<ColumnDescriptor>
  dictionaries: vector<DictDescriptor>
  transforms: vector<TransformDescriptor>
  transform_graphs: vector<TransformGraph>
  failed_streams: vector<FailedStreamManifest>
  search_extensions?: SearchExtensionManifest
  branch_payloads?: vector<BranchPayloadDescriptor>
  backend_hint: BackendSpec
}

ChunkHeader {
  magic: "UCHK1"
  slot_specs: map<SlotRef, SpecBlob>
  column_index: vector<ColumnDescriptor>
  dict_index: vector<DictDescriptor>
  typed_streams: vector<TypedStreamDescriptor>
  format_schema?: LogFormatSchema
  regex_table_name?: string
}

TypedStream {
  name: StreamName
  kind: "encoded_ints" | "raw_bytes" | "utf8_lines" | "dict_mapping" | "json_descriptor" | "search_metadata"
  file: string
  payload?: bytes
}

TypedStreamDescriptor {
  name: StreamName
  kind: string
  file: string
  codec?: CodecDescriptor
  byte_order?: "little" | "native" | "text"
  count?: uint64
}

CodecDescriptor {
  kind: "zigzag_varint_7bit_le" | "uvarint_7bit_le" | "plaintext_decimal" | "fixed64_native" | "utf8_lines" | "raw_bytes" | "lzma_member" | "zstd_stream" | "passthrough"
  signed_mode: "zigzag64" | "unsigned" | "none"
  byte_order: "little" | "native" | "text" | "not_applicable"
  count_prefix: "uvarint" | "none" | "native_header" | "metadata_count"
  plaintext_line_framing: "lf" | "crlf" | "none" | "not_text"
  count?: uint64
}

TypedStreamSlice {
  stream: StreamName
  offset: uint64
  length: uint64
  logical_role: "primary_values" | "row_refs" | "dict_ids" | "dict_mapping" | "all_same_value" | "index_slice" | "template_ids" | "failed_records" | "branch_payload" | "masked_batch" | "raw_span_map" | "occurrence_cells"
}

RegexExtractionDescriptor {
  masked_batch_ref?: TypedStreamSlice
  extracted_column_refs: vector<{ tag: Tag, column_key: ColumnKey }>
  application_order: vector<Tag>
  raw_span_map_ref: TypedStreamSlice
  table_name?: string
}

OccurrenceStreamDescriptor {
  stream_slice: TypedStreamSlice
  codec: CodecDescriptor
  count: uint64
  cell_ref_layout: "line_placeholder_column_occurrence" | "tag_occurrence" | "signature_tag_occurrence" | "path_residual_occurrence" | "capsule_occurrence"
}

IdStreamDescriptor {
  encoding: "S7"
  stream_slices: vector<TypedStreamSlice>
  codec: CodecDescriptor
  id_base: 0 | 1
  count: uint64
}

TransformDescriptorRef { transform_id: string }

DescriptorRef { descriptor_id: string }

SchemaManifest {
  templates?: vector<TemplateSetDescriptor>
  patterns?: vector<PatternPoolDescriptor>
  framing?: FramingSchemaDescriptor
  parser_rules?: vector<ParserRuleDescriptor>
  relations?: vector<RelationDescriptor>
  external_side_inputs?: vector<ExternalSideInputDescriptor>
}

TemplateSetDescriptor {
  descriptor_id: string
  stream_ref: StreamName
  grammar: "template_col_commit_markers" | "newline_mapping" | "urt_init_tree" | "logfold_template_dictionary" | "canonical_template_json" | "event_template_mapping_json" | "dictionary_template_mapping"
  template_count?: uint64
}

PatternPoolDescriptor {
  descriptor_id: string
  stream_ref: StreamName
  grammar: "length_bucket_pool" | "signature_mapping" | "urt_variable_trees" | "runtime_subpattern_table" | "canonical_pattern_json"
}

FramingSchemaDescriptor {
  descriptor_id: string
  stream_ref: StreamName
  grammar: "head_format" | "timestamp_pattern_table" | "log_format_capture" | "canonical_framing_json"
}

ParserRuleDescriptor {
  descriptor_id: string
  stream_ref: StreamName
  grammar: "logreducer_basic_rule" | "logreducer_string_rule" | "logreducer_num_rule_graph" | "canonical_parser_rule_json"
  applies_to: vector<TemplateId>
}

RelationDescriptor {
  descriptor_id: string
  stream_ref: StreamName
  grammar: "logshrink_relations_json" | "logshrink_relations_int_array" | "canonical_relation_json"
  applies_to: "header" | "variables" | "both"
}

ExternalSideInputDescriptor {
  descriptor_id: string
  role: "template_set" | "framing_schema" | "parser_rules" | "relation_metadata" | "urt_tree"
  path: string
  version?: string
  hash: string
  required_for_decode: bool
}

ColumnDescriptor {
  key: ColumnKey
  type: ColumnType
  encoding_route: EncodeRoute
  encode_mode: "bypass" | "chain" | "dag"
  encode_ops: vector<EncoderOp>
  encode_graph_ref?: TransformGraphRef
  persisted_output_ref?: string
  codec: CodecDescriptor
  stream_slices: vector<TypedStreamSlice>
  transform_ref?: TransformDescriptorRef
  dictionary_ref?: string
  all_same_value?: Value
  row_ref_mode: "row_order" | "template_occurrence" | "column_row_refs" | "branch_defined"
  row_ref_stream?: TypedStreamSlice
  decode_order?: vector<string>
  source_profile?: string
}

DictDescriptor {
  dict_id: string
  scope: "trunk_chunk" | "trunk_template" | "trunk_tag" | "trunk_column"
  id_base: 0 | 1
  ordering: "first_occurrence" | "sorted_by_id" | "pattern_segment_order" | "source_order"
  mapping_slices: vector<TypedStreamSlice>
  id_stream_descriptor: IdStreamDescriptor
}

SearchExtensionManifest {
  dictionaries: vector<SearchDictionaryDescriptor>
  branch_payloads: vector<BranchPayloadDescriptor>
  query_metadata?: StreamName
}

BranchPayloadDescriptor = ClpSegmentChunkDescriptor | LogGrepCapsuleBoxDescriptor | ExternalByteStreamDescriptor

ClpSegmentChunkDescriptor {
  kind: "clp_segment_chunk"
  segment_ids: vector<uint64>
  segment_streams: vector<{ segment_id: uint64, stream_ref: TypedStreamSlice, compression: CodecDescriptor, uncompressed_size: uint64 }>
  native_widths: map<string, uint8>
  endian_policy: "host_native_recorded" | "little" | "big"
  column_offsets: vector<{ segment_id: uint64, role: "timestamp" | "logtype_id" | "encoded_variables", offset: uint64, count: uint64, native_type: string }>
  dict_descriptors: vector<SearchDictionaryDescriptor>
  segindex_descriptors: vector<SegmentIndexDescriptor>
  metadata_db_binding: MetadataDbBinding
}

SegmentIndexDescriptor {
  stream_ref: StreamName
  dictionary_ref: string
  record_layout: "segment_id_then_sorted_ids"
}

MetadataDbBinding {
  stream_ref: StreamName
  tables: vector<string>
  required_columns: vector<string>
}

LogGrepCapsuleBoxDescriptor {
  kind: "loggrep_capsulebox"
  header_word_size: uint8
  header_endian_policy: "host_native_recorded" | "little" | "big"
  meta_compression: "lzma_props_5b"
  capsules: vector<LogGrepCapsuleDescriptor>
  outlier_semantics: "template_failed_or_variable_fallback"
}

LogGrepCapsuleDescriptor {
  packed_name: string
  template_id: TemplateId
  variable_position?: uint32
  sub_position?: uint32
  capsule_type: "var" | "svar" | "dic" | "entry" | "outlier" | "template" | "variablelist"
  meta_line: { filename: string, compressed: bool, offset: uint64, destLen: uint64, srcLen: uint64, lines: uint64, eleLen: int64 }
  offset: uint64
  dest_len: uint64
  src_len: uint64
  lines: uint64
  ele_len: int64
  padding_mode: "left_space_fixed_width" | "newline_delimited" | "special_structure"
  lzma_props: bytes
}

ExternalByteStreamDescriptor {
  kind: "external_byte_stream"
  producer: string
  stream_ref: StreamName
  decode_contract: "external_cascade" | "reference_only"
}
```

`manifest.json` 与 `header.json` 是 reader 的唯一入口；reader 不靠文件名猜测 codec、dict scope、row order、transform、schema side input、S4 extraction streams、common-flow sampling/block/execution 或 branch metadata。`.uvi` / `.uenc` 是 typed stream 的 file suffix，不是 codec；真实 codec 在 `CodecDescriptor` 与 `ColumnDescriptor.stream_slices` 中。DeLog `index.normal.csv` 的 offset/storage type、LogReducer count-prefix-vs-plaintext、LogShrink `.ee` physical-to-logical suffix、CLP native segment arrays、LogGrep native `size_t` CapsuleBox header 都必须先归一为 descriptor 字段，不能由 `algorithm_profile` 触发隐藏 parser（delog §4；logreducer §3/§4；logshrink §4；clp §4；loggrep §4）。

`ColumnDescriptor.encode_ops` 是 encode 路径的权威表达：`encode_mode="chain"` 时，`encode_ops` 按拓扑顺序列出单列算子链；`encode_mode="dag"` 时，`encode_ops` 结合 `encode_graph_ref` 指向的 `TransformGraph` 表达跨列依赖，chain 只是 DAG 的退化特例；`encode_mode="bypass"` 则表示该列不进入 encode 算子编排，只保留 raw/all_same/metadata 旁路。`decode_order` 若保留，只是 `encode_ops` 的逆拓扑 `op_id` 视图，用于 reader quick path；它不再用 `SlotRef` 粗粒度复写语义。`dictionary_ref` / `transform_ref` 若出现，只是便于定位 materialized artifact 的 shortcut；reader 仍必须以 `slot_label + operator + inputs + params` 为准，而不是把同一个 `SlotRef` 的语义寄托在旁挂引用上。

因此 logshrink `.var_dict` 可以直接写成 `encode_mode="chain" + encode_ops=[dict_lookup, successive_delta, zigzag_varint]`，其中 `dict_lookup` 产出 dictionary IDs、`successive_delta` 消费该 ID stream、`zigzag_varint` 负责最终字节编码，`decode_order=[zigzag_varint, successive_delta, dict_lookup]`。Denum 的普通 numeric 列可写成单算子 chain `[zigzag_varint]`；raw/all_same 旁路列则写成 `encode_mode="bypass"`；LogReducer correlation residual 则写成 `encode_mode="dag" + encode_graph_ref=<num.rule graph> + terminal S7 op`，从而保留原有 `TransformGraph` 跨列表达力。

`ChunkManifest.pipeline` 记录 resolved `PipelineSpec`，其中 `sampling`、`block`、`execution` 均携带 `source_default` / `experiment_override` / `effective` envelope；`common_param_profile` 记录 profile 与 override provenance。writer 必须让 `BlockPlan.effective` 与 `row_count` 或 stream size 相符；reader/evaluator 不得从 algorithm name、CLI default、chunk filename 推断 common params。`manifest.pipeline.execution.effective` 当前必须是 `single_threaded` + `sequential` + `parallelism_enabled=false`；chunk/block ID 表达 data/archive grouping，不表达 scheduling。

`RegexExtractionDescriptor` 是 S4 的持久化 decode contract。Denum 的 tag streams、`variablesetids.bin` 与 `allids.bin` 表达为 `extracted_column_refs` + trunk `DictDescriptor`/`ColumnDescriptor`，DeLog 的 modified lines、compact-id mappings 与 normal/fast byte slices表达为 `masked_batch_ref`、`extracted_column_refs` 与 `raw_span_map_ref`；reader 只按 descriptor resolve streams，不按 Denum/DeLog 文件名重建隐式 regex result（denum §4/§6；delog §4/§6）。没有 S4 的 trunk 算法直接省略 `regex_extraction`，这是 empty fork，不是 wrapper。

`SchemaManifest` 把 template/schema side inputs 变成 chunk/manifest data：LogReducer `template.col`、`E<eid>basic.rule`、`E<eid>string.rule`、`head.format`；LogShrink dataset-level `template.col`、`head.format` 与 chunk `relations.*`；LogNexus `initTree_optimized.txt`、`MegaVarTrees.txt` 都是 decode-critical。若 artifact 按原算法故意保留在 chunk 外，必须写 `ExternalSideInputDescriptor{required_for_decode=true,path,version/hash}`；否则 reader 不允许从源码目录或算法名猜路径（logreducer §4/§6；logshrink §4/§6；lognexus §4/§6）。

`TemplateSetDescriptor.grammar="event_template_mapping_json"` 表达 general `EventId -> EventTemplate string` JSON maps，例如 LogZip `template_mapping.json`；`grammar="dictionary_template_mapping"` 表达 deduplicated skeleton dictionary 的 value->ID template-set role，例如 Denum `allmapping.txt` / `allids.bin`。这两个值都是通用 grammar，不是 logzip/denum 专名。LogZip treematch `_preprocess_template` / `message_split` 这类 canonical parser preprocessing 只能写入 `ParserRuleDescriptor{grammar="canonical_parser_rule_json"}`，不能混入 template grammar；LogZip base64 ParaID 仍属于既有 `SearchDictionaryDescriptor{branch="logzip_base64"}`。

## 3. Physical-Mapping Validation Table

本表把实际算法 on-disk 文件名映射到同一 `UnifiedChunk` 逻辑字段，用 field-level mapping 展示 source-backed trunk members 具备 shared contract feasibility。前三列是最强 source-backed evidence；后两列只添加算法文档明确钉住的行，不补未证实细节。

| 本框架逻辑字段 | LogReducer 物理 | DeLog 物理 | Denum 物理 | LogNexus 物理 | LogShrink 物理 |
|---|---|---|---|---|---|
| `row_order_ref` / `records/row_order.uvi` | `Eid.eid`：每行 template ID，`-1`=load failed，`0`=match failed，`>0`=template ID（logreducer §4/§6） | `<logname>_templates.ids.bin`：modified-line dictionary ID stream，隐式 line order（delog §4/§6） | `allids.bin` / dictionary ID streams 隐含 line order；numeric tag streams按出现顺序回填（denum §4/§6） | `tree_node_ids.bin`：按 `line_id` 排序的 final pathID stream（lognexus §3/§4/§6） | `eids.eid(.ee)`：全量 line-order event ID stream，含 failed sentinels（logshrink §4/§6） |
| `occurrence_stream` / `TemplateDescriptor.placeholders` | 可由 `template_id + var_position` 推导；descriptor 仍记录 placeholder position（logreducer §4/§6） | compact tag provider 与 `index.normal.csv` slices 映射 line/template placeholder 到 tag stream occurrence（delog §4/§6） | `allids.bin`/`variablesetids.bin` 与 tag `.bin` 需要 `CellRef` 表达每个 `<tag>` / `<*>` occurrence（denum §4/§6） | final `pathID`、fixed vars、residual columns 和 extracted placeholders 需要 path/residual occurrence mapping（lognexus §4/§6） | `eids.eid` + `var_list_E*_*` per-template position；failed sentinels 回填（logshrink §4/§6） |
| `columns/*` numeric stream | `E<eid>_<col>.dat`：`uvarint(count)+count×uvarint(zigzag)` 或 plaintext（logreducer §3/§4） | `processed_tags/all_variables.normal.bin` numeric slice / ALLSAME value（delog §3/§4） | `<sanitized_tag>.bin`，如 `<I>` -> `_I_.bin`，elastic integer stream（denum §3/§4） | `_var_<tag>.bin`、`extracted_<placeholder>.bin`、`MegaVariables_col_<i>.bin`（lognexus §3/§4） | `head_var_col*.diff.ee/.dat.ee`、`var_list_E*_* .diff.ee/.dat.ee`（logshrink §4） |
| `columns/*` raw/string stream | `E<eid>_<col>.str`（logreducer §4） | dictionary mapping payload or raw provider slices（delog §4/§6） | numeric variables dictionary mapping is string payload in `variablesetmapping.txt`（denum §4） | `_var_STRING.txt` residual template dictionary with `LNXLP1` prefix encoding（lognexus §3/§4） | `head_var_col*.str`、`var_list_E*_* .str`（logshrink §4） |
| `dictionaries/*.umap` | `Header_dictionary.headDict`（logreducer §4/§6） | `*_templates.mapping.txt`、`all_mappings.fast.txt`、`all_mappings.normal.txt`（delog §4） | `variablesetmapping.txt`、`allmapping.txt`（denum §4） | `_var_STRING.txt` is residual template mapping; tree files map pathID/template structure（lognexus §4） | `header.dict`、`var.dict`（logshrink §4/§6） |
| `dictionaries/*.uids` | `Head<k>.head` may encode string-header dictionary IDs（logreducer §3/§4） | `*_templates.ids.bin`、`all_ids.fast.bin`、normal mapping ID slices in `all_variables.normal.bin`（delog §4） | `variablesetids.bin`、`allids.bin`（denum §4） | `MegaVariables_col_<i>.bin` can carry residual template IDs（lognexus §3/§4） | `.var_dict.ee` logical dictionary-index diff streams（logshrink §4/§6） |
| `ColumnDescriptor.stream_slices` / `CodecDescriptor` | Filename grammar `E<eid>_<col>.*` plus count-prefix/plaintext codec descriptor（logreducer §3/§4） | `index.fast.csv` / `index.normal.csv` become typed slices with storage type and offsets（delog §4） | tag filename is stream key; dictionary filenames encode `variableset` / `all` scope（denum §4） | fixed filenames plus `packed_small_files.bin` optional pack manifest become stream slices（lognexus §4） | `.ee` physical file maps to logical `.diff/.dat/.var_dict/.eid` via codec + slice role（logshrink §4） |
| `SchemaManifest.templates` | `template.col` commit-marker grammar（logreducer §4/§6） | `<logname>_templates.mapping.txt` modified-line dictionary（delog §4/§6） | `allmapping.txt` logs-without-numbers dictionary（denum §4/§6） | `initTree_optimized.txt` structural tree（lognexus §4/§6） | dataset-level `template.col` external or embedded descriptor（logshrink §4/§6） |
| `SchemaManifest.parser_rules` | `E<eid>basic.rule`、`E<eid>string.rule`、`E<eid>num.rule` grammar（logreducer §4/§8） | `tags_mapping.txt` compact ID -> full signature and mode-detected index grammar（delog §4/§6） | regex/tag application order and numeric-variable delimiter policy（denum §1/§2） | `MegaVarTrees.txt` variable subtree grammar（lognexus §4/§6） | parser rules from LogReducer-derived training artifacts（logshrink §1/§4） |
| `SchemaManifest.relations` | not used except transform graph generated from `num.rule` | not used; offset index is stream-slice metadata | not used | not used | `relations.hdiff/hdict/pdiff/pdict/v_pat/h_pat` relation metadata（logshrink §4/§6） |
| `SchemaManifest.framing` / header format | `head.format` with `headLength/isMulti/headRegex/...`（logreducer §2/§4） | absent; modified logs/templates carry structure（delog §4） | absent; chunk-of-lines + tags（denum §0/§4） | dataset + placeholder shape in chunk metadata/tree files, not `head.format`（lognexus §1/§4） | dataset-level `head.format` side input（logshrink §1/§4） |
| `mode/profile_detection` | `-E Z` vs non-Z reflected as `CodecDescriptor`, not algorithm flag（logreducer §3/§6） | presence of `index.fast.csv` vs `index.normal.csv` sets `PipelineSpec`/index grammar（delog §4/§6） | C++ vs Python preservation/delta exceptions are profile data（denum §6/§8） | full vs `--s1-lzma` must mark restorable vs ablation profile（lognexus §4/§8） | `encode_mode=E` `.ee` and `relations.*` presence required by restore（logshrink §4/§6） |
| `TransformGraph` / `TransformDescriptorRef` | `E<eid>num.rule` with `direct/do/up/doi/upi/diff2/diff3`, `corre_t=600`; current restore limitation requires `profile=source_compat_nonrestorable` for residual-only combos（logreducer §3/§6） | `NUMERIC_DELTA` / `NUMERIC_NODELTA` / `ALLSAME` in `index.normal.csv`（delog §3/§4） | successive-delta per tag except C++ `<I>/<a>/<b>/<c>`（denum §2/§3） | delta or unsigned/no-delta selected by placeholder family（lognexus §3） | `.diff` and `.var_dict` both inverse via prefix sum; pattern split metadata in `relations.*`（logshrink §3/§6） |
| `stream_slice_index` | per-column file or plaintext line stream descriptor | `index.fast.csv`/`index.normal.csv` offsets become slices | one stream per tag/dictionary ID file | fixed files or packed-small-file entries become slices | logical suffix after `.ee` decode selects slice role |
| `failed_streams` | `load_failed.log` / `match_failed.log`（logreducer §4/§6） | no dedicated failed stream in main chunk; unmapped literal pieces remain in modified templates（delog §4/§6） | no search/user failed stream; unmapped text remains in logs-without-numbers dictionary（denum §4/§6） | residual/unclassified variables are first-class streams, not parser failed logs（lognexus §2/§3） | `load_failed.log` / `match_failed.log` preserved and replayed by `eids.eid`（logshrink §4/§6） |
| S11 archive | sequential per-block archive; block is archive/data unit, not a parallel execution contract; `7za ... -m0=LZMA` backend（logreducer §5） | sequential archive over libarchive PAX tar + xz/gzip/bzip2/lz4/zstd/none filter（delog §5） | sequential per-block `tar -cJf ... compressed<id>.xz`（denum §5） | sequential `tar ... -cJf` `.tar.xz` archive（lognexus §5） | sequential per-chunk `7z -m0=LZMA` / tar.gz / tar.bz2（logshrink §5） |

结论：这些物理命名差异都能归入 `ColumnDescriptor.stream_slices`、`DictDescriptor.id_stream_descriptor`、`TransformGraph`、`SchemaManifest`、`failed_streams`、`BackendSpec`。本表不是“证明所有实现已经同一格式”，而是 **field-level mapping demonstrating shared contract feasibility for source-backed trunk members**；reader 读 manifest/header 中的逻辑字段，不写 per-algorithm filename parser，这就是 shared data contract，不是 wrapper。LogFold 是 paper-only logical trunk fit，不参与 source-backed physical mapping validation。

Common-flow params 统一保存在 `ChunkManifest.pipeline.sampling` / `ChunkManifest.pipeline.block` / `ChunkManifest.pipeline.execution` 与 `common_param_profile` provenance 中；`row_count`、`chunk_id`、block 文件名只描述 data grouping，不描述 worker/thread scheduling。

## 4. Per-Slot Module Interfaces S1-S12

每个 slot 一个 interface。所有实现同签名；不同算法只换 `Spec`。每个 stateful slot 都有 reader/reconstruct/decode counterpart。

### S1 Line framing / header split

依据：共识 S1；LogReducer/LogShrink `head.format`；CLP timestamp boundary；LogBlock/LogZip/LogGzip `log_format`；LogLite newline-only outlier。

```text
interface LineFramer {
  frame(input: RawInput, spec: FramingSpec) -> RecordBatch
  reconstruct(input: RecordBatch, spec: FramingSpec) -> RawInput
}

FramingSpec {
  mode: "line" | "multiline_header" | "timestamp_boundary" | "log_format_capture" | "opaque_byte_line"
  log_format?: LogFormatSchema
  head_format?: HeadFormatDescriptor
  timestamp_patterns?: vector<TimestampPatternDescriptor>
  preserve_raw_line: bool
  newline_policy: "preserve" | "normalize_lf" | "drop"
}
```

LogLite 的 `opaque_byte_line` 可产出 `Record{status=opaque}`，但 semantic trunk 会拒绝继续进入 S2-S10，只允许接 S11 external cascade（共识 c/S1/S11）。

### S2 Tokenizer / delimiter splitter

依据：共识 S2；DeLog fixed delimiter；Denum/LogShrink mined delimiter；LogGrep fixed delimiter；CLP variable-boundary tokenization；LogFold/LogNexus skeleton tokenizer。

```text
interface Tokenizer {
  tokenize(input: RecordBatch, spec: TokenizerSpec) -> TokenizedBatch
  detokenize(input: TokenizedBatch, spec: TokenizerSpec) -> RecordBatch
}

TokenizerSpec {
  mode: "fixed_delimiters" | "mined_delimiters" | "skeleton" | "schema_boundary" | "none"
  delimiters?: string
  keep_delimiters: bool
  candidate_chars?: string
  fallback_delimiter?: string
}
```

`fixed_delimiters` 与 `mined_delimiters` 是同一 signature 的 spec 值，不是 adapter。

### S3 Template / pattern miner

依据：共识 S3；LogReducer length-bucket exact matcher；DeLog signature synthesis；Denum numeric stripping；LogNexus URT/pathID；LogGrep LengthParser；LogGzip NCD parser reference。

```text
interface TemplateMiner {
  train(samples: vector<TokenizedBatch>, spec: TemplateSpec) -> TemplatePool
  mine(input: TokenizedBatch, pool?: TemplatePool, spec: TemplateSpec) -> ParsedBatch
  render(input: ParsedBatch, schema: SchemaManifest, spec: TemplateSpec) -> TokenizedBatch
}

TemplateSpec {
  mode: "length_bucket_exact" | "signature_synthesis" | "numeric_stripping" | "urt" | "skeleton_matrix" | "drain_lenma_ncd" | "clp_schema"
  wildcard_token: string
  training_match?: "similarity_merge" | "external_templates" | "none"
  compression_match?: "exact_non_wildcard" | "schema" | "ncd_similarity"
  threshold?: float
  emit_templates: bool
  emit_failed_streams: bool
}
```

S3 不允许只返回私有 parse tree。URT、skeleton tree、LengthParser pool 可以进入 `ParsedBatch.metadata` 或 `SchemaManifest.patterns`，但每条 record 必须有 `pattern_id` 与 ordered `variables`。S3 的 `input` 已经由 `PipelineSpec.s3_input_stream` 选好：Denum/DeLog/LogNexus 这类先替换再解析的路径必须传 S4 `masked_batch`，LogReducer/LogShrink 这类 parser path 可传 original S2 stream。S3 不再同时接 original + optional regex result，否则 reader/writer 会把“该挖哪条 stream”藏在算法 glue 中（共识 S4/e3；denum §1/§2；delog §1；lognexus §1）。

`TemplateMiner.train(samples, spec)` 的 `samples` 必须来自 `PipelineSpec.sampling.effective` 中兼容的 sampling phase。存在 sampling phase 时，writer 按 profile 取样：LogReducer 用 `--SampleRate` / contiguous samples，LogGrep static template 用 `sampleRange=100` 的 random ratio，Denum delimiter mining 用 distinct-length cap，LogZip 用 `external_templates` side input，LogGzip 用 `parser_reference` oracle。没有 sampling phase 时，行为由 `TemplateSpec` 决定为 full scan、external pool 或 no training；不能由算法名临时分支。

### S4 Metadata / regex pre-extractor

依据：共识 S4/e3；DeLog/Denum/LogNexus 16-dataset regex table lineage；CLP timestamp-only known-pattern table。

```text
interface RegexExtractor {
  extract(input: RecordBatch | TokenizedBatch, spec: RegexExtractSpec) -> RegexExtractionResult
  restore(input: RegexExtractionResult, tokenized: TokenizedBatch, spec: RegexExtractSpec, preservation: PreservationProfile) -> TokenizedBatch
}

RegexExtractSpec {
  table_kind: "loghub_16" | "timestamp_only" | "custom" | "none"
  unknown_dataset_policy: "empty" | "warn" | "error"
  value_policy: "raw" | "digits" | "ip_pad3" | "dataset_specific"
  application_order: "source_order"
}
```

S4 只负责抽取/遮蔽 metadata，不决定 numeric/dict/string route；route 由 S5 决定。关闭 S4 时是 `table_kind=none` 的 empty result，不是 wrapper。`restore` 必须通过 `raw_span_map` 和 `extracted` 回填 masked placeholders；若 `PreservationProfile.raw_line_available=false` 且算法只承诺 token-level/normalized restoration，restore 不得声称 byte identity。

### S5 Variable typer

依据：共识 S5；CLP int/float/dict；DeLog ALLSAME/NUMERIC_DELTA/MAPPING；Denum tag exceptions；LogReducer `basic.rule`；LogShrink numeric/string split；LogGrep runtime real/nominal。

```text
interface VariableTyper {
  type_columns(input: ColumnStore, spec: TypeSpec) -> TypedColumnStore
  untype_columns(input: TypedColumnStore, spec: TypeSpec) -> ColumnStore
}

TypeSpec {
  numeric_predicate: "ascii_digits" | "int64_safe" | "float_or_int" | "tag_rule" | "column_rule" | "int32_non_float"
  dict_predicate: "unique_ratio" | "always" | "never" | "runtime_pattern"
  unique_threshold?: float
  all_same_enabled: bool
  tag_exception_policy: map<Tag, TypeOverride>
  route_policy: map<ColumnType, vector<EncodeRoutePolicy>>
}

EncodeRoutePolicy {
  when: "default" | "low_cardinality" | "tag_exception" | "column_rule" | "runtime_nominal"
  slot_labels: vector<SlotRef>
  plan_selector: string
  graph_policy: "none" | "reuse_transform_graph"
}
```

`route_policy` 例：`numeric -> [{when=default, slot_labels=[S9,S7], plan_selector="transform_then_numeric", graph_policy="reuse_transform_graph"}]`、`dict -> [{when=default, slot_labels=[S8,S7], plan_selector="dict_then_numeric", graph_policy="none"}, {when=low_cardinality, slot_labels=[S8,S9,S7], plan_selector="dict_then_transform_then_numeric", graph_policy="none"}]`、`all_same -> [{when=default, slot_labels=[], plan_selector="metadata_only", graph_policy="none"}]`。S5 只决定 resolved `EncodeRoute` 与 plan selector；具体 `EncoderOp` 由 encode layer 的 encoder pool 展开，因此它是 encode layer 的入口路由，不是固定 stage 调度器。

### S6 Column / field organizer

依据：共识 S6/e2；LogReducer/LogShrink per-template variables；Denum/DeLog tag groups；LogNexus path residual；CLP segment columns；LogGrep Capsule columns；LogBlock direct transpose。

```text
interface ColumnOrganizer {
  organize(parsed: ParsedBatch, regex?: RegexExtractionResult, spec: ColumnSpec) -> ColumnStore
  unorganize(input: ColumnStore, schema: SchemaManifest, occurrence?: OccurrenceStreamDescriptor, spec: ColumnSpec) -> ParsedBatch
}

ColumnSpec {
  key_mode: "template_position" | "tag" | "field" | "capsule" | "segment_three_column" | "path_residual"
  preserve_row_order: bool
  include_failed_streams: bool
  row_ref_mode: "template_position" | "column_row_refs" | "branch_defined"
  emit_occurrence_stream: bool
}
```

S6 是替换 seam：S5/S7/S8/S9 只看 `ColumnStore`，不关心上游是 LogReducer parser、URT miner、Denum numeric stripping 还是 DeLog signature synthesis。它在进入 encode layer 前必须把列拆到“该算法的最小 encode 单元”：`row_ref_mode=template_position` 只适合 LogReducer/LogShrink 这类 `(template_id,var_position)` 可由模板位置直接定位的列；`tag`、`signature_tag`、`path_residual`、`capsule` 等模式必须 `emit_occurrence_stream=true` 并为每个 placeholder emit `CellRef`。S6 `unorganize` 的 interface 显式接收 `SchemaManifest` 与 optional `OccurrenceStreamDescriptor`，因此模板、placeholder binding 与 cell mapping 都来自 manifest/header 描述；它只能读 `SchemaManifest.templates` 中解析出的 `TemplateDescriptor.placeholders`、`OccurrenceStreamDescriptor`/decoded `OccurrenceStream.cells`、`Column.row_refs/occurrence_refs`，不能写算法名分支。

从这里往后进入统一 encode layer：S5 负责为每个 encode unit 选 `EncodeRoute`，S7/S8/S9 只提供可调用的 operator categories；真正的编排顺序由 `encode_ops` / `encode_graph_ref` 决定，而不是写死在 trunk stage graph 里。

### S7 Numeric encoder (encode-layer numeric-codec category)

依据：共识 S7/e1；DeLog/Denum/LogNexus/LogReducer/LogShrink/LogFold 的 ZigZag + 7-bit varint 共享点；LogReducer count prefix / plaintext fallback；CLP fixed64 branch。

```text
interface NumericEncoder {
  encode(input: IntColumnBundle, spec: NumericColumnSpec) -> EncodedInts
  decode(input: EncodedInts, spec: NumericColumnSpec) -> vector<int64 | uint64>
}

NumericColumnSpec {
  int_spec: NumericSpec
  count_prefix: bool
  count_prefix_mode: "uvarint" | "none"
  plaintext_fallback: bool
}

NumericSpec {
  signed_mode: "zigzag64" | "unsigned" | "none"
  byte_order: "little_7bit" | "native_fixed64" | "text"
  continuation_bit: "msb_1_more_0_stop" | "none"
}
```

S7 是 encode layer 内的 `numeric-codec` 类算子，不再是全局固定“最后一道 stage”。direct numeric 列可以是单算子 chain `[S7]`；logshrink `.var_dict` 则可以把 S7 放在 `[S8 dict_lookup, S9 successive_delta]` 之后。LogReducer non-`Z` plaintext 是 `plaintext_fallback=true`，同样输出 `EncodedInts`（logreducer §3）。

### S8 String dictionary (encode-layer dictionary category)

依据：共识 S8/e1；Denum/DeLog/LogNexus/LogShrink first-occurrence IDs；CLP logtype/var dict + segindex；LogGrep capsule dict；LogZip base64 ParaID。

```text
interface StringDictionary {
  encode(input: StringColumnBundle, spec: DictSpec, int_spec: NumericColumnSpec) -> DictionaryResult
  decode(input: DictionaryResult, spec: DictSpec, int_spec: NumericColumnSpec) -> vector<string | bytes>
}

DictSpec {
  id_base: 0 | 1
  ordering: "first_occurrence" | "sorted_by_id" | "pattern_segment_order" | "source_order"
  scope: "trunk_chunk" | "trunk_template" | "trunk_tag" | "trunk_column"
  id_encoding: "S7"
}
```

S8 是 encode layer 内的 `dictionary` 类算子。它的输出可以直接进入 S7，也可以先成为 S9 的输入（例如 logshrink `.var_dict` 中对 dictionary IDs 做 successive delta）；这由 `EncoderOp.inputs` / `encode_ops` 决定，不由 slot 编号暗示。strict trunk 仍要求 `id_encoding=S7` 是 trunk 唯一取值，`DictionaryResult.id_stream` 永远是 `EncodedInts`。CLP `logtype.dict/var.dict + *.segindex`、LogGrep `.dic/.entry` fixed-width padded capsule、LogZip base64 ParaID、LogBlock inline H3 dictionary 都不进入 trunk `DictionaryResult`；它们进入 `SearchDictionaryDescriptor` 或 shallow branch descriptor。这样 S8/S7 的 ID-stream contract 不再自相矛盾（共识 S8/e1；clp §3/§4；loggrep §3/§4；logzip §3）。

### S9 Delta / correlation transform (encode-layer transform category)

依据：共识 S9；Denum/DeLog/LogNexus/LogShrink/LogFold successive delta；LogReducer multi-column residual；LogBlock H2 source caveat。

```text
interface NumericTransform {
  transform(input: TypedColumnStore, spec: TransformSpec) -> TransformStore
  inverse(input: TransformStore, spec: TransformSpec) -> TypedColumnStore
}

TransformSpec {
  default_mode: "none" | "successive_delta"
  first_value_mode: "raw_first"
  skip_policy: set<Tag | ColumnPredicate>
  graph?: TransformGraph
  source_compatibility?: "strict_lossless" | "logblock_h2_source_compat"
}
```

S9 是 encode layer 内的 `transform` 类算子。它可以对原始 numeric values 做 successive delta，也可以在 `EncoderOp.inputs` 指定下对上游 dictionary IDs 做 delta；跨列相关 residual 则通过 `TransformGraph` 进入 DAG 形态。LogReducer correlation 是 `TransformGraph`，不是 opaque `CorrelationSpec`。`direct/do/up/doi/upi/diff2/diff3`、`corre_t=600`、ID-column deps、derived residual filenames 与 coverage 都必须从 graph descriptor 可读；generic inverse 只在 `inverse_available=true` 且 coverage 完整时允许。若选择当前 LogReducer restore 不能逆的 residual-only 组合，或选择 LogBlock source-compatible H2 off-by-one，`TransformDescriptor.inverse_available=false`，writer 必须拒绝 strict round-trip profile 或降级为 non-strict profile（logreducer §3/§6；logblock §3/§6）。

### S10 Persistence container

依据：共识 S10；主干 multi-file chunk directory；CLP segment+SQLite；LogGrep CapsuleBox；LogBlock single text；LogLite bitset outlier。

```text
interface ChunkPersistor {
  write(input: EncodedColumnSet, manifest: ChunkManifest, spec: PersistenceSpec) -> UnifiedChunk
  read(chunk: UnifiedChunk, spec: PersistenceSpec) -> EncodedColumnSet + ChunkManifest
}

PersistenceSpec {
  layout: "unified_chunk_dir" | "search_segment" | "capsulebox" | "single_preprocessed_text" | "external_stream"
  include_manifest: true
  include_failed_streams: bool
  include_search_extensions: bool
}
```

主干必须使用 `layout=unified_chunk_dir`。CLP/LogGrep 使用 search-aware layout，但必须把 segment/capsule metadata 映射到 `search/*` 与 manifest。LogBlock 可以作为 shallow branch 使用 `single_preprocessed_text`，但不伪装成 S7/S8/S9 trunk。`PipelineSpec.block.effective` 定义 S10/S11 的 data/archive grouping：line block、byte block、segment、whole file、line window 或 none。它约束 `row_count`、stream sizes、archive scope 与 manifest provenance，不定义 thread scheduling。

### S11 Backend archiver

依据：共识 S11/e4；主干 tar/7z + LZMA/xz family；DeLog pluggable tar filters；CLP zstd/passthrough；LogGrep per-capsule LZMA；LogLite external cascade；LogGzip oracle-only。

```text
interface BackendArchiver {
  pack(input: BackendInput, spec: BackendSpec) -> ArchiveResult
  unpack(archive: ArchiveResult, spec: BackendSpec) -> BackendInput
}

BackendInput {
  kind: "unified_chunk" | "clp_segment_chunk" | "loggrep_capsulebox" | "external_byte_stream"
  unified_chunk?: UnifiedChunk
  branch_payload?: BranchPayloadDescriptor
  stream_directory?: map<StreamName, TypedStream>
}

ArchiveResult {
  path: string
  format: "tar" | "7z" | "capsulebox" | "raw_stream" | "none"
  kernel: "xz" | "lzma" | "gzip" | "bzip2" | "lz4" | "zstd" | "deflate" | "none"
  payload_descriptor?: BranchPayloadDescriptor
  size_bytes?: uint64
}

BackendSpec {
  format: "tar" | "7z" | "capsulebox" | "raw_stream" | "none"
  kernel: "xz" | "lzma" | "gzip" | "bzip2" | "lz4" | "zstd" | "deflate" | "none"
  compression_scope: "whole_directory" | "per_capsule" | "external_cascade" | "oracle_only"
}
```

`oracle_only` 只允许 LogGzip reference profile；它不能生成 trunk `ArchiveResult`，因为没有 storage-compression artifact（共识 c/e4）。当前 unified fair execution 中，S11 writer 按 `ExecutionPlan.block_processing="sequential"` 以 input order 处理 chunk/block/archive units；`parallelism_enabled=false`。Source 中的 per-block loop、thread pool、worker count 或 multiprocessing knob 只作为 profile provenance，不改变 S11 interface。

### S12 Searchable index

依据：共识 S12/e5；只有 CLP 与 LogGrep 有压缩态搜索；其他算法完整解压或没有 storage compressor 语义。

```text
interface SearchIndexBuilder {
  build(chunk: UnifiedChunk, spec: SearchIndexSpec) -> SearchIndexResult
  query(chunk: UnifiedChunk, index: SearchIndexResult, query: SearchQuery) -> SearchResult
}

SearchIndexSpec {
  mode: "none" | "clp_segment_index" | "loggrep_capsule_index"
  dictionary_refs: vector<string>
  column_refs: vector<ColumnKey>
  stamp_metadata?: StampSpec
  segment_metadata?: SegmentSpec
}

SearchResult {
  matching_rows: Bitmap | vector<LineId>
  materialization_contract: "full_decode" | "on_demand_message" | "capsule_materialization"
}
```

S12 只从同一 `UnifiedChunk` 的 dictionaries / columns / search extensions 建索引或查询；它不是单独数据路径，也不改变 S1-S11 trunk contract。

## 5. Interchangeability Demonstration

每个 swap 都满足：同一 slot/category、同一 input/output type、只改 spec。没有 shim，没有 adapter。对 encode layer 而言，最小 swap 单位是 `EncodeRoute.plan_selector` 或单个 `EncoderOp`；S7/S8/S9 编号仍保留，但它们现在表达的是 operator category，而不是固定 stage 位置。

### Swap 1: S3 LogReducer parser ↔ URT miner

同一 signature：

```text
TemplateMiner.mine(input: TokenizedBatch, pool?: TemplatePool, spec: TemplateSpec) -> ParsedBatch
```

配置 A：`PipelineSpec{s3_input_stream=original_s2}` + `TemplateSpec{mode=length_bucket_exact, compression_match=exact_non_wildcard}`，输出 `ParsedRecord.pattern_id=TemplateId` 与 wildcard variables（共识 S3；logreducer §1）。

配置 B：`PipelineSpec{s3_input_stream=s4_masked_batch}` + `TemplateSpec{mode=urt}`，输出 `ParsedRecord.pattern_id=PathId` 与 URT residual variables（共识 S3/S4；lognexus §1/§2）。

下游统一：

```text
ParsedBatch -> ColumnOrganizer{key_mode=template_position | path_residual} -> ColumnStore
```

S5/S7/S8/S9 只消费 `ColumnStore.columns`。替换 parser 不需要把 URT output 转成 LogReducer output；它们本来就产出同一 `ParsedBatch`。

### Swap 2: S7 numeric-codec operator `zigzag_varint_7bit_le` ↔ plaintext numeric codec

同一 signature：

```text
NumericEncoder.encode(input: IntColumnBundle, spec: NumericColumnSpec) -> EncodedInts
```

配置 A：`NumericColumnSpec{int_spec.signed_mode=zigzag64, byte_order=little_7bit, count_prefix=true|false, plaintext_fallback=false}`，覆盖 Denum/DeLog/LogNexus/LogReducer `-E Z` shared trunk（共识 S7；denum §3；delog §3；lognexus §3；logreducer §3）。

配置 B：`NumericColumnSpec{plaintext_fallback=true, int_spec.byte_order=text}`，覆盖 LogReducer non-`Z` plaintext integer fallback（logreducer §3）。

两者都占 `encode_ops` 里的 S7 terminal op 位置。S10 只看到 `EncodedInts`、`ColumnDescriptor.codec` 与 `ColumnDescriptor.encode_ops`，reader 按 `CodecDescriptor` 选 decoder，再按 `decode_order` / 逆拓扑 op 序恢复上游值。

### Swap 3: S9 transform operator `successive_delta` ↔ LogReducer correlation residual DAG

同一 output：

```text
NumericTransform.transform(input: TypedColumnStore, spec: TransformSpec) -> TransformStore
TransformStore.transforms[*] = DeltaResult{values, descriptor}
```

配置 A：`EncodeRoute{slot_labels=[S9,S7], plan_selector="transform_then_numeric"}` + `TransformSpec{default_mode=successive_delta, first_value_mode=raw_first, skip_policy={<I>,<a>,<b>,<c>}}`，覆盖 Denum C++ path 及 DeLog/LogNexus/LogShrink 类 delta（共识 S9；denum §2/§3；delog §3；lognexus §3；logshrink §3）。

配置 B：`EncodeRoute{slot_labels=[S9,S7], plan_selector="correlation_graph"}` + `TransformGraph{edges={direct,do,up,doi,upi,diff2,diff3}, deps.id_columns=..., profile=correlation_graph_lossless|source_compat_nonrestorable}`，覆盖 LogReducer multi-column residual（共识 S9；logreducer §3）。

两者都把 `DeltaResult.values` 交给 terminal S7 op；`TransformDescriptorRef` 与 `TransformGraph` 写入 manifest，`ColumnDescriptor.encode_mode` 则区分这是单列 chain 还是跨列 DAG。只有 `inverse_available=true` 的 graph 是可互换 strict-lossless output；否则只能写 non-strict compression profile（logreducer §6）。

### Swap 4: S8 trunk chunk dictionary remains strict; search dictionaries branch explicitly

同一 signature：

```text
StringDictionary.encode(input: StringColumnBundle, spec: DictSpec, int_spec: NumericColumnSpec) -> DictionaryResult
```

配置 A：`DictSpec{id_base=1, ordering=first_occurrence, scope=trunk_chunk|trunk_tag, id_encoding=S7}`，覆盖 Denum/DeLog incremental dictionary（共识 S8；denum §3/§4；delog §4）。

配置 B 不再冒充 S8 trunk：CLP segment dict 与 LogGrep capsule nominal dict 写为 `SearchDictionaryDescriptor`，并由 `ClpSegmentChunkDescriptor` / `LogGrepCapsuleBoxDescriptor` 解释 native segment columns、fixed-width `.entry`、`.dic` pattern segments、segindex/capsule offsets（共识 S8/S12；clp §3/§4/§7；loggrep §3/§4/§7）。

因此 trunk S8 的输出始终是 `DictionaryResult{id_stream: EncodedInts}`；它在 encode layer 中可以作为 `dict_lookup` op 单独出现，也可以出现在 `[S8,S9,S7]` 这类 chain 的起点。search branch 的输出是 typed branch descriptor。S12 读取 branch dictionary metadata 生成 search view，不污染 trunk encode contract。

### Denum as one slot-config

Denum 是一组 slot specs，不是一个 adapter。融合 A/B 后的 canonical Denum config：

```text
denum_config = FrameworkConfig {
  S1  = FramingSpec{ mode="line", newline_policy="preserve" }
  S4  = RegexExtractSpec{ table_kind="loghub_16", unknown_dataset_policy="error", value_policy="dataset_specific" }
  S2  = TokenizerSpec{ mode="mined_delimiters", keep_delimiters=true, fallback_delimiter=" " }
  S3  = TemplateSpec{ mode="numeric_stripping", wildcard_token="<*>", emit_templates=true }
  S6  = ColumnSpec{ key_mode="tag", preserve_row_order=true, include_failed_streams=false, row_ref_mode="column_row_refs", emit_occurrence_stream=true }
  S5  = TypeSpec{ numeric_predicate="tag_rule", tag_exception_policy={ <I>, <a>, <b>, <c> -> no_delta }, route_policy={ numeric -> [{when="default", slot_labels=[S9,S7], plan_selector="transform_then_numeric", graph_policy="none"}], dict -> [{when="default", slot_labels=[S8,S7], plan_selector="dict_then_numeric", graph_policy="none"}], all_same -> [{when="default", slot_labels=[], plan_selector="metadata_only", graph_policy="none"}] } }
  S9  = TransformSpec{ default_mode="successive_delta", first_value_mode="raw_first", skip_policy={<I>, <a>, <b>, <c>} }
  S7  = NumericColumnSpec{ int_spec.signed_mode="zigzag64", int_spec.byte_order="little_7bit", count_prefix=false, plaintext_fallback=false }
  S8  = DictSpec{ id_base=1, ordering="first_occurrence", scope="trunk_chunk" | "trunk_tag", id_encoding="S7" }
  S10 = PersistenceSpec{ layout="unified_chunk_dir", include_manifest=true, include_failed_streams=false }
  S11 = BackendSpec{ format="tar", kernel="xz", compression_scope="whole_directory" }
  S12 = SearchIndexSpec{ mode="none" }
}
```

依据：Denum 用 numeric token parsing、tag `.bin`、`variablesetids.bin`/mapping、`allids.bin`/mapping、`tar -cJf` xz backend；C++ delta skip set 为 `<I>/<a>/<b>/<c>`（denum §1-§5）。统一格式把这些写成 `columns/`、`dictionaries/`、`manifest.json`，不是复制 Denum 私有目录名。

## 6. Branch Integration

### 6.1 Searchable branch: CLP / LogGrep

接入规则：searchable branch 附加 S12，不改变 S1-S11 的 shared contract。

```text
ColumnStore.branch="search"
UnifiedChunk.search_extensions.dictionaries = vector<SearchDictionaryDescriptor>
UnifiedChunk.search_extensions.branch_payloads = vector<ClpSegmentChunkDescriptor | LogGrepCapsuleBoxDescriptor>
S11 BackendInput = BackendInput{kind="unified_chunk"|"clp_segment_chunk"|"loggrep_capsulebox"|"external_byte_stream", branch_payload=...}
S12 SearchIndexResult exposes query -> matching_rows/materialization_contract
```

CLP：timestamp/logtype/variables 三列可表示为 `ColumnKey{key_mode=segment, branch=search}`；`metadata`、`metadata.db`、`logtype.dict`、`var.dict`、`*.segindex`、`s/<id>` native arrays 必须映射到 `ClpSegmentChunkDescriptor{segment_ids,segment_streams,native_widths,endian_policy,column_offsets,dict_descriptors,segindex_descriptors,metadata_db_binding}`。`segment_streams` 把每个 `segment_id` 绑定到实际 `TypedStreamSlice`、后端 compression 与 uncompressed size；`column_offsets` 再在同一 `segment_id` 内声明 timestamp/logtype/variables 三段 decompressed offsets/counts，所以 reader 不推断 `s/<id>` 文件名，也不从 metadata DB 旁路寻找 segment stream。S11 使用 zstd/passthrough branch backend（clp §4/§5/§7）。注意 CLP source-backed logtype ID 是 `int64_t`，segment file 是 host-native raw arrays + metadata DB offsets，core archive writer/reader path 是 zstd/passthrough，不应硬写 paper-level 32-bit ID 或未接入的 LZMA reader（clp §0/§4/§5/§8）。

LogGrep：`.var/.svar/.dic/.entry/.outlier` 是 `ColumnKey{key_mode=capsule, branch=search}` 下的 typed streams；native `size_t` header word size、meta LZMA props、per-Capsule `packed_name` 与 `meta_line{filename,compressed,offset,destLen,srcLen,lines,eleLen}` 必须映射到 `LogGrepCapsuleBoxDescriptor` / `LogGrepCapsuleDescriptor`。`packed_name`/`template_id`/`variable_position`/`sub_position`/`capsule_type` 保存 filename 编码出的 native identity，使 generic reader 能把 `.entry` 连到对应 `.dic`、把 `.svar` 连到 subvar、把 `.template`/`.variablelist` 连到静态模板与 runtime subpattern metadata；reader 不重读 LogGrep filename grammar 文档。S11 是 per-capsule LZMA / CapsuleBox（loggrep §3/§4/§5/§7）。`.entry` 是 fixed-width padded branch ID stream，不是 S7 ID stream；`.outlier` 是 correctness fallback，不是普通 error log；source-compatible `.var` tag filtering 有 C++ precedence 问题，profile 必须声明 `length_filter_only` 或实现 paper-intended fixed behavior（loggrep §7/§8）。

### 6.2 LogLite: S11 external cascade only

LogLite 是 same-length byte window + XOR/RLE + bitset；它不产生 token/template/column/dictionary。因此唯一合法接入：

```text
RawInput -> ExternalByteStream{producer="loglite", format="bitset_xor_rle"}
         -> S11 BackendSpec{format="raw_stream", compression_scope="external_cascade", kernel="zstd"|"lzma"|"none"}
```

这不是 wrapper，因为 framework 没有假装 LogLite 有 `ColumnStore`。它只在 byte-stream 层与 S11 相邻（共识 c/e4；loglite §4/§5）。

### 6.3 LogGzip: reference-only

LogGzip 的 gzip 是 NCD similarity oracle；落盘是 plaintext parser CSV；无 variable streams、无 decompressor、无 storage-compression artifact。因此它只可作为 S3 `TemplateSpec{mode=drain_lenma_ncd}` 的 parser reference，不可进入 S10/S11 trunk（共识 c/e4；loggzip §3/§4/§5）。

### 6.4 LogBlock: shallow branch

LogBlock 使用 S1 `log_format` 与 S6 direct transpose，列级 H1-H4 可视为 S5/S9-ish heuristics，但不共享 S7 varint trunk，也没有完整 inverse implementation；source H2 wire format 不可逆（logblock §1-§6）。合法表达：

```text
S1{mode=log_format_capture}
  -> S6{key_mode=field, branch=direct_transpose}
  -> S5{dict_predicate=unique_ratio, route_policy={raw -> []}}
  -> S10{layout=single_preprocessed_text, preservation.strict_writer_allowed=false if source H2 enabled}
  -> S11{kernel=gzip|deflate|lz4}
```

若要求 strict unified round-trip，LogBlock profile 必须禁用 source-compatible H2；否则必须写 `PreservationProfile{strict_writer_allowed=false, byte_identity=false}` 且 `TransformDescriptor.inverse_available=false`。

## 7. Writer / Reader Inverse Contract

### 7.1 Writer pipeline pseudocode

```text
WRITE pipeline:
  execution = manifest.pipeline.execution.effective
  assert execution.mode == "single_threaded"
  assert execution.block_processing == "sequential"
  assert execution.parallelism_enabled == false

  for each block in input order according to manifest.pipeline.block.effective:
    records    = S1.frame(raw, framing_spec)
    tokenized  = S2.tokenize(records, tokenizer_spec)
    extracted  = S4.extract(tokenized or records, regex_spec) if enabled else empty RegexExtractionResult
    s3_input   = tokenized if pipeline.s3_input_stream == "original_s2" else extracted.masked_batch
    parsed     = S3.mine(s3_input, parser_pool, template_spec)
    colstore   = S6.organize(parsed, extracted, column_spec)
    typedstore = S5.type_columns(colstore, type_spec)

    for each typed_column in typedstore.typed_columns:
      route = typed_column.encoding_route
      plan = resolve_encode_plan(route.plan_selector, type_spec.route_policy, algorithm_profile)

      if route.slot_labels == []:
        ops = []
        store all_same value, raw stream, or metadata stream
      else:
        graph = materialize_or_reuse_transform_graph(typed_column.key, typedstore subset, transform_spec) if plan.graph_policy == "reuse_transform_graph" else none
        ops = instantiate_encode_ops(typed_column, route, plan, graph)

        for op in ops in topological order:
          if op.slot_label == S8:
            run S8.encode(...) and record DictDescriptor when op.artifact_ref is dict_id
          else if op.slot_label == S9:
            run S9.transform(...) and record TransformDescriptor / TransformGraph coverage when op.artifact_ref is transform_id
          else if op.slot_label == S7:
            run S7.encode(...) and record CodecDescriptor + stream_slices

      record ColumnDescriptor{encoding_route=route, encode_mode=(route.slot_labels == [] ? "bypass" : plan.mode or "chain"), encode_ops=ops, encode_graph_ref=graph?, persisted_output_ref, decode_order=reverse_topological(ops)}

    encoded_set = assemble EncodedColumnSet(columns, dictionaries, row_order, failed_streams, metadata)
    schema = assemble SchemaManifest(templates, patterns, framing, parser_rules, relations, external_side_inputs)
    occurrence_descriptor = assemble OccurrenceStreamDescriptor from colstore.occurrence_stream when present
    regex_descriptor = assemble RegexExtractionDescriptor from extracted when pipeline.s4_enabled
    manifest = assemble ChunkManifest(pipeline, slot_specs, schema, descriptors, occurrence_descriptor, regex_descriptor, preservation, backend_hint)
    chunk = S10.write(encoded_set, manifest, persistence_spec)
    archive = S11.pack(chunk or branch input, backend_spec)
```

Writer 处理 block/chunk 的顺序必须是 input order；block 是 `PipelineSpec.block.effective` 声明的数据/归档单位，不是 worker 分片。所有当前 profiles 的 `parallelism_enabled=false`。

### 7.2 Writer obligations

| Obligation | 说明 |
|---|---|
| `manifest.json` first | Reader 不依赖文件名猜测 slot、codec、dict scope、transform 或 branch mode。 |
| Stable `ColumnKey` | 同一 logical column 在 encode/decode 中由 `ColumnKey` 定位。 |
| `encode_ops` authoritative | `ColumnDescriptor.encode_ops`（+ `encode_graph_ref` when DAG）是精确 encode 表达；`decode_order` 若存在，只是逆拓扑 `op_id` shortcut。 |
| `row_order` single source of truth | `records/row_order.uvi` 或 equivalent typed stream 保存 line_id -> pattern/status 顺序。 |
| `occurrence_stream` for non-position modes | tag/signature/path/capsule columns 必须写 `OccurrenceStreamDescriptor` + `CellRefStream`；不能靠 template var position 或 tag 顺序隐式消费。 |
| `SchemaManifest` complete | templates、parser rules、relations、framing side inputs 若解码必需，必须 embedded 或声明 `ExternalSideInputDescriptor{hash,path}`。 |
| S7 as sole integer gateway | S8 `id_stream` 与 S9 residual/delta values 都通过 S7 或 manifest 声明的 branch integer encoding。 |
| Explicit operator artifacts | 每个会 materialize side artifact 的 S8/S9 op 都必须可追到 `DictDescriptor` / `TransformDescriptor`；LogReducer correlation 还必须可追到 `TransformGraph`。 |
| Reject unavailable inverse | `inverse_available=false` 的 transform 不可写入 strict lossless profile；LogReducer residual-only restore gap 与 LogBlock H2 source bug 必须显式拒绝或降级 preservation。 |
| Dictionaries separated | mapping 与 id stream 分开保存；scope/id_base/ordering/id_encoding 全写入 `DictDescriptor`。 |
| Failed/outlier streams preserved | `load_failed`、`match_failed`、capsule outlier、unparsed lines 不混入普通 columns 后失语义。 |
| Branch extensions namespaced | CLP segment、LogGrep capsule、LogLite external stream 只能在 `search/` 或 `extensions/` 下出现。 |

### 7.3 Reader pipeline pseudocode

```text
READ pipeline:
  backend_input = S11.unpack(archive, backend_spec)
  chunk, manifest = S10.read(backend_input, persistence_spec)
  read manifest.json + header.json
  execution = manifest.pipeline.execution.effective
  assert execution.block_processing == "sequential"
  assert execution.parallelism_enabled == false
  schema = load SchemaManifest streams and verified external side inputs
  occurrence_stream = decode_occurrence_stream(manifest.occurrence_stream) when present
  regex_extraction = resolve_regex_extraction(manifest.regex_extraction, manifest.columns) when manifest.pipeline.s4_enabled

  for each ColumnDescriptor cd in manifest.columns:
    if cd.encode_mode == "bypass":
      values = expand cd.all_same_value or read_slices(cd.stream_slices)
    else:
      op_outputs = materialize_persisted_output(cd.persisted_output_ref, cd.stream_slices, cd.codec)
      reverse_ops = cd.decode_order if present else reverse_topological(cd.encode_ops)

      for op_id in reverse_ops:
        op = lookup_encode_op(cd.encode_ops, op_id)
        if op.slot_label == S7:
          op_outputs = apply_inverse_s7(op_outputs, op, numeric_spec_from(cd.codec))
        else if op.slot_label == S9:
          td = lookup_transform_descriptor(manifest.transforms, op.artifact_ref or cd.transform_ref.transform_id)
          graph = lookup_transform_graph(manifest.transform_graphs, cd.encode_graph_ref.transform_id) if cd.encode_graph_ref exists
          op_outputs = apply_inverse_s9(op_outputs, op, td, graph)
        else if op.slot_label == S8:
          dd = manifest.dictionaries[op.artifact_ref or cd.dictionary_ref]
          op_outputs = apply_inverse_s8(op_outputs, op, dd)

      values = op_outputs["column_values"]
    row_refs = decode cd.row_ref_stream if cd.row_ref_mode == "column_row_refs" else derive from row_order/template occurrence
    colstore.columns[cd.key] = Column{key=cd.key, values=values, row_refs=row_refs}

  colstore.row_order = decode row_order stream
  colstore.occurrence_stream = occurrence_stream
  colstore.failed_streams = read failed/outlier streams
  parsed = S6.unorganize(colstore, schema, manifest.occurrence_stream, column_spec)
  tokenized = S3.render(parsed, schema, template_spec)
  tokenized = S4.restore(regex_extraction, tokenized, regex_spec, manifest.preservation) if manifest.pipeline.s4_enabled
  records = S2.detokenize(tokenized, tokenizer_spec)
  raw = S1.reconstruct(records plus failed_streams, framing_spec, manifest.preservation)

  if query path:
    index = S12.build or load SearchIndexResult
    return S12.query(chunk, index, query) without full reconstruction when allowed
```

Reader 按 manifest row order 与 archive/chunk order 顺序恢复，不承诺 parallel decode；如果 source artifact 曾由多线程产生，该事实只在 profile/provenance 中记录，不进入 unified reader contract。

reader 的 decode contract 因而是：**按 `decode_order`（若无则按 `encode_ops` 逆拓扑序）逐算子执行 inverse**。也即 `decode = reverse(operator chain/DAG) + per-op inverse`；manifest/descriptor 决定这一切，文件名与算法名都不参与推断。

### 7.4 Round-trip symmetry guarantees

`UnifiedChunk` 必须能回答三个问题：

```text
Which records exist and in what order?       -> row_order + failed_streams
Which values fill each pattern/field slot?   -> TemplateDescriptor.placeholders + occurrence_stream + ColumnKey + decoded columns
How were bytes/strings normalized?           -> PreservationProfile + framing/tokenizer descriptors + raw_span_map
```

保证项：

| Guarantee | 机制 |
|---|---|
| Reader 由数据驱动 | `ChunkHeader.slot_specs`、`ColumnDescriptor.encoding_route/encode_ops/encode_graph_ref/decode_order` 决定逆过程，不由 algorithm name 决定。 |
| `row_order` 是唯一行序真源 | 所有列式转置逆过程只依赖 row order typed stream；failed streams 按 line_id/status 回填。 |
| S7 是唯一整数出入口 | 写时 S8 IDs 与 S9 values 进入 S7；读时统一 S7 decode，整数对称性集中验证一处。 |
| Occurrence mapping 是唯一占位符真源 | 非 `template_position` 模式不能隐式“取下一个 tag 值”；必须读 `occurrence_stream`。 |
| Schema side inputs 是 manifest 数据 | `S3.render` 与 S1/S2 reconstruct 不从算法目录猜 `template.col/head.format/relations/MegaVarTrees`。 |
| Failed-stream bypass 对称 | `load_failed`、`match_failed`、outlier records 原样保存，不参与 S3-S9 transform，读时原样归位。 |
| Losslessness boundary explicit | `PreservationProfile` 声明 byte/line/token/whitespace/timestamp/non-UTF8/final-newline 边界。 |
| 不可逆组合拒绝 | LogReducer current restore 不逆 `num.rule` residual-only combos（logreducer §6）；LogBlock source H2 不可逆（logblock §3/§6）；writer 对 strict profile 拒绝。 |

## 8. Algorithm Fit Summary

共识 (c) 给出 trunk/branch/outlier map。本框架保留 A 的 per-algorithm fit 表，并修正覆盖口径：5 个 source-backed trunk/trunk-dominant implementations（`denum`、`delog`、`logreducer`、`logshrink`、`lognexus`）+ 1 个 paper-only logical trunk fit（`logfold`）；LogBlock 是 shallow branch，CLP/LogGrep 是 searchable branch-but-rejoin；LogLite/LogGzip 不入 trunk。source-backed physical mapping validation 只要求覆盖前 5 个实现，LogFold 只做 logical contract fit。

所有算法 profile 都必须在 `PipelineSpec.sampling` / `PipelineSpec.block` / `PipelineSpec.execution` 中携带 `source_default` common params，即使是 branch/outlier：LogGzip 使用 `parser_reference` sampling 与 `reference_only` block；LogLite 使用 `none` sampling 与 `line_window` / `external_stream` block；LogZip 使用 external template sampling 与 whole-file block。source 中的 `num_threads`、worker、thread pool、multiprocessing 或 block dispatcher knob 是 provenance/source fact，不进入 unified fair execution；当前有效执行一律 `single_threaded`、`sequential`、`parallelism_enabled=false`。

| Algorithm | Fit | Slot expression | 关键边界 |
|---|---|---|---|
| `denum` | trunk | S4 regex tags + S2 mined delimiters + S3 numeric stripping + S6 tag columns + S5 tag-rule typing + encode layer{S7 numeric codec, S8 strict dictionary, S9 delta exceptions} + S10/S11 | C++ no full decompressor CLI；losslessness 需标 `normalized` 或 profile-specific（denum §6/§8）。 |
| `delog` | trunk | S3 signature synthesis + S4 regex + S6 tag/signature groups + S5 ALLSAME/MAPPING/NUMERIC + encode layer{ALLSAME/raw passthrough, S8 dictionary, S9 transform, S7 numeric codec} + tar backend | `index.fast/normal.csv` 是 decode offset metadata，不是 search index（delog §4/§7）。 |
| `logreducer` | trunk with inverse caveat | S1 `head.format` + S3 length parser + S6 template columns + encode layer{S9 delta/correlation DAG, S7 numeric codec} + S10/S11 7z/LZMA | Current restore 不读取 `num.rule` residual；strict writer 拒绝不可逆 residual-only combos（logreducer §6）。 |
| `logshrink` | trunk | LogReducer-derived S3 + S6 event variables + S5 commonality/variability + encode layer{`.var_dict` chain S8->S9->S7 plus direct S9->S7 and S7-only paths} + S10/S11 | `head.format/template.col` 是 dataset-level side input；relations 必须进入 manifest/profile（logshrink §4/§6）。 |
| `lognexus` | trunk-dominant | S3 URT/pathID + S6 path residual columns + encode layer{S9 selected delta/no-delta, S7 unsigned/signed} + S10/S11 tar.xz | Artifact restoration 是 token-stream equality，不承诺 byte identity（lognexus §6/§8）。 |
| `logfold` | paper-only trunk fit | S2 skeleton tokenizer + S3 skeleton patterns + S6 sub-token matrices + encode layer(logical S8/S9/S7 operator combos) + tar/LZMA | 当前证据主要 paper-level；manifest 必须标 source-backed level（共识 S2/S3/S7/S11）。 |
| `logzip` | partial trunk / shallow | S1 `log_format` + S3 external template matching + S6 parameter objects + S8 mapping + backend | Code lacks reliable decompressor and has mapping caveats per consensus;不作为 pure trunk。 |
| `logblock` | shallow branch | S1 `log_format` + S6 direct transpose + S5 column heuristics + S10 single preprocessed text + S11 gzip/deflate/lz4 | 不共享 S7 trunk；source H2 wire 不可逆；无 full inverse implementation（logblock §3/§6）。 |
| `clp` | searchable branch | S1 timestamp + S3 schema/logtype + S6 segment columns + branch fixed64/dicts + S10/S12 segment index + S11 zstd/passthrough | Branch rejoin at S10/S11；logtype IDs source-backed `int64_t`，LZMA reader not core path（clp §4/§5/§7/§8）。 |
| `loggrep` | searchable branch | S3 LengthParser + S5 runtime real/nominal + S6 capsule columns + S10 CapsuleBox + S12 stamp/bitmap search + S11 per-capsule LZMA | `.outlier` correctness fallback；source-compatible stamp tag filtering caveat（loggrep §4/§7/§8）。 |
| `loglite` | S11 external only | Byte-level XOR/RLE/bitset stream -> S11 external cascade | No semantic `ColumnStore`；do not force into trunk（共识 c/e4）。 |
| `loggzip` | reference-only | S3 NCD parser reference only | gzip length oracle + plaintext CSV; no storage-compression artifact（共识 c/e4）。 |

Coverage statement：`denum`、`delog`、`logreducer`、`logshrink`、`lognexus` 是 5 个 source-backed trunk/trunk-dominant implementations；`logfold` 是 1 个 paper-only logical trunk fit，不能用于 physical-mapping proof；`logblock` 可作为 shallow branch 复用 S1/S6/S10/S11 但 preservation 通常不能 strict；`clp`、`loggrep` 在 S10/S11 rejoin 并保留 S12；`loglite` 只接 S11 external cascade；`loggzip` reference-only。

## 9. Final Design Decision

本框架的 authoritative decision 是：**可替换性来自 identical slot in/out types，而不是 wrapper/adapter**。

若一个实现能直接实现 `S3: TokenizedBatch -> ParsedBatch`、`S6: ParsedBatch -> ColumnStore`、`S7: IntColumnBundle -> EncodedInts`、`S8: StringColumnBundle -> DictionaryResult`、`S9: TypedColumnStore -> TransformStore`、`S10: EncodedColumnSet -> UnifiedChunk`，并把 decode-critical facts 写入 `PipelineSpec`、`SchemaManifest`、`ColumnDescriptor`、`DictDescriptor`、`OccurrenceStream`、`TransformGraph` 与 branch descriptors，它就是 interchangeable module。若它不能产出这些共享 IR，就按共识分类为 branch、external cascade 或 reference-only。这个规则同时解释了为什么 source-backed trunk 可以吸收 five strong implementations + one paper-only logical fit，为什么 CLP/LogGrep 必须保留 searchable branch，为什么 LogLite/LogGzip 不能被强行塞进 trunk。
