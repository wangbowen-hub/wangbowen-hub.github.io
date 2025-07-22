---
title: "LLM Structured Ouputs"
date: 2025-07-11T18:54:20+08:00
# bookComments: false
# bookSearchExclude: false
---

# LLM 结构化输出

Structured Outputs和JSON mode是模型结构化输出的两种方式，Structured Outputs是JSON mode的演进版本，能够保证模型产出的 JSON完全符合提供的JSON Schema（字段名、是否必填、类型、取值范围、枚举等都对得上），Open AI官方更建议使用Structured Outputs

## Structured Outputs

Structured Outputs方式实现结构化输出有objects和手动模式两种方法，objects通过pydantic定义basemodel，手动模式需要自己手动构建json_schema

### SDK Objects

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()

class Step(BaseModel):
    explanation: str
    output: str

class MathReasoning(BaseModel):
    steps: list[Step]
    final_answer: str

completion = client.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    response_format=MathReasoning,
)

math_reasoning = completion.choices[0].message.parsed
```

### 手动模式

```python
response = client.chat.completions.create/parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "math_response",
            "schema": {
                "type": "object",
                "properties": {
                    "steps": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "explanation": {"type": "string"},
                                "output": {"type": "string"}
                            },
                            "required": ["explanation", "output"],
                            "additionalProperties": False
                        }
                    },
                    "final_answer": {"type": "string"}
                },
                "required": ["steps", "final_answer"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
)
## 注意是content，不是跟sdk objects一样的parsed，这里打印parsed会返回None，即使content里是遵循schema的json
print(response.choices[0].message.content)
```

#### strict

* 将 `strict` 设为 `false` ，模型生成过程仍受 schema 派生的语法约束，但生成完成后不会强制比对，允许模型返回模式中未指定的额外字段，类似于 JSON 模式。
*  `strict` 设为 `true` ,对模型生成强制约束，不符合schema会立刻拒绝（`refusal` 字段）或抛 400
* strict默认为false，一旦模型“卡壳”生成不出闭合 JSON，严格模式下更容易触发长时间的token 死循环，浪费配额——这是官方没有默认开启的原因
* strict必须配合Structured Outputs一起使用，JSON mode不受strict影响
* Structured Outputs的SDK Objects方式，strict默认为true
* strict mode有两个限制
  * `additionalProperties` must be set to `false` for each object in the `parameters`.
  * All fields in `properties` must be marked as `required`(You can denote optional fields by adding `null` as a `type` option ).

## JSON mode

将response_format中的type字段设置为`{ "type": "json_object" }`可启用json_object

## 两种结构化输出方式的对比

|                        | Structured Outputs                                           | JSON Mode                                                    |
| :--------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| **Outputs valid JSON** | Yes                                                          | Yes                                                          |
| **Adheres to schema**  | Yes (see [supported schemas](https://platform.openai.com/docs/guides/structured-outputs?api-mode=chat&format=without-parse#supported-schemas)) | No                                                           |
| **Compatible models**  | `gpt-4o-mini`, `gpt-4o-2024-08-06`, and later                | `gpt-3.5-turbo`, `gpt-4-*` and `gpt-4o-*` models             |
| **Enabling**           | `response_format: { type: "json_schema", json_schema: {"strict": true, "schema": ...} }` | `response_format: { type: "json_object" }`                   |
| 约束表达               | prompt中的自然语言描述                                       | JSON Schema表达：`required`、`type`、`enum`、`minItems`、正则 `pattern`…… |

**Adheres to schema**的具体意思是：

* JSON mode 只保证输出JSON的结构（花括号、逗号、引号）正确。

* Structured Outputs除了保证输出是合法 JSON，还强制要求模型产出的 JSON 必须完全符合你提供的 JSON Schema（字段名、是否必填、类型、取值范围、枚举等都要对得上）。输出在返回给你之前会被自动按 JSON Schema 校验，不合格就报错/返回 `None`。
