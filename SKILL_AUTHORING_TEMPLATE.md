# Agent Skill 编写标准模板

基于本仓库（RHClaw / RunningHub Skill）实践提炼：一套可被 OpenClaw、DeepSeek Harness、Cursor 等 Agent 加载的 Skill，应遵循 **契约层 → 流程层 → 执行层 → 数据层** 的分层。

---

## 1. 目录结构（标准骨架）

```
your-skill/
├── SKILL.md                 # 必填：总契约 + 意图路由（保持短）
├── references/              # 可选：按场景拆开的 SOP（按需再读）
│   ├── setup.md             # 鉴权 / 环境配置
│   ├── <feature-a>.md       # 某类任务的完整交互流程
│   ├── <feature-b>.md
│   └── output-delivery.md   # 结果交付与错误处理
├── scripts/                 # 可选：唯一合法执行器
│   ├── <client>.py          # 主业务 CLI
│   ├── <client_extra>.py    # 旁路能力（如应用/工作流）
│   └── build_*.py           # 仅开发构建，运行时 Agent 不调用
└── data/                    # 可选：结构化目录 / schema
    └── capabilities.json    # 由脚本生成，勿手改长期维护
```

**原则**

| 层 | 放什么 | 不放什么 |
|----|--------|----------|
| `SKILL.md` | 人设、硬规则、路由、脚本入口 | 长菜单、长 SOP、全量参数 |
| `references/` | 完整流程、固定话术、映射表 | 再嵌套 references |
| `scripts/` | 调 API / 本地副作用 | 给用户看的产品文案 |
| `data/` | 机器可读目录与 schema | 注释、手写菜单 |

---

## 2. `SKILL.md` 模板

```markdown
---
name: your-skill-name          # 小写、数字、连字符；≤64 字符
description: >-
  第三人称：做什么 + 覆盖范围 + 何时触发。
  含关键词便于 Agent 发现。≤1024 字符。
homepage: https://example.com  # 可选
metadata:
  {
    "openclaw":
      {
        "emoji": "🎬",
        "requires": { "bins": ["python3", "curl"] },
        "primaryEnv": "YOUR_API_KEY"
      }
  }
---

# Your Skill Name

Scripts: `python3 {baseDir}/scripts/xxx.py`
Data: `{baseDir}/data/xxx.json`

## Persona
- 对用户：语言、语气、是否报成本、是否隐藏内部 ID
- 交付后是否建议下一步

## CRITICAL RULES
1. **ALWAYS** …（必须做）
2. **NEVER** …（禁止做）
3. 长任务先通知再执行
4. 某类任务 → Read `{baseDir}/references/....md` 并 WAIT 用户确认
5. 结果交付约定（工具名 / 标记行）

## Setup
需要配置时 → Read `{baseDir}/references/setup.md`
快速检查: `python3 {baseDir}/scripts/xxx.py --check`

## Routing Table

| Intent | Action | Notes |
|--------|--------|-------|
| 意图 A | Read `references/a.md` 或 `endpoint/...` | 先菜单 / 可直调 |
| 意图 B | `scripts` 命令要点 | |

## Script Usage
慢任务：先 message 通知 → 再 exec
快任务：可直接 exec

\`\`\`bash
python3 {baseDir}/scripts/xxx.py \
  --endpoint ENDPOINT \
  --prompt "..." \
  -o /tmp/openclaw/<skill>-output/name_$(date +%s).ext
\`\`\`

## Output
细节 → Read `{baseDir}/references/output-delivery.md`
关键约定写在这里（如 OUTPUT_FILE / COST / NO_REPLY）
```

### `SKILL.md` 编写规范

1. Frontmatter 必有 `name`、`description`（description 写 WHAT + WHEN）。
2. 路径一律 `{baseDir}`，禁止写死本机绝对路径。
3. 硬规则用 `ALWAYS` / `NEVER` / `MUST`，编号列出。
4. 路由表：意图 → 端点或 reference；需交互的标 ⚠️。
5. 对用户话术（中文口语）与对 Agent 指令（英文技术约束）分开。
6. 主文件宜短；长流程一律外链 `references/`（一层深）。

---

## 3. `references/` 模板

每个文件 **一篇一事**。推荐骨架：

```markdown
# <场景名>

**Whenever** …（触发条件）… you MUST … and WAIT:

> 给用户看的固定话术 / 菜单（须原文复制，禁止改写）

**STRICT RULES**
1. Copy-paste 菜单 EXACTLY
2. Do NOT 从 data 自造选项
3. Do NOT 向用户暴露内部 ID / URL（若业务要求）

## 映射表（仅 Agent）
| # | Endpoint / 动作 |
|---|-----------------|
| 1 (default) | `...` |

## Matching Rules
- 数字 / 别名 / 「默认」「最快」→ 选项
- 何时可 Skip 菜单

## After Chosen
- 确认话术
- Smart defaults（分辨率、时长等）
- 特例与回退

## Prompt / 参数优化（如有）
- 用户短句如何增强；用何种语言调 API

## Errors & Retry
| Error | Action / 对用户话术 |
|-------|---------------------|
```

### `references/` 编写规范

1. 只从 `SKILL.md` 直接引用，不互相深链。
2. 用户可见文案放引用块 `>`；技术映射放表格。
3. 关键菜单写 BAD / GOOD 正反例。
4. 默认值、回退、失败重试写死，减少 Agent 自由发挥。
5. 命令示例可复制，路径用 `{baseDir}`。

---

## 4. `scripts/` 模板

```text
职责：Skill 的唯一合法执行器（Agent 禁止直接 curl/HTTP）
依赖：优先 stdlib + 系统已有工具（如 curl）
形态：argparse CLI，互斥主模式
```

### 推荐 CLI 模式

| 模式 | 用途 |
|------|------|
| `--check` | 鉴权 / 余额 / 环境健康 |
| `--list` | 浏览能力或资源 |
| `--info ID` | 单条详情 / 参数 schema |
| 执行标志 | `--endpoint` / `--run` / `--task` + `-o` |

### stdout 契约（给 Agent 解析）

```text
OUTPUT_FILE:/absolute/path     # 媒体成功
COST:¥x.xx                     # 可选费用
DURATION:Ns                    # 可选耗时
{"error":"CODE","message":"..."}  # 失败用稳定错误码
```

- 进度日志 → **stderr**
- 结果与错误码 → **stdout**

### Key 解析顺序（若需要）

`--api-key` → 环境变量（与 `primaryEnv` 一致） → 本地配置文件

### 脚本分工建议

| 文件 | 用途 |
|------|------|
| 主 client | 标准 API / 主路径 |
| 旁路 client | 应用、工作流等另一套 API |
| `build_*.py` | 仅生成 `data/`，不进对话执行路径 |

旁路脚本可 import 主脚本的 Key、轮询、下载等公共逻辑。

---

## 5. `data/` 模板

```json
{
  "version": "YYYY-MM-DD",
  "total": 0,
  "endpoints": [
    {
      "endpoint": "vendor/model/action",
      "name_cn": "",
      "name_en": "",
      "task": "text-to-image",
      "output_type": "image",
      "category": "",
      "popularity": 99,
      "tags": [],
      "params": [
        {
          "key": "prompt",
          "type": "STRING",
          "required": true,
          "default": null,
          "options": [],
          "maxLength": 2000
        }
      ]
    }
  ]
}
```

### `data/` 规范

1. 由 `build_*.py` 生成；不要手改后当长期源。
2. 标准 JSON，无注释；敏感 default 在构建时清洗。
3. 供脚本拼请求 / `--list` / `--info` / `--task` 选优。
4. **精选用户菜单不写在 data**；写在 `references/`，并在 `SKILL.md` 禁止从 data 自造菜单。

---

## 6. 分层协作（一句话）

```
用户自然语言
  → SKILL.md（人设 + 硬规则 + 路由）
  → references/*.md（完整 SOP / 固定菜单）
  → scripts/*.py（唯一执行）
  → data/*.json（参数与目录）
  → stdout 契约 → Agent 交付给用户
```

---

## 7. 编写检查清单

- [ ] `SKILL.md` 有清晰 description（WHAT + WHEN）
- [ ] 硬规则可执行，无「尽量」「也许」
- [ ] 长流程已拆到 `references/`，且一层深
- [ ] 用户菜单固定原文；内部 ID 有隐藏策略
- [ ] 脚本是唯一 API 入口；stdout 约定稳定
- [ ] Key / 输出目录 / 慢任务通知有统一约定
- [ ] `data` 可重建；精选能力不依赖全量目录展示
- [ ] 错误码 → 对用户话术表齐全

---

## 8. 最小可行 Skill（无脚本版）

若只需知识/流程、不调外部 API：

```
your-skill/
└── SKILL.md          # 规则 + 步骤 + 示例即可
```

需要鉴权交互、多步菜单、或外部副作用时，再按完整四层扩展。
