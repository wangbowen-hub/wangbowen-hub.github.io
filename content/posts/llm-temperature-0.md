---
title: "Llm Temperature 0"
date: 2025-07-11T10:14:06+08:00
# bookComments: false
# bookSearchExclude: false
---

# LLM中temperature=0的一些问题

## 为什么temperature=0，LLM的输出还是每次都不一样

理论为上temperature被引入到softmax函数中:
$$
P(xi) = exp(z_i/T) / Σ exp(z_j/T)
$$
当T等于0时，概率最大的token采样概率为1，变为贪婪解码，但在实际应用过程中，我们发现即使设置LLM的temperature为0，模型的每次输出还是有所不同，有一下几个原因：

### 1.处理策略不同

不同框架应对temperature=0的处理策略不同，实际生产框架处理常分为以下几种：

1.1 添加特殊处理

```python
# 处理 temperature=0 的情况
if temperature == 0:
    # 贪心解码：直接选择概率最高的 token
    idx_next = logits.argmax(dim=-1, keepdim=True)
else:
    # 应用 temperature 并采样
    logits = logits / temperature
    probs = F.softmax(logits, dim=-1)
    idx_next = torch.multinomial(probs, num_samples=1)
```

1.2 使用极小值代替

```python
# 避免除零，设置最小 temperature
temperature = max(temperature, 1e-8)
```

1.3 混合实现（huggingface）

```python
# 根据 temperature 选择采样策略
if temperature == 0 or temperature < 1e-8:
    # 贪心解码
    idx_next = logits.argmax(dim=-1, keepdim=True)
else:
    # 温度采样
    probs = F.softmax(logits / temperature, dim=-1)
    idx_next = torch.multinomial(probs, num_samples=1)
```

### 2.浮点数精度问题

浮点数运算可能会因为硬件、并行计算顺序（意力机制的并行计算、矩阵乘法的并行分块计算、Softmax的并行计算）等因素产生差异。

### 3.Dropout层（很少发生，model.eval() 会自动将dropout关闭）

Dropout 层在推理时未关闭，产生随机dropout

## 如果保证LLM的每次输出保持一致

设置temperature=0，并配合以下设置

* 使用top_k=1
* 使用top_p=极小值（top_p=0 在大多数实现中可能会被忽略或会被特殊处理，因此常用的是将top_p设置为非常小的值）
* 使用缓存机制
* 固定随机数种子

top_p和top_k一般不会组合使用，可能会产生冲突：

```python
概率分布 = {
    "药品A": 0.008,    # 最高概率但小于0.01
    "药品B": 0.007,
    "药品C": 0.006,
    # ... 很多低概率选项
}

# top_k=1：只保留"药品A"
# top_p=0.01：累积概率0.008 < 0.01，可能需要包含更多词
# 结果：可能产生冲突或异常行为
```

## 为什么不使用beam search来保证LLM输出的一致性？

1. beam search仍然会因为并行计算受到浮点数精度的影响
2. beam search倾向于产生重复的输出
3. beam search的计算成本对于大型LLM来说非常高，需要beam_size × sequence_length次前向传播，而一般的采样方式只需要sequence_length次前向传播

## 参考

https://community.openai.com/t/why-the-api-output-is-inconsistent-even-after-the-temperature-is-set-to-0/329541

https://community.openai.com/t/clarifications-on-setting-temperature-0/886447

https://cookbook.openai.com/examples/reproducible_outputs_with_the_seed_parameter



