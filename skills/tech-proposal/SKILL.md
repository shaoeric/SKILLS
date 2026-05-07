---
name: tech-proposal
description: "AI应用技术方案架构师。从需求对齐→技术调研→方案生成→审查→迭代优化，全流程产出可落地的技术方案报告。Triggers on: 技术方案, 架构设计, 方案设计, tech proposal, solution design, architecture design, 帮我设计系统, 出技术方案, /tech-proposal"
trigger: /tech-proposal
---

# Tech Proposal Architect — AI 应用技术方案架构师

端到端技术方案生成 pipeline：需求对齐 → 技术调研 → 方案生成 → 方案审查 → 迭代优化。

## Quick Start

```
/tech-proposal
```
或直接描述需求：
```
帮我设计一个智能客服系统的技术方案
```

## Trigger Conditions

**中文**: 技术方案, 架构设计, 方案设计, 帮我设计系统, 出技术方案
**English**: tech proposal, solution design, architecture design

### Does NOT Trigger

| 场景 | 使用技能 |
|------|---------|
| 纯学术研究 | `deep-research` |
| 代码实现 | 直接实现 |
| 论文写作 | `academic-paper` |

---

## Orchestration Workflow (4 Phases)

```
User: /tech-proposal
    |
=== Phase 0: 需求对齐 ===
    |-> 用户描述场景（1-2句话）
    |-> Agent 分析场景 → 推断特征 → 使用 AskUserQuestion 逐轮提问
    |   每轮 ≤4 题，每题 2-4 选项（选择题，非问答题）
    |   详见 references/questionnaire.md
    |-> 用户勾选 → Agent 总结为 requirements.md
    |-> ** 用户确认后进入 Phase 1 **
    |
=== Phase 1: 深度技术调研 ===
    |-> 检查 deep-research 是否安装（ls ~/.claude/skills/deep-research/SKILL.md）
    |   IF 未安装 → git clone https://github.com/Imbad0202/academic-research-skills.git /tmp/ars
    |             → cp -r /tmp/ars/deep-research ~/.claude/skills/deep-research
    |             → rm -rf /tmp/ars
    |-> 调用 Skill("deep-research")
    |-> 汇总调研结果 → research.md（含来源 URL、方案对比矩阵）
    |-> ** 用户选择方案或给出修改意见后进入 Phase 2 **
    |
=== Phase 2: 方案生成 ===
    |-> Agent("proposal-generator") → proposal-1.md
    |   定义见 agents/proposal_generator.md
    |   模板见 references/proposal_template.md
    |
=== Phase 3: 方案审查 ===
    |-> Agent("proposal-reviewer") → revision-1.md
    |   定义见 agents/proposal_reviewer.md
    |   审查维度见 references/review_dimensions.md
    |
    +-> 审查结论?
        |-> PASS → cp proposal-1.md proposal-final.md → 归档只读 ✅ 完成
        |-> NEEDS REVISION → Phase 4
        |
=== Phase 4: 迭代优化（最多 3 轮）===
    |
    LOOP (version = 1 → 3):
        cp proposal-{v}.md → proposal-{v+1}.md
        Agent("proposal-generator") 手术式修改（仅改审查指出的问题）
        Agent("proposal-reviewer") 审查新版
        IF PASS → cp proposal-{v+1}.md proposal-final.md → 归档只读 → BREAK
        IF v >= 3 → cp proposal-{v+1}.md proposal-final.md
                  → revision-final.md（标注未解决问题）
                  → 归档只读 → BREAK
```

### 最终归档协议

审查通过或达到最大迭代次数后，主 Agent 执行：

1. `cp proposal-{最新v}.md proposal-final.md`
2. 打印归档清单，告知用户最终输出文件路径

### 版本管理规则

| 文件 | 可写 | 说明 |
|------|------|------|
| requirements.md | ❌ | 需求基线，用户确认后锁定 |
| research.md | ❌ | 调研结果，用户确认后锁定 |
| proposal-{v}.md (历史) | ❌ | 已审查的历史版本 |
| proposal-{v}.md (最新) | ✅ | 当前待审查/修改中 |
| revision-{v}.md (历史) | ❌ | 历史审查报告 |
| revision-final.md | ❌ | 最终审查结论，归档后只读 |
| proposal-final.md | ❌ | 最终方案定稿，归档后只读 |


---

## Agent Team

| # | Agent | 角色 | 定义文件 |
|---|-------|------|---------|
| 1 | `proposal-generator` | 高级系统架构师，基于需求和调研生成技术方案 | `agents/proposal_generator.md` |
| 2 | `proposal-reviewer` | 资深技术审查专家，6 维度审查方案质量 | `agents/proposal_reviewer.md` |

---

## Reference Files

| 文件 | 用途 | 使用阶段 |
|------|------|---------|
| `references/questionnaire.md` | 6 类结构化问卷完整模板 | Phase 0 |
| `references/proposal_template.md` | proposal-{version}.md 完整模板 | Phase 2 |
| `references/revision_template.md` | revision-{version}.md 完整模板 | Phase 3 |
| `references/review_dimensions.md` | 6 维度审查打分细则 | Phase 3 |
| `references/failure_paths.md` | 失败场景与恢复策略 | 全流程 |

---

## Anti-Patterns

| # | 禁止行为 | 正确做法 |
|---|---------|---------|
| 1 | 跳过 Phase 0 直接出方案 | 必须先完成结构化问卷 |
| 2 | 不调用 deep-research 自己编方案 | 必须调用 `Skill("deep-research")` |
| 3 | proposal-generator 修改历史版本文件 | 仅编辑当前最新版本 |
| 4 | proposal-reviewer 直接修改方案 | 审查只输出 revision，不修改 proposal |
| 5 | 用户没确认就进入下一阶段 | 每个门控必须等待用户明确确认 |
| 6 | 方案引用未在 research.md 出现的方案 | 每个技术选型必须追溯到 research.md |
| 7 | 循环超过 3 次仍不终止 | 硬限制 max 3 轮，最终轮列出未解决问题后强制终止 |

## Quality Standards

1. ⚠️ **IRON RULE**: 每个门控点必须等待用户明确确认后才能进入下一阶段
2. **可追溯性**: 方案中每个技术选型必须能追溯到 research.md 的具体来源
3. **可量化**: 所有性能指标、人力估算必须有具体数字，不接受"大概""左右"
4. **完整性**: 每个模块必须有技术选型、输入/输出定义、流程图、corner case 列表
5. **硬件可量化**: 硬件选型必须基于 Q2 性能需求和 Q3 数据量计算，CPU/内存/存储/网络/GPU 给出最低和推荐配置
6. **版本一致**: proposal 和 revision 的版本号必须严格对应

## Task Tracking

每个 Phase 开始标记 in_progress，完成标记 completed：

| Phase | Task |
|-------|------|
| 0: 需求对齐 | "Phase 0: 分析场景，逐轮选择题提问，生成 requirements.md" |
| 1: 深度调研 | "Phase 1: 调用 deep-research，生成 research.md" |
| 2: 方案生成 | "Phase 2: Agent proposal-generator 生成 proposal-1.md" |
| 3: 方案审查 | "Phase 3: Agent proposal-reviewer 审查方案" |
| 4: 迭代优化 | "Phase 4: 迭代优化 Round {version}" |
