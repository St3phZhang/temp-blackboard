# LogShrink 原始实现 — round-trip 修复记录

原版 LogShrink(IntelligentDDS ICSE2024,`sources/logshrink/`)在本项目样本上开箱只有 Apache 能无损 round-trip。经诊断,其余 8 个数据集的失败**不是算法固有有损**,而是一批可定位的实现 bug。修复后 **9/9 数据集 value-equivalent round-trip 全通过,CR 保持原始水平**(所有修复都是正确性修复,不存原文、不牺牲压缩率)。

结果数据: [`results/logshrink_semantic_roundtrip.csv`](../results/logshrink_semantic_roundtrip.csv) + [说明](../results/logshrink_semantic_roundtrip.md)。

## 验收口径

- round-trip 判定 = **value-equivalent**:折叠空白 + 数字 token 按整数值归一化(strip 前导零,`09`==`9`、`00:01`==`0:1`),非数字内容保持不变。
- **前导零/字段宽度差异可接受**(不为此存原文,存原文会毁 CR);**整数值错误/内容损坏不可接受**,必须修。

## 修复清单(6 处,均在 `sources/logshrink/python_compression/`)

| # | 症状 | 数据集 | 根因(file) | 修复 |
|---|---|---|---|---|
| 1 | `24773`→`-18176`、`188978561024`→`0` 整数溢出 | Mac, Zookeeper | `parser/elastic.cpp` + `parser/elastic.h`:zigzag/varint 用 `int`/`uint32_t`,存不下 >2^31 / >2^32 的值 | zigzag/varint 链升 `int64_t`/`uint64_t`,varint 上限 5→10 字节(`ceil(64/7)`);`>>31`→`>>63`;`%d`→`%lld`。**注意 elastic.h 的签名改动会波及 main.cpp 数组类型;实测 pipeline 只走标准 `Elastic e ... z` 二进制(elastic.cpp),故只需改 elastic.cpp;elastic.h 保持不动以免连锁改 main.cpp** |
| 2 | 段末尾最后一行丢 header | Hpc, Ssh | `parser/main.cpp`:EOF 最后一条 match-failed 记录写 `linebuf`(仅 body)而非 `totalBuf`(完整行) | 改写 `totalBuf`,与非 EOF match-failed 一致 |
| 3 | `TID 1).`→`)..`、`(1.01`→`(1(.1` 分隔符重复 | Spark, Proxifier | `analyzer/property_miner.py`:变量内 pattern 拆分没消耗边界分隔符 → 分隔符既留在值里又被 pattern 回放 | 拆分改为消耗分隔符 + 允许空片段,`recover_pat()` 契约不变 |
| 4 | 非 ASCII 字节 mojibake(`\xc3\xb7`→`\xc3\x83\xc2\xb7`) | Linux | 分段/列 I/O 默认 UTF-8,把原始字节按 latin1 解成 Unicode 再按 UTF-8 写回(double-encoding) | `utils/util_code.py`(分段写)、`preprocess.py`、`compression/compress.py`(.str/dict)、`decompression/{restore,decompress}.py` 全部显式 `ISO-8859-1`,让字节透传 |
| 5 | `FA\|\|Url\|\|taskID[..]`→`FA` 中段整段丢失 | Mac | `parser/main.cpp` + `preprocess.py`:THULR 中间文件 `var.var` 用 `\|\|` 分隔字段,变量值本身含 `\|\|` → `read_var()` 把一个变量误切成多个,restore 只取第一个 | 写 `var.var` 时转义 `%`→`%25`、`\|`→`%7C`(`escapeVarField`),读时反转义 |
| 6 | NUL 前导行恢复成空(`\x00...日志`→``) | Healthapp | `parser/main.cpp`:以 `\x00` 开头的行 `strlen()==0`,被空行分支拦截,`load_failed.log` 只写了 `""` 而非原始字节;真正的 badChar 分支(逐字节写)永远到不了 | 空行判断加 `&& !badChar` 守卫,让 badChar 行落到 `failProcess(ff, line, readSize)` 逐字节存原行(与 match_failed 一致),解压按 `eids.eid` 的 `-1` 哨兵回填 |

## 关键教训

1. **"原始实现有损" 往往是 bug 不是设计。** 曾一度把 Healthapp NUL 行判为"设计固有损失",实为 `strlen==0` 误判 —— load_failed 本就该像 match_failed 一样存原始行。先假设可修,读代码定位,别轻易归因"算法固有"。
2. **修 round-trip 不该牺牲 CR。** 正解是修**正确性 bug**(编码位宽、分隔符转义、编码透传、分支误判),不是存原文兜底。存原文能强行 byte 无损但会让 CR 暴跌(实测 Zookeeper 124→37),是错误方向。
3. **诊断先隔离层次**:溢出/编码类先用**孤立微基准**(单独喂 Elastic 二进制、单行字节 diff)锁定根因 + 量化代价,再改;不要直接大改后跑全量。
4. **int64 修复的 CR 代价极小**:varint 层膨胀被 LZMA 吸收,6/9 数据集 ~0%,最差 Zookeeper 端到端 ~0.15%。
