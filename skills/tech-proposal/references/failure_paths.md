# Failure Paths — 失败场景与恢复策略

| # | 场景 | 触发条件 | 恢复策略 |
|---|------|---------|---------|
| 1 | 用户需求不清 | Phase 0 用户无法回答核心问题 | 给出 2-3 个假设场景，让用户选择最接近的作为需求基线 |
| 2 | 调研无结果 | deep-research 找不到相关方案 | 缩小范围，从相近领域借鉴；标注"探索性方案"并说明风险 |
| 3 | 审查反复不通过 | 3 轮迭代后仍有 BLOCKER | 列出所有未解决问题 + 缓解建议，让用户决策是否接受 |
| 4 | 用户中途改需求 | requirements.md 锁定后用户提新需求 | 评估影响范围：小改动→更新 requirements.md 重新生成；大改动→建议新建独立方案 |
| 5 | 技术栈不可行 | Phase 3 审查发现选型有致命缺陷 | 回退 Phase 1，重新调研替代方案，走完整流程 |
| 6 | 用户长时间不回复 | 门控等待用户确认超时 | 保留当前状态和所有中间文件，提醒用户从断点继续 |
| 7 | 版本号混乱 | proposal 和 revision 版本号不一致 | 检查各文件版本号，以 proposal-{max version}.md 为当前最新版本 |
| 8 | deep-research 未安装 | `ls ~/.claude/skills/deep-research/SKILL.md` 文件不存在 | 自动安装：`git clone https://github.com/Imbad0202/academic-research-skills.git` → 拷贝 deep-research 到 skills 目录 |
| 9 | deep-research 调用失败 | Skill("deep-research") 返回错误 | 重试 1 次；仍失败则改为手动搜索模式，但标注"未使用标准调研流程" |

## 断点恢复指南

每个 Phase 完成后，当前状态由以下文件决定：

| Phase | 状态文件 | 断点恢复动作 |
|-------|---------|-------------|
| 0 完成 | requirements.md | 直接进入 Phase 1 |
| 1 完成 | research.md | 用户确认后进入 Phase 2 |
| 2 完成 | proposal-1.md | 直接进入 Phase 3 |
| 3 完成 | revision-{v}.md 或 revision-final.md | 根据审查结论决定是否进入 Phase 4 |
| 4 循环中 | proposal-{v}.md + revision-{v}.md | 从当前 version 继续 |
