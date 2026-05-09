# LLM-as-Judge 评估协议

## 评估流程完整说明

### Phase 1: 评估准备

1. 主 agent 读取 Agent 输出的 `output/{train|val}/{turn}/sample-*.json`
2. 对每个样本，提取 `(user_prompt, expected, answer)` 三元组
3. 同步读取数据文件中的 `expected`（与 output JSON 中的 user_prompt 对应）

### Phase 2: 逐样本评估

对于每个样本：

1. **启动 llm_judge_agent**（独立子 agent）
2. 传入：`{user_prompt, expected, answer, calibration}`
3. 等待输出：`{task_type, tier, score, justification, error_analysis, error_category, sub_scores}`
4. 收集所有评分结果

### Phase 3: 评分聚合

```
train_avg = mean([sample.score for sample in train_samples])
val_avg = mean([sample.score for sample in val_samples])
train_tier_distribution = count_per_tier(train_samples)
val_tier_distribution = count_per_tier(val_samples)
```

### Phase 4: 映射到评分维度

- **维度 7（任务完成度 50 分）**：`round(train_avg * 0.5)` 和 `round(val_avg * 0.5)` 的平均
- **维度 9（留出测试 10 分）**：`round(test_avg * 0.1)`（仅最终验证）

## 任务类型检测详细方法

```
1. 读取 expected 的内容特征:
   - 是单个标签/类别？ → classification
   - 是纯数值？ → numerical_computation
   - 是 JSON/结构化对象？ → extraction 或 format_transform
   - 是多段文字分析？ → analysis 或 text_generation
   - 是可执行代码？ → code_generation
   - 是逻辑链推理？ → reasoning

2. 读取 user_prompt 的意图:
   - "分类"/"判断"/"识别" → classification
   - "计算"/"求"/"多少" → numerical_computation
   - "提取"/"找出"/"列出" → extraction
   - "分析"/"解释"/"为什么" → analysis
   - "写"/"生成"/"创建代码" → code_generation
   - "是否"/"对不对"/"验证" → qa

3. 如果两者一致 → 确认类型
4. 如果不一致 → 以 expected 的特征为主，user_prompt 的意图为辅
```

## 评分校准机制

### 校准数据格式

```json
{
  "calibrated_samples": [
    {
      "sample_id": "train_003",
      "original_tier": "D",
      "original_score": 45,
      "calibrated_tier": "C",
      "calibrated_score": 65,
      "rationale": "Agent 的输出虽然缺少具体数值，但推理方向正确，应给 65 而非 45。评判标准应更看重推理逻辑而非数值精确度。"
    }
  ]
}
```

### 校准如何影响后续评分

1. 主 agent 在每次启动 llm_judge_agent 时传入 `calibration` 字段
2. LLM-Judge 读取 rationale，理解用户的评判倾向：
   - 如用户说"推理逻辑优先于格式"→ 降低 format 维度的权重
   - 如用户说"数值精度最重要"→ 提高 correctness 维度的权重
3. 校准**影响评判方式**，但**不直接修改分数**

## 降级评估

当 LLM-Judge 子 agent 不可用时（超时、格式错误），采用降级评估：

### 精确匹配模式（仅适用于 classification 和 numerical_computation）

```
IF task_type == "classification":
  score = 100 if answer == expected else 0
  tier = "A" if score >= 90 else "E"

IF task_type == "numerical_computation":
  尝试从 answer 中提取数值
  IF 提取的数值与 expected 的数值误差 < 1%:
    score = 95, tier = "A"
  ELIF 误差 < 10%:
    score = 80, tier = "B"
  ELIF 误差 < 30%:
    score = 60, tier = "C"
  ELSE:
    score = 30, tier = "E"
```

降级评估的结果在 results.tsv 中标注 `eval_mode=degraded`。

## 评分一致性检查

每轮评分完成后检查：

1. 同一 tier 的样本，score 的 std 不应该 > 15
2. 同一 error_category 的样本，score 应该在合理范围内
3. 如果 score 分布异常（全部 >90 或全部 <30），标记并建议人工复核
