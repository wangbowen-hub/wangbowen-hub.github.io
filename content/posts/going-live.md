---
title: "Going Live"
date: 2025-08-08T18:42:49+08:00
# bookComments: false
# bookSearchExclude: false
---

# Going Live

## 延迟优化

延迟优化的八大原则（带*的是最有效和常用的方法）：

* Process tokens faster*（衡量标准是TPM/TPS）
  * 影响模型推理速度的主要因素是模型规模，在保证生成质量的同时使用更小的模型
    * 设计更精巧的提示词
    * few-shot
    * 微调/蒸馏
  * predicted outputs，主要应用于文本或代码的局部小幅修改替换
* Generate fewer tokens*，tokens生成是使用LLM延迟最高的环节，减少50%的tokens生成数，大概可以减少50%的使用延迟
  * 生成自然语言回复，通过提示词/微调让模型生成更简短的回复
  * 生成结构化输出，可缩短命名参数名称等
  * 通过max_tokens/stop_tokens参数提前结束生成
* Use fewer input tokens，更少的输入tokens能更降低延迟，但性价比不高，50%的输入tokens减少只能带来1%～5%的延迟降低
  * 微调模型以替代冗长的提示输入
  * 过滤上下文输入，例如精简 RAG 结果、清理 HTML 等操作
  * 将动态部分放置在提示词后部，提高缓存命中率
* Make fewer requests*，每次请求都会有往返延迟，这些延迟会逐渐累积
  * 在一个请求里获取多个任务结果，让模型输出json，根据json字段解析任务结果
* Parallelize
  * 如果步骤1和步骤2没有顺序关系，可以同时发起并行请求以降低延迟
* Make your user wait less
  * 流式传输
  * 分块处理
  * 向用户显示操作步骤，如工具调用细节等
  * 实时显示进度和加载状态
* Don't default to an LLM 不要总是使用LLM，如果能够通过硬编码完成的任务，尽量不要通过请求LLM来增加延迟
* 对于高价值用户可使用“优先处理”降低延迟，通过参数service_tier="priority"设置



## 成本优化

* batch api，适用于不需要即时响应的请求，如模型评估等
* flex，适用于低优先级任务，代价是响应时间更慢和偶尔的资源不可用，通过参数service_tier="flex"设置



## 准确率优化

![Accuracy mental model journey diagram](https://cdn.openai.com/API/docs/images/diagram-optimizing-accuracy-02.png)

**Context optimization**：

* the model lacks contextual knowledge because it wasn’t in its training set
* its knowledge is out of date
* it requires knowledge of proprietary information

**LLM optimization**：

* the model is producing inconsistent results with incorrect formatting
* the tone or style of speech is not correct,
* the reasoning is not being followed consistently

### 具体准确率优化步骤流程

* 提示词工程

* 构建评估集进行评估，评估方法：

  * ROUGE 或 BERTScore 等硬性指标评估
  * 使用LLM进行评估

* 经过提示词工程后的模型仍达不到预期效果：

  * Learned memory，解决方法：微调/few-shot

    * 微调适用场景：
      * 让模型具有稳定一致的行为模式，解决Learned memory问题
      * 提升效率，微调更小的模型使其达到更大模型的准确率，或微调模型减少输入和输出token，进而降低延迟
    * 微调优化：
      * 训练数据质量比数据数量更重要，从50条开始逐渐扩大数据规模
      * 训练数据应具有代表性，来源于真实生产环境，是LLM会真实面对的数据（prompt baking）
      * 如果应用用到了微调，应该在训练数据的context中添加检索数据
      * 使用微调需要额外管理训练集和验证集

  * In-context memory，解决方法：RAG

    * RAG适用场景：模型需要外部知识

    * RAG优化：

      * 使用RAG需要同时优化检索和LLM生成

      ![RAG evaluation diagram](https://cdn.openai.com/API/docs/images/diagram-optimizing-accuracy-05.png)

      

  * 这两种方法可共同叠加使用，不冲突

    ![Classifying memory problem diagram](https://cdn.openai.com/API/docs/images/diagram-optimizing-accuracy-03.png)

  

  
