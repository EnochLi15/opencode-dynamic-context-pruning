# OpenCode DCP 插件项目分析与对比测试方案（中文）

## 1) 项目特性、用法与原理

### 1.1 核心能力

- 自动上下文瘦身：在不改写历史会话的前提下，通过“压缩摘要 + 占位符替换”的方式减少送入模型的上下文 token。
- 双压缩模式：
  - `range`：按连续消息区间压缩，支持嵌套摘要保真。
  - `message`（实验）：按单条消息外科手术式压缩。
- 自动清理策略：
  - Deduplication：同工具同参数的重复调用只保留最近一次输出。
  - Purge Errors：错误工具调用的输入在达到 turn 阈值后被裁剪，保留错误信息。
- 安全保护：支持 protected tools、protected file patterns、可选保护用户消息。
- 可观测性：
  - `/dcp context` 查看当前会话 token 构成及“若无 DCP”的估算对比。
  - `/dcp stats` 查看会话/全局累计节省。
  - 配套脚本可离线分析 session token 与 DCP 事件影响。

### 1.2 使用方式（最短路径）

1. 在 `opencode.jsonc` 中启用插件：

```jsonc
{
  "plugin": ["@tarquinen/opencode-dcp@latest"]
}
```

2. 重启 OpenCode。
3. 常用命令：
   - `/dcp context`
   - `/dcp stats`
   - `/dcp compress [focus]`
   - `/dcp decompress <n>` / `/dcp recompress <n>`

### 1.3 机制原理（按数据流）

1. 插件初始化时注册系统提示词注入、消息变换、命令钩子与 `compress` 工具。
2. 每轮消息变换中执行：
   - 同步会话状态、消息 ID 与工具缓存。
   - 根据 state.prune 应用裁剪（工具输出/错误输入/压缩区间）。
   - 注入压缩提醒（nudge）与辅助元信息。
3. `compress` 工具执行时：
   - 解析并校验压缩范围。
   - 拼装摘要（含被保护内容）。
   - 写入压缩状态（block/run/message 映射、token 统计）。
   - 在后续上下文中用 synthetic user summary 注入替代原区间。
4. 统计口径：
   - 优先使用 API 回传 token 字段。
   - 对 user/tool 文本做 tokenizer 估算。
   - assistant 类别按“总量 - system - user - tools”残差计算。

---

## 2) “有插件 vs 无插件”测试报告方案

## 2.1 测试目标

量化 DCP 在真实开发会话中的收益与副作用：

- 主要收益：
  - 输入上下文 token 降幅（input + cache.read 维度）。
  - 工具相关 token 降幅（重复调用、错误输入、历史工具输出）。
  - 长会话稳定性（是否降低上下文超限风险）。
- 主要代价：
  - cache hit rate 变化。
  - 可能的质量风险（关键信息丢失、任务回归）。

## 2.2 实验设计

### A. 分组

- Control（对照组）：关闭 DCP（`enabled=false`）。
- Treatment（实验组）：开启 DCP（推荐默认配置起步）。

### B. 任务集（建议至少 20 个任务）

覆盖 4 类高上下文负载场景：

1. 大量文件检索与重复读取（read/list/search 密集）
2. 反复工具试错（会产生 error tool calls）
3. 多阶段重构（长链路、多轮确认）
4. 子任务并行（若启用 subagents，可单独成组）

每个任务应有：

- 固定输入 prompt 模板
- 固定验收标准（测试通过、变更正确性、关键输出完整）
- 统一上限（例如最多 25 轮或 45 分钟）

### C. 控制变量

- 同一模型/同一 provider
- 同一 OpenCode 版本
- 同一仓库快照（commit hash 固定）
- 同一任务执行脚本与验收脚本
- 同一时间窗口运行（减少外部波动）

### D. 随机化与配对

- 采用“配对交叉设计”：
  - 任务 i 先跑 Control 再跑 Treatment
  - 任务 i+1 先跑 Treatment 再跑 Control
- 目的：抵消顺序与环境漂移偏差。

## 2.3 指标体系

### 1) 成本/上下文指标（主指标）

- `total_input_context = input + cache.read`
- `input tokens` 总量
- `cache.read` / `cache.write`
- `cache_hit_rate = cache.read / (input + cache.read)`
- 每步平均上下文大小（按 step-finish）

### 2) DCP 专属节省指标

- `prunedTokens`（会话内累计）
- `prunedToolCount`
- `prunedMessageCount`
- compress 触发次数、平均每次压缩消息数

### 3) 质量与效率指标（护栏）

- 任务成功率
- 首次通过耗时（或步数）
- 回归测试通过率
- 人工评审分（可选）

## 2.4 数据采集实现

### 在线采集（会话内）

- 每个任务结束执行：
  - `/dcp context`
  - `/dcp stats`
- 保存终端输出或消息导出。

### 离线采集（脚本）

- `scripts/opencode-token-stats`：采样 session token 全量统计。
- `scripts/opencode-dcp-stats`：抓取 DCP 事件前后的 context/cache 变化。

推荐命令：

```bash
# 1) 对照/实验分批采样
scripts/opencode-token-stats --sessions 100 --json > token_stats.json
scripts/opencode-dcp-stats --sessions 100 --min-messages 5 --json > dcp_stats.json

# 2) 指定 session 回放（定位异常样本）
scripts/opencode-token-stats --session <SESSION_ID> --json
scripts/opencode-dcp-stats --session <SESSION_ID> --json
```

## 2.5 统计分析方法

- 主分析：配对样本差值（Treatment - Control）
  - 中位数差值（抗异常值）
  - 均值差值 + 95% CI（bootstrap）
- 显著性：
  - 非正态优先 Wilcoxon signed-rank
- 报告口径建议：
  - Token 节省率 = `1 - Treatment_input_context / Control_input_context`
  - 质量不下降判据：成功率差值 > -2%（阈值可调）

## 2.6 报告模板（你可直接套用）

1. 实验配置（模型、版本、任务数、日期）
2. 主结果（表 + 图）
   - 上下文 token 降幅
   - cache 命中率变化
3. 质量护栏
   - 成功率/回归通过率
4. 案例分析
   - 节省最高 3 个样本
   - 退化最明显 3 个样本
5. 结论与上线建议
   - 默认配置建议
   - 何时用 `range` / 何时尝试 `message`

## 2.7 风险与改进建议

- 风险：激进压缩导致关键上下文丢失。
  - 对策：启用 `compress.protectedTools` 与 `protectedFilePatterns`。
- 风险：cache hit rate 下降。
  - 对策：按任务类型调高 `maxContextLimit` 与 `nudgeFrequency`，减少过早压缩。
- 风险：复制粘贴大文本被保留过久。
  - 对策：谨慎开启 `protectUserMessages`。

