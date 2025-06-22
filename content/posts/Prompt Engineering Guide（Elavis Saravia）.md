+++
date = '2025-06-18T18:30:03+08:00'
draft = false
title = 'Prompt Engineering Guide（Elavis Saravia）'
tags = ['AI', 'Prompt Engineering']
+++



#  Prompt Engineering Guide（Elavis Saravia）

https://www.promptingguide.ai/introduction/tips

https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-the-openai-api#h_1951f30f08

https://platform.openai.com/docs/guides/reasoning-best-practices#7-evaluation-and-benchmarking-for-other-model-responses

## Prompt Elements

A prompt contains any of the following elements:

**Identity** - 赋予模型的身份以及需要模型完成的任务

**Context** - external information or additional context that can steer the model to better responses

**Input Format** -输入数据格式

**Output Format** -输出数据格式

**Rules** -操作规则

**Examples** -示例

**Input Data** - the input or question that we are interested to find a response for

**Output Indicator** -指示模型开始输出（可选）

示例：

```python
# Identity

你是一个信息提取助手，你的任务是从OCR识别结果中提取每一个问题的答案。

# Input Format

OCR识别结果使用```符号包围，按图片中从左至右、从上至下排序。问题列表使用[]符号包围。

# Output Format

* 输出以JSON格式返回，key为问题内容，value为对应答案列表。
* 若OCR识别结果中无法提取某个问题答案，请将对应value设为空列表。
* 只输出json格式的结果，并做json格式校验后返回，不要包含其它多余文字。
* Company names: <comma_separated_list_of_company_names>

# Rules

* 所有问题的答案必须完全依据表格中的内容进行作答。
* 回答时应尽可能详细和完整，不得省略或自行补充未在表格中明确出现的信息。
* 保持原文的格式、数字、正负号、单位、符号和标点符号完全一致。
* 答案必须逐字摘抄自 OCR 文本。
* 一个问题可能包含多个对应答案，将每个答案添加到问题对应value的列表中。
* 若OCR识别结果中无法提取某个问题答案，请将对应value设为空列表。
* 问题的答案为"其它"、"不详"、"拒绝提供"，也需要添加到问题对应value的列表中，不应该进行省略。
* 不得缩写、改写或断章取义。

# Examples

<user_query>
How do I declare a string variable for a first name?
</user_query>

<assistant_response>
var first_name = "Anna";
</assistant_response>

<product_review id="example-1">
I absolutely love this headphones — sound quality is amazing!
</product_review>

<assistant_response id="example-1">
Positive
</assistant_response>

<product_review id="example-2">
Battery life is okay, but the ear pads feel cheap.
</product_review>

<assistant_response id="example-2">
Neutral
</assistant_response>

<product_review id="example-3">
Terrible customer service, I'll never buy from them again.
</product_review>

<assistant_response id="example-3">
Negative
</assistant_response>


# Input Data

如下是需要提取信息的OCR
OCR识别结果```{ocr_result}```
问题列表{question_list}
提取结果：（Output Indicator）
```

###、##、"""、'''、"、'、<>、```可以表示强调。

## General Tips

1. 使用最新的模型

2. As you get started with designing prompts, you should keep in mind that it is really an iterative process that requires a lot of experimentation to get optimal results. (需要多次实验迭代达到更佳结果)（OpenAI playground、Cohere）

3. specificity、preciseness、conciseness

   ```cmd
   ### Specificity ###
   Less effective ❌:
     Write a poem about OpenAI. 
   Better ✅:
     Write a short inspiring poem about OpenAI, focusing on the recent DALL-E product launch (DALL-E is a text to image ML model) in the style of a {famous poet}
   
   ### Preciseness ###
   Less effective ❌:
     Explain the concept prompt engineering. Keep the explanation short, only a few sentences, and don't be too descriptive.
   Better ✅:
   	Use 2-3 sentences to explain the concept of prompt engineering to a high school student.
   
   ### Conciseness ###
   Less effective ❌:
     The description for this product should be fairly short, a few sentences only, and not too much more.
   Better ✅:
     Use a 3 to 5 sentence paragraph to describe this product.
   ```

4. Place instructions at the beginning of the prompt, use some clear separator like "###" to separate the instruction and context. (instruction放在prompt的开头，并用例如###的分隔符进行分割指令和上下文)

   ```cmd
   Less effective ❌:
     Summarize the text below as a bullet point list of the most important points.
   
   	{text input here}
    Better ✅:
     Summarize the text below as a bullet point list of the most important points.
   
     Text: """
     {text input here}
     """
   ```

5. Avoid saying what not to do but say what to do instead.(告诉模型应该做什么而不是不应该做什么)

6. Try zero shot first, then few shot if needed, neither of them worked, then fine-tune(首先尝试zero shot，如果需要复杂输出，可以通过few-shot给出示例告诉模型应该怎样输出(适用于reasoning model和gpt model)，也可以通过示例告诉模型应该怎么思考和解决问题(适用于gpt model)，如果实在都不行就需要微调了 )（注意示例应该与prompt中要求保持一致，不然会影响模型输出效果）

   ```python
   ### Few-shot ###
   Extract keywords from the corresponding texts below.
   
   Text 1: Stripe provides APIs that web developers can use to integrate payment processing into their websites and mobile applications.
   Keywords 1: Stripe, payment processing, APIs, web developers, websites, mobile applications
   ##
   Text 2: OpenAI has trained cutting-edge language models that are very good at understanding and generating text. Our API provides access to these models and can be used to solve virtually any task that involves processing language.
   Keywords 2: OpenAI, language models, text processing, API.
   ##
   Text 3: {text}
   Keywords 3:
   ```

7. 使用分隔符（如markdown, XML tags, and section titles ）来clarify prompt的不同部分，一方面可以是prompt层次结构更加清晰，另一方面也有助于模型更好的理解prompt

   ```python
   * 答案列表中元素为JSON格式，包含text和id两个个字段，text为答案内容，id为列表类型，列表中元素为答案所在BOX的id值。（worse,👎）
   
   * 答案列表中元素为 JSON 格式，包含 "text" 和 "id" 两个字段：
     - "text" 字段值表示答案内容。
     - "id" 字段值为列表类型，列表中元素表示答案所在 BOX 的 id 值。（better，👍）
   ```

   

8. 为了让模型更清晰地辨识语言边界，中文与英文之间建议空一格。如果提示词以中文为主，夹杂少量英文，使用中文标点符号；如果提示词以英文为主，夹杂少量中文，使用英文标点符号。在同一个提示词中，标点符号使用要统一，不要在同一句中混用两种标点体系。

9. prompt中的风格需要保持统一。

   ```python
   ### 如下列列表项要么末尾都加句号，要不都不加。###
   1. 请提取文章中的关键词。  
   2. 用三句话概述文章要点。  
   3. 翻译成英文。
   
   
   1. 请提取文章中的关键词  
   2. 用三句话概述文章要点 
   3. 翻译成英文
   
   ```

   

### GPT Model vs Reasoning Model

- A reasoning model is like a senior co-worker. You can give them a goal to achieve and trust them to work out the details.
- A GPT model is like a junior coworker. They'll perform best with explicit instructions to create a specific output.

**如何选择** ：

GPT Model：更快，cost更低，适合简单预先定义的工作，如总结、翻译等

Reasoning Model：

 	1. 适合用户提示不明确、用户输入模棱两可、背景信息有限的情形，能够理解用户意图，消除instruction gaps
 	2. 适合“大海捞针”，从大量无结构文档中找出所需的最相关信息
 	3. 查找多份/大量数据/文档中的相似处和细微差别
 	4. 适合planning，在复杂agent系统中，reasoning model做planner，gpt model做doer
 	5. 视觉推理，reasoning model在质量不高的图像中相较于gpt model会有更好的表现
 	6. 评估模型输出（Evaluation and benchmarking for other model responses）

**如何prompt reasoning model**：

general tips中的方法均适用于reasoning model，但注意对于reasoning model不要使用CoT方法，如"think step by step"和"explain your reasoning"，因为reasoning model是internal thinking，使用cot方法会影响模型能力。



## Learning

1. 模型的理解能力很强，标点符号、空格、分隔符这些东西不会改变它对你需求的理解，对模型输出结果影响不大



## TO DO

reasoning items是什么（https://platform.openai.com/docs/guides/reasoning-best-practices#7-evaluation-and-benchmarking-for-other-model-responses）

使用openai playground

https://platform.openai.com/docs/guides/reasoning?api-mode=chat

https://platform.openai.com/docs/guides/fine-tuning

openai cookbook很值得看

caffeinate

https://www.nature.com/articles/d41586-023-00288-7

https://arxiv.org/abs/2202.12837

