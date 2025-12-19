# 📘 Pipeline Directory Layout Specification

（工作流目录结构规范）
## V0.0
------

### 1. 设计动机（Motivation）

本规范用于定义 pipeline 各步骤（step/module）的标准输出目录结构，目标包括：

- 自描述性（Self-describing）：拿到任意 step 的输出目录即可重建上下文。
- 可复跑性（Reproducible）：配置与数据分离，不污染彼此。
- 可扩展性（Extensible）：未来 pipeline 新步骤可无缝加入此结构。
- 可审计性（Auditable）：所有行为（配置、日志、数据输出）都有明确位置。
- 跨语言兼容性（Interoperable）：Python / R / Web 前端都能一致读取。

------

### 2. 总原则（Core Principles）

1. 一个 step = 一个独立的 self-contained directory
    内含 configs / logs / out / tmp 四类固定结构。
2. configs/ 作为语义真源（Semantic Source of Truth）
    所有会影响分析行为的参数必须写在 configs.yaml，而不是其他文件。
3. out/ 作为正式产物目录（Deliverable Output）
    内含 parquet、tsv、meta.json 等可被下游或外部模块消费的文件。
4. meta.json 永远位于 out/ 根目录，用于入口索引（manifest）
5. tmp/ 为可随时清理的 scratch 空间，不可用于下游分析
6. logs/ 用于存放 step 的执行日志，便于 debug 与审计

------

### 3. Step 目录结构（General Form）

每一个 workflow step（如 intake / normalization / diff / enrichment）均遵循以下目录结构：

```text
<project_root>/tmp/<step_name>/
├── configs/
│   └── configs.yaml
│
├── logs/
│   └── <step_name>.log
│
├── out/
│   ├── meta.json
│   ├── <data files: *.parquet | *.tsv>
│   └── <other deliverables>
│
└── tmp/
    └── <scratch files, intermediate artifacts>
```

------

### 4. 各目录的职责（Directory Semantics）

------

#### 4.1 configs/

> 语义真源（Semantic Source of Truth）

存放本 step 的全部配置参数。
 必含文件：

```
configs/configs.yaml
```

职责：

- 决定本 step 的行为
- 包含平台信息、分析参数、项目元数据
- 必须可被下游读取
- 必须稳定、可审计、可复跑

不允许放：

- runtime 信息（时间戳、计数）
- 文件索引（属于 meta.json）
- 数据类输出

------

#### 4.2 logs/

> 执行日志（Execution Logs）

用于：

- 记录运行过程
- 错误溯源
- 审计
- 开发 debug

每个 step 至少有一个日志文件：

```
logs/<step>.log
```

可写为按日期/级别区分的更复杂结构，但核心是：日志必须独立于 out/。

------

#### 4.3 out/

> 对外可见产物（Deliverables）

所有下游模块、分析工具、前端展示使用的文件均放在此目录。

核心文件：

```
out/meta.json
```

其它典型文件：

```
out/info-samples.parquet
out/info-samples.tsv
out/info-contrasts.parquet
out/info-contrasts.tsv
out/qc-metrics.parquet
out/diff-results.parquet
out/plots/*.svg
```

原则：

- out 目录内容必须“干净、稳定、可供他人使用”
- 不允许放中间文件（它们属于 tmp/）
- 不允许放耗时产生的 cache（如 arrow IPC），放 tmp/

------

#### 4.4 meta.json（Manifest & Snapshot）

> 入口索引（entrypoint） + 摘要快照（summary）

位于：

```
out/meta.json
```

用于：

- 让任意模块只凭 `out/` 就能加载 step 结果
- 提供文件名索引
- 提供摘要信息（样本数、对比数、平台、项目名）

严格禁止：

- 存储分析参数（属于 YAML）
- 存储不可复原的数据
- 成为 pipeline 的语义真源

meta.json 示例（简化）：

```json
{
  "SchemaVersion": "1.0.0",
  "IntakeVersion": "0.2.1",
  "Files": {
      "ConfigsYaml": "../configs/configs.yaml",
      "InfoSamplesParquet": "info-samples.parquet",
      "InfoContrastsParquet": "info-contrasts.parquet"
  },
  "Samples": {
      "NSamples": 10
  },
  "Project": {
      "ProjectCode": "SNWD042825100601"
  }
}
```

------

#### 4.5 tmp/

> 临时文件（scratch space）

特点：

- 可随时删除
- 不允许下游读取
- 不需要 schema 或稳定命名
- 用于存放中间数据、cache、临时计算结果

例如：

```
tmp/__raw_diann_pg_matrix_intermediate.arrow
tmp/__filtered_matrix_temp.tsv
tmp/__cache_20251211_113500/
```

绝不能把 tmp 内容移入 out/
 out 是可交付、可复跑、结构稳定的，只能放最终结果。

------

### 5. Intake Step 的标准示例（完整展示）

```text
tmp/intake/
├── configs/
│   └── configs.yaml
│
├── logs/
│   └── intake.log
│
├── out/
│   ├── meta.json
│   ├── info-samples.parquet
│   ├── info-samples.tsv
│   ├── info-contrasts.parquet
│   └── info-contrasts.tsv
│
└── tmp/
    ├── __mqpar_raw.xml
    ├── __diann_intermediate.arrow
    └── __cache/
```

意义：

- configs.yaml = intake 生成的语义真源
- info-*.parquet = 下游分析的主数据源
- meta.json = 入口索引
- tsv = 利于人工审查与可视化
- tmp = 临时缓存

------

### 6. 下游如何读取 Step 输出？（标准调用范式）

推荐 loader：

```python
def load_step(dir_out: Path):
    meta = json.loads((dir_out / "meta.json").read_text())
    files = meta["Files"]

    cfg = yaml.safe_load((dir_out / files["ConfigsYaml"]).read_text())
    df_samples = pl.read_parquet(dir_out / files["InfoSamplesParquet"])
    df_contrasts = pl.read_parquet(dir_out / files["InfoContrastsParquet"])

    return cfg, df_samples, df_contrasts, meta
```

核心原则：

- 核心分析逻辑永不读 meta.json
- 核心分析逻辑永远读 configs.yaml
- 数据永远从 parquet 读，不从 tsv 读

------

### 7. Anti-Patterns（反模式）

这些做法绝对不要出现：

#### ❌ 1. 把 configs.yaml 放在 out/

configs 是参数，不是产物。

#### ❌ 2. 把 meta.json 放在 configs/

meta 是入口，不是配置。

#### ❌ 3. 把 FC/P threshold 放在 info-contrasts.parquet

那是分析参数，必须在 configs.yaml。

#### ❌ 4. tmp/ 中的临时文件作为下游输入

tmp 文件不具备长期稳定性。

#### ❌ 5. out/ 中放大量中间缓存

out 只存“干净的最终文件”。

------

### 8. Versioning（版本管理建议）

未来如果对 meta.json 或 configs.yaml 的 schema 修改，建议引入：

```
SchemaVersion: "1.1.0"
```

并保持向后兼容。

------

## 文档完毕。
