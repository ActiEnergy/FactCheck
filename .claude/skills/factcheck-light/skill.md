---
name: factcheck-light
description: "AI 调研报告可信度评估工具。触发词：/factcheck-light、调研、事实核查、信息核实、可信度评估、交叉验证、深度调研、详细调研、全面调研。"
---

# FactCheck Light

> **AI 帮你做调研，同时告诉你哪些可信、哪些要核实。**

大多数调研工具只告诉你"是什么"，FactCheck Light 额外告诉你"有多可信"。

---

## What can it do?

| 功能 | 说明 |
|------|------|
| 信息收集 | 搜索调研主题的相关信息，标注来源 |
| 双盲评估 | 正方支持 vs 反方质疑，找出潜在问题 |
| 可信度评估 | 每条信息标注可信度（★/◆/✗）并说明原因，高/中/低可信度文字染色支持 |
| 报告生成 | 输出带可信度标记的 HTML 报告（高/中/低可信度文字染色） |

## 核心 Agent

| Agent | 说明 |
|-------|------|
| `factcheck-collector` | 调研专家，生成含结论的完整调研报告（含角标） |
| `factcheck-pos-evaluator` | 正方评估专家，验证+补充+矛盾记录 |
| `factcheck-neg-evaluator` | 反方评估专家，质疑+支持性证据 |
| `factcheck-judge` | 裁判，分析评估→叠加可信度标注→生成信息核查说明 |
| `factcheck-report` | 报告生成专家，根据模板生成带可信度标记的 HTML 报告 |

## Agent 定义文件

本 skill 依赖的 agent 定义位于项目根目录的 `.claude/agents/` 目录：

| Agent | 定义文件 |
|-------|---------|
| `factcheck-collector` | `.claude/agents/collector.md` |
| `factcheck-pos-evaluator` | `.claude/agents/pos-evaluator.md` |
| `factcheck-neg-evaluator` | `.claude/agents/neg-evaluator.md` |
| `factcheck-judge` | `.claude/agents/judge.md` |
| `factcheck-report` | `.claude/agents/report.md` |

各 agent 通过 `@agent-{name}` 格式调用。

## 数据流与文件规范

```
workspace/
├── collector-{主题}-{时间戳}.md    ← Collector 输出
├── pos-{主题}-{时间戳}.md          ← PosEvaluator 输出
├── neg-{主题}-{时间戳}.md          ← NegEvaluator 输出
└── judge-{主题}-{时间戳}.md        ← Judge 输出
                                        │
                                        ▼
项目根目录 report-{主题}-{时间戳}.html   ← Report 输出（最终报告）
```

角标格式：【1】【2】【3】为主要来源（Collector），【P1】【P2】为支持性证据（PosEvaluator），【N1】【N2】为质疑性证据（NegEvaluator）

时间戳格式：`YYYYMMDD-HHMMSS`，例如 `20260510-143052`

## 工作流程图

```
用户输入主题
     │
     ▼
┌─────────────────────────┐
│   factcheck-collector   │
│   生成完整调研报告      │
│   （含结论+角标）      │
└───────────┬─────────────┘
            │
            ▼
    ┌───────┴───────┐
    ▼               ▼
┌──────────┐  ┌──────────┐
│PosEvaluator│  │NegEvaluator│  ← 并行执行
│  正方验证  │  │  反方质疑  │
└─────┬────┘  └─────┬────┘
      └──────┬──────┘
             ▼
┌─────────────────────────┐
│    factcheck-judge      │
│  裁判分析+可信度标注    │
│  生成信息核查说明      │
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│   factcheck-report      │
│   基于模板生成HTML报告  │
└─────────────────────────┘
```

## 触发方式

在 Claude Code 对话中输入 `/factcheck-light "调研xxx"` 即可触发本 Skill。

## 执行流程

### Step 1: 启动调研

```
@agent-factcheck-collector 调研"{主题}"
```

**输出文件：** `workspace/collector-{主题}-{时间戳}.md`
- Collector 完成后必须使用 Write 工具将结果写入文件
- 时间戳格式：`YYYYMMDD-HHMMSS`，例如 `20260510-143052`
- 文件保存路径：`workspace/collector-华为鸿蒙next现状-20260510-143052.md`

### Step 2: 双盲评估（并行）

正方和反方评估同时执行：

```
@agent-factcheck-pos-evaluator 基于Collector的结果进行正方评估
@agent-factcheck-neg-evaluator 基于Collector的结果进行反方评估
```

**输入文件：** `workspace/collector-{主题}-{时间戳}.md`

**输出文件：**
- `workspace/pos-{主题}-{时间戳}.md`
- `workspace/neg-{主题}-{时间戳}.md`

**重要：正方/反方评估必须为每条补充的证据提供完整URL**，格式为：
- 【P1】https://完整URL | 来源标题 | 说明
- 【N1】https://完整URL | 来源标题 | 说明

不能只提供来源名称，必须附上可点击的实际链接。

### Step 3: 综合评估

```
@agent-factcheck-judge 综合正反方评估的信息(不需要再进行联网搜索)，为collector的调研结果叠加可信度标注，并在后方生成信息核查说明。
```

**信息核查说明格式：每条信息以主数据源编号【1】【2】为主标题**

```markdown
### ★ 高可信度信息
**【1】信息描述**
- 可信原因：...
- 支持证据：【P1】【P2】(若有)

### ◆ 中可信度信息
**【2】信息描述**
- 不确定原因：...
- 质疑证据：【N1】(若有)

### ✗ 低可信度信息
**【3】信息描述**
- 问题原因：...
- 证据：【N1】(若有)
- 核实建议：...
```

**关键原则：**
- 以主数据源编号为主标题，P/N 作为该信息的支持/质疑证据
- 每条信息核查说明必须以主数据源编号【1】【2】开头，**禁止以【P】【N】开头**
- 缺少主数据源的信息不要单独列为一条，可归入相关主数据源下作为证据

**正确示例：**
```markdown
**【1】整机性能提升30%，功耗降低20%**
- 可信原因：官方发布，多方确认
- 支持证据：【P39】【P40】【P41】
```

**错误示例（禁止）：**
```markdown
**【P19】OpenAI恢复GPT-4o模型选择**  ← 不能以【P】开头
**【N5】用户体验存在反差**            ← 不能以【N】开头
```

**正确做法：**
- 如果【P19】没有对应的主数据源，将其归入相关的主数据源下
- 例如：`【14】【P19】微信鸿蒙版安装量1770万（2025年9月）` - 这条信息以【14】为主标题，【P19】作为支持证据

**输入文件：**
- `workspace/collector-{主题}-{时间戳}.md`
- `workspace/pos-{主题}-{时间戳}.md`
- `workspace/neg-{主题}-{时间戳}.md`

**输出文件：** `workspace/judge-{主题}-{时间戳}.md`
- Judge 完成后必须使用 Write 工具将结果写入文件

### Step 4: 生成报告

```
@agent-factcheck-report 基于Judge的报告生成HTML
```

**输入文件：** `workspace/judge-{主题}-{时间戳}.md`

**输出文件：** 项目根目录 `report-{主题}-{时间戳}.html`
- 必须使用 `templates/factcheck-report-template.html` 模板
- 不得修改 CSS 样式、类名、HTML 结构

**重要格式要求（信息核查说明部分）：**

1. **每条信息必须单独一个 `<li>` 元素**，不得合并多条信息
2. 参考 Judge 输出格式示例：
```markdown
**【1】整机性能提升30%，功耗降低20%**
- 可信原因：...
- 支持证据：【P39】【P40】【P41】
```
3. HTML 输出应对应：
```html
<li>
    <div class="badge-wrap">
        <span class="badge badge-high">高可信度</span>
        <span class="badge-src">【1】</span>
    </div>
    <div class="evidence-text">
        <strong>整机性能提升30%，功耗降低20%</strong><br>
        可信原因：...
        支持证据：【P39】【P40】【P41】
    </div>
</li>
```

**其他要求：**
- 角标使用 `<sup>` 上标，位置在标点符号之前
- 角标链接从 Judge 生成的 Markdown 参考资料区获取，不得使用 `#ref-N` 等内部链接
- 高/中/低可信度文字用 `span.evidence-high/medium/low` 包裹

## 设计原则

- **联网统一**：使用 `mcp__MiniMax__web_search`，如不可用则使用 `tavily-remote-mcp`，再不可用则使用claude code自带的Web Search。
- **本地运行**：纯本地运行，不上传用户数据
- **完整报告**：Collector 直接输出含结论的完整报告，不是半成品
- **叙事优先**：报告以连贯叙述为主，不是碎片化条目罗列
- **来源可追溯**：角标链接外部数据源，点击可跳转
- **来源充分性**：
  - 主要来源（【1】【2】【3】等）8~15 个
  - 核心信息至少各有1个可靠来源
  - 如搜索结果不足，标注"信息不足，待核实"即可，不强制补充

## 可信度标注说明

| 标注 | 含义 |
|------|------|
| **★** | 高可信度，信息可靠 |
| **◆** | 信息存在不确定性，参考时需注意 |
| **✗** | 信息存在显著问题，建议核实 |

## 输出格式示例

### Collector 输出（待评估的调研报告）

```markdown
# {主题}

## 执行摘要
（核心结论，3-5句话）

---

## [主题模块A]
（连贯叙述）

关键陈述【1】。另一关键陈述【2】。

---

## [主题模块B]
...

---

## 结论
（综合性论述，包括研究发现、启示、局限性说明）

---

## 参考资料
【1】https://xxx | 来源标题
【2】https://yyy | 来源标题
```

### Judge 输出（可信度标注后的报告）

```markdown
# {主题}

## 执行摘要
（核心结论）

---

## [主题模块A]
（连贯叙述，★/◆/✗标注已叠加）

关键陈述【1】★。参数规模约3-5万亿【2】✗。关键数据<span class="evidence-high">3000人</span>【1】【P2】【N3】★。

---

## 信息核查说明

### ★ 高可信度信息
**【1】【P2】关键陈述**
- 可信原因：官方来源/多方确认/逻辑自洽

### ◆ 中可信度信息
**【编号】信息描述**
- 不确定原因：...
- 核实建议：...

### ✗ 低可信度信息
**【编号】信息描述**
- 问题原因：...
- 核实建议：...

---

## 参考资料（必须完整列出所有 URL，不得省略或简化）

【重要】参考资料必须逐条列出，遗漏任何一条都是不合格输出。

### 主要来源（Collector 提供）- 共 X 条
【1】https://xxx | 来源标题
【2】https://yyy | 来源标题
...（必须与Collector输出的来源数量一致）

### 支持性证据（PosEvaluator 提供）- 共 X 条
【P1】https://ppp | 来源标题
【P2】https://qqq | 来源标题
...（必须与PosEvaluator输出的来源数量一致）

### 质疑性证据（NegEvaluator 提供）- 共 X 条
【N1】https://nnn | 来源标题
【N2】https://mmm | 来源标题
...（必须与NegEvaluator输出的来源数量一致）

【约束】不得使用"XX官方发布"、"XX综合报道"等描述替代具体URL，每个角标编号都必须有对应的完整URL。
```

## Report Agent 输出结构说明

最终由 Report Agent 生成的 HTML 报告按以下顺序组织：

1. **可信度图例** - 说明高/中/低可信度的含义
2. **执行摘要** - 核心结论，3-5句话
3. **主题模块** - 连贯叙述，可信度标注已叠加，角标可点击
4. **核心结论** - 根据调研内容总结生成
5. **信息核查说明** - 高可信/中可信/低可信 分类，逐条说明原因
6. **使用建议**
7. **参考资料** - 角标可点击，跳转外部数据源

## 模板文件

模板位于：`.claude/skills/factcheck-light/templates/factcheck-report-template.html`
- 模板包含的占位符说明详见 `.claude/agents/report.md`