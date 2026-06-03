下面把前面讨论的几类编码，按 **GPU decoder / encoder 怎么实现**展开。核心思想是：GPU 最擅长的是 **规则访存、bit unpack、prefix-sum/scan、gather/scatter、reduction**；最怕的是 **变长 header 串行解析、字符串链式依赖、Huffman/varint 这种 variable-length control flow**。Parquet 官方 encoding 包括 `PLAIN`、`RLE`、`RLE_DICTIONARY`、`DELTA_BINARY_PACKED`、`DELTA_LENGTH_BYTE_ARRAY`、`DELTA_BYTE_ARRAY`、`BYTE_STREAM_SPLIT` 等；Sprintz 不是 Parquet 标准 encoding，但它的 predictor + zigzag + bitpack + RLE pipeline 可以和 Parquet delta/bitpack 放在一起实现和比较。([Parquet][1])

---

## 0. 一个通用 GPU Parquet/Sprintz decoding pipeline

实际工程里最好不要为每种 encoding 写一个巨大 monolithic kernel，而是拆成多个 microkernel：例如 page decompression、level decode、value decode、dictionary gather、string offset 构造、final scatter。NVIDIA RAPIDS/cuDF 的 Parquet GPU reader 就强调过 microkernel 化可以降低 register pressure、提高 occupancy，比一个大 kernel 更容易优化。([NVIDIA Developer][2])

典型流程是：

```text
compressed page bytes
  -> page compression decode: Snappy/ZSTD/LZ4/Gzip 等
  -> parse page header / encoding header
  -> decode definition/repetition levels
  -> decode values: PLAIN / RLE_DICTIONARY / DELTA / BYTE_STREAM_SPLIT / Sprintz
  -> assemble validity bitmap / offsets / child columns
  -> write Arrow/cuDF column buffer
```

GPU 实现上通常会先生成一个 `PageDecodeTask[]`：

```cpp
struct PageDecodeTask {
    uint8_t* input;
    uint8_t* output;
    int num_values;
    int physical_type;
    int encoding;
    int bit_width;
    int page_id;
    int column_id;
    // offsets into def-level, rep-level, dict page, etc.
};
```

然后每个 page / miniblock / run 被分配给一个 CTA 或 warp。复杂 encoding 一般采用 **two-pass**：

```text
pass 1: parse + size estimation
        生成 run/block descriptor，计算每段输出长度、临时 buffer 大小

prefix-sum:
        为 variable-size output 分配 offset

pass 2: materialize
        真正 unpack / gather / scatter / copy bytes
```

这套思路对 `RLE`、`DELTA_BINARY_PACKED`、dictionary string、Sprintz zero-run 都很重要。

---

## 1. `PLAIN`

### 1.1 fixed-width `PLAIN`

适合类型：`INT32`、`INT64`、`FLOAT`、`DOUBLE`、fixed decimal、timestamp 等。

**Decode 算法：**

```text
thread i:
    load input[i]
    maybe endian convert / cast
    store output[i]
```

CUDA 风格伪代码：

```cpp
__global__ void decode_plain_i32(
    const int32_t* in,
    int32_t* out,
    int n
) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) out[i] = in[i];
}
```

fixed-width `PLAIN` 的 GPU decode 基本是 memory bandwidth bound。复杂点通常不在 value decode，而在 definition level 产生的 null scatter：如果有 null，需要根据 def-level 把 decoded value scatter 到非空位置，同时生成 validity bitmap。

**Encode 算法：**

```text
thread i:
    load column[i]
    write plain[i]
```

如果列没有 null，encode 也是纯 copy。
如果有 null，通常先 compact 非空值：

```text
valid_i = is_valid(i)
rank_i = prefix_sum(valid_i)
if valid_i:
    plain_values[rank_i] = column[i]
```

这里的 `prefix_sum` 是 GPU 上的关键 primitive。

---

### 1.2 `PLAIN` variable-length byte array / string

Parquet `BYTE_ARRAY` 的 plain 形式是：

```text
[length_0][bytes_0][length_1][bytes_1]...
```

**Decode 难点：** 每个字符串长度不同，无法直接 `thread i -> output[i]`。

常见 GPU decode：

```text
pass 1:
    解析每个 string 的 length，得到 len[i]
    由于 length header 也是变长位置，需要找到每个 string 起点

pass 2:
    offsets = exclusive_scan(len)

pass 3:
    parallel copy bytes:
        output_chars[offsets[i] + j] = input[string_data_pos[i] + j]
```

但 `PLAIN BYTE_ARRAY` 的 `length_i` 和 `bytes_i` 交错存储，`string_data_pos[i]` 本身需要前面所有 `length + bytes` 的累加，所以一般有两种做法：

1. **一个 warp/CTA 顺序扫描一个 page**，生成 `len[]` 和 `input_pos[]`；
2. **CPU 或单独 GPU parser 先把 string descriptors 解析出来**，后续 copy 交给 GPU 大规模并行。

**Encode：**

如果上游是 Arrow/cuDF string layout：

```text
offsets[i], offsets[i+1], chars[]
```

那么：

```text
len[i] = offsets[i+1] - offsets[i]
encoded_size[i] = 4 + len[i]
encoded_pos = exclusive_scan(encoded_size)

thread i:
    write 4-byte len at encoded_pos[i]
    copy chars[offsets[i] : offsets[i+1]]
```

---

## 2. `BYTE_STREAM_SPLIT`

`BYTE_STREAM_SPLIT` 把 fixed-width 值的第 0 个字节、第 1 个字节……分别连续存储；它本身不直接压缩数据，而是重排字节以利于后续 page compression。Parquet 文档把它列为适用于 `FLOAT`、`DOUBLE`、`INT32`、`INT64`、`FIXED_LEN_BYTE_ARRAY` 等 fixed-width 类型的 encoding。([Parquet][1])

以 `float32` 为例，原始：

```text
v0: b00 b01 b02 b03
v1: b10 b11 b12 b13
v2: b20 b21 b22 b23
```

`BYTE_STREAM_SPLIT` 后：

```text
b00 b10 b20 ... | b01 b11 b21 ... | b02 b12 b22 ... | b03 b13 b23 ...
```

### Decode：de-interleave

简单实现：

```cpp
template<int W>
__global__ void decode_bss(
    const uint8_t* in,
    uint8_t* out,
    int n
) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i >= n) return;

    #pragma unroll
    for (int b = 0; b < W; ++b) {
        out[i * W + b] = in[b * n + i];
    }
}
```

其中 `W=4` 对应 `float/int32`，`W=8` 对应 `double/int64`。

更优化的实现会做 **shared memory tiled transpose**：

```text
tile: values x bytes
global load:  coalesced from byte streams
shared transpose
global store: coalesced or vectorized store to output values
```

因为 `W` 很小，很多场景下简单 one-thread-per-value 已经不错；真正瓶颈常常是后续 page compression decode 和内存带宽。

### Encode：interleave

反过来：

```cpp
template<int W>
__global__ void encode_bss(
    const uint8_t* values,
    uint8_t* out,
    int n
) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i >= n) return;

    #pragma unroll
    for (int b = 0; b < W; ++b) {
        out[b * n + i] = values[i * W + b];
    }
}
```

`BYTE_STREAM_SPLIT` 是最容易 GPU 化的编码之一，因为没有 prefix dependency，也没有 variable-length parse。

---

## 3. `RLE` / bit-packed hybrid

Parquet 的 `RLE` encoding 实际是 **RLE + bit-packing hybrid**，常用于 definition levels、repetition levels、boolean、dictionary ids 等。Parquet 文档把 `RLE` 描述为 current encoding，并且 `BIT_PACKED` 旧 encoding 已经 deprecated。([Parquet][1])

### Decode

输入大致是多个 run：

```text
run header
    if RLE run:
        repeated value + run length
    if bit-packed run:
        packed groups of values
```

GPU decode 难点是：header 是 varint，run 长度不同。一个常见设计是两阶段：

#### pass 1：parse run descriptors

```text
input byte stream
  -> RunDesc[]
       type: RLE or BITPACK
       input_offset
       output_offset
       num_values
       bit_width
       repeated_value / packed_offset
```

这个 pass 可以：

* CPU parse；
* 或一个 warp/CTA 顺序 parse 一个 page；
* 或者 GPU 上 page-level parallel，但 run 内 header 仍然有串行性。

因为每个 page 的 run 数通常远小于 value 数，parse 串行不一定是瓶颈。

#### pass 2：parallel materialize

RLE run：

```cpp
__global__ void expand_rle(
    RunDesc* runs,
    int* out
) {
    int r = blockIdx.x;
    RunDesc run = runs[r];

    for (int k = threadIdx.x; k < run.num_values; k += blockDim.x) {
        out[run.output_offset + k] = run.value;
    }
}
```

Bit-packed run：

```text
thread handles one value or one group of 8 values:
    bit_pos = local_index * bit_width
    word = load bytes covering bit_pos
    value = extract bits
```

伪代码：

```cpp
uint32_t unpack_one(const uint8_t* base, int idx, int bit_width) {
    int bit = idx * bit_width;
    int byte = bit >> 3;
    int shift = bit & 7;

    uint64_t chunk = load_unaligned_64(base + byte);
    return (chunk >> shift) & ((1u << bit_width) - 1);
}
```

NVIDIA nvCOMP 的 cascaded compression 文档也把 RLE、delta、bitpacking 作为 GPU-friendly building blocks；其中 delta decode 可以用 exclusive prefix sum，bitpacking 每个 symbol 可以并行定位和 unpack。([NVIDIA Docs][3])

### Encode

RLE/bit-pack encode 更复杂，因为输出是 variable-size。

典型流程：

```text
input values
  -> equal_to_previous flags
  -> run boundary flags
  -> run_id = prefix_sum(boundary)
  -> run length / run value
  -> choose RLE or bitpack
  -> size per run
  -> output_offset = prefix_sum(size)
  -> emit headers + payload
```

RLE run detection：

```cpp
boundary[i] = (i == 0) || (x[i] != x[i-1]);
run_id[i] = exclusive_scan(boundary);
```

为了简单和高吞吐，很多 GPU writer 会对 dictionary ids / levels 采用 **bit-pack-first** 或者只对足够长的 run 使用 RLE，避免大量短 run 导致 header 开销和 branch divergence。

---

## 4. `RLE_DICTIONARY` / `PLAIN_DICTIONARY`

现代 Parquet dictionary encoding 一般是：

```text
dictionary page:
    dictionary values, usually PLAIN encoded

data page:
    dictionary ids, RLE/bit-packed encoded
```

Parquet 文档中 `RLE_DICTIONARY` 是当前 dictionary data page encoding，`PLAIN_DICTIONARY` 是 deprecated 旧标记。([Parquet][1])

### Decode

分三步：

```text
1. decode dictionary page
2. decode dictionary ids via RLE/bit-packed hybrid
3. gather dictionary values
```

#### fixed-width dictionary

```cpp
__global__ void dict_gather_i32(
    const int32_t* dict,
    const uint32_t* ids,
    int32_t* out,
    int n
) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) out[i] = dict[ids[i]];
}
```

这非常适合 GPU：`ids` 连续读，`out` 连续写，`dict` 是随机 gather。字典小的时候可以放进 shared memory 或 L2 cache 命中率很高。

#### string dictionary

string dictionary 通常是：

```text
dict_offsets[]
dict_chars[]
ids[]
```

decode 要做：

```text
len[i] = dict_offsets[ids[i]+1] - dict_offsets[ids[i]]
out_offsets = exclusive_scan(len)

parallel copy:
    out_chars[out_offsets[i] + j] =
        dict_chars[dict_offsets[ids[i]] + j]
```

这里的瓶颈是 variable-length copy。常见策略：

```text
短字符串:
    一个 warp 处理多个 string

长字符串:
    一个 CTA 处理一个或几个 string，线程协作 copy bytes

混合:
    根据 length bucket 分流，减少 divergence
```

### Encode

dictionary encode 的 GPU 难点在 **build dictionary**：

```text
input values
  -> hash table insert / lookup
  -> assign dictionary id
  -> compact unique values into dictionary
  -> encode ids with RLE/bitpack
```

fixed-width 类型可以用 GPU hash table：

```cpp
id = hash_table.insert_or_find(value)
ids[i] = id
```

但要保证 deterministic dictionary order、处理冲突、控制 load factor。string dictionary 更难，因为 hash 前要读 variable-length bytes，还要处理相等性比较。工程上常见策略：

* 低 cardinality fixed-width：GPU dictionary encode；
* 高 cardinality 或 string：采样估计是否值得 dictionary；
* 不值得时 fallback 到 `PLAIN` / `DELTA_LENGTH_BYTE_ARRAY`；
* 或者 CPU/GPU hybrid 构字典。

---

## 5. `DELTA_BINARY_PACKED`

`DELTA_BINARY_PACKED` 用于 `INT32` / `INT64`。Parquet 的 delta binary packed 会存 first value、block/miniblock 信息、min delta，以及每个 miniblock 的 bit width；相同值 block 可以用 0 bit width 表示。([Parquet][1])

### Decode

逻辑上：

```text
first_value
deltas = unpack(bitpacked_data) + min_delta
values = first_value + prefix_sum(deltas)
```

GPU decode 设计：

```text
pass 1:
    parse block headers:
        first value
        block size
        miniblock count
        min_delta per block
        bit_width per miniblock
        payload offset

pass 2:
    parallel bit-unpack normalized deltas

pass 3:
    add min_delta

pass 4:
    prefix-sum deltas to reconstruct original values
```

局部 block scan：

```cpp
// within one block
delta[i] = unpacked[i] + min_delta;
value[i] = first_value + inclusive_scan(delta)[i];
```

如果一个 page 很大，通常做 hierarchical scan：

```text
CTA-level scan:
    each CTA scans a segment
    output segment_sum[]

global scan:
    prefix-sum segment_sum[]

fixup kernel:
    add scanned segment prefix to each segment
```

NVIDIA nvCOMP 文档也明确把 delta decode 描述成 exclusive prefix sums；这正是 GPU 上很成熟的 primitive。([NVIDIA Docs][3])

### Encode

```text
input values x[i]

1. delta[i] = x[i] - x[i-1]
2. per block min_delta = reduce_min(delta)
3. normalized[i] = delta[i] - min_delta
4. per miniblock bit_width = bits_required(max(normalized))
5. size pass:
       header size + packed payload size
6. prefix-sum sizes -> output positions
7. emit headers + bit-packed payload
```

伪代码：

```cpp
delta[i] = x[i] - x[i-1];

for each block:
    min_delta = reduce_min(delta)
    norm[i] = delta[i] - min_delta

for each miniblock:
    bw = 32 - clz(reduce_or(norm))
    bitpack(norm, bw)
```

注意 encode 端有 variable-size output，因此必须有 size pass 和 prefix-sum。
另外，`DELTA_BINARY_PACKED` 对 sorted integer、timestamp、递增 id 很好；对随机整数会退化成接近 fixed-width bitpack。

---

## 6. `DELTA_LENGTH_BYTE_ARRAY`

`DELTA_LENGTH_BYTE_ARRAY` 把 byte array 的长度序列用 `DELTA_BINARY_PACKED` 编码，后面接连续拼接的 byte payload。Parquet 文档也是这样定义它的。([Parquet][1])

### Decode

```text
input:
    encoded lengths
    concatenated bytes

decode:
    len[] = delta_binary_packed_decode(length_stream)
    offsets[] = exclusive_scan(len)
    copy payload bytes to output chars
```

伪代码：

```cpp
// after lengths decoded
offsets = exclusive_scan(lengths);

__global__ void copy_payload(
    const uint8_t* payload,
    const int32_t* offsets,
    uint8_t* out_chars,
    int total_bytes
) {
    int p = blockIdx.x * blockDim.x + threadIdx.x;
    if (p < total_bytes) out_chars[p] = payload[p];
}
```

这里 payload 本身已经是连续拼接的，所以比 `PLAIN BYTE_ARRAY` 更适合 GPU：你不需要在 interleaved `[len][bytes]` 结构里解析每个 string 起点。

### Encode

如果输入是 Arrow-style strings：

```text
offsets[i], offsets[i+1], chars[]
```

那么：

```text
len[i] = offsets[i+1] - offsets[i]
encode len[] using DELTA_BINARY_PACKED
copy chars[] as payload
```

这类 encoding 对 GPU 比较友好，因为主要是 length delta encode + prefix/scan + contiguous bytes copy。

---

## 7. `DELTA_BYTE_ARRAY`

`DELTA_BYTE_ARRAY` 是前缀压缩：每个字符串存和前一个字符串共享的 prefix length，以及剩余 suffix。Parquet 文档说明它适用于 `BYTE_ARRAY` 和 `FIXED_LEN_BYTE_ARRAY`，常用于具有公共前缀的字符串。([Parquet][1])

### Decode

逻辑：

```text
prefix_len[i] = ...
suffix[i] = ...
value[i] = value[i-1][0 : prefix_len[i]] + suffix[i]
```

问题是 `value[i]` 依赖 `value[i-1]`。这对 GPU 很不友好。

#### 基础实现：page 内顺序 decode

```cpp
for i in page:
    copy prefix from previous decoded string
    append suffix
```

可以让一个 CTA 处理一个 page，CTA 内线程协作 copy 当前 string 的 bytes，但 string 顺序还是串行：

```text
for i = 0..n-1:
    parallel copy prefix bytes
    parallel copy suffix bytes
    sync
```

这能利用一些带宽，但无法充分利用 GPU 的 massive parallelism。

#### 分块 checkpoint 优化

如果是你自定义格式，可以每 K 个字符串存一个 full anchor：

```text
anchor at i = 0, K, 2K, ...
within each chunk:
    decode sequentially
chunks:
    parallel
```

但标准 Parquet 文件不一定有这种 checkpoint，所以对已有 `DELTA_BYTE_ARRAY` 文件不能直接假设。

#### Encode

encode 反而比 decode 更容易并行：

```text
for each i in parallel:
    lcp[i] = longest_common_prefix(value[i-1], value[i])
    suffix[i] = value[i][lcp[i]:]
```

每对相邻字符串的 LCP 可以由一个 warp 或 CTA 比较。之后：

```text
prefix_len[] -> DELTA_BINARY_PACKED
suffix[]     -> DELTA_LENGTH_BYTE_ARRAY
```

但是 LCP 比较本身对 string length 敏感，容易 warp divergence。

**结论：** `DELTA_BYTE_ARRAY` 可以 GPU encode，但 GPU decode 通常不如 `DELTA_LENGTH_BYTE_ARRAY` / dictionary string 友好。

---

## 8. Sprintz-delta

Sprintz 是一个 time-series compression algorithm，主要 pipeline 是 forecasting/prediction、residual、zigzag、bit packing、run-length encoding，并且原论文强调它面向 multivariate integer time series，内存开销低、server-side 单线程 decompression 可达多 GB/s。([arXiv][4]) Apache IoTDB 文档也把 SPRINTZ 描述为 prediction + zigzag + bit-packing + RLE，并指出它适合相邻值绝对差较小的时间序列。([IoTDB Website][5])

这里先讲最 GPU-friendly 的 **Sprintz-delta**，也就是 predictor 取前一个值：

```text
pred[i] = x[i-1]
residual[i] = x[i] - pred[i]
zz[i] = zigzag(residual[i])
bitpack zz[]
zero-block RLE
```

### Decode

```text
1. parse block header
2. if zero residual block:
       residuals = 0
   else:
       bit-unpack residuals
3. inverse zigzag
4. reconstruct:
       x[i] = x[i-1] + residual[i]
```

因为 delta reconstruction 是 prefix-sum：

```text
x[i] = x[0] + prefix_sum(residual)[i]
```

所以 GPU decode 可以这样做：

```cpp
residual[i] = inverse_zigzag(unpack(...));
x[i] = base + inclusive_scan(residual)[i];
```

对 multivariate time series：

```text
layout: time x dimension

策略 A:
    一个 warp 处理一个 dimension 的一个 block

策略 B:
    一个 CTA 处理一个 sensor/page，threads 覆盖 time 和 dimension

策略 C:
    如果 dimension 很多，按 dimension 并行；
    如果每条序列很长，按 segment 并行 + carry fixup
```

### Encode

```text
1. residual[i] = x[i] - x[i-1]
2. zz[i] = zigzag(residual[i])
3. per block:
       bw = bits_required(reduce_or(zz))
4. if bw == 0:
       emit zero-block run
   else:
       bitpack block
```

伪代码：

```cpp
resid[i] = x[i] - x[i-1];
zz[i] = (resid[i] << 1) ^ (resid[i] >> 31);

block_or = reduce_or(zz in block);
bw = 32 - clz(block_or);

if bw == 0:
    mark_zero_block
else:
    bitpack(zz, bw)
```

zero-block RLE encode：

```text
zero_flag[block] = (bw == 0)
run boundaries = zero_flag changes
run_id = prefix_sum(boundary)
emit zero-run descriptors
```

Sprintz-delta 和 `DELTA_BINARY_PACKED` 的实现非常像；区别是 Sprintz 更小 block、更强调 zero residual block，并且通常面向 multivariate time series。

---

## 9. Sprintz-FIRE predictor

Sprintz 论文里除了 delta predictor，还有 FIRE 这类 online forecasting 方法；论文摘要强调其 forecasting algorithm 可以 online training，并且相比 delta coding 改善压缩率。([arXiv][6])

GPU 实现的核心问题是：

```text
predict x[i]
observe residual
update predictor state
predict x[i+1]
```

也就是每条序列内部有状态依赖。

### Decode / encode 策略

#### 策略 1：跨序列并行

如果你有很多 sensor / metric / column：

```text
thread block 0 -> series 0
thread block 1 -> series 1
...
```

每条序列内部顺序跑 predictor，但不同序列并行。这适合 telemetry 场景。

#### 策略 2：chunk checkpoint

自定义格式可以每 K 个点存 predictor state checkpoint：

```text
checkpoint:
    last value
    predictor weights/state

chunk 0 decode independently
chunk 1 decode independently
...
```

这样可以牺牲一点压缩率换 GPU 并行度。

#### 策略 3：hybrid predictor

在 GPU scan-heavy 场景下，可以考虑：

```text
writer:
    使用 FIRE 压缩冷数据

reader:
    对 scan-heavy 热数据改用 Sprintz-delta / Parquet delta
```

因为 FIRE 的压缩率优势可能会被 decode 串行依赖抵消。

---

## 10. Sprintz + Huffman / entropy coding

Sprintz pipeline 里可以叠加 entropy coding。对 GPU 来说，这一层要谨慎。

Huffman decode 的问题是：

```text
variable-length code
bit boundary unknown
每个 symbol 的起点依赖前面 symbol 的 code length
```

可行方案：

1. **block-level Huffman**：每个 compressed block 独立解，block 之间并行；
2. **canonical Huffman + lookup table**：一次查 8~12 bits，减少 branch；
3. **checkpoint bit offsets**：每隔 K 个 symbol 存 bit offset，允许 chunk parallel decode；
4. **跳过 Huffman**：保留 Sprintz 的 predictor + zigzag + bitpack + RLE，把 page compression 交给 ZSTD/LZ4/Snappy。

如果目标是 GPU scan throughput，我通常建议先做 **Sprintz without Huffman**。先把 bitpack、zero-run、delta scan 做到很快，再看 entropy coding 是否值得。

---

## 11. definition / repetition levels 的 GPU 处理

Parquet nested data 里，definition level 和 repetition level 通常用 RLE/bit-packed hybrid。它们不是 value encoding，但对最终列构造非常关键。Parquet 文档也说明 `RLE` 用于 levels，dictionary ids 也使用 RLE/bit-packed hybrid。([Parquet][1])

### definition level -> validity bitmap

```text
is_valid[i] = def_level[i] == max_def_level
```

GPU 生成 bitmap：

```cpp
valid = (def[i] == max_def);
if (valid) atomic_or(bitmap[word], 1 << bit);
```

更高效做法是 warp-level ballot：

```cpp
uint32_t mask = __ballot_sync(FULL_MASK, valid);
if (lane == 0) validity_words[warp_id] = mask;
```

### repetition level -> list offsets

对于 repeated/list：

```text
new_list = rep_level[i] == 0
list_id = prefix_sum(new_list)
```

然后根据 `list_id` 构造 offsets。这个过程本质是 flag + prefix-sum + scatter，非常适合 GPU，但细节多。

---

## 12. 各编码的实现优先级

| 编码                           | Decode 实现难度 | Encode 实现难度 | GPU 关键 primitive                                 | 建议                  |
| ---------------------------- | ----------: | ----------: | ------------------------------------------------ | ------------------- |
| fixed-width `PLAIN`          |           低 |           低 | copy / scatter                                   | 最先做                 |
| `BYTE_STREAM_SPLIT`          |           低 |           低 | byte transpose                                   | 最先做                 |
| `RLE` / bit-pack             |           中 |          中高 | varint parse、bit unpack、run expansion、prefix-sum | 必做                  |
| `RLE_DICTIONARY` fixed-width |           中 |           高 | RLE decode、gather、GPU hash table                 | decode 先做           |
| `RLE_DICTIONARY` string      |          中高 |           高 | gather length、scan、variable copy                 | decode 可做，encode 谨慎 |
| `DELTA_BINARY_PACKED`        |           中 |          中高 | bit unpack、scan、min reduce                       | 很值得做                |
| `DELTA_LENGTH_BYTE_ARRAY`    |           中 |           中 | delta decode、scan、byte copy                      | string 场景值得做        |
| `DELTA_BYTE_ARRAY`           |           高 |          中高 | LCP、链式 prefix restore                            | decode 不优先          |
| Sprintz-delta                |           中 |           中 | zigzag、bitpack、scan、zero-run RLE                 | time-series 很值得做    |
| Sprintz-FIRE                 |           高 |           高 | per-series state machine                         | 压缩率优先时再做            |
| Sprintz + Huffman            |           高 |           高 | variable-length entropy decode                   | GPU scan 场景慎用       |

---

## 13. 我会采用的工程路线

如果目标是做一个高吞吐 GPU Parquet/Sprintz decoder，我会按这个顺序实现：

```text
Phase 1:
    fixed-width PLAIN
    BYTE_STREAM_SPLIT
    RLE/bit-packed levels
    RLE_DICTIONARY fixed-width

Phase 2:
    DELTA_BINARY_PACKED
    DELTA_LENGTH_BYTE_ARRAY
    dictionary string decode

Phase 3:
    Sprintz-delta numeric time-series
    zero-block RLE
    chunk-level scan + carry fixup

Phase 4:
    DELTA_BYTE_ARRAY
    Sprintz-FIRE
    optional entropy coding
```

一句话总结：**GPU 编解码的核心不是“把 CPU decoder 搬到 CUDA”，而是把每种 encoding 拆成 scan、reduce、bitpack/unpack、gather、scatter、byte-copy 这些 GPU primitive；`BYTE_STREAM_SPLIT`、fixed-width `PLAIN`、RLE/bitpack、dictionary id、delta-binary-packed、Sprintz-delta 最自然，`DELTA_BYTE_ARRAY`、FIRE、Huffman 这类有链式依赖或 variable-length control flow 的部分最需要谨慎。**

[1]: https://parquet.apache.org/docs/file-format/data-pages/encodings/?utm_source=chatgpt.com "Encodings | Parquet - Apache Parquet"
[2]: https://developer.nvidia.com/blog/accelerating-apache-parquet-scans-on-apache-spark-with-gpus/?utm_source=chatgpt.com "Accelerating Apache Parquet Scans on Apache Spark with GPUs"
[3]: https://docs.nvidia.com/cuda/nvcomp/cascaded.html?utm_source=chatgpt.com "Cascaded Compression — nvCOMP - NVIDIA Documentation Hub"
[4]: https://arxiv.org/pdf/1808.02515?utm_source=chatgpt.com "Sprintz: Time Series Compression for the Internet of Things"
[5]: https://iotdb.apache.org/UserGuide/latest/Technical-Insider/Encoding-and-Compression.html?utm_source=chatgpt.com "Compression & Encoding | IoTDB Website"
[6]: https://arxiv.org/abs/1808.02515?utm_source=chatgpt.com "Sprintz: Time Series Compression for the Internet of Things"
