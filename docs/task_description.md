# Clawlens Skill 开发任务描述

## 项目概述

Clawlens 是一个 OpenClaw Skill，读取 OpenClaw 的历史聊天数据，生成一份多维度的 Markdown 使用洞察报告。灵感来自 Claude Code 的 `/insights` 命令，但适配 OpenClaw"全能型个人 AI 助手"的定位（而非编程工具），输出纯 Markdown 而非 HTML。

## 当前进度

Skill 已完成初版开发，核心功能可用。文件结构：

```
clawlens/
├── _meta.json                    # Skill 元数据 (slug, version)
├── SKILL.md                      # Skill 核心定义 (触发条件、CLI 参数、使用说明)
├── references/
│   └── report-format.md          # 报告格式参考 (章节结构、分类标签定义)
└── scripts/
    └── clawlens.py              # 主脚本 (~1375 行)
```

---

## 重要参考文档

| 文档 | 路径 | 说明 |
|------|------|------|
| Claude Insights 功能介绍 | `docs/Claude_Code_Insights_Main_Contents.md` | 借鉴的原型，了解报告维度设计 |
| Claude Insights 实现深度解析 | `docs/claude-code-insights-deep-dive.html` | 4 阶段流水线架构、prompt 模板设计 |
| Clawlens PRD | `docs/openclaw-clawlens-prd.md` | 产品需求，定义了 7 个报告维度 |
| OpenClaw 对话数据格式 | `docs/openclaw-对话数据速查.md` | JSONL 数据结构、sessions.json 索引格式 |
| Skill 开发教程 | https://support.claude.com/en/articles/12512198-how-to-create-custom-skills | 文件夹结构、SKILL.md frontmatter 规范 |
| 示例 Skill | `examples/ontology-1.0.4/` | 一个真实的 Skill 参考实现 |

---

## 技术要求

### 基本约束

1. **语言**：Python
2. **LLM 调用**：使用 `litellm` 库，不直接调用任何厂商 SDK
3. **Skill 规范**：严格按照 OpenClaw Skill 开发要求组织文件夹（`SKILL.md` + `_meta.json` + `scripts/` + `references/`）
4. **输出格式**：Markdown 纯文本，用户在任意聊天渠道内直接阅读

### 报告维度（全部实现，不区分优先级）

1. **使用概览** — 基础统计（会话数、消息数、活跃天数、Token 消耗、活跃时间分布）
2. **任务分类** — 按意图归类（邮件管理/日程/信息检索/内容创作/编码辅助/自动化/智能家居/文件操作/通信/规划/个人助理），含占比、趋势、高价值任务标记
3. **摩擦分析** — 区分 Claw 侧（误解指令/执行错误/未遵循偏好/编造信息/过度操作）与用户侧（上下文不足/指令模糊/渠道不当/矛盾请求），附具体示例
4. **Skills 生态分析** — 已安装 vs 实际使用、高频排名、未使用清理建议、社区推荐
5. **自主行为审计** — 自主行为次数/类别、用户确认率 vs 否决率、异常标记、准确度趋势
6. **多渠道分析** — 各渠道使用量/任务分布、渠道错配检测、优化建议

### 模型选择策略

- **默认**：自动从 OpenClaw 配置中读取模型和 API key，无需用户指定任何参数
  - 读取 `~/.openclaw/openclaw.json` 中 `agents.defaults.model.primary`（如 `kimi-code/kimi-for-coding`）
  - 解析对应 provider 的 `baseUrl` 和 `api` 类型（`openai-completions` / `anthropic-messages`）
  - 从 `~/.openclaw/agents/{agentId}/agent/auth-profiles.json` 读取 API key
  - 转换为 litellm 格式：`openai/<model-id>` + `api_base` + `api_key`
- **手动指定**：`--model` 参数覆盖，模型名和环境变量必须遵循 litellm 的 provider 格式（如 `deepseek/deepseek-chat` + `DEEPSEEK_API_KEY`），详见 [litellm 文档](https://docs.litellm.ai/docs/providers)

### 数据读取

- **数据目录**：`~/.openclaw/agents/{agentId}/sessions/`
- **索引文件**：`sessions.json` 仅包含当前活跃 session，不包含全部历史
- **实际做法**：直接扫描目录下所有 `*.jsonl` 文件，按文件修改时间过滤时间窗口，`sessions.json` 作为补充元数据（channel、model 等字段）
- **Skills 目录**：`~/.openclaw/skills/` 扫描已安装 Skills

### 默认参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--days` | `180` | 默认看最近半年 |
| `--max-sessions` | `2000` | 上限 2000 个 session |
| `--concurrency` | `10` | 并发 LLM 调用数 |
| `--lang` | `zh` | 支持 `zh` / `en` |
| `--model` | 自动检测 | 从 OpenClaw 配置读取 |

---

## 架构设计（4 阶段流水线）

### Stage 1: 数据收集（本地）

- 扫描 `*.jsonl` 文件，按 mtime 过滤时间窗口
- 解析每个 JSONL，提取结构化 `SessionMeta`：消息数、工具调用统计、时间戳、首条用户消息、费用等
- 过滤规则：`user_message_count < 2` 或 `duration < 1min` 的 session 跳过
- 同时构建会话 transcript 文本（截断到 80K 字符），供 Stage 2 使用

### Stage 2: LLM Facet 提取（per-session，带缓存）

- 对每个 session 调用 LLM，提取结构化 JSON facets：任务分类、outcome、satisfaction、friction、skills、自主行为判断
- **缓存**：结果存到 `~/.openclaw/agents/{agentId}/sessions/.clawlens-cache/facets/{sessionId}.json`，重复运行直接跳过
- 长会话（>100K 字符）先分块摘要再提取
- asyncio 并发，最多处理 50 个未缓存 session
- **自主行为检测**：由于 session 元数据无明确标记，通过 LLM 分析对话内容判断（特征：首条消息来自 assistant、提到定时/自动检查等）

### Stage 3: 聚合统计（本地）

- 合并所有 `SessionMeta` + `SessionFacets`
- 计算：总量统计、时间分布（小时/星期）、任务分类占比、摩擦归因（Claw vs 用户）、Skills 生态（已安装 vs 使用）、渠道分布、自主行为统计
- LLM 返回值做类型校验（count 字段可能返回 dict 而非 int，需防御性处理）

### Stage 4: 报告生成（并行 LLM 调用）

- 6 个维度 + 1 个综合摘要，每个维度独立 prompt，asyncio 并发调用
- 每个 prompt 指定语言、输出格式（Markdown）、聚合数据 JSON
- 综合摘要（at_a_glance）依赖前 6 个维度的输出，最后生成
- 最终拼接为完整 Markdown 报告

---

## 关键实现细节

### 已解决的问题

1. **`sessions.json` 只索引活跃 session**：改为直接扫描目录 `*.jsonl` 文件，sessions.json 仅做元数据补充
2. **`sessionFile` 字段含绝对路径**：可能来自 Docker 环境（如 `/root/.openclaw/...`），取 `Path(sessionFile).name` 只用文件名
3. **`updatedAt` 时间戳单位不一致**：有的是毫秒（>1e12），有的是秒，做了归一化处理
4. **LLM 返回值类型不稳定**：`friction_counts` 的 value 有时是 dict 而非 int，加了类型校验
5. **文件权限问题**：某些 JSONL 可能因权限不可访问，加了 try/except 跳过
6. **未索引 session 缺少 token 统计**：从 JSONL 中 assistant message 的 `usage` 字段累加

### 依赖

- `litellm` — LLM 调用（唯一的第三方依赖）
- Python 标准库：`json`, `pathlib`, `asyncio`, `argparse`, `dataclasses`, `datetime`, `collections`

---

## 验证方式

```bash
# 1. 最简运行（自动检测模型）
python3 clawlens/scripts/clawlens.py --verbose

# 2. 指定模型运行
DEEPSEEK_API_KEY=sk-xxx python3 clawlens/scripts/clawlens.py --model deepseek/deepseek-chat --verbose

# 3. 保存报告到文件
python3 clawlens/scripts/clawlens.py -o /tmp/clawlens-report.md

# 4. 检查缓存是否生效（第二次运行 Stage 2 应全部 Cached）
python3 clawlens/scripts/clawlens.py --verbose 2>&1 | grep "Cached"

# 5. 清除缓存重新提取
python3 clawlens/scripts/clawlens.py --no-cache --verbose
```

验证要点：
- 4 个阶段日志正常输出
- 报告 Markdown 格式正确，6 个维度 + 概览摘要齐全
- 缓存文件生成在 `.clawlens-cache/facets/` 目录
- `--lang en` 输出英文报告
