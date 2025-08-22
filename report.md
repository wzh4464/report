# Weekly Report — Progress on Model Evaluation and Prompt Handling

**Presenter:** Zihan Wu  
**Date:** Today

---

## Slide 1: Executive Summary

本周我们主要完成了四个实验：

1. **Generalization**: 在未见过的 dev 集上达到 80% 准确率，说明模型有一定泛化能力
2. **Adding "correct" Label**: 加入 correct 类别后，和 others 类高度混淆，指标明显下降
3. **Binary-first Strategy**: 提出两阶段方案，先做二分类 (vulnerable vs. benign)，在第一阶段就能得到 F1 = 0.92
4. **Prompt Control**: 大规模运行时 prompt 长度持续增长。我们引入 5000-token 上限，超过就做总结压缩，保持任务与输出格式不变

---

## Slide 2: Findings & Confusion Analysis

### 1. Observations
- correct 类和 others 语义重叠，导致严重误分类
- 加入 correct 之后，整体指标不稳定

### 2. Key Numbers
- Dev Accuracy (unseen): 0.80
- Stage-1 F1: 0.92
- Token Cap: 5,000
- Current Scale: 1% run
- Planned Scale: 10%

### 3. Design Choice
我们采用两阶段流水线：
1. **Stage-1**: 先做二分类 (vulnerable vs. benign)
2. **Stage-2**: 只对 vulnerable 样本再做细粒度分类

---

## Slide 3: Binary-First Results

- **Effectiveness**: Stage-1 F1 达到 0.92，大幅减少后续分类的噪声
- **Impact on Confusion**: 先过滤掉 benign，可以有效缓解 correct 和 others 的混淆问题
- **Operational Simplicity**: 更干净的决策边界，让我们只需要在真正的 vulnerable 样本上优化 prompt，提升效率
- **Next Validation**: 目前在 1% 规模下运行稳定，接下来会扩展到 10%，进一步验证鲁棒性

---

## Slide 4: Prompt Management & Next Steps

### Prompt Growth Control
- 设置 5000 tokens 上限
- 超过上限时，进行总结压缩，但保持任务要求和输出格式不变
- 目标是：在 10% 规模下防止 prompt 爆炸，同时不牺牲准确率

### Next Steps
1. 完成 1% 运行，然后推广到 10%
2. 跟踪压缩前后模型准确率的变化
3. 进一步优化 correct vs. others 的区分，可能引入额外规则或校准器

---

## 汇报建议

建议你在汇报时：

- **Slide 1**: 快速过一遍亮点，交代问题和方案
- **Slide 2**: 重点讲 confusion 的根源，以及为什么要引入二分类
- **Slide 3**: 强调 Stage-1 的效果和带来的稳定性
- **Slide 4**: 讲大规模运行时的 prompt 管控方案和下一步计划