# 反过拟合算法

## 核心原理

过拟合 = skill 学会了 train 数据的特异性模式，但这些模式在 val/test 数据上不成立。

检测方法 = 交叉对比 train 和 val 的评分趋势。

## 检测算法

```
FUNCTION detect_overfitting(history, current_round, change_type, edit_diff, train_data):

    alerts = []

    // ===== 准则 1: 直接过拟合 =====
    IF len(history) >= 2:
        last_two = history[-2:]
        delta_train_1 = last_two[1].train_score - last_two[0].train_score
        delta_val_1 = last_two[1].val_score - last_two[0].val_score

        IF delta_train_1 > 2 AND delta_val_1 < -2:
            alerts.append({
                criterion: "direct_overfitting",
                level: "critical",
                detail: f"Round{last_two[0].round}→{last_two[1].round}: "
                        f"train {last_two[0].train_score}→{last_two[1].train_score} "
                        f"(+{delta_train_1}) but val {last_two[0].val_score}→"
                        f"{last_two[1].val_score} ({delta_val_1})"
            })

    // ===== 准则 2: 隐性过拟合 =====
    IF len(history) >= 3:
        last_three = history[-3:]
        val_scores = [h.val_score for h in last_three]
        train_scores = [h.train_score for h in last_three]
        val_range = max(val_scores) - min(val_scores)
        train_trend = train_scores[-1] - train_scores[0]

        IF val_range < 1.0 AND train_trend > 3.0:
            alerts.append({
                criterion: "hidden_overfitting",
                level: "warning",
                detail: f"val stagnant (range={val_range:.1f}) "
                        f"but train rising (+{train_trend:.1f} over 3 rounds)"
            })

    // ===== 准则 3: FORBIDDEN 改动类型 =====
    FORBIDDEN = [
        "specific_input_output", "case_specific_branch",
        "hardcoded_answer", "data_pattern_match",
        "example_embedding", "regex_overfit"
    ]
    IF change_type IN FORBIDDEN:
        alerts.append({
            criterion: "forbidden_change_type",
            level: "critical",
            detail: f"change_type '{change_type}' is FORBIDDEN"
        })

    // ===== 准则 4: 数据字符串泄漏 =====
    overlap_count = 0
    FOR each sample in train_data:
        FOR field in [sample.input, sample.expected]:
            IF len(field) > 20:
                // 滑动窗口，检查 edit_diff 中包含 field 的 ≥10 字符子串
                FOR i in range(0, len(field) - 9):
                    substring = field[i:i+10]
                    IF substring in edit_diff:
                        overlap_count += 1
                        break  // 同一个 field 只计数一次

    IF overlap_count > 2:
        alerts.append({
            criterion: "string_leakage",
            level: "warning",
            detail: f"edit_diff contains substrings from {overlap_count} "
                    f"training data fields (threshold: 2)"
        })

    // ===== 汇总 =====
    IF any alert.level == "critical" for alert in alerts:
        overall_level = "critical"
        recommendation = "stop_and_revert"
    ELIF any alert.level == "warning" for alert in alerts:
        overall_level = "warning"
        recommendation = "continue_with_constraints"
    ELSE:
        overall_level = "none"
        recommendation = "continue"

    RETURN { overall_level, alerts, recommendation }
```

## 语义变体测试

每 3 轮执行一次。

```
FUNCTION semantic_variant_test(val_samples, agent_py, skill_dir, output_dir, round):
    // 1. 随机选 2 个 val 样本
    test_samples = random.sample(val_samples, min(2, len(val_samples)))

    // 2. 对每个样本生成同义改写
    variants = []
    FOR each sample in test_samples:
        variant_input = paraphrase(sample.input)
        variants.append({
            original_sample: sample,
            variant_input: variant_input
        })

    // 3. 用改写后的输入跑 Agent
    // (写入临时数据文件，启动 Agent)
    variant_scores = run_agent_and_judge(variants, agent_py, skill_dir, output_dir, round)

    // 4. 对比原始分数和变体分数
    FOR each variant in variants:
        original_score = variant.original_sample.score
        variant_score = variant_scores[variant]
        delta = variant_score - original_score

        IF delta < -10:
            RETURN {
                alert: true,
                level: "warning",
                detail: f"Sample {variant.original_sample.id}: "
                        f"paraphrase caused score drop of {delta}"
            }

    RETURN { alert: false }
```

## 响应协议

### CRITICAL 响应

1. 主 agent 立即停止 Phase 2 的优化循环
2. 查找 history 中 val_score 最高的轮次 → 读取对应的 git commit
3. `git checkout <best_commit> -- SKILL.md` 恢复到最优状态
4. 将完整的 overfitting 证据写入 turn-N.md
5. 在 result-val.tsv 中标注当前行为 `status=overfitting_stop`

### WARNING 响应

1. 记录警告到 result-val.tsv 的 `overfitting_alert` 列
2. 下轮 hill_climber 的改动类型限制为 `general_rules` 或 `instruction_clarity`
3. 下轮字符串泄漏检查阈值从 2 降低到 1（更严格）
4. 如果连续 2 次 WARNING → 升级为 CRITICAL

## 字符串重叠计算方法

```
FUNCTION compute_overlap(edit_text, train_samples):
    // 提取 edit_text 中所有长度 ≥10 的 token 序列
    edit_tokens = tokenize(edit_text)
    edit_ngrams = set()
    FOR each token in edit_tokens:
        FOR length in [3, 5, 10]:
            ngram = token[start:start+length]
            IF len(ngram) == length:
                edit_ngrams.add(ngram)

    // 统计在 train 数据中出现的 ngram
    train_text = join([s.input + s.expected for s in train_samples])
    overlap = 0
    FOR each ngram in edit_ngrams:
        IF ngram in train_text:
            overlap += 1

    // 返回重叠比例
    return overlap / len(edit_ngrams)
```
