---
title: "Ai Reviewer"
date: 2025-08-21T17:00:57+08:00
# bookComments: false
# bookSearchExclude: false
---

# AI Reviewer

## 1. langchain_openai库中ChatOpenAI和OpenAI的区别

* ChatOpenAI面向现代聊天模型，这些模型支持多模态输入，工具调用和结构化输出等

* OpenAI面向旧的补全模型(gpt-3.5-turbo-instruct, text-davinci-003等)，这些模型只支持字符串输入和输出，不支持多模态输入，工具调用和结构化输出等现代特性

## 2. pydantic别名

* pydantic别名用于兼容历史/旧的字段名
* alias, ConfigDict, populate_by_name

```python

from pydantic import BaseModel, ConfigDict, Field


class User(BaseModel):
    user_id: str = Field(alias="id")


user1 = User(id="2")
# user2 = User(user_id="2") ❌ 用内部名初始化报错
print(user1)
# print(user2)


class User2(BaseModel):
    user_id: str = Field(alias="id")
    model_config = ConfigDict(
        populate_by_name=True
    )  # 设置 populate_by_name=True 后可用内部名进行初始化


user3 = User2(id="2")
user4 = User2(user_id="2")  # ✅
print(user3)
print(user4)


```

## 3. ChatOpenAI中model和model_name参数的区别

* model参数是model_name的别名，初始化用model或model_name都一样，最好是用model

## 4. langgraph图中如果有节点执行错误，并且重试超过最大次数，会有state输出吗？

* 不会有最终state输出，并且invoke/ainvoke会报错

## 5. 函数无返回值应该不写返回值类型，还是应该写->None?

* 如果其他函数有类型注解，就写->None,保持整体性。如果其他函数都没有类型注解，就不写返回值类型

## 6. any和typing.Any的区别

* any是python中的内置函数，判断可迭代对象中是否包含真值

```python

any([0, "", None, 5])   # True
any([])                 # False

```

* typing.Any表示任意类型
* 它们之间不是类似list和typing.List的关系

## 7. raise_for_status()

* raise_for_status()用来检查响应状态码，如果状态码是错误码，则会抛出异常，否则什么都不做

## 8. 0.0.0.0和127.0.0.1的区别

* 0.0.0.0上启动的服务外部可访问
* 127.0.0.1上启动的服务只本机可访问

## 9. python的真值和假值

* 假值: 数值零 (包括整数，浮点数，复数的零), False, None, 空序列/容器 (len(obj)==0)
* 真值: 除以上假值外的所有对象

## 10. langgraph的state schema类型的适用场景

* TypedDict: 最常用，大多数使用 (最快)
* dataclass: 需要设置默认值和必填字段时使用
* pydantic BaseModel: 需要递归数据校验时使用 (内部嵌套对象也需要进行数据校验) (最慢)

## 11

* docstring没有规范是三个单引号还是三个双引号，但最好是用三个双引号写

## 12

* 英文prompt中括号前后都要空一格，括号后面如果是标点符号则不需要空一格
* 括号内部：开括号后不空格，闭括号前不空格

```txt

* Compare the expected and actual values to determine if they match (True/False).✅

* Provide clear reasoning for your assessment in Chinese (limited to 250 characters), explaining why the values match or do not.✅

```

## 13. pytest单元测试

* 测试文件按功能模块进行划分，以test_开头
* 测试函数以test_开头
* 异步测试函数上方需要添加 @pytest.mark.asyncio注解
* 测试参数使用 @pytest.mark.parametrize注解

```python

import pytest
from utils import async_compress_and_encode_images_from_urls

@pytest.mark.parametrize(
    "image_urls",
    [
        [
            "https://trialedctest.oss-cn-hangzhou.aliyuncs.com/123.png?Expires=1755917451&OSSAccessKeyId=LTAItycD0loaPpSp&Signature=L2XaKIHaVoCgRKhPbsIZXmlO9Xw%3D"
        ]
    ],
)
@pytest.mark.asyncio
async def test_async_compress_and_encode_images_from_urls(image_urls):
    base64_images = await async_compress_and_encode_images_from_urls(
        request_id="123", image_urls=image_urls, quality=100
    )
    base64_images = [img for img in base64_images if img is not None]
    assert len(base64_images) > 0


```

## 14.
LangGraph实现按需并行扇出，需要多个节点就同时跑多个:

* 使用Send
* 使用add_conditional_edges

详见：[LangGraph并行扇出实现](/posts/LangGraph并行扇出实现.html)


## 15. LangGraph的节点同名问题

* LangGraph中同一个 StateGraph 内，节点名必须唯一，同名会报错。 
* 不同的 StateGraph 之间可以使用相同的节点名（互不影响）。

## 16. python导入文件名

* python文件名就是它对应的模块名
* 导入时文件中的所有顶层代码都会被执行一次，并创建一个以模块名命名的命名空间

```python

# demo_module.py
print("模块正在被导入...")

CONSTANT_VALUE = "这是一个常量"
counter = 0

def increment():
    global counter
    counter += 1
    return counter

def get_info():
    return f"当前计数器值: {counter}"

# 这行代码在import时就会执行
print(f"模块加载完成，初始计数器值: {counter}")

```

```python

import demo_module
# 输出：
# 模块正在被导入...
# 模块加载完成，初始计数器值: 0

# 使用模块内容
print(demo_module.CONSTANT_VALUE)  # 这是一个常量
demo_module.increment()
print(demo_module.get_info())      # 当前计数器值: 1

```

## 17. LangSmith不开源，其开源替代工具为Langfuse

### LangChain中的回调作用于以下几类事件：

* Chain/Runnable相关: on_chain_start / on_chain_end / on_chain_error
* LLM相关: on_llm_start / on_llm_new_token (流式输出时可用)/ on_llm_end / on_llm_error
* tool相关: on_tool_start / on_tool_end / on_tool_error
* retriver相关: on_retriever_start / on_retriever_end
* agent相关: on_agent_action / on_agent_finish

### 触发规则

* 把 handlers 传给最顶层的 runnable（链/模型），会自动传播给内部一切子调用；不必逐层手动挂
* 父级先触发 `on_*_start`，再进入子级；收尾时逆序触发 `on_*_end`（像函数调用栈一样“后进先出”）
* 出现异常则触发 `on_*_error`，不会再触发对应的 `on_*_end`。异常会“冒泡”，父级随后收到它自己的 on_chain_error
* RunnableMap / RunnableParallel 等并行执行的子步骤，它们的 start/end 事件会交错出现，顺序不保证稳定

### 调用顺序：

**提示词 + LLM（Prompt | LLM）**

```python

on_chain_start                  # 最外层 sequence
  on_chain_start                # Prompt 格式化（把输入拼成 messages/prompts）
  on_chain_end                  # Prompt 完成
  on_llm_start
    on_llm_new_token * N
  on_llm_end
on_chain_end

```

### 使用技巧

* 重操作应该使用异步回调 AsyncCallbackHandler
* 回调参数可以带入标签/元数据便于观测和过滤排查
* 可以配合LangSmith和Langfuse进行日志追踪和观测
* .with_config(...) 绑定的是运行时配置，会自动传播到链里的每个子组件，可以用来绑定局部组件和长期复用绑定
* .invoke 中的 config 适合临时调用绑定

## 18

* URL路径中的分词符应该用连字符-，而不是下划线_

## 19

* dict等价于dict[Any,Any]

## 20. 项目现存竞态问题

* 回调handler存在竞态，原来所有请求用的都是同一个ai_review_handler，每次请求开始都会执行 review_handler.records.clear()。解决方法: 每次请求初始化新的handler

* woodylogger中多进程写入日志文件存在竞态。Python 标准 logging 在 handler 级别是加锁的，所以多线程并发输出不会有竞态问题。解决方法: 将woodylogger中的TimedRotatingFileHandler替换成concurrent_log_handler.ConcurrentTimedRotatingFileHandler

* ReviewerAgent存在竞态，langgraph图fanout 时会并发触发同一个 REVIEW 节点的同一实例，`ReviewerAgent.__call__` 内把链路存到实例属性 `self.chain`，多协程并发时会互相覆盖，导致 A 调用可能使用到 B 的链。解决方法: 将`self.chain`改为`chain`

详见：[项目竞态问题](/posts/cursor2.md)



