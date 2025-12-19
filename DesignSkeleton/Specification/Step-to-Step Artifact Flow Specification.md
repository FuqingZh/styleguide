# 📘 Step-to-Step Artifact Flow Specification

## v1.0

------

### 1. Overview（总览）

本规范旨在明确多步骤（step-based）工作流中：

- 每个步骤输出哪些产物（artifacts）
- 哪些产物可供下游使用（public artifacts）
- 哪些产物仅内部使用（private artifacts）
- 各步骤之间如何通过 artifact 发生数据传递

该文档确保：

- Pipeline 每一部分可独立复跑
- 串联部分语义清晰、无歧义
- 下游模块不需要访问上游的 tmp/
- 所有模块通过可复现数据与语义真源通信
- Artifact 版本化与 schema 稳定性可保证长期演进

------

### 2. 总体 Artifact 分层模型（Three-Layer Model）

每个步骤产生的产物分为三类：

```
┌────────────────────────────┐
│   Layer 1: Semantic Truth   │  ← configs.yaml（参数真源）
└────────────────────────────┘
┌────────────────────────────┐
│   Layer 2: Data Truth       │  ← parquet（数据真源）
└────────────────────────────┘
┌────────────────────────────┐
│   Layer 3: Manifest Layer   │  ← meta.json（索引与摘要）
└────────────────────────────┘
```

- configs.yaml：影响分析逻辑的语义真源（semantic source of truth）
- parquet / parquet.dataset：作为下游数据分析的主数据源
- meta.json：告知“这个 step 输出了什么”（入口 manifest），但不记录分析参数

------

### 3. Step Graph（步骤依赖图）

针对 Proteomics，典型步骤链如下：

```
Intake
   ↓
Normalize
   ↓
Filter (optional)
   ↓
Diff (t-test / ANOVA)
   ↓
Annotation
   ↓
Enrichment (GO / KEGG)
   ↓
Visualization / Report
```

每个步骤之间依赖的 不是整个目录，而是明确的 artifact 子集。

------

### 4. 标准 Step-to-Step Artifact Flow

下面对每个步骤进行规范说明：

------

#### 4.1 Step: Intake

##### Input

- sample_info.json
- contrast_info.json
- project_info.json
- MQ/DIA-NN 配置文件
- options.xlsx

##### Output / Public Artifacts

```
out/
  meta.json
  info-samples.parquet
  info-contrasts.parquet
  info-samples.tsv
  info-contrasts.tsv
configs/
  configs.yaml
```

##### Downstream Consumption

| 下游 step  | 读取内容                                               |
| ---------- | ------------------------------------------------------ |
| Normalize  | configs.yaml（平台信息）、info-samples、info-contrasts |
| Diff       | configs.yaml、info-contrasts                           |
| Annotation | configs.yaml、project/ref 信息                         |

##### 不可被下游读取的内容

- intake/tmp/（任何临时文件）
- intake/logs/

------

#### 4.2 Step: Normalize（归一化）

##### Input Artifacts

来自 Intake：

- configs.yaml（必须）
- info-samples.parquet（定义 sample 列顺序与所属组）
- 原始定量矩阵（从 MQ 或 DIA-NN 原始文件路径中读取，或通过 intake 解析的路径）

##### Output / Public Artifacts

```
normalize/out/
  meta.json
  matrix-normalized.parquet       ← 数据真源
  matrix-normalized.tsv           ← 可读（可选）
configs/
  configs.yaml                    ← 派生于上一 step
```

##### Downstream Consumption

| Step   | 读取内容                  |
| ------ | ------------------------- |
| Filter | matrix-normalized.parquet |
| Diff   | matrix-normalized.parquet |
| QC     | matrix-normalized.parquet |

Normalize 的产出是整个 pipeline 的第一份“标准化分析矩阵”，是后续步骤的统一输入。

------

#### 4.3 Step: Filter（可选，缺失值过滤 / low-quality filtering）

##### Input

- configs.yaml（读取过滤参数）
- matrix-normalized.parquet
- info-samples.parquet

##### Output / Public Artifacts

```
filter/out/
   meta.json
   matrix-filtered.parquet
configs/
   configs.yaml
```

##### Downstream Consumption

| Step | 输入文件                        |
| ---- | ------------------------------- |
| Diff | matrix-filtered.parquet（若有） |

注意：
 如果 Filter step 被跳过，下游 Diff step 直接读取“归一化”矩阵。

------

#### 4.4 Step: Diff（差异分析 t-test / ANOVA）

##### Input

- configs.yaml（阈值、平台信息）
- info-contrasts.parquet
- matrix-(normalized or filtered).parquet

##### Output / Public Artifacts

```
diff/out/
  meta.json
  diff-results.parquet
  diff-significant.parquet
  diff-results.tsv
configs/
  configs.yaml
```

- diff-results.parquet：所有蛋白 + 所有对比的完整差异分析表
- diff-significant.parquet：符合阈值的差异蛋白列表

##### 下游消费：

| Step       | Input                    |
| ---------- | ------------------------ |
| Annotation | diff-significant.parquet |
| Enrichment | diff-significant.parquet |

------

#### 4.5 Step: Annotation（注释）

##### Input

- configs.yaml（物种、ref 信息）
- diff-significant.parquet

##### Output

```
annotation/out/
  meta.json
  protein2go.parquet
  protein2kegg.parquet
configs/
  configs.yaml
```

##### 下游消费：

- Enrichment (ORA/GSEA)
- Report 生成

------

#### 4.6 Step: Enrichment（富集）

##### Input

- diff-significant.parquet
- ref 信息（来自 configs.yaml）
- annotation 表（GO/KEGG/eggNOG 等）

##### Output

```
enrich/out/
  meta.json
  go_enrich.parquet
  kegg_enrich.parquet
  enrichment-plots/*.svg
configs/
  configs.yaml
```

##### 下游消费：

- Visualization
- Report

------

#### 4.7 Step: Report（交付报告）

##### Input

来自所有步骤的公共 artifact：

- configs.yaml
- diff-results.parquet
- enrichment 表
- normalized/filter matrix（用于 QC）

##### Output

```
report/out/
  meta.json
  summary.html
  summary.pdf
  imgs/
configs/
  configs.yaml
```

Report 是最终交付，没有下游依赖。

------

### 5. Artifact Dependency Graph（产物依赖图）

非常关键：
 this is the *source of truth* for pipeline orchestration.

```
Intake/info-samples.parquet ───────► Normalize ─────► Filter ───► Diff ───► Annotation ───► Enrichment ───► Report
Intake/info-contrasts.parquet ─────┘
Intake/configs.yaml ───────────────► all subsequent steps
```

换成逐文件流：

| 文件                         | 产生于          | 消费于                | 说明                           |
| ---------------------------- | --------------- | --------------------- | ------------------------------ |
| configs.yaml             | every step      | every next step       | 每步继承并传递，不新增分析参数 |
| meta.json                | every step      | next-step loader      | 仅作索引，不传递语义           |
| info-samples.parquet     | intake          | normalize/filter/diff | 仅 intake 生成                 |
| normalized matrix        | normalize       | filter/diff           | 规范化后统一入口               |
| filtered matrix          | filter          | diff                  | optional                       |
| diff-significant.parquet | diff            | annotation/enrichment | 标准前景集                     |
| annotation tables        | annotation      | enrichment            | 用于 enrich                    |
| enrich tables            | enrichment      | report                | 报告依赖                       |
| plots                    | enrich / report | report                | 可选                           |

------

### 6. Step Boundary Rules（步骤边界规则）

这些是整个 pipeline 能否长期稳定的关键：

------

#### Rule 1：下游永远不能读取上游 tmp/

tmp 属于：

- 不稳定
- 不可复跑
- 不保证 schema
- 不保证生成

下游只能读 out/ 与 configs/。

------

#### Rule 2：分析参数永远来自 configs.yaml，而不是 meta.json 或 parquet

阈值 / 策略必须是语义真源。

------

#### Rule 3：数据永远来自 parquet，而不是 tsv

tsv 仅用于人类阅读与绘图。

------

#### Rule 4：meta.json 永远不包含决定分析行为的内容

它只告诉你“东西在哪”，不告诉你“怎么分析”。

------

#### Rule 5：每个步骤都输出自己的 meta.json

形成完整的 “step-level manifest chain”。

------

#### Rule 6：parquet schema 必须稳定

任何 schema change 必须 bump version。

------

### 7. Cross-Step Loader（标准加载器设计）

下游 loader 应当这样写：

```python
def load_step(dir_out: Path):
    meta = json.loads((dir_out / "meta.json").read_text())
    cfg = yaml.safe_load((dir_out / meta["Files"]["ConfigsYaml"]).read_text())

    dfs = {}
    for key, fname in meta["Files"].items():
        if fname.endswith(".parquet"):
            dfs[key] = pl.read_parquet(dir_out / fname)

    return cfg, dfs, meta
```

------

### 8. 未来扩展（Extensibility）

本规范可自然扩展至：

- Multi-omics（RNAseq / ATAC / GWAS / PWAS）
- Cloud execution（WDL / Cromwell / NF）
- Module registry（step plugin system）
- Artifact versioning system（schema registry）

你可以用此文档作为未来版本化 pipeline 的基础。
