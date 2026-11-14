# ClusterBooks

批量中文文档（书籍）自动领域分类工具：读取一批 `.txt` / `.md` 文本，提取文档特征，用 MinHash + LSH 快速召回相似文档对，再以 Leiden 社区发现把整个语料库划分成主题簇（如"法律""历史""物理"），最后把文件归入对应的文件夹。

纯 Swift + Foundation 实现，**零第三方依赖**，单文件即可编译。

## 项目结构

| 文件 | 说明 |
|---|---|
| `CoreWordKit.swift` | 共享算法库：N-gram + NPMI + 左右信息熵的核心词提取器（内存优化的紧凑实现） |
| `ExtractCoreWords.swift` | 核心词提取 CLI，可对单本书/目录独立运行 |
| `ClusterBooks.swift` | 批量聚类主程序：MinHash → LSH → 精确 Jaccard → Leiden → 簇命名 → 归档移动 |

## 算法流水线

```
文档文本
  │
  ├─ 方案一 core-words    ：N-gram + NPMI（凝固度）+ 左右信息熵（自由度）
  │                         提取每本书的核心词 → 词集合
  ├─ 方案二 char-shingles ：清理文本（汉字+小写字母）→ 字符 4-shingle
  │                         → 去重 / 停用词 / bottom-k 采样 → 特征集合
  │
  ▼
MinHash(128 维签名，确定性哈希族，可复现)
  ▼
LSH(32 bands × 4 rows)          ← 候选对召回，超大盘自动跳过防 O(n²)
  ▼
精确 Jaccard(词 ID 双指针)       ← sim > threshold 才建边，sim 为边权
  ▼
加权稀疏图 → Leiden 社区发现
  ▼
簇命名(簇内文档核心词按文档频次合并取 top3)
  ▼
归类移动（默认 dry-run，--apply 才真正移动，manifest 支持 --undo 回退）
```

关键设计：

- **MinHash**：128 个独立哈希函数 `h(x) = (a·x + b) mod (2^61-1)`，参数由固定种子 SplitMix64 生成，跨运行可复现。
- **Leiden**：局部移动（加权模块度）→ refinement 细化 → 图聚合多层迭代；孤立节点单独成簇。
- **目标簇数**：γ（分辨率）越大簇越多，在 `[γmin, γmax]` 上对数二分搜索逼近期望簇数；必要时 `--force-clusters` 层级贪心硬合并，超大簇可再拆分。
- **内存**：汉字映射 UInt16 ID 压成单条流，N-gram 打包成 16 字节紧凑键；熵计算用扁平计数 + `Σc·log2c` 累加，避免每候选一个小字典。
- **安全**：默认 dry-run；每次移动写入 manifest，`--undo` 可完整回退。

## 编译

```bash
# 核心词提取 CLI
swiftc -O -o ecw CoreWordKit.swift ExtractCoreWords.swift

# 批量聚类
swiftc -O -o bookcluster CoreWordKit.swift ClusterBooks.swift
```

要求 macOS（用到 GB18030 编码的 CoreFoundation 支持）与 Swift 5.x。

## 使用

### 1. 单本书核心词提取

```bash
./ecw book.md --format words --top 500
./ecw 某目录 --format json --output words.json   # 目录会递归收集 txt/md
cat book.txt | ./ecw - --format table            # 支持管道输入
./ecw --self-test                                # 内置断言自检
```

常用选项：`--min-npmi`（凝固度下限）、`--min-entropy`（左右熵下限）、`--stopwords`（追加自定义停用词）、`--length-bonus`（长词加权）、`--encoding gb18030`（自动检测 UTF-8/GB18030/Latin1）。

### 2. 批量聚类归类书籍

```bash
# 预览（默认 dry-run，不移动任何文件）
./bookcluster /Volumes/SSD/books-md --report /tmp/report.json

# 指定期望簇数（γ 二分搜索 + 硬合并兜底）
./bookcluster /Volumes/SSD/books-md --target-clusters 30 --apply

# 对比两种特征方案
./bookcluster /Volumes/SSD/books-md --scheme char-shingles

# 撤销上一次移动
./bookcluster --undo /Volumes/SSD/books-md/.bookcluster_manifest.json

# 内置自检
./bookcluster --self-test
```

完整参数见 `--help`（特征方案、缓存、阈值、目标簇数搜索、拆分/合并策略、输出与归类路径等均可调）。

### 自检

两个工具均内置 `--self-test`，覆盖切分/编码往返/NPMI 公式/熵公式/过滤开关/子串碎片判定/端到端提取与文件加载等断言。

## License

MIT
