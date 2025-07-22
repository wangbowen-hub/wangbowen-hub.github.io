---
title: "Function Calling"
date: 2025-07-17T16:14:04+08:00
# bookComments: false
# bookSearchExclude: false
---

# Function Calling

## Overview

### 什么是function calling？

通过function calling，模型能够访问自定义代码

function calling有两种作用：

* fetching data 获取数据：Retrieve up-to-date information to incorporate into the model's response (RAG).
* taking action 采取行动：Perform actions like submitting a form, calling APIs, modifying application state, or taking agentic workflow actions.

## function calling 步骤

* 1. 定义function

     ```python
     from openai import OpenAI
     import json
     
     client = OpenAI()
     
     tools = [{
         "type": "function",
         "function": {
             "name": "get_weather",
             "description": "Get current temperature for provided coordinates in celsius.",
             "parameters": {
                 "type": "object",
                 "properties": {
                     "latitude": {"type": "number"},
                     "longitude": {"type": "number"}
                 },
                 "required": ["latitude", "longitude"],
                 "additionalProperties": False
             },
             "strict": True
         }
     }]
     
     messages = [{"role": "user", "content": "What's the weather like in Paris today?"}]
     
     completion = client.chat.completions.create(
         model="gpt-4.1",
         messages=messages,
         tools=tools,
     )
     ```

     

  2. 模型决定调用function，返回函数名称及参数

     ```python
     # 模型返回function调用信息
     [{
         "id": "call_12345xyz",
         "type": "function",
         "function": {
           "name": "get_weather",
           "arguments": "{\"latitude\":48.8566,\"longitude\":2.3522}"
         }
     }]
     ```

     

  3. 解析模型响应并执行对应function

     ```python
     tool_call = completion.choices[0].message.tool_calls[0]
     args = json.loads(tool_call.function.arguments)
     
     result = get_weather(args["latitude"], args["longitude"])
     ```

     

  4. 向模型提供function执行结果，再次调用模型将执行结果整合进响应中

     ```python
     messages.append(completion.choices[0].message)  # append model's function call message
     messages.append({                               # append result message
         "role": "tool",
         "tool_call_id": tool_call.id,
         "content": str(result)
     })
     
     completion_2 = client.chat.completions.create(
         model="gpt-4.1",
         messages=messages,
         tools=tools,
     )
     ```

     

  5. 模型输出---整合了function执行结果

     ```python
     # completion_2.choices[0].message.content
     "The current temperature in Paris is 14°C (57.2°F)."
     ```

     

## 定义function

function定义的token是会被记入input token中的，并占用模型上下文

### JSON Schema 定义

![image-20250717162732491](/Users/bowenwang/Library/Application Support/typora-user-images/image-20250717162732491.png)

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Retrieves current weather for the given location.",
    "parameters": {
      "type": "object",
      "properties": {
        "location": {
          "type": "string",
          "description": "City and country e.g. Bogotá, Colombia"
        },
        "units": {
          "type": "string",
          "enum": [
            "celsius",
            "fahrenheit"
          ],
          "description": "Units the temperature will be returned in."
        }
      },
      "required": [
        "location",
        "units"
      ],
      "additionalProperties": false
    },
    "strict": true
  }
}
```

### 通过Pydantic或Zod定义

```python
from openai import OpenAI, pydantic_function_tool
from pydantic import BaseModel, Field

client = OpenAI()

class GetWeather(BaseModel):
    location: str = Field(
        ...,
        description="City and country e.g. Bogotá, Colombia"
    )

tools = [pydantic_function_tool(GetWeather)]

completion = client.chat.completions.create(
    model="gpt-4.1",
    messages=[{"role": "user", "content": "What's the weather like in Paris today?"}],
    tools=tools
)

print(completion.choices[0].message.tool_calls)
```

### function定义的最佳实践

1. 明确描述function的作用、参数和返回值含义（返回值含义描述在function的description字段中）
2. 在system prompt明确什么时候应该调用function，哪种情况不应该调用function
3. system prompt中包含examples和edge cases
4. 使用枚举来确保无效状态无法被表示
5. 不要让模型填写你已知的参数
6. 合并总是顺序调用的functions，如每次调用query_location()后都会调用mark_location()，那么考虑将mark_location()里的逻辑代码合并到query_location()中
7. 保持较少的函数数量以提高准确性。（建议20个以内）
8. 对于大量函数，可以考虑通过finetune提高调用准确率

## 处理模型function calling响应

模型响应的function calling信息在tool_calls数组中:

* id: 后续用于提交函数结果
* name: 调用函数名
* arguments: 函数调用参数

```python
[
    {
        "id": "call_12345xyz",
        "type": "function",
        "function": {
            "name": "get_weather",
            "arguments": "{\"location\":\"Paris, France\"}"
        }
    },
    {
        "id": "call_67890abc",
        "type": "function",
        "function": {
            "name": "get_weather",
            "arguments": "{\"location\":\"Bogotá, Colombia\"}"
        }
    }
]
```

### 使用方式

```python
for tool_call in completion.choices[0].message.tool_calls:
    name = tool_call.function.name
    args = json.loads(tool_call.function.arguments)

    result = call_function(name, args)
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": str(result)
    })
```

```python
def call_function(name, args):
    if name == "get_weather":
        return get_weather(**args)
    if name == "send_email":
        return send_email(**args)
```

### 函数执行结果提交

* 函数执行结果提交给模型，必须保证是string类型的
* 如果函数没有执行结果，返回一个字符串表示成功或失败即可（例如 `"success"` ）。

## 额外配置

### tool_choice参数

指定模型函数调用行为

* **auto:** 默认值，调用0，1或多个函数

* **required:** 调用一个或多个函数
* **Forced Function:** 必须调用单个特定函数（无法指定多个），tool_choice: {"type": "function", "function": {"name": "get_weather"}}
* **none**: 不调用任何函数

### parallel_tool_calls参数

* true: 模型一轮对话中可以调用多个函数
* false: 模型一轮对话中最多可调用一个函数

### strict参数

* true: 保证tool_calls响应严格遵循function schema
* false: 尽力保证tool_calls响应遵循function schema

### stream参数

* true：流式传输
* false：非流式传输

## 参考

https://platform.openai.com/docs/guides/function-calling?api-mode=chat
