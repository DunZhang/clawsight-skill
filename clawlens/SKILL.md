---
name: clawlens
description: Analyze OpenClaw conversation history and generate a deep usage insights report covering usage stats, task classification, friction analysis, skills ecosystem, autonomous behavior audit, and multi-channel analysis.
user-invocable: true
dependencies:
  - litellm
required-env:
  - DEEPSEEK_API_KEY or OPENAI_API_KEY or ANTHROPIC_API_KEY (depending on --model)
reads:
  - ~/.openclaw/agents/{agentId}/sessions/sessions.json
  - ~/.openclaw/agents/{agentId}/sessions/*.jsonl
  - ~/.openclaw/skills/
writes:
  - ~/.openclaw/agents/{agentId}/sessions/.clawlens-cache/
external-api:
  - LLM provider specified by --model via litellm (sends conversation transcript summaries for analysis)
---

# Clawlens - OpenClaw Usage Insights

Generate a comprehensive usage insights report by analyzing conversation history.

## When to Use

| User Says | Action |
|-----------|--------|
| "show me my usage report" | Run full report |
| "analyze my conversations" | Run full report |
| "how am I using Claw" | Run full report |
| "clawlens" / "claw lens" | Run full report |
| "usage insights" / "usage analysis" | Run full report |

## How to Run

Execute the analysis script:

```bash
python3 scripts/clawlens.py [OPTIONS]
```

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--agent-id` | `main` | Agent ID to analyze |
| `--days` | `180` | Analysis time window in days |
| `--model` | **required** | LLM model in litellm format (e.g. `deepseek/deepseek-chat`). API key must be set via env var. |
| `--lang` | `zh` | Report language: `zh` or `en` |
| `--no-cache` | false | Ignore cached facet extraction results |
| `--max-sessions` | `2000` | Maximum sessions to process |
| `--concurrency` | `10` | Max parallel LLM calls |
| `--verbose` | false | Print progress to stderr |
| `-o` / `--output` | stdout | Output file path |

### Examples

```bash
# DeepSeek, 180 days, Chinese
DEEPSEEK_API_KEY=sk-xxx python3 scripts/clawlens.py --model deepseek/deepseek-chat

# OpenAI, English, last 7 days
OPENAI_API_KEY=sk-xxx python3 scripts/clawlens.py --model openai/gpt-4o --lang en --days 7

# Verbose, save to file
ANTHROPIC_API_KEY=sk-xxx python3 scripts/clawlens.py --model anthropic/claude-sonnet-4-20250514 --verbose -o /tmp/clawlens-report.md
```

## Output

The script outputs a **Markdown** report to stdout (or to the file specified by `-o`). Progress messages go to stderr when `--verbose` is set.

The report includes all dimensions: usage overview, task classification, friction analysis, skills ecosystem, autonomous behavior audit, and multi-channel analysis.

**Present the Markdown output directly to the user.** Do not summarize or truncate it.

## Model Configuration

`--model` is required. The model name and API key must follow [litellm's provider format](https://docs.litellm.ai/docs/providers):

| Provider | `--model` value | Required env var |
|----------|----------------|------------------|
| DeepSeek | `deepseek/deepseek-chat` | `DEEPSEEK_API_KEY` |
| OpenAI | `openai/gpt-4o` | `OPENAI_API_KEY` |
| Anthropic | `anthropic/claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| OpenAI-compatible | `openai/<model-id>` + set `OPENAI_API_BASE` | `OPENAI_API_KEY` |

The format is always `<provider>/<model-id>`. Refer to litellm docs for the full list of supported providers and their env var naming conventions.

## Data Source

The script reads conversation data from:
- `~/.openclaw/agents/{agentId}/sessions/sessions.json` (session index)
- `~/.openclaw/agents/{agentId}/sessions/*.jsonl` (per-session logs, including unindexed historical files)
- `~/.openclaw/skills/` (installed skills directory for ecosystem analysis)

Cache is written to `~/.openclaw/agents/{agentId}/sessions/.clawlens-cache/facets/` to avoid re-analyzing the same sessions.

## Privacy Notice

This skill sends conversation transcript data to an **external LLM provider** (specified by `--model`) for analysis. Specifically:

- **Stage 2 (Facet Extraction)**: Each session's conversation transcript (truncated to ~80K chars) is sent to the LLM to extract structured analysis (task categories, friction points, etc.). Results are cached locally so each session is only sent once.
- **Stage 4 (Report Generation)**: Aggregated statistics and session summaries (not raw transcripts) are sent to the LLM to generate the report sections.

**No API keys or credentials are read from or stored by this skill.** The user must provide the LLM API key via environment variables before running the script. This skill does not access `openclaw.json` or `auth-profiles.json`.
