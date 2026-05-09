# 故障场景与恢复策略

## Agent 启动相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| Agent Python 文件不存在 | `python agent.py` → FileNotFoundError | **MAJOR** | 告知用户路径不存在，让用户确认 `--agent` 参数是否正确 |
| Agent 缺少依赖 | ImportError / ModuleNotFoundError | **MAJOR** | 输出错误信息。告知用户安装缺失的依赖，不擅自 `pip install` |
| Agent 参数不兼容 | Agent 报错 "unrecognized arguments" | **MAJOR** | 输出 Agent 的 help 信息，告知用户 Agent 支持的参数列表。对比 auto-opti 期望的参数 |
| Agent 执行超时 | subprocess > 600s | **MINOR** | 杀掉进程。提示用户 Agent 可能卡死或数据量过大。建议检查 Agent 或减少数据量 |
| Agent 返回非 0 但无错误信息 | returncode != 0, stderr 为空 | **MINOR** | 告知用户 Agent 异常退出但无错误信息。建议用户手动运行 Agent 排查 |
| Agent 输出目录为空 | output JSON 数量 < 数据样本数 | **MAJOR** | 告知用户输出文件数量不匹配。列出哪些样本有输出、哪些没有 |

## 数据文件相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| train.json 缺失 | 默认路径和用户指定路径均不存在 | **MAJOR** | 提示用户创建，展示 `templates/data_sample.json` 样例格式 |
| val.json 缺失 | 同上 | **MAJOR** | 同上 |
| test.json 缺失 | 同上 | **MINOR** | 提示用户创建。如仅做 Phase 1 可不需 test.json |
| JSON 格式错误 | `JSON.parse()` 失败 | **MAJOR** | 输出解析错误的具体行号，让用户修复 |
| 缺少字段 | 某条记录无 `input` 或 `expected` | **MAJOR** | 列出缺失字段的样本索引，跳过该样本。如 >50% 样本有问题则中止 |
| 数据量不足 | 某数据集 <3 条 | **MAJOR** | 提示至少需要 3 条样本用于评分 |
| train/val 数据重叠 | train 和 val 中有完全相同的 input | **WARNING** | 警告用户可能存在数据泄漏，建议去重 |
| 文件权限问题 | 文件存在但无法读取 | **MAJOR** | 提示权限错误，让用户 `chmod +r` 修复 |

## LLM-Judge 相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| Judge 子 agent 超时 | 120s 无响应 | **MINOR** | 重试 1 次。仍失败则降级为 `degraded` 模式 |
| Judge 输出非 JSON | JSON.parse 失败 | **MINOR** | 让 Judge 重试，附带格式纠正提示。最多 2 次。仍失败降级 |
| Judge 输出缺字段 | JSON 校验失败 | **MINOR** | 同上 |
| Judge 评分分布异常 | 所有样本 >90 或 <30 | **WARNING** | 提示用户评分可能偏离，建议人工复核 2-3 个样本的评分 |
| Judge 连续 N 次失败 | 子 agent 连续 3 次返回错误 | **MAJOR** | 中止当前轮，建议用户检查 LLM 可用性 |

## Git 相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| 不在 git 仓库 | `git rev-parse` 失败 | **MINOR** | 提示用户。如拒绝 git init，降级为文件备份 |
| 分支名冲突 | `git checkout -b` 失败 | **MINOR** | 追加 `-2`/`-3`。3 次失败则切回现有分支询问用户 |
| `git revert` 失败 | 冲突或工作目录脏 | **MAJOR** | `git stash` → 重试。仍失败则从上一个 commit 读取 SKILL.md 覆盖 |
| `git commit` 失败 | 无改动 / hook 失败 | **MINOR** | 输出 git 错误信息。如无改动则跳过 |
| 仓库空间不足 | `git` 写入失败 | **MAJOR** | 提示用户清理空间 |

## SKILL.md 相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| SKILL.md 不存在 | 目标路径无 SKILL.md | **MAJOR** | 告知用户路径不正确，让用户确认 --skill 参数 |
| 文件体积超限 | 编辑后 > 原始 ×1.5 | **MINOR** | 拒绝提交。精简编辑计划中次要的改动 |
| 编辑冲突 | 两个同时进行的编辑修改同一行 | **MINOR** | change_synthesizer 自动协调。如无法自动解决则标记需人工裁决 |
| FORBIDDEN 改动被检测到 | 编辑内容触发准则 3 或 4 | **MAJOR** | 拒绝该编辑。标记到 overfitting_alert。要求重新生成 |

## 用户交互相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| Phase 1 用户中断 | Ctrl+C 或会话断开 | **MINOR** | 保存当前进度（已分析的样本列表 + 校准记录）。告知用户下次如何恢复 |
| 用户连续跳过 | 用户对 >50% 样本选择"跳过" | **WARNING** | 询问用户是否希望结束 Phase 1 直接进入 Phase 2 |
| 用户输入无效选择 | 输入的不是 1/2/3 | **MINOR** | 提示有效选项范围，重新询问 |

## Phase 2 相关

| 场景 | 触发条件 | 严重度 | 处理 |
|------|---------|:------:|------|
| 连续 revert | 连续 3 轮 val_score 下降 → revert | **WARNING** | 提示：优化似乎进入瓶颈，建议停止或手动介入 |
| val_score 不增不减 | 5 轮 val_score delta < 0.5 | **WARNING** | 提示：可能需要改变优化策略或停止 |
| 10 轮上限 | Phase 2 达到 10 轮 | **MINOR** | 不强制继续。输出全流程摘要 |
| test_score 大幅低于 val | test_score < best_val_score - 10 | **WARNING** | 输出过拟合终验警告。在最终报告中明确标注 |

---

## 通用处理原则

1. **先告知，再处理**：遇到任何异常，先输出清晰的信息告知用户发生了什么
2. **不静默跳过**：绝不在日志中默默记录错误然后跳过样本
3. **不猜测修复**：数据问题和 Agent 问题让用户修复，不自行修改
4. **保存中间状态**：长时间运行的流程在中止时应保存进度
5. **降级而非崩溃**：某个子功能失败时尝试降级运行，而非整个流程中止
