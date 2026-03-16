# Claude Code `/insights` 报告内容概览

> `/insights` 是 Claude Code 的内置命令，输出一份交互式 HTML 报告，分析用户过去 30 天与 Claude Code 的协作情况。以下是该报告包含的全部章节与维度。

---

## 1. Statistics Dashboard（统计仪表盘）

报告顶部的数字概览，包含以下指标：

- 总会话数（Total Sessions）
- 总消息数（Total Messages）
- 总时长（Total Duration）
- 输入 / 输出 Token 消耗量
- Git 提交数与推送数
- 活跃天数（Active Days）
- 最长连续活跃天数（Activity Streaks）
- 高峰活跃时段（Peak Activity Hours）

---

## 2. Interactive Visualizations（交互式图表）

报告内嵌六种可视化图表：

- **Daily Activity Chart** — 每日活动趋势图，支持多时区切换（PT / ET / London / CET / Tokyo / 自定义偏移）
- **What You Asked For** — Goal Categories 柱状图，展示各类任务意图的分布
- **Tools Claude Used** — 工具使用分布（Read、Edit、Bash、Write、Grep、TodoWrite 等）
- **Languages You Worked In** — 编程语言使用占比
- **Session Type Distribution** — 会话类型分布
- **Satisfaction / Outcome Tracking** — 用户满意度与任务完成度追踪

---

## 3. "At a Glance" Executive Summary（总览摘要）

四个子模块：

- **What's Working** — 当前有效的工作流和交互风格
- **What's Hindering You** — 阻碍因素，区分 Claude 侧问题与用户侧摩擦
- **Quick Wins** — 快速可落地的改进建议
- **Ambitious Workflows** — 面向未来的前瞻性工作流构想

---

## 4. "What You Work On"（项目领域卡片）

为每个检测到的项目领域生成独立卡片，每张卡片包含：会话数量、项目摘要、常见任务类型、工具分布、编程语言、会话类型构成。

---

## 5. "How You Use Claude Code"（交互风格叙述）

2–3 段叙事性文字，描述用户的交互模式与习惯（如是否倾向微管理、是否偏好并行任务等）。

---

## 6. "Impressive Things You Did"（亮点回顾）

展示从会话历史中挑选的三个突出工作流或成就。

---

## 7. "Where Things Go Wrong"（摩擦分析）

列出用户的 Top 3 摩擦类别，每个类别附带两个来自真实会话的具体示例。

---

## 8. CLAUDE.md Suggestions（CLAUDE.md 配置建议）

提供可直接复制的 `CLAUDE.md` 规则，每条附带一键 Copy 按钮和解释说明。

---

## 9. Feature Recommendations（功能推荐）

推荐尚未充分利用的 Claude Code 功能（MCP Servers、Custom Skills、Hooks、Headless Mode、Task Agents），附带示例命令。

---

## 10. "Usage Patterns to Adopt"（推荐使用模式）

根据用户实际模式定制的可复制 Prompt 模板和工作流技巧。

---

## 11. "On the Horizon"（前瞻性工作流提案）

三个自主工作流构想（如自修复循环、并行迁移、自动 QA），每个附带即用型 Prompt。

---

## 12. Memorable Moment（趣味亮点）

报告收尾，展示一个从会话记录中提取的有趣或令人印象深刻的片段。

---

## 附录：Facet 维度枚举

报告的分析基于对每个会话提取的结构化维度（Facet），各维度的完整枚举值如下：

### Goal Categories（目标分类）

`debug/investigate` | `implement feature` | `fix bug` | `write script/tool` | `refactor code` | `configure system` | `create PR/commit` | `analyze data` | `understand codebase` | `write tests` | `write docs` | `deploy/infra` | `warmup_minimal`

### Outcome（成果评定）

`fully_achieved` | `mostly_achieved` | `partially_achieved` | `not_achieved` | `unclear_from_transcript`

### User Satisfaction（用户满意度）

`happy` | `satisfied` | `likely_satisfied` | `dissatisfied` | `frustrated` | `unsure`

### Claude Helpfulness（Claude 帮助度）

`unhelpful` | `slightly_helpful` | `moderately_helpful` | `very_helpful` | `essential`

### Session Type（会话类型）

`single_task` | `multi_task` | `iterative_refinement` | `exploration` | `quick_question`

### Primary Success Factor（主要成功因素）

`none` | `fast_accurate_search` | `correct_code_edits` | `good_explanations` | `proactive_help` | `multi_file_changes` | `good_debugging`

### Friction Categories（摩擦点分类）

`misunderstood_request`（误解意图）| `wrong_approach`（方法错误）| `buggy_code`（代码有 Bug）| `user_rejected_action`（用户拒绝操作）| `claude_got_blocked`（Claude 卡住）| `user_stopped_early`（用户提前终止）| `wrong_file_or_location`（文件/位置错误）| `excessive_changes`（过度修改）| `slow_or_verbose`（慢或冗长）| `tool_failed`（工具失败）| `user_unclear`（用户表述模糊）| `external_issue`（外部问题）
