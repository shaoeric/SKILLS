---
name: auto-opti-skill
description: "Auto-Opti Skill: 数据驱动的全自动 prompt/skill 优化系统。通过 LLM-as-Judge 评估 Agent 输出与期望输出的匹配度，对 skill 进行两阶段优化——Phase 1 交互式诊断（人在回路）+ Phase 2 全自动爬山优化。内置反过拟合机制确保泛化性。Use when user asks \"自动优化skill\", \"auto optimize\", \"skill调优\", \"prompt优化\", \"优化提示词\", \"提高准确率\", \"优化agent\", \"skill calibration\", \"auto-opti\"."
---

# Auto-Opti Skill

> 数据驱动的全自动 prompt/skill 优化系统。
> 核心理念：**train 用于发现问题 → 修改 skill → val 判断是否真的变好 → test 最终验证泛化性**
> 绝对禁止：将数据样本写进 skill（过拟合），修改黑盒 Agent 代码

---

## 设计哲学

1. **数据驱动** — 用 train/val/test 数据而非主观感受衡量 skill 质量
2. **LLM-as-Judge** — 自适应评估 answer vs expected，5 档评分
3. **优化与验证分离** — train 用于分析问题，val 用于判断优化是否有效，test 用于最终验证
4. **人在回路仅在第一轮** — Phase 1 交互诊断后，Phase 2 全自动
5. **反过拟合是核心约束** — FORBIDDEN 改动类型 + 趋势检测
6. **黑盒不可改** — Agent 应用只启动不修改，问题告知用户
7. **棘轮机制** — 只在 val 分数提升时保留改动，否则回滚

---

## 触发条件

### 触发关键词

**中文**：自动优化skill, skill调优, prompt优化, 优化提示词, 提高准确率, 优化agent, 自动调优, 数据驱动优化

**英文**：auto optimize skill, auto-opti, skill calibration, prompt optimization, improve accuracy, agent tuning

### 使用方式

```
# 全量优化（默认路径）
auto-optimize <target_skill_path> --agent <agent_py_path>

# 指定数据路径
auto-optimize <target_skill_path> --agent <agent_py_path> \
    --train ./data/train.json --val ./data/val.json --test ./data/test.json

# 仅评估不修改
auto-optimize <target_skill_path> --agent <agent_py_path> --eval-only
```

---

## 运行架构

```
┌─────────────────────────────────────────────────────────┐
│  Claude Code (auto-opti-skill) — 优化器                  │
│                                                         │
│  1. 读取当前 SKILL.md                                    │
│  2. 启动黑盒 Agent:                                      │
│     python agent.py --data-path train.json              │
│         --skill /path/to/skill-dir                      │
│         --output-dir output --turn N                    │
│  3. 读取输出文件 output/train/{turn}/sample-{idx}.json   │
│  4. LLM-Judge 评估 answer vs expected                    │
│  5. 分析问题 → 修改 SKILL.md                             │
│  6. 重复                                                │
└─────────────────────────────────────────────────────────┘
         │                    │
         │ python subprocess   │ 读取 JSON
         ▼                    ▼
┌─────────────────┐   ┌──────────────────────────┐
│ 黑盒 Agent       │   │ output/                   │
│ 不可修改！        │   │ ├── train/{turn}/         │
│ 仅可启动和传参    │   │ │   ├── sample-0.json     │
└─────────────────┘   │ │   └── sample-1.json     │
                       │ ├── val/{turn}/           │
                       │ └── test/final/           │
                       └──────────────────────────┘
```

### Agent 启动协议

```
python <agent_py_path> \
    --data-path <train|val|test.json> \
    --skill <target_skill_dir> \
    --output-dir <output_dir> \
    --turn <round_number>

参数说明：
  --data-path:  数据文件路径（train.json / val.json / test.json）
  --skill:      目标 skill 的目录路径（包含 SKILL.md）
  --output-dir: 输出文件目录基路径
  --turn:       当前优化轮次（1, 2, 3, ... "final"）
```

### Agent 输出格式

每个样本输出一个 JSON 文件：`<output_dir>/train/<turn>/sample-<idx>.json`

```json
{
  "system_prompt": "当前使用的系统提示词（取自 SKILL.md，可为空字符串）",
  "user_prompt": "用户问题（即数据文件中该样本的 input 字段）",
  "answer": "Agent 的最终回答"
}
```

### 关键约束

- Agent 应用是**黑盒**，不允许修改其代码
- 仅通过 CLI 启动 Agent
- 执行失败、传参不对、输出有误 → **告知用户如何修改，不擅自修改 Agent**
- Claude Code 只做：读取输出 JSON → 分析差距 → 修改 SKILL.md

---

## 数据管理

### 数据格式

```json
[
  {"input": "用户给Agent的任务输入", "expected": "期望Agent输出的结果"}
]
```

### 路径解析

- 默认路径：`./train.json` `./val.json` `./test.json`
- 用户可通过 `--train` `--val` `--test` 参数指定自定义路径
- 数据文件为**严格只读**，不修改、不 chmod
- 文件缺失且未指定路径 → 提示用户并提供 `templates/data_sample.json` 样例
- 验证要求：每条记录含 `input` 和 `expected`，每集 ≥3 条

---

## 评分体系（8 维度，100 分）

### 结构维度（30 分）— 静态分析

| # | 维度 | 权重 | 评分标准 |
|---|------|:----:|---------|
| 1 | **Frontmatter 质量** | 5 | name 规范、description 包含"做什么+何时用+触发词"、≤1024 字符 |
| 2 | **工作流清晰度** | 8 | 步骤明确可执行、有序号、每步有明确输入/输出 |
| 3 | **边界条件覆盖** | 5 | 处理异常情况、有 fallback 路径、错误恢复 |
| 4 | **检查点设计** | 4 | 关键决策前有用户确认或验证、防止自主失控 |
| 5 | **指令具体性** | 5 | 不模糊、有具体参数/格式/示例、可直接执行 |
| 6 | **资源整合度** | 3 | references/scripts/assets 引用正确、路径可达 |

### 效果维度（70 分）— 需要实测

| # | 维度 | 权重 | 评分标准 |
|---|------|:----:|---------|
| **7** | **任务完成度** | **50** | **LLM-Judge 评估 answer vs expected，在 train+val 数据上实测取平均** |
| 8 | **整体架构** | 10 | 结构层次清晰、不冗余不遗漏 |
| 9 | **留出测试表现** | 10 | LLM-Judge 在 test.json 上最终验证（仅最终评一次） |

### 评分规则

- 维度 1-6, 8：每个维度打 1-10 分，乘以权重得到该维度得分
- 维度 7：LLM-Judge 对每个样本评分（0-100）→ 全部样本取平均 → 映射为 50 分制 `round(avg_score * 0.5)`
- 维度 9：最终验证时，LLM-Judge 对 test 集评分取平均 → 映射为 10 分制 `round(avg_score * 0.1)`
- **总分 = Σ(维度分 × 权重) / 10**（维度 1-6 实际得分 × 权重 / 10，维度 7 直接 0-50，维度 8 得分 × 10 / 10 = 得分）
- 总分满分 100，保留 1 位小数
- 改进后总分必须**严格高于**改进前才保留

---

## LLM-as-Judge 自适应评估

### 评估输入（3 个字段）

- `user_prompt`：数据文件中的 input
- `expected`：数据文件中的 expected
- `answer`：Agent 输出 JSON 中的 answer

### 5 档语义量表

| 档位 | 名称 | 语义标准 | 分数范围 |
|:----:|------|---------|:--------:|
| **A** | 优秀 | 任务完全达成，输出与期望高度一致，核心内容无遗漏无错误 | 90-100 |
| **B** | 良好 | 任务基本达成，存在轻微偏差（措辞、格式等）但不影响核心结论 | 75-89 |
| **C** | 一般 | 任务部分达成，核心方向正确但有明显偏差（遗漏要点、错误细节） | 60-74 |
| **D** | 较差 | 任务完成度不足，仅达成部分次要目标，核心目标未完全实现 | 40-59 |
| **E** | 很差 | 任务基本未达成，输出与期望差距大，或答非所问 | 0-39 |

### 评估流程

1. **任务类型检测**：LLM Judge 自动判断任务类型（分类/生成/抽取/推理/代码/QA 等）
2. **自适配评估**：根据任务类型选择最合适的评判标准
3. **结构化输出**：`{tier, score, justification, error_analysis, error_category}`
4. **独立执行**：用子 agent 评分，不能同一上下文"改完直接评"
5. **反谄媚规则**：不得受轮次数、历史分数影响

### 错误分类（error_category）

| 分类 | 含义 |
|------|------|
| `format_mismatch` | 输出格式与期望不符 |
| `content_omission` | 缺少必要信息 |
| `hallucination` | 编造了不存在的事实/数据 |
| `logic_error` | 推理步骤错误 |
| `instruction_drift` | 偏离任务指令 |
| `over_constraint` | skill 过度约束导致输出僵硬 |
| `under_specification` | skill 指令不够具体导致输出模糊 |

### 评分独立性（IRON RULE）

效果维度必须使用独立子 agent 评分。如果子 agent 不可用，退化为 `dry_run` 并在 tsv 中标注 `eval_mode=dry_run`。

---

## 人类评分校准与全量重打分

在 Phase 1 用户对展示样例调整评分后：

1. 用户给出校准后的分数 + **打分依据**（为什么这个分数更合理、正确的评判标准是什么）
2. LLM-Judge 以用户校准分数和打分依据作为**参考锚点**，对**全部样本**（train + val）重新打分
3. 重打分后重新排序，最差 10 个可能发生变化
4. 对新进入最差 10 的样本，补充启动 problem_analyzer_agent 分析
5. 校准记录和重打分结果保存到对应的 tsv 文件

---

## 实验记录管理

训练集和验证集评分**分开记录**。

### result-train.tsv

```
timestamp	commit	skill	old_score	new_score	status	dimension	note	eval_mode	change_type	overfitting_alert
2026-05-09T10:00	abc1234	my-skill	-	68	baseline	-	init	auto	-	none
2026-05-09T10:15	def5678	my-skill	68	72	keep	task_completion	added_reasoning_steps	auto	reasoning_steps	none
```

### result-val.tsv（判断优化是否有效的唯一依据）

```
timestamp	commit	skill	old_score	new_score	status	eval_mode	val_score	train_score	overfitting_alert
2026-05-09T10:00	abc1234	my-skill	-	62	baseline	auto	62	68	none
2026-05-09T10:15	def5678	my-skill	62	68	keep	auto	68	72	none
2026-05-09T10:30	ghi9012	my-skill	68	64	revert	auto	64	76	warning
```

### 核心原则：train 用于优化，val 仅用于判断

| 用途 | train | val |
|------|:-----:|:---:|
| 分析问题、诊断根因 | ✅ | ❌ |
| 生成优化方案 | ✅ | ❌ |
| 判断优化是否有效 | ❌ | ✅（唯一标准） |
| 过拟合检测 | ❌ | ✅（与 train 趋势对比） |
| 停止条件判断 | ❌ | ✅ |

---

## Phase 1：交互式诊断（人在回路）

```
Step 0: 初始化
  - 读取目标 SKILL.md，确认存在
  - 读取 train.json + val.json + test.json
  - 验证数据文件：每条含 input 和 expected，每集 ≥3 条
  - git checkout -b auto-optimize/YYYYMMDD-HHMM
  - 初始化 result-train.tsv 和 result-val.tsv（如不存在则写入表头）

Step 1: 运行 Agent + LLM-Judge 首轮评估（全量）
  - 启动 Agent 跑 train:
    python <agent_py> --data-path <train.json> --skill <skill_dir> --output-dir output --turn 1
  - 读取 output/train/1/sample-*.json → {system_prompt, user_prompt, answer}
  - 启动 Agent 跑 val:
    python <agent_py> --data-path <val.json> --skill <skill_dir> --output-dir output --turn 1
  - 读取 output/val/1/sample-*.json
  - 逐样本启动 llm_judge_agent (user_prompt, expected, answer)
    → {tier, score, justification, error_category}
  - 按 score 升序排列全部样本（最差在前）
  - 写入 result-train.tsv 和 result-val.tsv baseline 行

Step 2: 挑选最差 10 个
  - train 最低分 5 个 + val 最低分 5 个 = 10 个样例

Step 3: 10 个并行子 agent 分析
  启动 10 个 problem_analyzer_agent，每个接收:
    - user_prompt（该样本的 input）
    - expected（期望回答）
    - answer（Agent 实际输出）
    - score + tier + error_category（LLM-Judge 评分结果）
  每个独立分析 → 输出:
    - 根本原因: 为什么 Agent 产出偏离期望？根因在 SKILL.md 哪一段？
    - 原因分类: format_mismatch | content_omission | hallucination | logic_error
               | instruction_drift | over_constraint | under_specification
    - Option A: 优化方式 + 改动位置 + 改动类型 + 预期效果 + 风险
    - Option B: 备选优化方式（同上结构）
  检查是否有重复根因 → 标记"高频根因"

Step 4: 逐样例用户预览 + 评分校准
  对每个样例依次展示:
    - user_prompt / expected / answer / LLM-Judge 评分（tier + score）
    - 子 agent 分析: 根本原因 / 分类
    - Option A（推荐）/ Option B（备选）
  用户交互:
    - 是否调整评分？→ 输入新分数 + 打分依据
    - 选择优化方式: [1] A  [2] B  [3] 自定义输入

Step 5: 全量重打分（关键步骤）
  ⚠️ 用户校准评分后，必须触发:
  - 以用户校准的分数和打分依据作为参考锚点
  - LLM-Judge 对 train + val 全部样本重新评分
  - 全部样本重新排序
  - 如果新进入最差 10 的样本尚未被分析:
    → 补充启动 problem_analyzer_agent
    → 再次向用户展示新增的分析结果
  - 用户确认后进入 Step 6

Step 6: change_synthesizer 汇总合成
  输入所有分析 + 用户选择 + 最终校准评分 → 合并为编辑计划:
    a. 去重: 指向同一根因的多个分析 → 合并为一个编辑
    b. 冲突检测: 修改同一段落的不同编辑 → 按优先级协调
    c. 排序: 影响面最大的编辑优先执行
    d. 校验: 无 FORBIDDEN 改动类型、文件总变动 ≤150% 原始大小

Step 7: 应用编辑 + 记录
  - 按编辑计划逐项修改 SKILL.md → git add + commit
  - 启动 Agent 重新跑 train + val（turn=1-post）
  - LLM-Judge 重新评分 → result-train.tsv + result-val.tsv 各追加 keep 行
  - 写入 turn-1.md（记录全部样本的评分、分析、优化思路，见下方格式要求）
```

### turn-{round}.md 记录要求

**必须记录全部样本**（不只是最差 10 个），包含：

- **全部样本评分表**：train 和 val 每个样本的 `user_prompt` / `expected` / `answer` / LLM-Judge 初始评分 / 校准后评分（如有）/ 分析摘要 / 优化思路
- **最差 10 个详细分析**：完整问题分析 + Option A/B + 用户选择
- **评分校准记录**：用户调整了哪些评分、调整依据
- **全量重打分结果**：校准前后的分数变化对比
- **编辑计划**：具体修改了 SKILL.md 哪些行、改动类型、预期效果
- **分数变化**：修改前后的结构分数 + 任务完成度分数 + 总分对比

---

## Phase 2：全自动优化（train 分析 + val 判断）

```
round = 2..10:

  Step 1: 运行 Agent 跑 train + val（当前 skill）
    python <agent_py> --data-path <train.json> --skill <skill_dir> --output-dir output --turn {round}
    读取 output/train/{round}/sample-*.json
    python <agent_py> --data-path <val.json> --skill <skill_dir> --output-dir output --turn {round}
    读取 output/val/{round}/sample-*.json

  Step 2: LLM-Judge 分别评分
    - 对 train 全部样本评分 → train_score（平均），写入 result-train.tsv
    - 对 val 全部样本评分 → val_score（平均），写入 result-val.tsv
    - 评分时参考 Phase 1 人类校准的打分依据

  Step 3: 检查停止条件（仅看 val）
    - 对比 result-val.tsv 历史：val_score 连续 3 轮 delta < 1% → STOP
    - 达到 10 轮 → STOP
    - 过拟合检测触发 → STOP（见反过拟合机制）

  Step 4: 仅分析 train 的问题（不分析 val）
    ⚠️ IRON RULE: 只从 train 找问题来优化 skill，
       禁止通过分析 val 的具体内容寻找优化方向。
       val 的唯一作用是：通过 val_score 的涨跌判断优化是否有效。

    - 从 train 中找出最低分样本（至少 3 个）
    - 分析问题根因（仅在 train 数据上）
    - 基于 train 的问题生成改进方案（仅限 ALLOWED 改动类型）
    - 每次只改一个维度，只做一种改动

  Step 5: 应用改动 → git add + commit

  Step 6: 重新跑 Agent（train + val）+ LLM-Judge 评分

  Step 7: 决策（仅看 val）
    新 val_score > 旧 val_score（严格大于）→ status=keep，更新 best_val_score
    否则 → git revert HEAD, status=revert
    val 是判断优化是否有效的唯一标准，train 分数仅供参考

  Step 8: 过拟合检测
    交叉对比 result-train.tsv 和 result-val.tsv 最近 3 轮趋势
    - train↑ + val↓ → overfitting 嫌疑（见反过拟合机制）

  Step 9: 更新 result-train.tsv + result-val.tsv
    写入 turn-{round}.md（记录全部样本评分变化 + 改动内容 + 决策依据）

最终: 跑 test 集（一次性，留出集，从未参与优化分析）
  python <agent_py> --data-path <test.json> --skill <skill_dir> --output-dir output --turn final
  LLM-Judge 评估全部 test 样本 → test_score
  写入 result-test.tsv
  如果 test_score < best_val_score - 10 → 过拟合终验警告
```

---

## problem_analyzer_agent 规格

### 输入

```
- user_prompt:    "请帮我分析这份化学实验报告的产率偏低的原因"
- expected:       "产率偏低的主要原因是...（期望的完整分析回答）"
- answer:         "产率偏低可能有以下原因...（Agent 实际输出）"
- score:          45
- tier:           "D"
- error_category: "content_omission"
```

### 输出

```
1. 根本原因
   描述为什么 Agent 产出偏离期望。追溯到 SKILL.md 的具体位置。
   示例: "Agent 缺少对具体变量的定量分析。SKILL.md 第 42 行只说'分析原因'
         但没有要求引用报告中的数值数据，导致输出空泛。"

2. 原因分类
   format_mismatch | content_omission | hallucination | logic_error
   | instruction_drift | over_constraint | under_specification

3. Option A（推荐方案）
   - 改动描述: "在分析流程中增加'引用实验数据'步骤"
   - 改动位置: SKILL.md 第 40-45 行
   - 改动类型: reasoning_steps
   - 预期效果: Agent 在分析中会引用具体数值，不再空泛
   - 风险: low

4. Option B（备选方案）
   - 改动描述: "在输出格式中要求每条结论标注数据来源"
   - 改动位置: SKILL.md 第 78-82 行
   - 改动类型: output_format_spec
   - 预期效果: 每条结论有数据支撑，可追溯
   - 风险: low
```

---

## change_synthesizer_agent 职责

将 10 个并行子 agent 的分析结果和用户选择**合成为一个无冲突的编辑计划**：

1. **去重**：3 个子 agent 都发现 SKILL.md 第 45 行是同一个根因 → 合并为一个编辑，标注"3 个分析共同指向"
2. **冲突检测**：子 agent A 建议第 40-50 行插入内容，子 agent B 建议删除第 42-48 行 → 优先保留低分样本对应的建议
3. **优先级排序**：先改核心推理流程（reasoning_steps），再改输出格式（output_format_spec），最后改措辞（instruction_clarity）
4. **输出**：有序编辑计划列表 `[(行号范围, 新内容, 改动类型, 来源分析), ...]`

---

## 反过拟合机制

### 改动类型约束

**ALLOWED**（优化思考过程，被允许的改动）：
- `reasoning_steps` — 添加/明确中间推理步骤
- `verification_checkpoints` — 在关键行动前添加验证检查
- `general_rules` — 添加通用规则/约束（不包含任何具体数据值）
- `output_format_spec` — 明确输出结构/格式要求
- `error_handling` — 添加故障回退路径
- `instruction_clarity` — 将模糊表述替换为具体指令

**FORBIDDEN**（特化/作弊，禁止的改动）：
- `specific_input_output` — 添加数据中的具体输入/输出对
- `case_specific_branch` — 创建匹配特定样例的条件分支
- `hardcoded_answer` — 在 skill 中嵌入期望答案的具体内容
- `data_pattern_match` — 添加对数据中特定字符串的检测逻辑
- `example_embedding` — 将训练样例（或改写后的变体）复制到 skill
- `regex_overfit` — 添加过度特定的正则或模式匹配

### 过拟合检测准则

| 准则 | 触发条件 | 级别 | 响应 |
|------|---------|:----:|------|
| 直接过拟合 | train_score↑ + val_score↓ 连续 2 轮 | **CRITICAL** | 立即停止 Phase 2，回滚到 best_val 对应的 commit |
| 隐性过拟合 | val_score 停滞（delta <1%）+ train_score 持续上升 3 轮 | WARNING | 限缩下轮改动范围，仅允许 `general_rules` |
| 禁用改动 | 改动用到了 FORBIDDEN 类型 | **CRITICAL** | 立即回滚该轮，标记到 overfitting_alert |
| 字符串重叠 | 编辑内容与 train 数据中的字符串重合 >30% | WARNING | 分数惩罚 -5 分，重新评估是否值得保留 |

---

## 停止条件

1. **val 停滞**：val_score 连续 3 轮无提升（delta < 1%）→ 以 result-val.tsv 为准
2. **最大轮次**：达到 10 轮强制停止
3. **过拟合检测**：train↑ + val↓ 连续 2 轮触发 CRITICAL → 停止并回滚到 best_val

---

## 约束规则

1. **不改变 SKILL 的核心功能** — 只优化"怎么写"和"怎么执行"，不改"做什么"
2. **不引入新依赖** — 不添加 target skill 原本没有的 scripts 或 references 文件
3. **每轮只改一个维度** — 避免多个变更导致无法归因
4. **文件大小 ≤150% 原始** — 优化后 SKILL.md 不超原始 150%
5. **可回滚** — 所有改动在 git 分支上，用 `git revert` 而非 `git reset --hard`
6. **评分独立性** — 效果维度必须用独立子 agent，不能同一上下文改完直接评
7. **黑盒不可改** — Agent 应用只启动不修改；任何 Agent 相关问题告知用户
8. **数据只读** — train/val/test.json 严格只读，不修改不 chmod
9. **train/val 用途分离** — train 用于分析问题，val 仅用于判断优化效果

---

## 异常处理

| 场景 | 触发条件 | 处理 |
|------|---------|------|
| 不在 git 仓库 | `git rev-parse` 失败 | 提示用户 `git init`；拒绝则用 `cp SKILL.md SKILL.md.bak.YYYYMMDD-HHMM` 备份 |
| train.json 缺失 | 默认路径不存在且未指定 | 提示用户创建，提供 `templates/data_sample.json` 样例格式，暂停等待 |
| val.json 缺失 | 默认路径不存在且未指定 | 同上 |
| test.json 缺失 | 默认路径不存在且未指定 | 同上 |
| 数据格式错误 | JSON 解析失败或缺少 `input`/`expected` | 输出具体行号和错误原因，让用户修复，暂停等待 |
| 数据量不足 | 某数据集 <3 条 | 提示至少需要 3 条样本，暂停等待 |
| Agent 启动失败 | subprocess 返回非 0 | 输出完整错误信息（stdout+stderr），告知用户修复方式。**不擅自修改 Agent 代码** |
| Agent 输出文件缺失 | output JSON 不存在或无法解析 | 输出期望的文件路径和格式，告知用户检查 Agent 输出逻辑 |
| Agent 输出字段缺失 | JSON 中缺少 `user_prompt` 或 `answer` | 输出缺字段的样本 ID 和缺失字段名，告知用户修复 |
| LLM-Judge 超时 | 子 agent 120s 未返回 | 重试 1 次；仍失败降级为 `degraded`，用主 agent 评分并在 tsv 中标注 |
| LLM-Judge 输出格式错误 | JSON schema 校验失败 | 让 Judge 重新输出，最多 2 次；仍失败降级为 `degraded` |
| `git revert` 失败 | 冲突或工作目录脏 | 先 `git stash` → 重试 revert；仍失败则从上一个 commit 的 SKILL.md 读取覆盖 |
| 分支名冲突 | `auto-optimize/*` 分支已存在 | 追加 `-2`/`-3`；3 次失败则切回现有分支询问用户 |
| 体积超 150% | 新文件 > 原始 × 1.5 | 拒绝提交，精简编辑计划中次要的改动 |
| result-*.tsv 缺失 | 文件不存在 | 新建并写入对应表头 |
| result-*.tsv 损坏 | 列数不匹配或非 TSV | 备份为 `.bak.YYYYMMDD-HHMM` 后重建，告知用户 |
| Phase 2 轮次达上限 | 10 轮后仍无显著提升 | 不强制继续，输出摘要："已运行 10 轮，建议手动检查当前 skill 质量" |
| 用户中断 | Phase 1 中用户按 Ctrl+C | 保存当前进度（已分析的样例 + 校准记录），提供恢复路径 |

**黄金原则**：异常先告知用户，再按规则处理；绝不静默跳过或静默失败。

---

## 文件结构

```
auto-opti-skill/
├── SKILL.md                          # 本文件
├── agents/
│   ├── llm_judge_agent.md            # LLM-as-Judge 评估 agent
│   ├── problem_analyzer_agent.md     # 单问题分析 agent
│   ├── change_synthesizer_agent.md   # 10 分析 → 1 编辑计划
│   ├── overfitting_detector_agent.md # 过拟合检测 agent
│   └── hill_climber_agent.md         # Phase 2 自动优化 agent
├── references/
│   ├── evaluation_protocol.md        # LLM-Judge 完整评估协议
│   ├── change_type_catalog.md        # ALLOWED / FORBIDDEN 改动类型目录
│   ├── anti_overfitting_algorithms.md
│   └── failure_paths.md              # 故障场景与恢复策略
└── templates/
    └── data_sample.json              # 数据样例
```

---

## 子 Agent 速查

| Agent | 文件 | 用途 | 何时启动 |
|-------|------|------|---------|
| llm_judge | `agents/llm_judge_agent.md` | 评估 answer vs expected | 每轮评分、全量重打分 |
| problem_analyzer | `agents/problem_analyzer_agent.md` | 分析单样本问题根因 + 推荐优化 | Phase 1 最差 10 个 |
| change_synthesizer | `agents/change_synthesizer_agent.md` | 合并分析为编辑计划 | Phase 1 汇总时 |
| overfitting_detector | `agents/overfitting_detector_agent.md` | 过拟合检测 | Phase 2 每轮结束时 |
| hill_climber | `agents/hill_climber_agent.md` | 自动诊断 + 生成改进 | Phase 2 每轮 Step 4 |
